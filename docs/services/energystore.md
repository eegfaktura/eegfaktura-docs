# energystore (v1)

!!! warning "Two implementations in active use"
    This page describes **energystore v1**, the embedded-Badger implementation that runs in production today. A parallel **[energystore-v2](energystore-v2.md)** based on TimescaleDB is live in the pilot cluster and is the planned cutover target. See ADR-0010 in the `eegfaktura-platform` repo for the architectural decision and [energystore-v2](energystore-v2.md) for the v2-specific design.

    v1 will continue to exist as the production binary until the cutover is finalised. The two implementations share the same wire-format expectations from the EDA pipeline; storage layer and report APIs differ.

Go service that stores per-period energy time-series and answers report queries. Backed by an embedded Badger KV store on a PVC, not by PostgreSQL.

## At a glance

| | |
|---|---|
| Language | Go 1.24 (module `at.ourproject/energystore`) |
| Storage | BadgerDB v4 (embedded KV) on PVC, values MessagePack-encoded |
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

energystore writes to an embedded BadgerDB v4 KV store with per-tenant / per-EC isolated instances; values are encoded with MessagePack (`tinylib/msgp`). The storage root is the container `VOLUME /opt/rawdata` (production `config-prod.yaml` sets `persistence.path: /opt/energy/rawdata`). Logical structure:

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

energystore uses `eclipse/paho.mqtt.golang` v1.5.1 (MQTT 3.1.1), with QoS 1 by default. The broker connection, client id, QoS and topics are read from config (`mqtt.*`).

!!! note "Production topic and payload differ from the README"
    The legacy README documented `eegfaktura/<tenant>/energy/<meterId>` with a plain-JSON payload. The actual energy subscription pattern (config `mqtt.energySubscriptionTopic`) is:

    - **Topic**: `eda/response/+/protocol/cr_msg`. Note a config/doc inconsistency: `config-prod.yaml` may still carry the older `eda/response/energy/+` pattern, so confirm the deployed ConfigMap value rather than assuming one.
    - **Inverter topic**: `mqtt.inverterSubscriptionTopic` (e.g. `eda/response/+/protocol/inverter_msg`) is present in config but the server code only subscribes to the energy topic — the inverter topic is **configured but not actually subscribed** in v1.
    - **Payload**: `base64(gzip(json))`. The producer ([eda-xp](eda-xp.md)) gzip-compresses the per-message JSON and base64-encodes it; energystore base64-decodes, gunzips, then parses JSON. There is **no encryption** in the v1 code path despite the misleadingly named `decryptMessage` helper.

    The plain-JSON pattern below describes the **logical** payload after base64-decode + decompression.

After base64-decode + decompression the payload is JSON:

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

Badger persists to the configured `persistence.path` (container `VOLUME /opt/rawdata`), mounted from a PVC. The PVC is the single source of truth for energy data — energystore does **not** own a PostgreSQL schema.

## Config

Config is loaded via viper from `config.yaml` (under `-configPath`) and overridden by environment variables. Viper uses the prefix `ENERGYSTORE_` with `.` → `_`, so `mqtt.host` maps to `ENERGYSTORE_MQTT_HOST`, etc. A few variables (`PORT`, `KEYCLOAK_CONFIG`) are read directly and are **not** prefixed.

| Variable | Purpose |
|----------|---------|
| `ENERGYSTORE_MQTT_HOST` | broker connection (`host:port`) |
| `ENERGYSTORE_MQTT_ID` | MQTT client id |
| `ENERGYSTORE_MQTT_QOS` | subscription QoS (default 1) |
| `ENERGYSTORE_MQTT_ENERGYSUBSCRIPTIONTOPIC` | energy-data topic |
| `ENERGYSTORE_MQTT_INVERTERSUBSCRIPTIONTOPIC` | inverter topic (configured, not subscribed) |
| `ENERGYSTORE_PERSISTENCE_PATH` | Badger storage path |
| `ENERGYSTORE_JWT_PUBKEYFILE` | JWT public key file |
| `ENERGYSTORE_SERVICES_MAIL_SERVER` | mail service address |
| `ENERGYSTORE_SERVICES_MASTER_SERVER` | masterdata service address |
| `PORT` | HTTP listen port (default 8080, read directly) |
| `KEYCLOAK_CONFIG` | path to `keycloak.json` (default `./keycloak.json`, read directly) |

## Build and image

- Source: `eegfaktura-energystore`
- Build: **single-stage** `Dockerfile` on `golang:1.24` (not multi-stage, not distroless). The Go builder image is also the runtime image; `go build` produces the `energystore` binary at `/usr/local/bin/energystore`.
- gRPC/protobuf code is generated at build time via `protoc` (`make protoc`).
- Entrypoint: `CMD ["energystore", "-configPath", "/etc/energystore/", "-logtostderr=true", "-stderrthreshold=INFO"]` — the `-logtostderr` / `-stderrthreshold` flags come from `glog`.
- Tag: commit SHA

!!! note "Binary name discrepancy"
    The Docker build produces a binary named `energystore` (no hyphen), but the `Makefile` `make build` target produces `energy-store` (with hyphen, `BINARY_NAME=energy-store`). The image and `CMD` use the un-hyphenated `energystore`.

Because the build uses a single-stage `golang` image as the runtime, the image is larger and carries a wider CVE surface than a multi-stage / distroless build would. Migrating to multi-stage is a potential future improvement.

## Known limitations (drove the v2 design)

- **No multi-replica scaling**: BadgerDB embedded + ReadWriteOnce-PVC technically prevent running more than one replica.
- **Full-range read-modify-write per EDA-message**: an incoming CR_MSG triggers reading the entire time range from disk, merging, and writing back — produces large RAM and CPU spikes under load.
- **In-process tenant lock**: a `Turns.lock(tenant)` mutex serialises all EDA processing per tenant, capping throughput.
- **base64+gzip payload envelope (no encryption)**: incoming `cr_msg` payloads are `base64(gzip(json))` — the v1 code path base64-decodes and gunzips, then parses JSON. The `decryptMessage` helper performs no cryptography despite its name.

These points are the explicit motivation for [energystore-v2](energystore-v2.md). See ADR-0010 in `eegfaktura-platform` for the architecture justification.

## Related

- **[energystore-v2](energystore-v2.md)** — the TimescaleDB-based replacement
- [Architecture / Messaging](../architecture/messaging.md) — energy-data MQTT pipeline
- [Architecture / Databases](../architecture/databases.md) — energystore vs PG split
- [reference/obis-codes](../reference/obis-codes.md) — code → slot table
