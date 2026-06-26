# energystore-v2

Stateless Go service that stores per-period energy time-series in **TimescaleDB** and serves report queries. Designed as a drop-in replacement for [energystore (v1)](energystore.md). Live in the pilot cluster; production cutover is planned.

!!! info "Status"
    - **Pilot**: live, in production-equivalent setup (PR #41 + ADR-0010/0011)
    - **Production cutover**: planned, blocked by clarifying open architecture decisions — see [Cutover open questions](#cutover-open-questions) below
    - **Public source**: [`vfeeg-development/eegfaktura-energystore-v2`](https://github.com/vfeeg-development/eegfaktura-energystore-v2)

## At a glance

| | |
|---|---|
| Language | Go 1.26 |
| Storage | TimescaleDB (PostgreSQL extension) in a separate cluster |
| Bus | MQTT subscriber (paho.golang, MQTT 5) |
| Subscription | Shared Subscription `$share/energystore-v2/<topic>` (multi-replica capable) |
| Auth | JWT verify, X-Tenant header / claim fallback chain |
| State | Stateless (no PVC) |
| Replicas | Multi-replica capable (currently 1 in pilot) |

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
│   eda-comm       │ ─────────────────────►│   (1..N replicas)   │
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

Two tables in the `postgres-energy` cluster:

### `energy_data` — hypertable

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | text | partition key (hash, currently 16 buckets in pilot, planned 128 for prod) |
| `ec_id` | text | EEG identifier |
| `metering_point` | text | OBIS-style ID (AT…) |
| `meter_code` | text | full OBIS code, e.g. `1-1:1.9.0 G.01` (T-/R-variants accepted; see [reference/obis-codes](../reference/obis-codes.md)) |
| `ts` | timestamptz | slot timestamp |
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
| `direction` | smallint | 1=consumer, 2=generator |
| `source_idx` | int | original v1 index (compat) |
| `period_start` / `period_end` | timestamptz | active window |
| `payload` | jsonb | extension fields beyond the wire-protocol |
| `updated_at` | timestamptz | |

Compaction / retention policies (per ADR-0011): compression after 7 days, continuous aggregates for hourly / daily / monthly rollups.

## MQTT input

Subscribes (multi-replica capable) to:

| Env | Default in pilot |
|---|---|
| `ESV2_MQTT_TOPIC_PATTERN` | `eegfaktura/+/energy/+` (pilot) — **prod uses `eda/response/+/protocol/cr_msg`** |
| `ESV2_MQTT_SHARE_GROUP` | `energystore-v2` (becomes `$share/energystore-v2/<topic>` at subscribe time) |
| `ESV2_MQTT_QOS` | `1` |

!!! note "MQTT-5 Shared Subscriptions need paho.golang"
    v2 uses `github.com/eclipse/paho.golang` (MQTT 5). An earlier attempt with `paho.mqtt.golang v1` (MQTT 3.1.1) failed: Mosquitto silently accepts `$share/…` subscribes from a 3.1.1 client but never matches a publish to them. See also the [mosquitto operational notes](mosquitto.md#mqtt-5-vs-311-shared-subscriptions-caveat).

    Also: MQTT 5 §4.8.2 forbids `NoLocal=true` on Shared Subscriptions — the v2 subscriber sets it to `false`. Earlier dev versions had `NoLocal=true` and were disconnected by Mosquitto with `reason_code=130 (Protocol Error)`.

### Optional payload decrypt

The production v1 stack wraps `cr_msg` payloads in an encrypted envelope — see [eda-xp.md#cr_msg-payload-encryption](eda-xp.md#cr_msg-payload-encryption) for the wire format. For drop-in compatibility v2 ships an optional pre-decode step, env-configurable:

| Env | Default | Effect |
|---|---|---|
| `ESV2_MQTT_DECRYPT_KEY_HEX` | empty | 32-byte AES key as hex |
| `ESV2_MQTT_DECRYPT_IV_HEX` | empty | 16-byte IV as hex |
| `ESV2_MQTT_DECRYPT_GZIP` | `false` | enable gzip pre-step |

Empty key → pass-through (pilot default). All three env vars set → full v1-prod pipeline replication. Both key and IV must be set together; one without the other is a config error.

The decrypt module (`internal/decode`) lives in the public v2 repo; the actual prod key/IV intentionally do *not*.

## Auth

JWT verify against Keycloak. Tenant resolution falls back in order:

1. `X-Tenant` HTTP header (REST + GraphQL)
2. `tenant` header (back-compat with prod v1 callers)
3. `ecid` path parameter (REST)
4. Single-tenant JWT claim (when the JWT carries exactly one tenant)

Explicit-but-wrong tenant always → 403 (no fallback past a wrong explicit value). See `feedback_v2_tenant_from_jwt` for the security gate rationale.

## REST + GraphQL APIs

REST surface mirrors v1's `/eeg/v2/{ecid}/...` endpoints with the same wire shape (intentional — the customer SPA was reused unchanged). GraphQL surface adds an `/query` endpoint with a tenant fallback consistent with REST.

The `/excel/report/download` endpoint produces an XLSX with the v1-compatible sheet layout; wire-shape compatibility with v1 was verified during the pilot rollout.

## Scaling note

The v2 app tier is stateless and intentionally lightweight: per-message work is decode + upsert, no in-process state, no range-read-merge. The load-bearing component is `postgres-energy` (TimescaleDB), not the v2 pod.

## Cutover open questions

The v2 implementation is feature-complete for the cutover, but three open architecture decisions remain:

1. **MQTT-payload encryption** — keep the current envelope (v1 status quo), replace with proper crypto (per-tenant keys, AES-GCM, Vault), or remove entirely (cluster-internal traffic is already isolated). Tracked as an internal architecture decision.
2. **`processhistory` retention** — backend audit table without a retention policy. Outside the energystore-v2 scope but related to the cutover (tracked in the backend repository).
3. **Eventual ELWG-Q4-2026 alignment** — multi-participation, peer-to-peer contracts, self-supply across multiple meters. T-/R-OBIS variants already accepted (PR #39); deeper schema changes tracked as energystore-v2 issue #40.

## Related

- [energystore (v1)](energystore.md) — the production implementation being replaced
- [Architecture / Messaging](../architecture/messaging.md) — MQTT topic + payload conventions
- [Architecture / Databases](../architecture/databases.md) — TimescaleDB role
- [reference/obis-codes](../reference/obis-codes.md) — meter-code mapping including T-/R-variants
- ADR-0010 (TimescaleDB choice) and ADR-0011 (production sizing) in the `eegfaktura-platform` repo
