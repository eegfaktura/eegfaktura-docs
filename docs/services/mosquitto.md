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

!!! warning "The legacy topic for energystore is outdated"
    The legacy README listed `eegfaktura/<tenant>/energy/<meterId>` as the energy topic. The actual production v1 stack uses `eda/response/<tenant>/protocol/cr_msg` (and `…/inverter_msg` for PV inverter feeds). The mismatch was discovered 2026-06-07 during the v2 cutover analysis. See [eda-xp.md#mqtt-output](eda-xp.md#mqtt-output).

| Topic (prod) | Publisher | Subscriber | Payload |
|---|---|---|---|
| `eda/<tenant>/protocol/<process_lower>` | eda-xp / eda-mock | backend | `EbMsMessage` (JSON via circe) |
| `eda/response/<tenant>/protocol/cr_msg` | eda-xp | energystore | **AES-256-CBC + gzip + base64** envelope around `MqttEnergyMessage` JSON |
| `eda/response/<tenant>/protocol/inverter_msg` | eda-xp | energystore | plain JSON `MqttEnergyMessage` (PV inverter; not encrypted) |
| `eegfaktura/<tenant>/energy/<meterId>` | pilot energy publishers only | [energystore-v2](energystore-v2.md) | plain `MqttEnergyMessage` JSON (pilot convention) |

`<tenant>` is the EEG's `community_id`. `<process_lower>` is the EDA process code lowercased (`cm_rev_sp`, `ec_req_onl`, …).

The `cr_msg` topic is the one carrying actual energy values per metering point, hence the encryption. The `cr_req_pt` topic (outbound period requests) is cleartext — request metadata only. See [eda-xp.md#cr_msg-payload-encryption](eda-xp.md#cr_msg-payload-encryption) for the full crypto pipeline.

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

### MQTT 5 vs 3.1.1 — Shared Subscriptions caveat

The broker accepts both MQTT 3.1.1 and MQTT 5 clients. **Shared Subscriptions (`$share/<group>/<topic>`) are an MQTT 5 feature.** Mosquitto silently accepts the subscribe pattern from a 3.1.1 client but then never matches any publish to it — you get a connected client that receives zero messages and no error.

[energystore-v2](energystore-v2.md) ran into this with `paho.mqtt.golang v1` (MQTT 3.1.1) and was migrated to `eclipse/paho.golang` (MQTT 5). Anything new that wants Shared Subscriptions must do the same.

Also: MQTT 5 §4.8.2 forbids `NoLocal=true` on a Shared Subscription. Setting it results in disconnect with `reason_code=130 (Protocol Error)`. Subscribers should leave the flag at default `false`.

### Burst-load caveat

Mosquitto's default per-client `max_inflight_messages` and `max_queued_messages` are low enough that a sustained burst on `qos=1` can result in silent drops: the broker returns PUBACK to the publisher but the subscriber sees only a subset. The chart provides an `extraConfig` value to raise the limits for environments that need it. See the chart values for the current defaults.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full topic flow
- [services/backend](backend.md) — main subscriber on EDA topics
- [services/energystore](energystore.md) — subscriber on energy topics
- [services/eda-xp](eda-xp.md) — main publisher
