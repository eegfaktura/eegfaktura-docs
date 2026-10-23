# energystore-v2

Stateless Go service that stores per-period energy time-series in **TimescaleDB** and serves report queries. Designed as a drop-in replacement for [energystore (v1)](energystore.md). **Not rolled out**: production and the pilot zone both run v1, and the repository has no released version yet (its CHANGELOG has only `[Unreleased]`).

!!! info "Status"
    - **Deployments**: none in eegfaktura (production and pilot zone run v1); the v2 README refers to an upstream pilot setup and to ADR-0010 in `gemeinstrom/eegfaktura-platform`
    - **Production cutover**: not scheduled; the wire-format gap and the open architecture decisions below must be resolved first — see [Cutover open questions](#cutover-open-questions)
    - **Public source**: [`vfeeg-development/eegfaktura-energystore-v2`](https://github.com/vfeeg-development/eegfaktura-energystore-v2)

## At a glance

| | |
|---|---|
| Language | Go 1.26 (module `github.com/gemeinstrom/eegfaktura-energystore-v2`) |
| Storage | TimescaleDB (PostgreSQL extension) in a separate cluster |
| Bus | MQTT subscriber (`eclipse/paho.golang` v0.23.0, MQTT 5) |
| Subscription | Shared Subscription `$share/energystore/<topic>` (default share group, multi-replica capable) |
| Auth | JWT verify (go-oidc), X-Tenant header / claim fallback chain; fail-closed |
| State | Stateless (no PVC) |
| Replicas | Multi-replica capable |

## Why a rewrite?

The v1 stack had three structural problems that couldn't be tuned away (see [energystore v1: Known limitations](energystore.md#known-limitations-drove-the-v2-design)):

1. Embedded BadgerDB + RWO-PVC → no scaling beyond single replica
2. Full-range read-modify-write per EDA message → RAM/CPU spikes
3. In-process `Turns.lock(tenant)` → tenant-level serialisation

v2 addresses all three by externalising storage to TimescaleDB (multi-replica DB-side concurrency), using long-schema with `INSERT ... ON CONFLICT DO UPDATE` (only the actually changed cells are written), and removing the in-process mutex (the DB handles concurrency).

## Architecture

```
┌──────────────────┐                       ┌─────────────────────┐
│   eda-xp /       │  MQTT 5 + Shared Sub  │   energystore-v2    │
│                  │ ─────────────────────►│   (1..N replicas)   │
│   (publishers)   │   eda/response/+/...  │                     │
└──────────────────┘                       │  - decode + decrypt │
                                           │  - upsert (pgx)     │
                                           │  - REST + GraphQL   │
                                           └─────────┬───────────┘
                                                     │ SQL
                                                     ▼
                                           ┌─────────────────────┐
                                           │ postgres-energy     │
                                           │ TimescaleDB         │
                                           │  - hypertable       │
                                           │    energy_data      │
                                           │    (hash on tenant) │
                                           │  - counterpoint_meta│
                                           └─────────────────────┘
```

## Storage shape (long schema)

Three tables in the `postgres-energy` cluster (`migrations/0001_init.sql`, `0002_mqtt_dlq.sql`):

### `energy_data` — hypertable

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | text | space partition key (hash, `number_partitions => 16` in `0001_init.sql`; planned 128 for prod) |
| `ec_id` | text | EEG identifier |
| `metering_point` | text | OBIS-style ID (AT…) |
| `meter_code` | text | full OBIS code, e.g. `1-1:1.9.0 G.01` (T-/R-variants accepted; see [reference/obis-codes](../reference/obis-codes.md)) |
| `ts` | timestamptz | slot timestamp (time dimension, 7-day chunks) |
| `value` | double precision | meter reading for the slot |
| `qov` | smallint | quality marker, 1=measured / 2=replaced / 3=estimated / 0=unknown |
| Primary key | `(tenant_id, ec_id, metering_point, meter_code, ts)` | |

Insert path: `INSERT ... ON CONFLICT (PK) DO UPDATE` writes only the cells that arrived in the latest EDA message. No range-read-merge.

### `counterpoint_meta`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | text | |
| `ec_id` | text | |
| `metering_point` | text | |
| `direction` | smallint | 1=consumer, 2=producer |
| `source_idx` | int | original v1 index (compat) |
| `period_start` / `period_end` | timestamptz | active window |
| `payload` | jsonb | extension fields beyond the wire-protocol |
| `updated_at` | timestamptz | |

### `mqtt_dlq`

Dead-letter queue for MQTT messages that failed to `decode` or `upsert`: `topic`, `failure`, `error`, raw `payload` (bytea), `ts`, `replayed_at`.

Policies in `0001_init.sql`: chunk interval 7 days, compression after 30 days (segmented by tenant / EC / metering point / meter code), continuous aggregates `energy_hourly` and `energy_daily` (sum, slot count, `MIN(qov)`). No retention policy.

## MQTT input

Subscribes (multi-replica capable) to:

| Env | Default in pilot |
|---|---|
| `ESV2_MQTT_TOPIC_PATTERN` | no default, required (the upstream pilot used `eegfaktura/+/energy/+`; eda-xp publishes to `eda/response/+/protocol/cr_msg`) |
| `ESV2_MQTT_SHARE_GROUP` | `energystore` (becomes `$share/energystore/<topic>` at subscribe time) |
| `ESV2_MQTT_QOS` | `1` |
| `ESV2_MQTT_BROKER_URL` | no default, required |
| `ESV2_MQTT_INVERTER_TOPIC_PATTERN` | empty = inverter subscription off (share group `energystore-inverter`) |

!!! warning "Tenant extraction does not fit the eegfaktura topic"
    The handler takes the tenant from the topic (`tenantFromTopic` in `cmd/server/main.go`): for `eda/response/energy/<tenant>` it uses the fourth segment, otherwise the **second** segment. For `eda/response/<tenant>/protocol/cr_msg` that yields `response`, not the tenant. This must be adapted before v2 can consume the topic eda-xp publishes.

!!! note "MQTT-5 Shared Subscriptions need paho.golang"
    v2 uses `github.com/eclipse/paho.golang` (MQTT 5). An earlier attempt with `paho.mqtt.golang v1` (MQTT 3.1.1) failed: Mosquitto silently accepts `$share/…` subscribes from a 3.1.1 client but never matches a publish to them. See also the [mosquitto operational notes](mosquitto.md#mqtt-5-vs-311-shared-subscriptions-caveat).

    Also: MQTT 5 §4.8.2 forbids `NoLocal=true` on Shared Subscriptions — the v2 subscriber sets it to `false`. Earlier dev versions had `NoLocal=true` and were disconnected by Mosquitto with `reason_code=130 (Protocol Error)`.

### Optional payload decrypt

The `cr_msg` wire format published by eda-xp is gzip + Base64 and is **not** encrypted — see [eda-xp.md#cr_msg-payload-encoding](eda-xp.md#cr_msg-payload-encoding). v2 ships an optional, disabled-by-default pre-decode step (`internal/decode/decrypt.go`) that implements base64 → AES-256-CBC (static IV, PKCS#7) → optional gunzip. It was built against an assumed AES envelope that eda-xp does not produce:

| Env | Default | Effect |
|---|---|---|
| `ESV2_MQTT_DECRYPT_KEY_HEX` | empty | 32-byte AES key as hex |
| `ESV2_MQTT_DECRYPT_IV_HEX` | empty | 16-byte IV as hex |
| `ESV2_MQTT_DECRYPT_GZIP` | `false` | enable gzip pre-step |

Empty key → pass-through: the payload is parsed as plain JSON. Both key and IV must be set together; one without the other is a config error.

!!! warning "Not wire-compatible with current eda-xp output"
    v2 on `main` has no mode for plain `base64(gzip(json))` (gunzip only runs after AES decryption), and `internal/decode` expects an outer envelope `{"message": {"meter", "energy", "ecId"}}` with a single `energy` object. eda-xp publishes the message without that envelope and with `energy` as an array (see [energystore v1](energystore.md#mqtt-input)). The decode path needs adapting before a cutover.


## Auth

JWT verify against Keycloak via go-oidc discovery (`ESV2_AUTH_APP_ISSUER`, `ESV2_AUTH_APP_CLIENT_ID`). The audience check is off unless `ESV2_AUTH_APP_AUDIENCE` / `ESV2_AUTH_API_AUDIENCE` is set (then `aud` must contain it). Auth is **on by default** (fail-closed): startup fails without issuer and client id unless `ESV2_DEV_NO_AUTH=true` (dev only). The Basic-auth routes (`/query/rawdata`, `/query/{ecid}/metadata`) use a password grant via `ESV2_AUTH_API_ISSUER`, `ESV2_AUTH_API_CLIENT_ID`, `ESV2_AUTH_API_CLIENT_SECRET`.

Tenant resolution (`pickTenant` in `internal/auth/middleware.go`) falls back in order:

1. `X-Tenant` HTTP header (REST + GraphQL)
2. `tenant` header (back-compat with prod v1 callers)
3. `tenant` path parameter (`/api/v1/energy/{tenant}/{ec}/…`)
4. `ecid` path parameter (REST)
5. Single-tenant JWT claim (when the JWT carries exactly one tenant)

Explicit-but-wrong tenant always → 403 (no fallback past a wrong explicit value).

## REST + GraphQL APIs

REST surface mirrors v1's `/eeg/v2/{ecid}/...` and `/eeg/{ecid}/...` endpoints with the same wire shape (intentional — the customer SPA was reused unchanged), plus `/query/rawdata`, `/query/{ecid}/metadata`, `/api/v1/energy/{tenant}/{ec}/range` and `…/last-record-date`, `/healthz`, `/readyz` and `/metrics` (Prometheus). Not ported: v1's `rawdata/delete` and the GET variants of `load-curve-report` / `combined-report`. GraphQL is served at `POST /query` with a tenant fallback consistent with REST.

The `/excel/report/download` endpoint produces an XLSX with the v1-compatible sheet layout; wire-shape compatibility with v1 was verified during the pilot rollout.

## Cutover open questions

Besides the wire-format and topic gaps above, three open architecture decisions remain:

1. **MQTT-payload encryption** — keep the current envelope (v1 status quo), replace with proper crypto (per-tenant keys, AES-GCM, Vault), or remove entirely (cluster-internal traffic is already isolated). Tracked as an internal architecture decision.
2. **`processhistory` retention** — backend audit table without a retention policy. Outside the energystore-v2 scope but related to the cutover (tracked in the backend repository).
3. **Eventual ELWG-Q4-2026 alignment** — multi-participation, peer-to-peer contracts, self-supply across multiple meters. T-/R-OBIS variants already accepted (PR #39); deeper schema changes tracked as energystore-v2 issue #40.

## Related

- [energystore (v1)](energystore.md) — the production implementation being replaced
- [Architecture / Messaging](../architecture/messaging.md) — MQTT topic + payload conventions
- [Architecture / Databases](../architecture/databases.md) — TimescaleDB role
- [reference/obis-codes](../reference/obis-codes.md) — meter-code mapping including T-/R-variants
- ADR-0010 (TimescaleDB choice) and ADR-0011 (production sizing) in the upstream `gemeinstrom/eegfaktura-platform` repo (not the `vfeeg-development` platform repo, whose ADR-0010/0011 cover other decisions)
