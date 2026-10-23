# mosquitto

Eclipse Mosquitto MQTT broker. The platform's message bus, used for EDA inbound dispatch (eda-xp → backend), outbound EDA requests (backend → eda-xp) and energy-data ingestion (eda-xp → energystore).

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-mosquitto:latest` |
| Port | 1883 (plain), no TLS by default; internal only in the local stack (not published to the host) |
| Persistence | `persistence true`, data under `/mosquitto/data/` |

The local docker-compose stack uses a custom image, `ghcr.io/eegfaktura/eegfaktura-mosquitto:latest` (with `mosquitto.conf` baked in, started with `mosquitto -c /mosquitto/config/mosquitto.conf`), rather than an upstream `eclipse-mosquitto` image. Port `1883` is reachable only inside the compose network; it is not published to the host. The dev-zone manifest in `eegfaktura-platform` runs the upstream `eclipse-mosquitto:2.0.22` image with the config from a ConfigMap.

## Topic catalogue

!!! warning "The legacy topic for energystore is outdated"
    The legacy README listed `eegfaktura/<tenant>/energy/<meterId>` as the energy topic. The actual production v1 stack uses `eda/response/<tenant>/protocol/cr_msg`. The mismatch was discovered 2026-06-07 during the v2 cutover analysis. See [eda-xp.md#mqtt-output](eda-xp.md#mqtt-output).

| Topic (prod) | Publisher | Subscriber | Payload |
|---|---|---|---|
| `eda/response/<tenant>/protocol/<process_lower>` | eda-xp | backend (`eda/response/+/protocol/#`) | EDA message as JSON (circe) |
| `eda/response/<tenant>/protocol/cr_msg` | eda-xp | energystore (`eda/response/+/protocol/cr_msg`); the backend receives it via its wildcard but skips it | gzip + Base64 around the EbMsMessage JSON (energystore decodes it as `MqttEnergyMessage`), not encrypted ([wire format](eda-xp.md#cr_msg-payload-encoding)) |
| `eda/response/<tenant>/protocol/cr_msg_history` | energystore (after each import) | backend (process history) | plain JSON: meter, conversation id, message code, periods |
| `eda/response/<tenant>/command/<command>` | eda-xp | backend (`eda/response/+/command/#`) | JSON |
| `eda/response/error` | eda-xp (`errorTopic`, request stream errors) | backend (logs it, no further handling) | error messages |
| `eda/request` | backend | eda-xp | outbound EDA requests (JSON) |
| `eegfaktura/<tenant>/energy/<meterId>` | upstream pilot publishers only | [energystore-v2](energystore-v2.md) | plain `MqttEnergyMessage` JSON (upstream pilot convention, not used in eegfaktura deployments) |

`<tenant>` is the community's market-participant id (RC/CC/GC number) as eda-xp puts it into the topic — the receiver of an inbound message, the sender of a request answer — lower-cased; energystore and backend upper-case it again. It is not `base.eeg.tenant`, which can differ (GEA). `<process_lower>` is the EDA process code lowercased (`cm_rev_sp`, `ec_req_onl`, …; `cr_msg_03.03` is normalised to `cr_msg`).

The `cr_msg` topic is the one carrying actual energy values per metering point, hence the gzip + Base64 encoding (it is **not** encrypted). All other topics are cleartext JSON. energystore v1 has a configured `…/protocol/inverter_msg` topic, but nothing publishes to it and v1 does not subscribe to it. See [eda-xp.md#cr_msg-payload-encoding](eda-xp.md#cr_msg-payload-encoding) for the wire format.

Subscriber-side details are documented per service: see [backend](backend.md) and [energystore](energystore.md).

## QoS and sessions

All eegfaktura clients use **QoS 1**: eda-xp publishes with `AtLeastOnce` and subscribes `eda/request` with QoS 1; backend and energystore subscribe with `mqtt.qos: 1`. Retained messages are not used. backend, energystore and eda-xp connect with persistent sessions (`cleanSession false`), so the broker queues QoS-1 messages for them while they are disconnected (with `persistence true` this survives a broker restart).

Consequence: a consumer that is down during a publish receives the message after reconnecting, within the broker's queue limits (see below). A message that energystore cannot decode is acknowledged and dropped; its data has to be requested again via EDA. Re-publishing the same period overwrites the values in energystore.

## Auth

The broker configurations in `eegfaktura-platform` (dev zone, pilot runbook) set `allow_anonymous true`, and none of the clients (backend, energystore, eda-xp) configures MQTT credentials. Access control relies on the broker being reachable only inside the cluster / compose network; there are no per-service users or topic ACLs.

## Config

The main `mosquitto.conf` (dev-zone manifest in `eegfaktura-platform`; the local-stack image bakes in its own):

| Directive | Value | Purpose |
|-----------|-------|---------|
| `listener` | `1883` | bind port |
| `allow_anonymous` | `true` | no client authentication |
| `persistence` | `true` | enable on-disk persistence |
| `persistence_location` | `/mosquitto/data/` | data directory |
| `log_dest` / `log_type` | `stdout` / `error`, `warning`, `notice` | logging |

## Delivery semantics

- The broker does **not** distinguish "no subscriber" from "delivered" — publishing to a topic nobody listens to silently succeeds. This is the dominant failure mode when adding new EDA protocol support; see [Architecture / Messaging](../architecture/messaging.md#gap-points) gap point #6.

### MQTT 5 vs 3.1.1 — Shared Subscriptions caveat

The broker accepts both MQTT 3.1.1 and MQTT 5 clients. **Shared Subscriptions (`$share/<group>/<topic>`) are an MQTT 5 feature.** Mosquitto silently accepts the subscribe pattern from a 3.1.1 client but then never matches any publish to it — you get a connected client that receives zero messages and no error.

[energystore-v2](energystore-v2.md) ran into this with `paho.mqtt.golang v1` (MQTT 3.1.1) and was migrated to `eclipse/paho.golang` (MQTT 5). Anything new that wants Shared Subscriptions must do the same.

Also: MQTT 5 §4.8.2 forbids `NoLocal=true` on a Shared Subscription. Setting it results in disconnect with `reason_code=130 (Protocol Error)`. Subscribers should leave the flag at default `false`.

### Burst-load caveat

The config sets neither `max_inflight_messages` nor `max_queued_messages`, so Mosquitto's defaults apply (20 in-flight, 1000 queued messages per client). During a sustained QoS-1 burst towards a slow or disconnected subscriber, messages beyond the queue limit are dropped silently: the publisher still gets its PUBACK, the subscriber sees only a subset. Raise the limits in `mosquitto.conf` where large EDA bursts are expected.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full topic flow
- [services/backend](backend.md) — main subscriber on EDA topics
- [services/energystore](energystore.md) — subscriber on energy topics
- [services/eda-xp](eda-xp.md) — main publisher
