# backend

The central Go service. Owns the master-data domain (participants, metering points, EEGs, tariffs) and subscribes to the MQTT-delivered EDA messages that affect participant state.

## At a glance

| | |
|---|---|
| Language | Go 1.26 (`go 1.26.0` in `go.mod`, toolchain 1.26.9) |
| Framework | `net/http` + gorilla/mux v1.8.1; GraphQL via gqlgen v0.17.90 at `/query` |
| Runtime | `golang:1.26.9` container (not distroless) |
| State | PostgreSQL, schema `base` |
| Bus | MQTT subscriber (eda-xp publishes) and publisher (`eda/request`), eclipse/paho.mqtt.golang v1.5.1 |
| Auth | JWT verify via coreos/go-oidc v3.18.0 (RS256) |
| Internal API | gRPC server (default port 9092) |

## Responsibilities

- CRUD for master data: `base.participant` (with `base.contactdetail`, `base.address`, `base.bankaccount`), `base.meteringpoint`, `base.metering_partition_factor`, `base.eeg`, `base.tariff`
- EDA inbound dispatch: subscribes to MQTT topics for the protocols it cares about and updates state accordingly
- Process-history append: every EDA message the backend receives over MQTT (including the echo of its own requests) is logged to `base.processhistory`; UI actions are not
- Member-self lookups: matches the JWT `email` claim (case-insensitive) against the participant's e-mail
- Member mails via eda-xp's gRPC `SendMailService`: activation, metering point active, metering point no longer part of the community (`zp-inactive-mail-template`), metering-point list (Excel attachment)
- Energy-data request summary: a background check (every 5 minutes) writes one summary notification per multi-meter energy-data request (process `ENERGY_REQUEST`)

## REST API

Routes are mounted under flat, resource-scoped path prefixes (gorilla/mux subrouters): `/eeg`, `/participant`, `/meteringpoint`, `/process`, `/master`, and `/query` (GraphQL). There is no per-tenant URL prefix. CORS is open (`AllowedOrigins: *`); the HTTP server uses 180 s read and write timeouts and sets no request-body limit of its own.

The tenant is supplied out-of-band via the `tenant` (or `X-Tenant`) HTTP header. The middleware verifies the header value is present in the JWT's `tenant` claim (a string array, compared case-insensitively) before dispatching; tokens with the realm role `superuser` skip this check. The handler receives the tenant upper-cased.

Representative endpoints:

| Path | Method | Auth | Purpose |
|------|--------|------|---------|
| `/eeg` | GET / POST | user or admin / admin | read or update the EEG record |
| `/participant` | GET | admin (all) or user (self) | list members or member-self lookup |
| `/participant/{id}` | PUT | admin | update a member |
| `/meteringpoint/changepartitionfactor` | POST | admin | bulk request: sends one `ANFORDERUNG_CPF` (`EC_PRTFACT_CHANGE`) per grid operator; 404 if the EEG is not online |
| `/process/history` | GET | admin | paginated process history (`ps` = page size; negative → 400) |
| `/query` | POST | token + tenant header (`GQLProtect`) | GraphQL endpoint; the admin group is not enforced yet — operations with a non-admin token are logged (`gql-admin-check: non-admin token`) |
| `/master/updatepartfact`, `/master/masterdata` | POST / GET | Basic auth (`ProtectApi`), `/EEG_ADMIN` group, `X-Tenant` header | external API |

!!! note
    There are no dedicated `/health` or `/ready` endpoints — no HTTP health endpoint exists for k8s probes.

## Auth

The JWT is verified with coreos/go-oidc (RS256, issuer-side validation against the realm of the `app` client in `keycloak.json`; `SkipClientIDCheck: true`, so `aud` is not checked) and parsed into a `PlatformClaims` struct. Authorization is driven by the `access_groups` claim (`/EEG_ADMIN`, `/EEG_USER`) since 1.0.0; the only realm role read from `realm_access.roles` is `superuser`, which skips the tenant-header check but does not grant the admin group. The middlewares:

| Middleware | Effect |
|------------|--------|
| `Protect(handler)` | requires the `/EEG_ADMIN` group, otherwise 401 |
| `ConditionProtect(adminHandler, userHandler)` | calls `adminHandler` for `/EEG_ADMIN`, `userHandler` for `/EEG_USER`, otherwise 401 |
| `GQLProtect(handler)` | `/query`: token + tenant header; non-admin tokens are only marked for the log |
| `ProtectApi(handler)` | `/master/*`: Basic auth, exchanged for a token via password grant against the `api` client in `keycloak.json`; requires `/EEG_ADMIN` and the `X-Tenant` header in the `tenant` claim |

A missing or malformed bearer token gives 403, a token that fails verification 401. The `azp` claim is mapped to an `Authorized` field. `PlatformClaims` embeds golang-jwt v3 `StandardClaims`, whose `Audience` field is a `string`: a token whose `aud` is a JSON array fails claim parsing (401).

## EDA subscriptions

The backend subscribes to `eda/response/+/protocol/#` (EDA messages), `eda/response/+/command/#` (commands from eda-xp: `pontononlinestate` sets the EEG online flag, `register-eeg` creates an EEG) and `eda/response/error`; outbound EDA requests are published to `eda/request`. The protocol handler list is hard-coded at startup. Anything not in the list is dropped silently (broker publishes succeed but no consumer reads). A panic in a handler is recovered and only that message is dropped.

When a community has moved to another zone, eda-xp forwards its inbound Ponton messages to that zone's adapter (`app.forward`, eda-xp 1.0.9); this zone's backend then receives no MQTT messages for that community.

| Protocol | Handler | Effect |
|----------|---------|--------|
| `CR_MSG_HISTORY` (Go constant `model.CR_MSG`) | `protocolCrMsgHandler` | published by the energystore on `eda/response/<tenant>/protocol/cr_msg_history` after it imported a `CR_MSG`; writes one process-history entry (meter, from, to) per energy block. The raw `CR_MSG` itself is dropped in the backend's MQTT route — the energystore consumes it directly |
| `CR_REQ_PT` | `protocolCrReqPtHandler` | energy-data request (`ANFORDERUNG_PT`) answers `ANTWORT_PT` / `ABLEHNUNG_PT`; only writes history / notifications, does **not** change metering-point status |
| `EC_REQ_ONL` | `protocolEcReqOnlHandler` | Zählpunkt online-activation lifecycle |
| `EC_REQ_OFF` | `protocolEcReqOnlHandler` | offline registration lifecycle (same handler as `EC_REQ_ONL`) |
| `CM_REV_IMP` | `protocolCmRevImpHandler` | revocation, import direction |
| `CM_REV_CUS` | `protocolCmRevImpHandler` | revocation by customer |
| `CM_REV_SP` | `protocolCmRevImpHandler` | revocation by service provider |
| `EC_PRTFACT_CHANGE` | `protocolEcPrtChangeHandler` | grid-operator response to a participation-factor change; three variants: `ANTWORT_CPF` (accepted, appends to `base.metering_partition_factor`), `ABLEHNUNG_CPF` (rejected, notification with response code), `ANFORDERUNG_CPF` (outbound echo, history only) |
| `EC_PODLIST` | `protocolEcPodListHandler` | per-grid-operator metering-point list response; exported to Excel and mailed to the EEG's e-mail address |

Handlers pattern-match on `MessageCode` (not version). Depending on the message they append to `base.processhistory`, create notifications, and — for the lifecycle handlers — update `base.meteringpoint.status` / `process_state`. Not every handler touches metering-point status (e.g. `CR_REQ_PT` only writes history and notifications).

### Owner change (moved metering points)

After an owner change a metering point exists twice in a community: the old member's entry with `flag = 0` (moved) and the new member's with `flag = 1` (assigned); `flag = 2` marks deleted. Existing rows are tagged, not deleted, because invoices depend on them. An `ANTWORT_CPF` names no member, so the new factor is written only to the current (`flag = 1`) entry; the member form changes only that member's own entry.

### Outbound EDA caveats

The grid operator validates inbound `ANFORDERUNG_*` payloads on receipt. Response codes seen on outbound requests (texts from `ECON_RESPONSE_CODES` in `model/mqtt.go`):

| Code | Meaning | Trigger |
|------|---------|---------|
| 70 | Änderung/Anforderung akzeptiert | nominal success |
| 82 | Prozessdatum falsch | request submitted outside the process's time window (for `EC_PRTFACT_CHANGE`: 09:00–17:00 Austrian time) |
| 90 | Kein Smart Meter | meter not eligible for the requested process |
| 94 | Keine Daten im angeforderten Zeitraum vorhanden | requested period is empty for this meter |

For `EC_PRTFACT_CHANGE` specifically the grid operator only accepts requests between 09:00 and 17:00 Austrian time, and the change takes effect the next calendar day. The backend does **not** enforce this window: an `ANFORDERUNG_CPF` sent outside it receives an `ABLEHNUNG_CPF` with code 82 even when the payload is otherwise correct. The web form warns outside the window (eegfaktura-web 1.8.0) but still allows sending.

When adding support for a new EDA protocol or message code, the subscription list **and** the message-code switch inside the handler both need updating — see [Architecture / Messaging](../architecture/messaging.md) for the seven gap points.

## gRPC API

The gRPC server (`services/grpcServer.go`, port `grpc-provider.port`) registers `RegisterEegService` (`register`, `eegExists`), `ApiService` (`MasterData_MeteringPoint`) and `AdminEegService` (`updateValue`) from `proto/*.proto`; admin-backend uses it for EEG registration and admin updates. As a client the backend calls eda-xp's `SendMailService` (`services.mail-server`).

## Database access

| Schema | Tables (key) | Operation |
|--------|--------------|-----------|
| `base` | `eeg`, `participant`, `contactdetail`, `address`, `bankaccount`, `meteringpoint`, `metering_partition_factor`, `tariff`, `processhistory`, `notification`, `gridoperators`; views `activeMeteringPartition`, `activeTariff`, `billing_masterdata` (read by billing) | read/write |

The Go layer talks to PostgreSQL via the jackc/pgx v5 driver together with jmoiron/sqlx, and SQL is built with doug-martin/goqu. Schema migrations are embedded from `migrations/` and run automatically on startup via golang-migrate v4 (`db.MigrateDB()` in `database/database.go`, called from `main()` in `server.go`; migration table in schema `base`). The target schema lives in `schema.sql` / `schema.hcl`; migration files are generated with Atlas (`atlas.hcl`, golang-migrate format, `migrations/atlas.sum`) and the three must be kept in sync. Latest migration: `20260711120000_zvt_time_tariff` (time-of-use tariff columns on `base.tariff`, 1.6.0).

Caveats from operational experience:

- `goqu` `.Select(&struct)` does **not** respect `db:"-"` on nested struct fields the way `sqlx` does. Nested structs trigger LEFT JOINs on sub-selects. Use explicit field lists for nested types.
- `.Prepared(true).ToSQL()` emits `?` placeholders with goqu's default dialect. Build prepared queries with `goqu.Dialect("postgres")` (registered by the blank import in `database/databaseHelper.go`) to get `$1` placeholders; use `.ToSQL()` without `Prepared` for queries that are concatenated into the final SQL.
- The Excel master-data import caps the unzipped workbook at 128 MiB (`excelUnzipSizeLimit` in `database/excel.go`).

## Config

Configuration is loaded with spf13/viper from `config.yaml` in the `-configPath` directory. Environment variables override keys with the prefix `VFEEG_BACKEND_`; only dots are mapped to underscores, so keys containing hyphens (`grpc-provider.port`, `services.mail-server`, `file-content.templates`, `eda-process-versions.*`) keep the hyphen in the variable name and are in practice set in `config.yaml`. Map-valued keys (`grid-operator-alias`) are read from the file only. Logging is via logrus.

| Variable | Purpose |
|----------|---------|
| `VFEEG_BACKEND_DATABASE_HOST` / `_PORT` / `_USER` / `_PASSWORD` / `_DBNAME` | PostgreSQL connection |
| `VFEEG_BACKEND_DATABASE_MAXOPENCONNS` / `_MAXIDLECONNS` / `_CONNMAXLIFETIME` | connection-pool tuning |
| `VFEEG_BACKEND_DATABASE_STATSLOGINTERVAL` | interval of the pool-stats log line (default `1m`; negative disables) |
| `VFEEG_BACKEND_DATABASE_PASSWORD_FILE` | path to a file the entrypoint reads the DB password from |
| `VFEEG_BACKEND_MQTT_HOST` / `_ID` / `_QOS` | MQTT broker connection (no broker credentials) |
| `VFEEG_BACKEND_PORT` | HTTP listen port (default 9080) |
| `grpc-provider.port` | gRPC listen port (default 9092) |
| `services.mail-server` | gRPC endpoint of `eda-xp`'s `SendMailService` (member notifications; see [architecture/email](../architecture/email.md)) — **not** a direct SMTP host |
| `file-content.templates` | directory with global and per-tenant mail templates on the data volume (overrides the templates embedded in the binary) |
| `eda-process-versions.<MessageCode>` | EDA schema version stamped on outbound messages per message code (e.g. `ANFORDERUNG_CPF`) |
| `grid-operator-alias` | map of grid operator numbers to the number they are reachable under (read once at start) |
| `KEYCLOAK_CONFIG` | path to the Keycloak client JSON (clients `app` and `api`; default `./keycloak.json`) |
| `LOG_LEVEL` | logrus level, e.g. `info` / `debug` (default `debug`) |

## Secrets

| Secret | Holds |
|--------|-------|
| `eegfaktura-postgresql-eegfaktura-secret` | DB user, password and database name (keys `username`, `password`, `database` → `VFEEG_BACKEND_DATABASE_USER` / `_PASSWORD` / `_DBNAME`) |

The MQTT client connects without credentials, so there is no MQTT secret. `keycloak.json` may carry the `api` client secret.

## Build and image

- Go module path: `at.ourproject/vfeeg-backend`
- Binary: the Docker build produces `vfeeg-backend` (image `CMD ["vfeeg-backend","-configPath","/etc/backend/"]`); the local `Makefile` builds it as `master-backend`
- Build: multi-stage Docker (`golang:1.26.9` in both stages, final image not distroless); the build stage runs `make protoc` and `go generate ./...` (gqlgen, go-jsonschema) before compiling. The container `EXPOSE`s 8080, while the app listens on the configurable `port` (default 9080 in the repo `config.yaml`) plus gRPC on 9092; it runs as UID 1000 with `TZ=Europe/Berlin`
- Tag scheme: `sha-<short sha>` for every build, plus semver tags (`v1.6.0`, `1.6`, `1`) from `v*` git tags and `latest` on the default branch

## Local development

The service expects PostgreSQL, Mosquitto, and Keycloak to be reachable. The local docker-compose stack provides all three in a working configuration.

For pure code work, many tests use `sqlmock`; the database tests start PostgreSQL 18 (`postgres:18.6-alpine`) via testcontainers-go, so they need Docker. Note: handlers that open + defer-close the DB twice in one call break sqlmock (the second call sees "database is closed"). The fix-pattern is to keep `OpenDbXConnection` for public API and add a `*DB`-variant for handler batching.

## Authorization gaps

The backend's auth-Z coverage is incomplete. All REST routes except `GET /eeg` and `GET /participant` (`ConditionProtect`) require the admin group, and single-participant operations check the participant's tenant (1.1.1); finer owner checks do not exist. When adding routes, default to `Protect` (admin group) unless there is a deliberate reason to use `ConditionProtect`, and document the reason. Known gaps: `GET /master/test` is mounted without any auth (it only logs), `/query` does not enforce the admin group yet, and the gRPC server has no authentication (cluster-internal only).

## Related

- [Architecture / Authentication](../architecture/auth.md)
- [Architecture / Messaging](../architecture/messaging.md) — seven gap points
- [Architecture / Databases](../architecture/databases.md) — `base` and `eda` schema layout
- [services/eda-xp](eda-xp.md) — the MQTT producer side
