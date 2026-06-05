# backend

The central Go service. Owns the master-data domain (participants, metering points, EEGs, tariffs, contracts) and subscribes to the MQTT-delivered EDA messages that affect participant state.

## At a glance

| | |
|---|---|
| Language | Go |
| Framework | net/http + chi (or comparable router) |
| Runtime | distroless container |
| State | PostgreSQL schemas `base` and `eda` |
| Bus | MQTT subscriber (eda-xp publishes) |
| Auth | JWT verify against Keycloak JWKS |

## Responsibilities

- CRUD for master data: `base.participant`, `base.metering_point`, `base.eeg`, `base.tariff`, `base.contract`
- EDA inbound dispatch: subscribes to MQTT topics for the protocols it cares about and updates state accordingly
- Process-history append: every interaction (UI + EDA) is logged to `base.processhistory`
- Member-self lookups: matches JWT `email` / `preferred_username` against the participant row

## REST API

The customer SPA addresses the backend with a tenant-scoped URL prefix:

```
/eeg/v2/<ecId>/...
```

`<ecId>` is the EEG's `community_id`. The middleware extracts it, verifies it is in the JWT's `tenant` claim, and dispatches.

Conventional endpoints:

| Path | Method | Auth | Purpose |
|------|--------|------|---------|
| `/eeg/v2/<ecId>/participants` | GET | EEG_ADMIN | list members |
| `/eeg/v2/<ecId>/participants/<id>` | GET | self or EEG_ADMIN | one member |
| `/eeg/v2/<ecId>/metering-points` | GET / POST | EEG_ADMIN | manage Zählpunkte |
| `/eeg/v2/<ecId>/metering-points/<id>/activate` | POST | EEG_ADMIN | triggers `EC_REQ_ONL` |
| `/eeg/v2/<ecId>/metering-points/<id>/participation-factor` | PATCH | EEG_ADMIN | triggers `EC_REQ_PRZ` |
| `/eeg/v2/<ecId>/processhistory` | GET | EEG_ADMIN | paginated history |
| `/health`, `/ready` | GET | none | k8s probes |

## Auth

The JWT is parsed into a `PlatformClaims` struct. `access_groups` is checked against an `adminGroups` map. Two middlewares wrap routes:

| Middleware | Effect |
|------------|--------|
| `ConditionProtect(adminHandler, userHandler)` | calls `adminHandler` if `IsAdmin()`, otherwise `userHandler` |
| `AdminOnly(handler)` | returns 403 to non-admins |

`adminGroups` contains `EEG_ADMIN` and `vfeeg-superuser`. Both with and without leading slash are accepted.

`aud` is parsed as a `string`. A user holding client roles on a foreign Keycloak client will have `aud` resolved as an array, breaking the parse. See [Architecture / Authentication](../architecture/auth.md) for the "audience caveat" details.

## EDA subscriptions

The MQTT subscription list is hard-coded at startup. Anything not in the list is dropped silently (broker publishes succeed but no consumer reads).

| Protocol | Handler | Effect |
|----------|---------|--------|
| `CR_MSG` | `protocolCrMsgHandler` | generic notification |
| `CR_REQ_PT` | `protocolCrReqPtHandler` | participant-list response |
| `EC_REQ_ONL` | `protocolEcReqOnlHandler` | Zählpunkt activation lifecycle |
| `CM_REV_IMP` | `protocolCmRevImpHandler` | revocation, import direction |
| `CM_REV_CUS` | `protocolCmRevImpHandler` | revocation by customer |
| `CM_REV_SP` | `protocolCmRevImpHandler` | revocation by service provider |

Handlers pattern-match on `MessageCode` (not version) and update `base.metering_point.status`, append to `base.processhistory`, and create notifications.

When adding support for a new EDA protocol or message code, the subscription list **and** the message-code switch inside the handler both need updating — see [Architecture / Messaging](../architecture/messaging.md) for the six gap points.

## Database access

| Schema | Tables (key) | Operation |
|--------|--------------|-----------|
| `base` | `participant`, `metering_point`, `eeg`, `tariff`, `contract`, `processhistory`, `notification` | read/write |
| `eda` | message storage shared with eda-xp | read/write (process-history side) |

DB connection comes from a Kubernetes Secret. The Go layer uses `database/sql` with `lib/pq`. SQL is built with `goqu`.

Caveats from operational experience:

- `goqu` `.Select(&struct)` does **not** respect `db:"-"` on nested struct fields the way `sqlx` does. Nested structs trigger LEFT JOINs on sub-selects. Use explicit field lists for nested types.
- `.Prepared(true).ToSQL()` emits `?` placeholders, which `lib/pq` rejects. Use `.ToSQL()` without `Prepared` for queries that are concatenated into the final SQL.

## Config

Environment variables (typical):

| Variable | Purpose |
|----------|---------|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | PostgreSQL connection |
| `KEYCLOAK_URL`, `KEYCLOAK_REALM` | JWKS source |
| `MQTT_HOST`, `MQTT_PORT`, `MQTT_USER`, `MQTT_PASS` | broker connection |
| `LOG_LEVEL` | `info` / `debug` |

The exact names vary by version; consult the deployed `ConfigMap` for the authoritative list.

## Secrets

| Secret | Holds |
|--------|-------|
| `backend-db-secret` | DB user + password |
| `backend-mqtt-secret` | MQTT user + password |

## Build and image

- Source repository: `eegfaktura-backend` in the operating organization
- Build: multi-stage Docker, distroless final image
- Tag scheme: commit SHA

## Local development

The service expects PostgreSQL, Mosquitto, and Keycloak to be reachable. The provisioning pipeline's bootstrap chart provides all three in a working configuration.

For pure code work, the test suite uses `sqlmock`. Note: handlers that open + defer-close the DB twice in one call break sqlmock (the second call sees "database is closed"). The fix-pattern is to keep `OpenDbXConnection` for public API and add a `*DB`-variant for handler batching.

## Authorization gaps

The backend's auth-Z coverage is incomplete. A meaningful share of routes accept any authenticated user without an admin / owner check. When adding routes, default to `AdminOnly` unless there is a deliberate reason to use `ConditionProtect`, and document the reason.

## Related

- [Architecture / Authentication](../architecture/auth.md)
- [Architecture / Messaging](../architecture/messaging.md) — six gap points
- [Architecture / Databases](../architecture/databases.md) — `base` and `eda` schema layout
- [services/eda-xp](eda-xp.md) — the MQTT producer side
