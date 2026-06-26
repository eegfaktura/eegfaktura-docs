# Messaging

Mosquitto MQTT is the message bus between the EDA gateway (`eda-xp`), the energystore, and the backend. There is no Kafka, no RabbitMQ — just Mosquitto with topic-based routing.

## Broker

| Item | Value |
|------|-------|
| Service | `mosquitto` (Eclipse Mosquitto) |
| Default port | 1883 (in-cluster), no TLS by default |
| Persistence | PVC-backed, on by default |
| Auth | username/password by default; TLS / mTLS optional |

See [services/mosquitto](../services/mosquitto.md) for sizing and operational notes.

## Topic catalogue

| Topic pattern | Publisher | Subscriber | Payload |
|---|---|---|---|
| `eda/<tenant>/protocol/<process_lower>` | eda-xp | backend | EbMsMessage (cleartext JSON via circe) |
| `eda/response/<tenant>/protocol/cr_msg` | eda-xp | [energystore (v1)](../services/energystore.md), [energystore-v2](../services/energystore-v2.md) | gzip + Base64 around `MqttEnergyMessage` JSON (no encryption) ([wire format](../services/eda-xp.md#cr_msg-payload-encryption)) |
| `eda/response/<tenant>/protocol/inverter_msg` | eda-xp | energystore / energystore-v2 | cleartext `MqttEnergyMessage` (PV inverter telemetry) |
| `eda/response/<tenant>/protocol/cr_req_pt` | eda-xp | backend | cleartext EbMsMessage (period-request workflow event) |
| `eegfaktura/<tenant>/energy/<meterId>` | pilot energy publishers only | energystore-v2 in pilot | cleartext `MqttEnergyMessage` (pilot convention) |

The tenant slot is the EEG's `community_id` / `ecId`. `<process_lower>` is the EDA process code lowercased: `cm_rev_sp`, `ec_req_onl`, `cm_rev_imp`, etc.

!!! note "Legacy energy topic pattern in prod"
    The energy topic pattern is `eda/response/+/protocol/cr_msg`. Some production configs may still reference the legacy form `eda/response/energy/+` — check the deployed config when reconciling subscriptions.

!!! warning "CR_MSG is the only compressed topic"
    `cr_msg` is the topic carrying actual energy values per metering point — and the only topic with a non-cleartext wire format. The payload is **gzip-compressed then Base64-encoded; it is not encrypted** (the eda-xp source carries a `// Todo: Encrypt` comment — encryption is a documented TODO, not implemented). All other topics are cleartext JSON. Subscribers must Base64-decode then gunzip before the JSON parse — see [services/eda-xp#cr_msg-payload-encryption](../services/eda-xp.md#cr_msg-payload-encryption) for the wire format. The pilot convention `eegfaktura/+/energy/+` is cleartext.

## EDA inbound pipeline

eda-xp bridges EDA market communication to MQTT. It has **two inbound transports**: the Ponton X/P (KEP) adapter delivering AS4 over HTTP (port 6090), and email (IMAP/SMTP). Both feed the same handler chain below; responses are published to MQTT (`energy` / `cm` / `cp` / `error` topic prefixes), where the backend and energystore subscribe.

The end-to-end path of an inbound EDA message (shown for the Ponton transport; email follows the same chain from the upload step onward):

```
PONTON adapter (external, AS4 over HTTP on port 6090)
   │ HTTP POST multipart/form-data
   ▼
eda-xp: FileService.handleUpload
   │ parseProcessName → (protocol, version)
   ▼
eda-xp: MessageHelper.getEdaMessageFromHeader(protocol, version)
   │ selects scalaxb handler by (protocol, schema-version)
   ▼
Handler.fromXML(xmlNode): Try[EdaMessage]
   │ scalaxb.fromXML with per-version schema
   ▼
MessageStorage.MergeMessage (actor)
   │ loads prior conversation state from DB
   │ enriches new message with stored context
   ▼
mergeEbmsMessage (pattern match on messageCode)
   │ wraps the message in the appropriate domain type
   ▼
MqttPublisher → topic eda/<tenant>/protocol/<process>
   ▼
backend: EDA subscription handler
   │ hard-coded subscriptions per protocol
   │ pattern-matches messageCode
   ▼
DB write (metering-point status, process history, ...)
```

## EDA-process codes used

The backend hard-codes subscriptions for these protocols. Anything not in this list is silently dropped on the broker side.

| Process | Direction | Purpose |
|---------|-----------|---------|
| `EC_REQ_ONL` | outbound | "Anmeldung Teilnahme Online" — Zählpunkt activation |
| `EC_REQ_OFF` | outbound | Zählpunkt deactivation |
| `EC_REQ_ENE` | outbound | Energy-data request |
| `EC_PRTFACT_CHANGE` | bidirectional | Participation-factor change. Outbound `ANFORDERUNG_CPF`, inbound `ANTWORT_CPF` (accepted) or `ABLEHNUNG_CPF` (rejected). Grid operator only accepts requests Mon-Fri 09:00–17:00, effective the next day. |
| `EC_REQ_LST` | outbound | Request participant list |
| `EC_PODLIST` | inbound | Per-grid-operator metering-point list response |
| `CM_REV_SP` | inbound | Revocation by service provider |
| `CM_REV_IMP` | inbound | Revocation, import direction |
| `CM_REV_CUS` | inbound | Revocation by customer |
| `CR_MSG` | bidirectional | Energy data (per-slot values) and generic notifications |
| `CR_REQ_PT` | inbound | Participant-list response |

Message codes inside an EDA message (e.g. `AUFHEBUNG_CCMS`, `ANTWORT_ECON`, `ONLINE_REG_COMPLETION`) determine the concrete state change. They are version-independent and matched in the backend handler with a `switch` on the message code.

## Schema versions

EDA distinguishes two kinds of "version":

- **Process version** (`messageCodeVersion`) — e.g. `02.30`, `01.30`. Drives handler selection in `MessageHelper`.
- **Schema version** — XML attribute on the document. Independent of the process version.

A given process can have multiple supported versions in parallel. scalaxb-generated code is namespaced per (protocol, version) to keep them isolated.

The ebUtilities schema versions currently dispatched in eda-xp are:

| Schema | Versions |
|--------|----------|
| CMNotification | 01p11, 01p12, 01p20 |
| CMRevoke | 01p00, 01p10 |
| CMRequest | 01p10, 01p20, 01p21, 01p30 |
| CPNotification | 01p13 |
| ECMPList | 01p00, 01p10 |
| ConsumptionRecord | 01p30, 01p40, 01p41 |
| GCRequest | 01p00 |
| Ponton KEP envelope | v320 |

(There is no ConsumptionRecord 01p20 or 01p31 wired into the dispatcher, even where an XSD file happens to be present.)

## Gap points

Six places where a new EDA version can silently drop:

1. **MessageHelper dispatcher** — no case branch for the new `messageCodeVersion` → default branch → wrong handler.
2. **Handler `fromXML`** — scalaxb cannot parse an unknown XSD namespace → upstream catches the exception but the content is lost.
3. **MessageStorage merge** — if the conversation-ID key does not match a prior outbound (e.g. partner-initiated message), no prior state is available.
4. **`mergeEbmsMessage`** — no case branch for the message code → default → message goes to MQTT without enrichment.
5. **backend subscription list** — protocol not in the hard-coded subscriptions → broker receives the publish, no consumer reads it.
6. **backend handler `switch`** — specific message code not handled → returns without DB effect.

When adding support for a new EDA version, all six points must be verified end-to-end. The pattern is documented in the platform repo as the "XSD backfill chain check".

## Energy data pipeline

Energy-data MQTT messages (`MqttEnergyMessage`) carry consumption / production values per metering point and quarter-hour slot:

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

v1 prod subscribes to `eda/response/<tenant>/protocol/cr_msg` (and `inverter_msg`), Base64-decodes then gunzips the payload (`energyExecutor.go`), decodes the `meterCode` into a storage slot, and writes the values to Badger. v2 follows the same wire-format but writes to TimescaleDB instead — see [services/energystore-v2](../services/energystore-v2.md). The pilot convention `eegfaktura/+/energy/+` is cleartext. See [reference/obis-codes](../reference/obis-codes.md) for the full code-to-slot mapping including T-/R-variants.

Direction detection: a metering point is registered as a **producer** only if its payload contains a `P.01` or `P.01T` code. Receiving only `G.01` is ambiguous (consumers and producers both have a `G.01`) and the meter is then registered as a consumer.

## MQTT QoS and retention

The bus uses QoS 0 by default. Retained messages are not used. A consumer that is down during a publish loses that message — recovery relies on the producer re-sending (eda-xp) or on the original network operator re-issuing the EDA process.

Production deployments typically increase persistence + QoS for the EDA inbound topic.

## MQTT protocol version

For non-shared subscriptions either MQTT 3.1.1 or MQTT 5 works. For **Shared Subscriptions** (`$share/<group>/<topic>`, used by [energystore-v2](../services/energystore-v2.md) to spread the load across replicas), MQTT 5 is mandatory — the broker silently accepts the subscribe pattern from a 3.1.1 client but never matches any publish to it. New Go subscribers should use `github.com/eclipse/paho.golang`, not `github.com/eclipse/paho.mqtt.golang`. See [services/mosquitto](../services/mosquitto.md#operational-notes).

## Related

- [services/eda-xp](../services/eda-xp.md) — gateway implementation
- [services/eda-mock](../services/eda-mock.md) — local stub
- [services/mosquitto](../services/mosquitto.md) — broker config
- [services/energystore](../services/energystore.md) — energy-data consumer
- [reference/obis-codes](../reference/obis-codes.md) — OBIS code semantics
