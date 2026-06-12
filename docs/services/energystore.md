# energystore (v1)

!!! warning "Two implementations in active use"
    This page describes **energystore v1**, the embedded-Badger implementation that runs in production today. A parallel **[energystore-v2](energystore-v2.md)** based on TimescaleDB is live in the pilot cluster and is the planned cutover target. See ADR-0010 in the `eegfaktura-platform` repo for the architectural decision and [energystore-v2](energystore-v2.md) for the v2-specific design.

    v1 will continue to exist as the production binary until the cutover is finalised. The two implementations share the same wire-format expectations from the EDA pipeline; storage layer and report APIs differ.

Go service that stores per-period energy time-series and answers report queries. Backed by an embedded Badger KV store on a PVC, not by PostgreSQL.

## At a glance

| | |
|---|---|
| Language | Go |
| Storage | Badger (embedded KV) on PVC |
| Bus | MQTT subscriber |
| Auth | JWT verify, `X-Tenant` header check |
| Per-EEG bucket | one logical bucket per EEG `tenant` |
| Replicas | Single (BadgerDB lock + RWO PVC) |
| Source repo | `eegfaktura-energystore` |

## Responsibilities

- Subscribe to MQTT energy-data topic, decode `MqttEnergyMessage`, persist to Badger
- Serve per-period reports (consumption / production / EEG share) to the customer SPA and billing
- Serve the network-operator's allocation values via the `AllocDynamicV2` pass-through (energystore does not compute allocations — see below)
- Register new metering points with the correct direction (consumer vs producer) on first publish

## Storage shape

energystore writes to a Badger KV store at `/data`. Logical structure:

```
<tenant>/
├── <meter-id>/
│   ├── consumer[0]  → consumption (G.01)
│   ├── consumer[1]  → G.02 (allocation)
│   ├── consumer[2]  → G.03 (utilization)
│   ├── producer[0]  → production (G.01)
│   └── producer[1]  → P.01 (allocation)
```

Each slot stores values per quarter-hour for the configured time range. There is no PostgreSQL involvement.

## MQTT input

!!! note "Production topic and payload differ from the README"
    The legacy README documented `eegfaktura/<tenant>/energy/<meterId>` with a plain-JSON payload. The actual production subscription pattern, as configured in the `eegfaktura-energystore-config` ConfigMap, is:

    - **Topic**: `eda/response/+/protocol/cr_msg` (energy data) and `eda/response/+/protocol/inverter_msg` (PV inverter)
    - **Payload**: an encrypted envelope, *not* plain JSON. The producer ([eda-xp](eda-xp.md)) wraps the per-message JSON; energystore unwraps it before the JSON parse — see [eda-xp.md#cr_msg-payload-encryption](eda-xp.md#cr_msg-payload-encryption) for the wire format.

    The plain-JSON pattern below describes the **logical** payload after decryption.

After decryption + decompression the payload is JSON:

```json
{
  "meterId": "AT003000...",
  "data": [
    {"meterCode": "1-1:1.9.0 G.01", "value": [0.12, 0.15, ...]},
    {"meterCode": "1-1:2.9.0 G.02", "value": [...]},
    {"meterCode": "1-1:2.9.0 G.03", "value": [...]}
  ]
}
```

The `meterCode` decoder maps each code to a storage slot. See [reference/obis-codes](../reference/obis-codes.md) for the complete table.

### Direction detection

A new metering point is registered as a **producer** only if its first published payload contains a `P.01` or `P.01T` code. A payload with only `G.01` is ambiguous (consumers also have a `G.01`) and the meter is registered as a consumer.

For sample-data publishers: a producer's first publish must include a `P.01` value, even if zero. Otherwise the meter is permanently classified as a consumer (correction requires deleting the bucket and re-publishing).

## AllocDynamicV2 — pass-through, not a calculation

The `AllocDynamicV2` allocation is implemented as a filter over the stored matrix, not a derivation. energystore expects all three consumer values (`G.01`, `G.02`, `G.03`) and both producer values (`G.01`, `P.01`) to be pre-computed by the network operator.

Practical consequences:

- For real instances, this is what arrives via EDA.
- For mock / sample data, the publisher must produce all five codes consistently and keep them coherent (`G.03 ≤ G.02`, `G.03 ≤ G.01`, etc).
- Missing G.02 / G.03 → consumers see "0 EEG share" in the customer SPA.

## REST API

Customer SPA addresses energystore with tenant scoping. The most common endpoints serve per-period reports:

```
GET /report?tenant=<tenant>&from=<iso8601>&to=<iso8601>
```

with header:

```
X-Tenant: <tenant>
```

The header value must be in the JWT's `tenant` array; otherwise 403.

The report response is the matrix shape consumed by the SPA's chart code. See [reference/obis-codes](../reference/obis-codes.md) for the field-name caveat (`allocated` = EEG share, `consumed` = EVU share — counter-intuitive).

## Auth

JWT verify against Keycloak's JWKS. The middleware checks the `X-Tenant` header against the JWT's `tenant` claim (array). Mismatch → 403.

There is no role check beyond authenticated-and-tenant-matches — `EEG_USER` and `EEG_ADMIN` see the same data scope (their own tenant).

## Persistence

Badger persists to `/data`, mounted from a PVC. The PVC is the single source of truth for energy data. Two operational consequences:

- Wipe-replay of the namespace destroys this data; restore requires a PVC snapshot or re-import via EDA.
- The PVC is RWO (single-writer). energystore is a single-replica deployment.

The PVC dominates the storage footprint for the namespace.

## Config

| Variable | Purpose |
|----------|---------|
| `MQTT_HOST`, `MQTT_PORT`, `MQTT_USER`, `MQTT_PASS` | broker connection |
| `KEYCLOAK_URL`, `KEYCLOAK_REALM` | JWKS source |
| `BADGER_DIR` | typically `/data` |
| `LOG_LEVEL` | log verbosity |

## Build and image

- Source: `eegfaktura-energystore`
- Build: multi-stage Docker, distroless final
- Tag: commit SHA

The build image should not be shipped as the runtime image. The image-size and CVE-surface impact of using the Go builder image directly is significant. Multi-stage distroless is the convention.

## Known limitations (drove the v2 design)

- **No multi-replica scaling**: BadgerDB embedded + ReadWriteOnce-PVC technically prevent running more than one replica.
- **Full-range read-modify-write per EDA-message**: an incoming CR_MSG triggers reading the entire time range from disk, merging, and writing back — produces large RAM and CPU spikes under load.
- **In-process tenant lock**: a `Turns.lock(tenant)` mutex serialises all EDA processing per tenant, capping throughput.
- **Topic-scoped payload encryption**: the encryption envelope applies to the `cr_msg` topic only. See [eda-xp.md#cr_msg-payload-encryption](eda-xp.md#cr_msg-payload-encryption).

These points are the explicit motivation for [energystore-v2](energystore-v2.md). See ADR-0010 in `eegfaktura-platform` for the architecture justification.

## Related

- **[energystore-v2](energystore-v2.md)** — the TimescaleDB-based replacement
- [Architecture / Messaging](../architecture/messaging.md) — energy-data MQTT pipeline
- [Architecture / Databases](../architecture/databases.md) — energystore vs PG split
- [reference/obis-codes](../reference/obis-codes.md) — code → slot table
