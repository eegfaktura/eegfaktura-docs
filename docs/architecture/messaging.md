# Messaging

Mosquitto MQTT is the message bus between the EDA gateway (`eda-xp`), the energystore, and the backend. There is no Kafka, no RabbitMQ — just Mosquitto with topic-based routing.

## Broker

| Item | Value |
|------|-------|
| Service | `mosquitto` (Eclipse Mosquitto) |
| Default port | 1883 (in-cluster), no TLS by default |
| Persistence | `persistence true` (QoS-1 queues of persistent sessions survive a broker restart) |
| Auth | none: `allow_anonymous true`, no client credentials, no topic ACLs (broker reachable only inside the cluster / compose network) |

See [services/mosquitto](../services/mosquitto.md) for configuration and delivery semantics.

## Topic catalogue

| Topic pattern | Publisher | Subscriber | Payload |
|---|---|---|---|
| `eda/request` | backend | eda-xp | EbMsMessage request (cleartext JSON) |
| `eda/response/<participant>/protocol/<process_lower>` | eda-xp | backend (`eda/response/+/protocol/#`) | EbMsMessage (cleartext JSON via circe) — inbound EDA messages and answers to requests |
| `eda/response/<participant>/protocol/cr_msg` | eda-xp | [energystore (v1)](../services/energystore.md); [energystore-v2](../services/energystore-v2.md) (not rolled out) cannot consume it yet | gzip + Base64 around EbMsMessage JSON (no encryption) ([wire format](../services/eda-xp.md#cr_msg-payload-encoding)); the backend skips it |
| `eda/response/<tenant>/protocol/cr_msg_history` | energystore (v1) | backend | cleartext JSON: metering point, `ecId`, conversation, message code and the stored periods (`start`/`end`) |
| `eda/response/<participant>/protocol/error` | eda-xp | backend (logs it, no further handling) | EbMsMessage with `errorMessage` (request not built/sent, or inbound message type without process) |
| `eda/response/error` | eda-xp | backend (logs it, no further handling) | error for a request payload that could not be parsed |
| `eda/response/<tenant>/command/<command>` | eda-xp | backend (`eda/response/+/command/#`) | command JSON, e.g. `pontononlinestate` after a tenant registration |
| `eegfaktura/<tenant>/energy/<meterId>` | upstream pilot publishers only, not used in eegfaktura deployments | energystore-v2 (upstream pilot) | cleartext `MqttEnergyMessage` (upstream pilot convention) |

The participant slot is the message's market-participant id — for inbound messages the `receiver` (the community's RC/CC/GC number), for request answers the `sender` — lowercased by eda-xp; the backend and energystore uppercase it again. It is not the `ecId` (community id), which travels in the payload. `<process_lower>` is the EDA process code lowercased: `cm_rev_sp`, `ec_req_onl`, `cm_rev_imp`, etc.

!!! note "Legacy energy topic pattern"
    The energy topic pattern is `eda/response/+/protocol/cr_msg`. The energystore repo's `config-prod.yaml` still carries the legacy form `eda/response/energy/+`; eda-xp never publishes there (its `energyTopic` config key is unused) — check the deployed config when reconciling subscriptions.

!!! warning "CR_MSG is the only compressed topic"
    `cr_msg` is the topic carrying actual energy values per metering point — and the only topic with a non-cleartext wire format. The payload is **gzip-compressed then Base64-encoded; it is not encrypted** (eda-xp contains no encryption code). All other topics are cleartext JSON. Subscribers must Base64-decode then gunzip before the JSON parse — see [services/eda-xp#cr_msg-payload-encoding](../services/eda-xp.md#cr_msg-payload-encoding) for the wire format. The pilot convention `eegfaktura/+/energy/+` is cleartext.

## EDA inbound pipeline

eda-xp bridges EDA market communication to MQTT. It has **two inbound transports**: the Ponton X/P (KEP) EDA adapter POSTing SOAP envelopes over HTTP (port 6090, `/pontonxp/message`), and email (IMAP polling per tenant mailbox). Both feed the same mapping below; results are published under `eda/response/…`, where the backend and energystore subscribe.

The end-to-end path of an inbound EDA message (shown for the Ponton transport; email uses the same `XmlParseHandler` mapping and publish step):

```
PONTON EDA adapter (external, HTTP on port 6090)
   │ HTTP POST SOAP envelope → /pontonxp/message
   ▼
eda-xp: PontonRoute
   │ ReceiverId listed in app.forward → forwarded unchanged to the other zone, stop here
   ▼
eda-xp: XmlParseHandler.reponseEbMsMessage
   │ scalaxb.fromXML[Envelope], then pattern match on the document type
   │ (per-version scalaxb binding); unknown type → ERROR_MESSAGE
   ▼
PontonRoute.notificationFor
   │ MessageHelper.EDAMessageCodeToProcessCode (message code → process);
   │ no mapping → protocol "error"
   ▼
MqttPublisher → EbMsAggregator → ConversationEntity (actor)
   │ loads the stored request by conversation id (eda.conversation)
   │ mergeEbmsMessage (pattern match on messageCode) copies ecId, meter, …
   ▼
MqttSystem → topic eda/response/<receiver>/protocol/<process>
   ▼
backend: MessageBroker route eda/response/+/protocol/#
   │ callback per protocol (eda/edaSubscription.go)
   │ switch on messageCode
   ▼
DB write (metering-point status, process history, ...)
```

Outbound requests take the reverse way: the backend publishes on `eda/request` (with the version label from `eda-process-versions`), eda-xp assigns ids, stores the conversation, sends the XML via Ponton or mail, and answers on `eda/response/<sender>/protocol/<process>`.

## EDA-process codes used

The backend registers callbacks for these protocols (`eda/edaSubscription.go`, `mqtt/subscriptionHandler.go`). A message on any other protocol topic finds no callback and is dropped in the backend without a log.

| Process | Direction | Purpose |
|---------|-----------|---------|
| `EC_REQ_ONL` | bidirectional | "Anmeldung Teilnahme Online" — Zählpunkt activation (`ANFORDERUNG_ECON` → `ANTWORT_ECON`, `ZUSTIMMUNG_ECON`, `ABSCHLUSS_ECON`, …) |
| `EC_REQ_OFF` | bidirectional | Offline registration (`ANFORDERUNG_ECOF` → `…_ECOF`) |
| `EC_PRTFACT_CHANGE` | bidirectional | Participation-factor change. Outbound `ANFORDERUNG_CPF`, inbound `ANTWORT_CPF` (accepted) or `ABLEHNUNG_CPF` (rejected). Grid operator only accepts requests between 09:00 and 17:00 Austrian time, effective the next day. |
| `EC_PODLIST` | bidirectional | Metering-point list: outbound `ANFORDERUNG_ECP`, inbound `SENDEN_ECP` / `ABLEHNUNG_ECP` |
| `CR_REQ_PT` | bidirectional | Energy-data request: outbound `ANFORDERUNG_PT`, inbound `ANTWORT_PT` / `ABLEHNUNG_PT` |
| `CM_REV_SP` | bidirectional | Revocation by the service provider (`AUFHEBUNG_CCMS` → `ANTWORT_CCMS` / `ABLEHNUNG_CCMS`) |
| `CM_REV_IMP` | inbound | Revocation (`AUFHEBUNG_CCMI`) |
| `CM_REV_CUS` | inbound | Revocation by customer (`AUFHEBUNG_CCMC`) |
| `CR_MSG` | inbound | Energy data (`DATEN_CRMSG`) — consumed by energystore; the backend handles the derived `cr_msg_history` from energystore |
| `ERROR` | inbound | Errors from eda-xp; logged by the backend |

eda-xp also sends `ANFORDERUNG_GN` (process `MD_REQ_GN`); the backend registers no callback for `md_req_gn`, so its answers are dropped there.

Message codes inside an EDA message (e.g. `AUFHEBUNG_CCMS`, `ANTWORT_ECON`, `ABSCHLUSS_ECON`) determine the concrete state change. They are version-independent and matched in the backend handler with a `switch` on the message code.

## Schema versions

EDA distinguishes two kinds of "version":

- **Process version** — e.g. `EC_REQ_ONL` 03.00 on ebUtilities.
- **Schema set version** — e.g. `EC_REQ_ONL_02.40`; since the EDA change of 2026-10-05 it differs from the process version.

For outbound requests the backend config `eda-process-versions.<MESSAGE_CODE>` selects the label (the schema set version); the backend stamps it onto `messageCodeVersion`, eda-xp picks the builder in `getVersion()` and refuses a label without its own case. Inbound documents are dispatched by their XML namespace (scalaxb type), not by a version label. scalaxb-generated code is namespaced per (schema, version) to keep them isolated.

The ebUtilities schema versions with scalaxb bindings in eda-xp are:

| Schema | Versions |
|--------|----------|
| CMNotification | 01p11, 01p12, 01p20 |
| CMRevoke | 01p00, 01p10 |
| CMRequest | 01p10, 01p20, 01p21, 01p30 |
| CPNotification | 01p13 |
| CPRequest / CPDocument | 01p12 |
| ECMPList | 01p00, 01p10, 01p20 |
| ConsumptionRecord | 01p30, 01p31, 01p40, 01p41 |
| GCRequest / GCRequestAP | 01p00 |
| Ponton KEP envelope | v320 |

(There is no ConsumptionRecord 01p20 in the code.)

## Gap points

Places where a new EDA version can silently drop:

1. **scalaxb binding** — the new namespace is not mapped in `build.sbt` (`scalaxbPackageNames`) → no generated type.
2. **Inbound dispatcher** — no case in `XmlParseHandler.mapDataRecordToEbms` → the message becomes an `ERROR_MESSAGE` and goes to `protocol/error`; a document that does not parse answers 500 to Ponton.
3. **Outbound builder** — no `getVersion()` case for the label → the request is refused (ERROR log, error to the sender).
4. **Process mapping** — message code missing in `MessageHelper.EDAMessageCodeToProcessCode` → `protocol/error`.
5. **Conversation merge** — conversation id does not match a stored request (e.g. partner-initiated message), or no case in `ConversationEntity.mergeEbmsMessage` → published without enrichment.
6. **backend callback list** — protocol not registered → the backend reads the publish but drops it.
7. **backend handler `switch`** — specific message code not handled → returns without DB effect.

When adding support for a new EDA version, all points must be verified end-to-end. The pattern is documented in the platform repo as the "XSD backfill chain check".

## Energy data pipeline

The `cr_msg` payload is the EbMsMessage JSON of a `DATEN_CRMSG`; energystore decodes it into `MqttEnergyMessage` (`model/mqtt.go`), which carries consumption / production values per metering point and slot:

```json
{
  "conversationId": "...",
  "messageCode": "DATEN_CRMSG",
  "ecId": "AT00300000000RC100...",
  "meter": {"meteringPoint": "AT003000...", "direction": "CONSUMPTION"},
  "energy": [
    {
      "start": 1730415600000,
      "end": 1730502000000,
      "data": [
        {"meterCode": "1-1:1.9.0 G.01", "value": [{"from": 1730415600000, "to": 1730416500000, "method": "L1", "value": 0.12}, ...]},
        {"meterCode": "1-1:2.9.0 G.02", "value": [...]},
        {"meterCode": "1-1:2.9.0 G.03", "value": [...]}
      ]
    }
  ]
}
```

Timestamps are epoch milliseconds (circe encodes `java.util.Date` as `getTime`). v1 subscribes to the configured `mqtt.energySubscriptionTopic` (`eda/response/+/protocol/cr_msg`), Base64-decodes then gunzips the payload (`mqttclient/energyExecutor.go`), decodes the `meterCode` into a storage slot, writes the values to Badger, and publishes the stored periods on `eda/response/<tenant>/protocol/cr_msg_history` for the backend. v2 (not rolled out) writes to TimescaleDB instead, but cannot consume this topic yet: its decoder expects a different envelope and its tenant extraction does not fit the topic — see [services/energystore-v2](../services/energystore-v2.md). The pilot convention `eegfaktura/+/energy/+` is cleartext. See [reference/obis-codes](../reference/obis-codes.md) for the full code-to-slot mapping including T-/R-variants.

Direction detection: a new metering point is registered as a **producer** if its first payload contains a generation code — `1-1:2.9.0 G.01` / `G.01T` or `1-1:2.9.0 P.01` / `P.01T` (`utils.ExamineDirection`); the two `G.01` codes differ by their OBIS prefix (`2.9.0` production, `1.9.0` consumption). Otherwise it is registered as a consumer.

## MQTT QoS and retention

eda-xp publishes with QoS 1 (at least once) and subscribes to `eda/request` with QoS 1; the backend (`mqtt.qos`) and energystore v1 are configured with QoS 1 as well. Retained messages are not used. backend, energystore and eda-xp connect with persistent sessions (`cleanSession false`), so the broker queues QoS-1 messages for a disconnected consumer within its queue limits; messages beyond the limit are dropped (see [services/mosquitto](../services/mosquitto.md#qos-and-sessions)). eda-xp has no re-send of its own; lost energy data has to be requested again via EDA.

## MQTT protocol version

For non-shared subscriptions either MQTT 3.1.1 or MQTT 5 works. For **Shared Subscriptions** (`$share/<group>/<topic>`, used by [energystore-v2](../services/energystore-v2.md) to spread the load across replicas), MQTT 5 is mandatory — the broker silently accepts the subscribe pattern from a 3.1.1 client but never matches any publish to it. New Go subscribers should use `github.com/eclipse/paho.golang`, not `github.com/eclipse/paho.mqtt.golang`. See [services/mosquitto](../services/mosquitto.md#mqtt-5-vs-311-shared-subscriptions-caveat).

## Related

- [services/eda-xp](../services/eda-xp.md) — gateway implementation
- [services/eda-mock](../services/eda-mock.md) — Dev-zone HTTP echo (no MQTT)
- [services/mosquitto](../services/mosquitto.md) — broker config
- [services/energystore](../services/energystore.md) — energy-data consumer
- [reference/obis-codes](../reference/obis-codes.md) — OBIS code semantics
