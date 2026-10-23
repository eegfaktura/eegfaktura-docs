# energystore (v1)

!!! warning "Two implementations"
    This page describes **energystore v1**, the embedded-Badger implementation that runs in production (and in the pilot zone) today. A successor, **[energystore-v2](energystore-v2.md)** based on TimescaleDB, exists as a separate repository but is **not rolled out** in any eegfaktura deployment. See ADR-0010 in the upstream `gemeinstrom/eegfaktura-platform` repo (referenced from the v2 README) for the architectural decision and [energystore-v2](energystore-v2.md) for the v2-specific design.

    v1 remains the production binary until a cutover is decided. Both implementations are meant to consume the same EDA energy messages, but v2 on `main` does not yet decode the current `cr_msg` wire format (see [energystore-v2](energystore-v2.md#optional-payload-decrypt)); storage layer and report APIs differ.

Go service that stores per-period energy time-series and answers report queries. Backed by an embedded Badger KV store on a PVC, not by PostgreSQL.

## At a glance

| | |
|---|---|
| Language | Go 1.26 (`go 1.26.0`, toolchain `go1.26.9`; module `at.ourproject/energystore`) |
| Storage | BadgerDB v4 (`badger/v4` v4.9.1, embedded KV) on PVC, values JSON-encoded |
| Bus | MQTT subscriber (paho.mqtt.golang, MQTT 3.1.1) |
| APIs | REST (gorilla/mux) + GraphQL (gqlgen, `/query`); gRPC clients to backend and eda-xp |
| Auth | JWT verify via go-oidc, `tenant` / `X-Tenant` header check |
| Per-EEG store | one Badger database per (`tenant`, `ecId`) |
| Replicas | Single (BadgerDB lock + RWO PVC) |
| Source repo | `eegfaktura-energystore` |

## Responsibilities

- Subscribe to the MQTT energy-data topic, decode `MqttEnergyMessage`, persist to Badger
- After each successful import, publish a short history record (metering point, conversation id, message code, periods) to `eda/response/<tenant>/protocol/cr_msg_history`, which the [backend](backend.md) writes into its process history
- Serve per-period reports (consumption / production / EEG share) to the customer SPA and billing
- Serve the network-operator's allocation values via the `AllocDynamicV2` pass-through (energystore does not compute allocations — see below)
- Register new metering points with the correct direction (consumer vs producer) on first publish
- Excel import (GraphQL `singleUpload`) and Excel energy export

## Storage shape

energystore writes to an embedded BadgerDB v4 KV store with one database per tenant and EC, in the directory `<persistence.path>/<tenant>/<ecId>`. `OpenStorage` rejects any tenant or ecId that is not strictly alphanumeric before it touches the filesystem, and the connection pool is keyed by tenant **and** ecId. Values are encoded with the default JSON codec of the `store/ebow` wrapper (a MessagePack codec, `tinylib/msgp`, exists in `store/ebow/codec/msgp` but is not wired in). The storage root is the container `VOLUME /opt/rawdata` (production `config-prod.yaml` sets `persistence.path: /opt/energy/rawdata`).

Logical structure inside one database:

```
<persistence.path>/<tenant>/<ecId>/   (one Badger DB)
├── cpmeta/0          → metadata of all metering points of the community
│                       (direction, source index, active periods)
└── CP/<row id>       → one row per quarter hour (row id encodes local wall-clock time)
    ├── consumers[]   → 3 slots per consumer at its source index:
    │                    +0 consumption (1.9.0 G.01), +1 G.02 (allocation), +2 G.03 (utilization)
    ├── producers[]   → 2 slots per producer: +0 production (2.9.0 G.01), +1 P.01 (surplus)
    └── qovconsumers[] / qovproducers[] → quality per slot (L1/L2/L3 → 1/2/3, 0 = none)
```

Each row holds the values of all metering points of the community for one quarter hour. There is no PostgreSQL involvement.

## MQTT input

energystore uses `eclipse/paho.mqtt.golang` v1.5.1 (MQTT 3.1.1, `SetProtocolVersion(4)`) with a persistent session (`CleanSession(false)`) and auto-reconnect. Broker address, client id, QoS (`1` in the shipped `config.yaml`) and topic are read from config (`mqtt.*`). Messages are dispatched to one worker per tenant (tenant = third topic segment, upper-cased); a worker closes its databases after 15 s without messages.

!!! note "Production topic and payload differ from the README"
    The legacy README documented `eegfaktura/<tenant>/energy/<meterId>` with a plain-JSON payload. The actual energy subscription pattern (config `mqtt.energySubscriptionTopic`) is:

    - **Topic**: `eda/response/+/protocol/cr_msg` (shipped `config.yaml`). The repository's `config-prod.yaml` still carries the older `eda/response/energy/+` pattern; production does not use that file's topic.
    - **Inverter topic**: `mqtt.inverterSubscriptionTopic` (e.g. `eda/response/+/protocol/inverter_msg`) is present in config but the server code only subscribes to the energy topic — the inverter topic is **configured but not actually subscribed** in v1.
    - **Payload**: `base64(gzip(json))`. The producer ([eda-xp](eda-xp.md)) gzip-compresses the per-message JSON and base64-encodes it; energystore base64-decodes, gunzips, then parses JSON. There is **no encryption** in the v1 code path despite the misleadingly named `decryptMessage` helper. A message that cannot be decoded is logged and dropped.

After base64-decode + decompression the payload is JSON (`model/mqtt.go`, no outer envelope):

```json
{
  "conversationId": "...",
  "messageCode": "...",
  "ecId": "...",
  "meter": {"meteringPoint": "AT003000...", "direction": "..."},
  "energy": [
    {
      "start": 1700000000000,
      "end": 1700086400000,
      "data": [
        {"meterCode": "1-1:1.9.0 G.01", "value": [{"from": 1700000000000, "to": 1700000900000, "method": "L1", "value": 0.12}, ...]},
        {"meterCode": "1-1:2.9.0 G.02", "value": [...]},
        {"meterCode": "1-1:2.9.0 G.03", "value": [...]}
      ]
    }
  ]
}
```

Each `energy` block is split into day blocks, which are stored one after another. The `meterCode` decoder maps each code to a storage slot. See [reference/obis-codes](../reference/obis-codes.md) for the complete table.

### Direction detection

A new metering point is registered as a **producer** if its first published payload contains a generation code — `1-1:2.9.0 G.01` / `G.01T` or `1-1:2.9.0 P.01` / `P.01T` (`utils.ExamineDirection`). Otherwise (e.g. only `1-1:1.9.0 G.01`, `G.02`, `G.03`) it is registered as a consumer.

For sample-data publishers: a producer's first publish must include at least one of these generation codes. Otherwise the meter is permanently classified as a consumer (correction requires deleting the data and re-publishing).

## AllocDynamicV2 — pass-through, not a calculation

The `AllocDynamicV2` allocation is implemented as a filter over the stored matrix, not a derivation. energystore expects all three consumer values (`G.01`, `G.02`, `G.03`) and both producer values (`G.01`, `P.01`) to be pre-computed by the network operator.

Practical consequences:

- For real instances, this is what arrives via EDA.
- For mock / sample data, the publisher must produce all five codes consistently and keep them coherent (`G.03 ≤ G.02`, `G.03 ≤ G.01`, etc).
- Missing G.02 / G.03 → consumers see "0 EEG share" in the customer SPA.

## REST API

The customer SPA addresses energystore per EC; the tenant comes from a header (see [Auth](#auth)). Routes in `rest/restServer.go`, all behind `ProtectApp`:

| Method | Path | Purpose |
|---|---|---|
| POST | `/eeg/v2/{ecid}/report` | participant report for a period (optional time-of-use `timeWindows`, see below) |
| GET | `/eeg/v2/{ecid}/meta` | metering-point metadata |
| POST | `/eeg/v2/{ecid}/raw` | raw values for metering points |
| POST | `/eeg/v2/{ecid}/rawdata/delete` | zero the raw data of one metering point in a range (`dryRun` supported); requires realm role `superuser` |
| POST | `/eeg/v2/{ecid}/intra-day-report` | intra-day report |
| GET, POST | `/eeg/v2/{ecid}/load-curve-report` | load curve |
| GET, POST | `/eeg/v2/{ecid}/combined-report` | several reports in one call |
| POST | `/eeg/v2/{ecid}/summary` | period summary |
| POST | `/eeg/v2/{ecid}/billing-check` | checks before billing (L3 quarter hours, gaps, community totals); read only |
| GET | `/eeg/{ecid}/lastRecordDate` | end of the last stored period |
| POST | `/eeg/{ecid}/excel/export/{year}/{month}` | energy export, sent by mail (via the eda-xp gRPC mail service) |
| POST | `/eeg/{ecid}/excel/report/download` | energy export as XLSX download |

Two further routes, `POST /query/rawdata` and `POST /query/{ecid}/metadata`, use Basic auth (`ProtectApi`, see [Auth](#auth)). GraphQL (`/query`) offers the query `lastEnergyDate` and the mutation `singleUpload` (Excel import).

Since 1.7.0 the participant report accepts up to two time windows per metering point (`timeWindows: [{key: T1|T2, from: "HH:MM", to: "HH:MM"}]`, 15-min raster); the response then carries per-window sums (`report.buckets: [{key: BASE|T1|T2, kWh}]`). Invalid windows → 400. energystore has no tariff or price knowledge.

The report response is the matrix shape consumed by the SPA's chart code. See [reference/obis-codes](../reference/obis-codes.md) for the field-name caveat (`allocated` = EEG share, `consumed` = EVU share — counter-intuitive).

## Auth

The bearer JWT is verified with go-oidc against the Keycloak realm configured in `keycloak.json` (entry `app`); `iss` is checked, the audience check is off (`SkipClientIDCheck: true`). The tenant is read from the `tenant` header, falling back to `X-Tenant`, and must be in the JWT's `tenant` claim (array, compared case-insensitively). Missing token or tenant → 403, invalid token → 401, tenant mismatch → 403 on REST and 401 on GraphQL. The GraphQL resolvers also compare their `tenant` argument with the verified tenant.

Users with the realm role `superuser` skip the tenant-membership check. Apart from that there is no role check — `EEG_USER` and `EEG_ADMIN` see the same data scope (their own tenant); only `rawdata/delete` explicitly requires `superuser`.

The `/query/rawdata` and `/query/{ecid}/metadata` routes take `Authorization: Basic …` credentials, run a password grant against the `api` client from `keycloak.json`, and check `X-Tenant` against the resulting token's `tenant` claim.

## Persistence

Badger persists to the configured `persistence.path` (container `VOLUME /opt/rawdata`), mounted from a PVC. The PVC is the single source of truth for energy data — energystore does **not** own a PostgreSQL schema.

### Value-log garbage collection

Since 1.5.0 the service can run Badger's `RunValueLogGC` in the background (**off by default**). The community-wide `cpmeta/0` record exceeds the 128 KB `ValueThreshold` above roughly 750 metering points, and every period extension writes a complete new copy into the value log, which only the value-log GC frees. The run happens inside the service (not as a CronJob) because Badger locks each directory exclusively; it opens the largest databases one after another through the pool, within a time window and a byte budget. Settings under `persistence.vlogGC` (see [Config](#config)).

## Config

Config is loaded via viper from `config.yaml` (under `-configPath`) and overridden by environment variables. Viper uses the prefix `ENERGYSTORE_` with `.` → `_` (other characters such as `-` are kept), so `mqtt.host` maps to `ENERGYSTORE_MQTT_HOST`, etc. A few variables (`PORT`, `KEYCLOAK_CONFIG`) are read directly and are **not** prefixed.

| Variable | Purpose |
|----------|---------|
| `ENERGYSTORE_MQTT_HOST` | broker URL (`tcp://<host>:1883`) |
| `ENERGYSTORE_MQTT_ID` | MQTT client id |
| `ENERGYSTORE_MQTT_QOS` | subscription QoS (`1` in `config.yaml`) |
| `ENERGYSTORE_MQTT_ENERGYSUBSCRIPTIONTOPIC` | energy-data topic |
| `ENERGYSTORE_MQTT_INVERTERSUBSCRIPTIONTOPIC` | inverter topic (configured, not subscribed) |
| `ENERGYSTORE_PERSISTENCE_PATH` | Badger storage path |
| `ENERGYSTORE_PERSISTENCE_VLOGGC_ENABLED` | value-log GC on/off (default `false`) |
| `ENERGYSTORE_PERSISTENCE_VLOGGC_DRYRUN` | only log what a run would do (default `false`) |
| `ENERGYSTORE_PERSISTENCE_VLOGGC_WINDOW`, `…_CHECKEVERY`, `…_DISCARDRATIO`, `…_PROBERATIO`, `…_MINVLOGMB`, `…_MAXGBPERRUN` | GC window (`02:00-04:30`, Europe/Vienna), check interval (`10m`), discard ratio (`0.5`), probe ratio (`0.1`), minimum value-log size (`100` MB), budget per run (`50` GB) |
| `ENERGYSTORE_SERVICES_MAIL-SERVER` | gRPC address of the eda-xp mail service (config key `services.mail-server`; the env name keeps the hyphen) |
| `ENERGYSTORE_SERVICES_MASTER-SERVER` | gRPC address of the backend masterdata service (config key `services.master-server`) |
| `PORT` | HTTP listen port (default 8080, read directly) |
| `KEYCLOAK_CONFIG` | path to `keycloak.json` (default `./keycloak.json`, read directly) |

The `jwt.*` keys in `config-prod.yaml` are not read by the current code (the key-file JWT middleware is commented out; tokens are verified through OIDC discovery).

## Build and image

- Source: `eegfaktura-energystore`
- Build: **single-stage** `Dockerfile` on `golang:1.26.9` (not multi-stage, not distroless). The Go builder image is also the runtime image; `go build` produces the `energystore` binary at `/usr/local/bin/energystore` and the `estore` ops CLI (cobra) at `/usr/local/bin/estore`.
- The container runs as the non-root user `app` (UID/GID 1000), which owns `/opt/rawdata` and `/opt/energy`.
- gRPC/protobuf code (`protoc/*.pb.go`) is checked in; `make protoc` regenerates it. The image build does not run `protoc`.
- Entrypoint: `CMD ["energystore", "-configPath", "/etc/energystore/", "-logtostderr=true", "-stderrthreshold=INFO"]` — the `-logtostderr` / `-stderrthreshold` flags come from `glog`.
- Tags: `vX.Y.Z` for releases, `sha-<short>` per commit, `latest` on the default branch.

!!! note "Binary name discrepancy"
    The Docker build produces a binary named `energystore` (no hyphen), but the `Makefile` `make build` target produces `energy-store` (with hyphen, `BINARY_NAME=energy-store`). The image and `CMD` use the un-hyphenated `energystore`.

Because the build uses a single-stage `golang` image as the runtime, the image is larger and carries a wider CVE surface than a multi-stage / distroless build would. Migrating to multi-stage is a potential future improvement.

## Known limitations (drove the v2 design)

- **No multi-replica scaling**: BadgerDB embedded + ReadWriteOnce-PVC technically prevent running more than one replica.
- **Full-range read-modify-write per EDA-message**: an incoming CR_MSG triggers reading the entire time range from disk, merging, and writing back — produces large RAM and CPU spikes under load.
- **In-process tenant lock**: a `Turns.lock(tenant)` mutex serialises all EDA processing per tenant, capping throughput.
- **base64+gzip payload envelope (no encryption)**: incoming `cr_msg` payloads are `base64(gzip(json))` — the v1 code path base64-decodes and gunzips, then parses JSON. The `decryptMessage` helper performs no cryptography despite its name.

These points are the explicit motivation for [energystore-v2](energystore-v2.md). See ADR-0010 in `gemeinstrom/eegfaktura-platform` for the architecture justification.

## Related

- **[energystore-v2](energystore-v2.md)** — the TimescaleDB-based replacement
- [Architecture / Messaging](../architecture/messaging.md) — energy-data MQTT pipeline
- [Architecture / Databases](../architecture/databases.md) — energystore vs PG split
- [reference/obis-codes](../reference/obis-codes.md) — code → slot table
