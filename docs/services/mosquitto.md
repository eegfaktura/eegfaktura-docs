# mosquitto

Eclipse Mosquitto MQTT broker. The platform's message bus, used for EDA inbound dispatch (eda-xp → backend) and energy-data ingestion (publishers → energystore).

## At a glance

| | |
|---|---|
| Image | `eclipse-mosquitto:2.0.x` |
| Topology | single-instance StatefulSet |
| Port | 1883 (plain), no TLS by default |
| Persistence | PVC-backed |
| Optional sidecar | `mosquitto-exporter` (Prometheus metrics) |

## Topic catalogue

| Topic | Publisher | Subscriber | Payload |
|-------|-----------|------------|---------|
| `eda/<tenant>/protocol/<process_lower>` | eda-xp / eda-mock | backend | `EbMsMessage` (JSON via circe) |
| `eegfaktura/<tenant>/energy/<meterId>` | energy publisher / eda-xp | energystore | `MqttEnergyMessage` JSON |

`<tenant>` is the EEG's `community_id`. `<process_lower>` is the EDA process code lowercased (`cm_rev_sp`, `ec_req_onl`, …).

Subscriber-side details are documented per service: see [backend](backend.md) and [energystore](energystore.md).

## QoS and retention

The bus uses QoS 0 by default. Retained messages are not used.

Consequence: a consumer that is down during a publish loses that message. Recovery options:

- For EDA inbound, the producer (eda-xp) re-sends after a configurable timeout, **or** the original network operator re-issues the EDA process.
- For energy data, the publisher is expected to be idempotent for the same period (re-publishing the same period overwrites in energystore).

For production deployments, increase QoS and enable persistence + retention on the inbound EDA topic.

## Auth

Username + password by default. Optional TLS / mTLS.

The auth file is mounted via `ConfigMap` / `Secret`. Each connecting service has its own username:

| Service | Role |
|---------|------|
| eda-xp / eda-mock | publisher on `eda/+/protocol/#` |
| backend | subscriber on `eda/+/protocol/#` |
| energy publisher | publisher on `eegfaktura/+/energy/#` |
| energystore | subscriber on `eegfaktura/+/energy/#` |

ACLs are configured in `mosquitto.conf` (or an `acl_file` referenced from it).

## Persistence

Mosquitto persists its queue to a PVC. Without persistence (or with an undersized PVC), a broker restart loses in-flight messages.

The PVC is RWO; Mosquitto is a single-replica StatefulSet.

## Config

The main `mosquitto.conf` is typically mounted from a `ConfigMap`. Key directives:

| Directive | Typical value | Purpose |
|-----------|---------------|---------|
| `listener` | `1883` | bind port |
| `persistence` | `true` | enable on-disk persistence |
| `persistence_location` | `/mosquitto/data/` | PVC mount |
| `allow_anonymous` | `false` | require auth |
| `password_file` | `/mosquitto/auth/passwords` | mounted from a Secret |
| `acl_file` | `/mosquitto/auth/acl` | per-user topic ACLs |

## Metrics

An optional `mosquitto-exporter` sidecar (e.g. `sapcc/mosquitto-exporter`) exposes Prometheus metrics on a separate port. Useful for connection-count, message-rate, and queue-depth dashboards.

## Operational notes

- A "connection refused" from a backend that just started usually means the broker is reachable on TCP but has not yet accepted the auth — surface this in liveness/readiness probes by checking subscription state, not raw TCP.
- ACL changes require a Mosquitto restart in default config.
- The broker does **not** distinguish "no subscriber" from "delivered" — publishing to a topic nobody listens to silently succeeds. This is the dominant failure mode when adding new EDA protocol support; see [Architecture / Messaging](../architecture/messaging.md) gap point #5.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full topic flow
- [services/backend](backend.md) — main subscriber on EDA topics
- [services/energystore](energystore.md) — subscriber on energy topics
- [services/eda-xp](eda-xp.md) — main publisher
