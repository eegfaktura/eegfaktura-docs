# energystore

Go service that stores per-period energy time-series and answers report queries. Backed by an embedded Badger KV store on a PVC, not by PostgreSQL.

## At a glance

| | |
|---|---|
| Language | Go |
| Storage | Badger (embedded KV) on PVC |
| Bus | MQTT subscriber |
| Auth | JWT verify, `X-Tenant` header check |
| Per-EEG bucket | one logical bucket per EEG `tenant` |

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

Subscribes to `eegfaktura/<tenant>/energy/<meterId>`. The payload is JSON:

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

In production instances the PVC is large (hundreds of GiB). Plan for it in cluster sizing.

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

## Sizing

The energystore pod is historically one of the largest in the stack:

- Idle RAM is dominated by Badger's table cache; scales with the data size.
- CPU is light at idle, can spike when serving large reports.
- PVC growth is roughly linear in `(meters × time × codes)`. A small EEG might fit in 10 GiB; a large one needs hundreds of GiB.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — energy-data MQTT pipeline
- [Architecture / Databases](../architecture/databases.md) — energystore vs PG split
- [reference/obis-codes](../reference/obis-codes.md) — code → slot table
