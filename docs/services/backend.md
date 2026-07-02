# backend

The central Go service. Owns the master-data domain (participants, metering points, EEGs, tariffs, contracts) and subscribes to the MQTT-delivered EDA messages that affect participant state.

## At a glance

| | |
|---|---|
| Language | Go 1.25 |
| Framework | `net/http` + gorilla/mux v1.8.1; GraphQL via gqlgen v0.17.90 at `/query` |
| Runtime | `golang:1.25` container (not distroless) |
| State | PostgreSQL, schema `base` |
| Bus | MQTT subscriber (eda-xp publishes), eclipse/paho.mqtt.golang v1.5.1 |
| Auth | JWT verify via coreos/go-oidc (RS256) |

## Responsibilities

- CRUD for master data: `base.participant`, `base.metering_point`, `base.eeg`, `base.tariff`, `base.contract`
- EDA inbound dispatch: subscribes to MQTT topics for the protocols it cares about and updates state accordingly
- Process-history append: every interaction (UI + EDA) is logged to `base.processhistory`
- Member-self lookups: matches JWT `email` / `preferred_username` against the participant row

## REST API

Routes are mounted under flat, resource-scoped path prefixes (gorilla/mux subrouters): `/eeg`, `/participant`, `/meteringpoint`, `/process`, `/master`, and `/query` (GraphQL). There is no per-tenant URL prefix.

The tenant is supplied out-of-band via the `tenant` (or `X-Tenant`) HTTP header. The middleware verifies the header value is present in the JWT's `tenant` claim (a string array, compared case-insensitively) before dispatching.

Representative endpoints:

| Path | Method | Auth | Purpose |
|------|--------|------|---------|
| `/eeg` | GET / POST | user / admin | read or update the EEG record |
| `/participant` | GET | admin (all) or user (self) | list members or member-self lookup |
| `/participant/{id}` | PUT | admin | update a member |
| `/meteringpoint/changepartitionfactor` | POST | admin | bulk request: triggers `EC_PRTFACT_CHANGE` / `ANFORDERUNG_CPF` per grid operator |
| `/process/history` | GET | user | paginated process history |
| `/query` | POST | per-resolver | GraphQL endpoint |

!!! note
    There are no dedicated `/health` or `/ready` endpoints — no HTTP health endpoint exists for k8s probes.

## Auth

The JWT is verified with coreos/go-oidc (RS256, issuer-side validation) and parsed into a `PlatformClaims` struct. Authorization is driven by the `realm_access.roles` claim, with the roles `admin`, `user`, and `superuser`. Three middlewares wrap routes:

| Middleware | Effect |
|------------|--------|
| `Protect(handler)` | requires the `admin` role |
| `ConditionProtect(adminHandler, userHandler)` | calls `adminHandler` if the `admin` role is present, otherwise `userHandler` |
| `AdminOnly(handler)` | returns 403 to callers without the `admin` role |

The `access_groups` claim is parsed into the struct but is **not** used for authorization. The `azp` claim is mapped to an `Authorized` field; the service does **not** parse the `aud` claim itself, so there is no audience-array parse hazard.

## EDA subscriptions

The MQTT subscription list is hard-coded at startup. Anything not in the list is dropped silently (broker publishes succeed but no consumer reads).

| Protocol | Handler | Effect |
|----------|---------|--------|
| `CR_MSG` | `protocolCrMsgHandler` | energy data + generic notifications; bridges per-slot values to the `eda/response/energy/<tenant>` topic for the energystore |
| `CR_REQ_PT` | `protocolCrReqPtHandler` | participant-list response; only writes history / notifications, does **not** change metering-point status |
| `EC_REQ_ONL` | `protocolEcReqOnlHandler` | Zählpunkt online-activation lifecycle |
| `EC_REQ_OFF` | `protocolEcReqOffHandler` | Zählpunkt offline-deactivation lifecycle |
| `CM_REV_IMP` | `protocolCmRevImpHandler` | revocation, import direction |
| `CM_REV_CUS` | `protocolCmRevImpHandler` | revocation by customer |
| `CM_REV_SP` | `protocolCmRevImpHandler` | revocation by service provider |
| `EC_PRTFACT_CHANGE` | `protocolEcPrtChangeHandler` | grid-operator response to a participation-factor change; three variants: `ANTWORT_CPF` (accepted, appends to `base.metering_partition_factor`), `ABLEHNUNG_CPF` (rejected, notification with response code), `ANFORDERUNG_CPF` (outbound echo, history only) |
| `EC_PODLIST` | `protocolEcPodListHandler` | per-grid-operator metering-point list response |

Handlers pattern-match on `MessageCode` (not version). Depending on the message they append to `base.processhistory`, create notifications, and — for the lifecycle handlers — update `base.metering_point.status`. Not every handler touches metering-point status (e.g. `CR_REQ_PT` only writes history and notifications).

### Outbound EDA caveats

The grid operator validates inbound `ANFORDERUNG_*` payloads on receipt. Two recurring rejection causes:

| Code | Meaning | Trigger |
|------|---------|---------|
| 70 | Akzeptiert | nominal success |
| 82 | Prozessdatum falsch | request submitted outside Mon–Fri 09:00–17:00, or `DateActivate` not the next working day |
| 90 | Kein Smart Meter | meter not eligible for the requested process |
| 94 | Keine Daten im angeforderten Zeitraum vorhanden | requested period is empty for this meter |

For `EC_PRTFACT_CHANGE` specifically the regulator only accepts requests during business hours and the change takes effect the following day. UI flows that trigger an `ANFORDERUNG_CPF` from outside the window will receive an `ABLEHNUNG_CPF` with code 82 even when the payload is otherwise correct.

When adding support for a new EDA protocol or message code, the subscription list **and** the message-code switch inside the handler both need updating — see [Architecture / Messaging](../architecture/messaging.md) for the six gap points.

## Database access

| Schema | Tables (key) | Operation |
|--------|--------------|-----------|
| `base` | `participant`, `metering_point`, `eeg`, `tariff`, `contract`, `processhistory`, `notification` | read/write |

The Go layer talks to PostgreSQL via the jackc/pgx v5 driver together with jmoiron/sqlx, and SQL is built with doug-martin/goqu. Schema migrations are embedded from `migrations/` and run automatically on startup via golang-migrate v4 (`MigrateDB()` in `server.go`).

Caveats from operational experience:

- `goqu` `.Select(&struct)` does **not** respect `db:"-"` on nested struct fields the way `sqlx` does. Nested structs trigger LEFT JOINs on sub-selects. Use explicit field lists for nested types.
- `.Prepared(true).ToSQL()` emits `?` placeholders. Use `.ToSQL()` without `Prepared` for queries that are concatenated into the final SQL.

## Config

Configuration is loaded with spf13/viper. Environment variables use the prefix `VFEEG_BACKEND_` with config-key dots mapped to underscores. Logging is via logrus.

| Variable | Purpose |
|----------|---------|
| `VFEEG_BACKEND_DATABASE_HOST` / `_PORT` / `_USER` / `_PASSWORD` / `_DBNAME` | PostgreSQL connection |
| `VFEEG_BACKEND_DATABASE_MAXOPENCONNS` / `_MAXIDLECONNS` / `_CONNMAXLIFETIME` | connection-pool tuning |
| `VFEEG_BACKEND_DATABASE_PASSWORD_FILE` | path to a file the entrypoint reads the DB password from |
| `VFEEG_BACKEND_MQTT_HOST` / `_ID` / `_QOS` | MQTT broker connection |
| `VFEEG_BACKEND_PORT` | HTTP listen port (default 9080) |
| `VFEEG_BACKEND_GRPC_PROVIDER_PORT` | gRPC listen port (default 9092) |
| `VFEEG_BACKEND_SERVICES_MAIL_SERVER` | gRPC endpoint of `eda-xp`'s `SendMailService` (member notifications; see [architecture/email](../architecture/email.md)) — **not** a direct SMTP host |
| `VFEEG_BACKEND_FILE_CONTENT_BASEDIR` | base directory for file content |
| `VFEEG_BACKEND_EDA_PROCESS_VERSIONS_*` | per-process EDA schema versions |
| `KEYCLOAK_CONFIG` | path to the Keycloak client JSON |
| `LOG_LEVEL` | `info` / `debug` |

Consult the deployed `ConfigMap` for the authoritative list.

## Secrets

| Secret | Holds |
|--------|-------|
| `backend-db-secret` | DB user + password |
| `backend-mqtt-secret` | MQTT user + password |

## Build and image

- Go module path: `at.ourproject/vfeeg-backend`
- Binary: the Docker build produces `vfeeg-backend` (image `CMD ["vfeeg-backend","-configPath","/etc/backend/"]`); the local `Makefile` builds it as `master-backend`
- Build: multi-stage Docker with a `golang:1.25` final image (not distroless); the container `EXPOSE`s 8080, while the app listens on the configurable `port` (default 9080) plus gRPC on 9092
- Tag scheme: commit SHA

## Local development

The service expects PostgreSQL, Mosquitto, and Keycloak to be reachable. The local docker-compose stack provides all three in a working configuration.

For pure code work, the test suite uses `sqlmock`. Note: handlers that open + defer-close the DB twice in one call break sqlmock (the second call sees "database is closed"). The fix-pattern is to keep `OpenDbXConnection` for public API and add a `*DB`-variant for handler batching.

## Authorization gaps

The backend's auth-Z coverage is incomplete. A meaningful share of routes accept any authenticated user without an admin / owner check. When adding routes, default to `AdminOnly` unless there is a deliberate reason to use `ConditionProtect`, and document the reason.

## Related

- [Architecture / Authentication](../architecture/auth.md)
- [Architecture / Messaging](../architecture/messaging.md) — six gap points
- [Architecture / Databases](../architecture/databases.md) — `base` and `eda` schema layout
- [services/eda-xp](eda-xp.md) — the MQTT producer side
