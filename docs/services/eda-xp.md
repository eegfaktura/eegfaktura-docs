# eda-xp

Scala / Pekko service that bridges PONTON (the external EDA adapter) with the rest of the platform. Accepts inbound EDA XML over two transports — the Ponton X/P (KEP) EDA adapter, which POSTs SOAP envelopes over HTTP, and per-tenant IMAP/SMTP email polling — parses it via scalaxb, enriches it with stored conversation state, and republishes it on MQTT for the backend and energystore to consume. In the other direction it takes EDA requests from the backend over MQTT, builds the XML and sends it via Ponton or email.

## At a glance

| | |
|---|---|
| Language | Scala 2.13.18 |
| Framework | Apache Pekko 1.2.1 (HTTP 1.2.0 + typed actors + persistence, migrated from Akka 2026-06-13), Pekko Connectors MQTT 1.1.0, Pekko gRPC (sbt plugin 1.1.1) |
| Inbound | (1) Ponton X/P (KEP) EDA adapter — SOAP envelope POSTed to `/pontonxp/message` on port 6090; (2) email IMAP polling per tenant mailbox (tenant type `MAIL`) |
| Outbound | EDA XML in a SOAP request to the Ponton X/P endpoint (`app.kepserver.url`) for tenants of type `KEP`, or by SMTP for tenants of type `MAIL`; inbound results published to MQTT |
| Ports | HTTP 6090 (Ponton routes `/pontonxp/*` + `/admin/upload`), gRPC 9093 |
| Auth | none — no JWT/bearer check on the HTTP routes or the gRPC server (reached cluster-internally only) |
| State | PostgreSQL `eda` schema (tenant config, conversations, mail inbox/outbox) via Slick + slick-pg, Flyway migrations; outgoing message counter in a Pekko persistence LevelDB journal under `/storage/prod` |
| XML parsing | scalaxb 1.12.0 (per-namespace package mapping) |
| Main class | `XpAdapter` |

Also known as `eda-xp-connector` (sbt `packageName`) or `xp-adapter` in legacy configurations.

## Responsibilities

- Accept inbound EDA messages from PONTON (`/pontonxp/message`) and from tenant mailboxes
- Parse the XML payload via the matching scalaxb binding (dispatch on the document type/namespace)
- Merge the inbound message with the stored conversation of the original request (`eda.conversation`)
- Publish the enriched `EbMsMessage` to MQTT
- Take requests from the backend on MQTT `eda/request`, assign message/conversation ids, store the conversation and send the request
- Forward inbound messages of a community that moved to another zone (see below)
- gRPC services for the backend/admin: tenant registration (`RegisterPontonService`), member mails (`SendMailService`). `ExcelAdminService` is defined in `excel.proto` but **not** registered in `AdminServer.scala` — energystore's Excel-export mail call to it fails

## Inbound pipeline

```
PONTON EDA adapter (external) ─HTTP POST SOAP envelope─→ PontonRoute /pontonxp/message
                                              │  (app.forward hit → forwarded unchanged, see below)
                                              ▼
                          scalaxb.fromXML[Envelope] → XmlParseHandler.reponseEbMsMessage
                                              │  pattern match on the scalaxb document type
                                              ▼
                       PontonRoute.notificationFor (message code → process code)
                                              │  unknown message type → protocol "error"
                                              ▼
                       MqttPublisher → EbMsAggregator → ConversationEntity.MergeNotification
                                              │  mergeEbmsMessage (stored request + answer),
                                              │  ecId from base.eeg if still missing
                                              ▼
                       MqttSystem → eda/response/<receiver>/protocol/<process>
```

The email path (`mailer/Fetcher`, `TenantMailActor`) and the admin multipart upload (`/admin/upload`, `FileService.handleUpload`) feed the same `XmlParseHandler` mapping and `MqttPublish`.

Answer codes to Ponton: 204 when the message was handled (also for an unknown message type, which is logged as ERROR and published to the receiver's `protocol/error` topic); 500 when a known document cannot be parsed or mapped (e.g. a new MessageCode) — Ponton keeps it and redelivers it. `/pontonxp/status` and `/pontonxp/notification` are only logged.

See [Architecture / Messaging](../architecture/messaging.md) for the end-to-end view including the backend subscriber side, and the "gap points" where a new EDA version can silently drop.

### Forwarding for a moved community

A community that moved to another zone (e.g. the pilot zone) keeps its Ponton agreements on the prod adapter. `app.forward` maps an RC number (the Ponton header `ReceiverId`, not the tenant — all GEA tenants of one RC number move together) to the URL of the other adapter's `/pontonxp/message`. A message for a listed RC number is forwarded unchanged (body, content type, SOAPAction) and not processed or published here (`InboundForwarding`, `PontonRoute`).

- Ponton gets 204 only when the target answered 2xx; an error, an unreachable target or no answer within `app.forward-timeout` gives 500, so Ponton redelivers (at-least-once).
- Forwarded requests carry the header `X-Eegfaktura-Forwarded`; an adapter that would forward such a request again refuses it (500) to avoid a loop.
- With entries, every message (forwarded or not) is read completely and must arrive within `app.forward-read-timeout`. Empty `app.forward` (default) keeps the plain message route.
- An entry that is not an absolute http(s) URL with a host, or a duplicate RC number, stops the start.

### Outgoing message counter

Message, conversation and request ids are built from a per-adapter counter (`PrepareMessageActor`, an event-sourced actor persisted in the LevelDB journal). `app.message-seq-floor` sets a lower bound for that counter, so the adapter of a second zone does not repeat Ponton header ids (`MSG<counter>@<RC>`) that prod already sent for a moved community. Default 0 (unchanged); at most `9000000000` (10 digits), other values stop the start. The start log line `Message counter at …` shows counter, floor and next header id.

## Process and schema versions

EDA distinguishes:

- **Process version** — e.g. `EC_REQ_ONL` 03.00 on ebUtilities.
- **Schema set version** — e.g. `EC_REQ_ONL_02.40`. Since the EDA change of 2026-10-05 the two differ.

For outbound requests the backend stamps the version label from its config `eda-process-versions.<MESSAGE_CODE>` onto `messageCodeVersion` (`mqtt/messageBroker.go`). eda-xp selects the builder by string match in the message class's `getVersion()` and sends the label as Ponton header `MessageVersion`. The label is the **schema set version**, not the process version. A missing label, or a label without its own case, is refused with an ERROR log and an error to the sender (no silent fallback since 1.0.8).

The Ponton header `MessageType` (`MessageHelper.pontonHeaderMessageType`) carries the message type from the schema set definition for ECP, ECON, ECOF and CPF (e.g. `ANFORDERUNG_ECP`); PT and CCMS still send the process code.

scalaxb-generated code is namespaced per schema version (`scalaxbPackageNames` in `build.sbt`, e.g. `cmrequest.v01p30`) to keep the versions isolated.

### Supported ebUtilities message schema versions

Derived from `build.sbt` (scalaxb per-namespace bindings) and `XmlParseHandler.scala`:

| Message type | Supported versions |
|--------------|--------------------|
| CMNotification | 01p11, 01p12, 01p20 |
| CMRevoke | 01p00, 01p10 |
| CMRequest (outbound) | 01p10, 01p20, 01p21, 01p30 |
| CPNotification | 01p13 |
| CPRequest / CPDocument | 01p12 |
| ECMPList | 01p00, 01p10, 01p20 |
| ConsumptionRecord (CR_MSG) | 01p30, 01p31, 01p40, 01p41 |
| GCRequest / GCRequestAP | 01p00 |
| CommonTypes | 01p20 |
| Ponton KEP envelope | v320 |

Inbound documents dispatched in `XmlParseHandler.mapDataRecordToEbms`: ECMPList 01p00/01p10/01p20, CMNotification 01p11/01p12/01p20, CMRevoke 01p00/01p10, CPNotification 01p13, ConsumptionRecord 01p30/01p31/01p40/01p41. Anything else becomes an `ERROR_MESSAGE`.

!!! note
    There is **no** ConsumptionRecord 01p20 in the code — do not assume it is supported.

## Outbound

When the backend triggers an EDA flow (Zählpunkt activation, energy-data request, participation-factor change, …) it publishes the request on `eda/request`; eda-xp builds the XML (`MessageHelper.getEdaMessageByTypeTry`) and sends it to PONTON or by mail. Routed requests:

| Message code | Process | Version labels accepted | Purpose |
|--------------|---------|-------------------------|---------|
| `ANFORDERUNG_ECON` | `EC_REQ_ONL` | 02.00, 02.10, 02.30, 02.40 | Zählpunkt activation (online registration) |
| `ANFORDERUNG_ECOF` | `EC_REQ_OFF` | 02.00, 02.10, 02.20, 02.30 | Offline registration |
| `ANFORDERUNG_ECP` | `EC_PODLIST` | 02.00, 02.10 | Request the community's metering-point list |
| `ANFORDERUNG_PT` | `CR_REQ_PT` | 03.00 | Energy-data request |
| `ANFORDERUNG_GN` | `MD_REQ_GN` | 03.12 | Master data request |
| `AUFHEBUNG_CCMS` | `CM_REV_SP` | 01.02, 01.10, 01.30 | Revocation by the service provider |
| `ANFORDERUNG_CPF` | `EC_PRTFACT_CHANGE` | 01.00 (ECMPList 01p10), 01.10 (ECMPList 01p20) | Change participation factor |

The backend's default labels are in its `config.yaml` (`eda-process-versions`).

`EC_PRTFACT_CHANGE` is only accepted by the grid operator between 09:00 and 17:00 Austrian time and the change takes effect the following day. The XML builder sets `ProcessDate = today + 1` and `DateFrom = today + 1` (Europe/Vienna); `DateActivate` is the metering point's activation date. Triggering outside that window returns an `ABLEHNUNG_CPF` with response code 82 (`Prozessdatum falsch`).

## XXE hardening

`scala.xml.XML.loadString` on attacker-controlled XML without XXE defenses is unsafe: scala-xml versions < 2.2 do not have secure-by-default parsing. eda-xp uses **scala-xml 2.3.0**, which is secure by default (DOCTYPE declarations disallowed), so inbound XML parsing is XXE-hardened.

This protection applies anywhere in eda-xp that consumes inbound XML; keep scala-xml at ≥ 2.2 to retain it.

## Database

Owns state in the `eda` schema (Flyway migrations under `src/main/resources/db/migration`):

| Table | Holds |
|-------|-------|
| `eda.tenantconfig` | per tenant: transport type (`KEP` or `MAIL`), mail domain/hosts/ports/credentials, active flag |
| `eda.conversation` | the sent request as JSON by conversation id (stored before the send) |
| `eda.inbox` / `eda.outbox` | mail content per tenant |

The `ConversationEntity` actor reads the stored request on merge (`mergeEbmsMessage`): depending on the message code it copies e.g. `ecId`, the metering point, the meter list or the consent end into the answer. If the conversation id does not match a stored request (e.g. a partner-initiated message), the message is published unenriched — the backend handler then deals with that case. For answers that need an `ecId` and still lack one, `EbMsAggregator` looks it up in `base.eeg` by tenant.

!!! note "Shared table read"
    `base.eeg` belongs to the backend's schema; this read is a direct cross-component SQL access, not an API call.

## MQTT output

All topics are built in `MqttPaths` / `MqttSystem` with the base `eda/response`, QoS 1, not retained. eda-xp publishes:

### EDA workflow events (cleartext JSON)

```
eda/response/<participant>/protocol/<process_lower>
```

`<participant>` is the message's `receiver` (for inbound messages: the community's market-participant id, e.g. its RC number), lowercased; an empty receiver gives `error`. `<process_lower>` is the process code lowercased: `cm_rev_sp`, `ec_req_onl`, `cm_rev_imp`, `cr_req_pt`, etc.; inbound messages without a process mapping go to `protocol/error`.

Payload is `EbMsMessage` serialized as JSON via circe (null fields dropped) — cleartext, no encryption.

Further topics:

| Topic | When |
|-------|------|
| `eda/response/<sender>/protocol/<process_lower>` | answer to a request taken from `eda/request` (the prepared request with its ids) |
| `eda/response/<tenant>/protocol/error` | a request could not be built or sent |
| `eda/response/error` | request payload could not be parsed, or another stream error |
| `eda/response/<tenant>/command/pontononlinestate` | Ponton online state after a tenant registration/update over gRPC |

The keys `epmsmail.mqtt.topics.energyTopic` / `cmTopic` / `cpTopic` are read from config but not used for publishing; only `errorTopic` (`eda/response/error`) is used.

### CR_MSG (energy data — gzip + Base64)

```
eda/response/<participant>/protocol/cr_msg
```

`cr_msg` (network-operator data response — contains per-meter, per-slot energy values) is the **only** topic that receives special encoding: the JSON is gzip-compressed then Base64-encoded (see below). It is **not** encrypted.

Subscribers: [energystore (v1)](energystore.md) and [energystore-v2](energystore-v2.md). The backend subscribes to `eda/response/+/protocol/#` but skips `cr_msg`.

### CR_MSG payload encoding

`cr_msg` payloads are gzip-compressed and Base64-encoded before publishing (`MqttSystem.encodeProtocolPayload`, pinned by `MqttPayloadCompatibilitySpec`). This is the canonical description of the wire format — other pages link here instead of repeating it.

```
EbMsMessage as JSON (UTF-8)
  ↓
gzip                          ← compression (java.util.zip.GZIPOutputStream)
  ↓
Base64 (standard alphabet)    ← MQTT-safe transport (java.util.Base64.getEncoder)
  ↓
MQTT publish
```

Wire-format parameters:

| | |
|---|---|
| Compression | gzip |
| Transport encoding | Base64 (standard alphabet, not URL-safe) |
| Encryption | **none** |
| Scope | the `cr_msg` topic only; every other topic is published as cleartext JSON (`case _ => value`) |

Subscribers reverse the pipeline: Base64-decode → gunzip → JSON parse.

!!! warning "No encryption"
    There is **no encryption** anywhere in eda-xp — only the gzip + Base64 path for `CR_MSG` is implemented; there are zero references to AES, `Cipher`, or any encryption primitive in the source. Earlier revisions of this page described an AES-256-CBC envelope with a pre-shared key/IV; that was never implemented. Treat the wire format as gzip + Base64 encoding only, not a confidentiality control.

#### Architecture status

Encryption of cluster-internal MQTT traffic is not implemented and not a Defense-in-Depth measure. The long-term design of this layer (implement, keep as-is, or replace) is tracked in an internal architecture decision record in the platform repository.

eda-xp publishes `cr_msg` as plain `base64(gzip(json))`. The [energystore-v2](energystore-v2.md) decoder has no mode for that format (it expects base64 → AES → gunzip and a wrapped single `energy` object), so v2 cannot consume today's `cr_msg` as is.

## Config

Configuration is layered: `reference.conf` (defaults) ← `application.conf` (mounted at `/conf`, passed as `-Dconfig.file`) ← environment overrides.

| Key / variable | Purpose |
|----------------|---------|
| `app.interface.mode` | `SIMU` or `PROD` (case ignored); any other value is an error |
| `app.kepserver.url` | outbound Ponton X/P endpoint |
| `app.server.host` / `app.server.port` | HTTP server (port 6090) |
| `app.grpc.host` / `app.grpc.port` | gRPC server (port 9093) |
| `app.forward` | RC number → URL of the target adapter's `/pontonxp/message` (empty by default) |
| `app.forward-read-timeout` / `app.forward-timeout` | deadlines while `app.forward` has entries (8 s / 10 s, with unit) |
| `app.max-body-size` | largest request body on `/pontonxp/*` and `/admin/upload` (default `128m`); larger requests get 413 |
| `app.message-seq-floor` | lower bound of the outgoing message counter (default 0) |
| `epmsmail.mail.*` | mail inbox dir and per-domain IMAP/SMTP settings + poll interval |
| `epmsmail.mqtt.host/port/sub-topic/qos/consumer-id` | MQTT broker connection; `sub-topic` = `eda/request` |
| `epmsmail.mqtt.topics.errorTopic` | error topic (`eda/response/error`); `energyTopic`/`cmTopic`/`cpTopic` unused |
| `epmsmail.admin.*` | admin/member mail SMTP and sender address |
| `slick.pgsql.local.db.*` | PostgreSQL connection (for `eda` schema) |
| `flyway.*` | DB migration settings |

Environment overrides: `SMTP_ADMIN_SERVER_HOST` / `SMTP_ADMIN_SERVER_PORT`, `EMAIL_ADMIN_USER`, `EMAIL_ADMIN_PWD`, `EMAIL_ADMIN_FROM`, `FORWARD_READ_TIMEOUT`, `FORWARD_TIMEOUT`, `MAX_BODY_SIZE`, `MESSAGE_SEQ_FLOOR`, `TZ`, `JAVA_OPTS`.

Two volumes are declared: `/conf` (holds `application.conf`) and `/storage/prod` (Pekko persistence journal and snapshots).

## Build and image

- Build: sbt 1.10.11 with `sbt-native-packager` (no Dockerfile); `sbt compile` runs scalaxb and Pekko gRPC codegen
- Docker image name `eegfaktura-kep`; base `eclipse-temurin:17-jre`; binary `xpadapter`; main class `XpAdapter`
- Runtime defaults: `JAVA_OPTS=-Xmx4g`, `TZ=Europe/Vienna`
- Key libraries: scalaxb 1.12.0 (per-namespace XML binding), Courier 3.0.1 (mail), Slick 3.5.1 + slick-pg 0.22.2, Flyway 10.20.0, Circe 0.14.3, scala-xml 2.3.0 (XXE-safe by default), PostgreSQL JDBC 42.7.3, Logback
- Final image is substantially larger than the Go services (JRE + scalaxb-generated classes for every supported EDA protocol version)

## When PONTON is not deployed

In environments without a real PONTON link (dev / mock instances), an [eda-mock](eda-mock.md) placeholder stands in for the PONTON side: a `traefik/whoami` HTTP echo rather than a functional EDA stub — see that page for details.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full inbound pipeline + gap points
- [services/eda-mock](eda-mock.md) — stub for dev
- [services/backend](backend.md) — MQTT consumer side
