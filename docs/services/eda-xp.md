# eda-xp

Scala / Pekko service that bridges PONTON (the external EDA adapter) with the rest of the platform. Accepts inbound EDA XML over two transports — Ponton X/P (KEP) AS4 over HTTP and per-domain IMAP/SMTP email polling — parses it via scalaxb, enriches with stored conversation state, and republishes on MQTT for the backend to consume.

## At a glance

| | |
|---|---|
| Language | Scala 2.13.18 |
| Framework | Apache Pekko 1.2.1 (HTTP 1.2.0 + actors, migrated from Akka 2026-06-13), Pekko Connectors MQTT, Pekko gRPC |
| Inbound | (1) Ponton X/P (KEP) AS4 over HTTP — multipart upload to port 6090; (2) email IMAP/SMTP polling per-domain mailboxes |
| Outbound | EDA XML POSTed to the Ponton X/P endpoint (`app.kepserver.url`); responses published to MQTT |
| Ports | HTTP 6090 (Ponton multipart upload + admin), gRPC 9093 |
| State | PostgreSQL `eda` schema (conversation state) via Slick + slick-pg, Flyway migrations |
| XML parsing | scalaxb 1.12.0 (full per-namespace generation) |
| Main class | `XpAdapter` |

Also known as `eda-xp-connector` or `xp-adapter` in legacy configurations.

## Responsibilities

- Accept multipart uploads from PONTON
- Identify the EDA process and version from the message envelope
- Parse the XML payload via the appropriate scalaxb-generated handler
- Merge the inbound message with prior conversation state in `eda.*`
- Publish the enriched `EbMsMessage` to MQTT

## Inbound pipeline

```
PONTON (external) ─HTTP POST multipart─→ FileService.handleUpload
                                              │
                                              ▼
                          MessageHelper.getEdaMessageFromHeader(protocol, version)
                                              │
                                              ▼
                                  Handler.fromXML(xmlNode)
                                              │
                                              ▼
                                 MessageStorage.MergeMessage
                                              │
                                              ▼
                                       mergeEbmsMessage
                                              │
                                              ▼
                                     MqttPublisher → broker
```

See [Architecture / Messaging](../architecture/messaging.md) for the end-to-end view including the backend subscriber side, and the six "gap points" where a new EDA version can silently drop.

## Process and schema versions

EDA distinguishes:

- **Process version** (`messageCodeVersion`) — e.g. `02.30`. Drives handler selection in `MessageHelper`.
- **Schema version** — XML attribute. Independent of the process version.

Multiple supported versions exist in parallel. scalaxb-generated code is namespaced per (protocol, version) to keep them isolated. The shared "targeted hybrid" approach (single package for multiple namespaces) does not scale beyond a small set of XSDs; the convention is **full-per-namespace** generation with `scalaxbProtocolPackageName := Some("xmlprotocol")`.

### Supported ebUtilities message schema versions

Derived from `build.sbt` (scalaxb per-namespace bindings) and `XmlParseHandler.scala`:

| Message type | Supported versions |
|--------------|--------------------|
| CMNotification | 01p11, 01p12, 01p20 |
| CMRevoke | 01p00, 01p10 |
| CMRequest (outbound) | 01p10, 01p20, 01p21, 01p30 |
| CPNotification | 01p13 |
| CPRequest / CPDocument | 01p12 |
| ECMPList | 01p00, 01p10 |
| ConsumptionRecord (CR_MSG) | 01p30, 01p40, 01p41 |
| GCRequest | 01p00 |
| CommonTypes | 01p20 |
| Ponton KEP envelope | v320 |

!!! note
    There is **no** ConsumptionRecord 01p31 and **no** CR 01p20 in the code — do not assume they are supported.

## Outbound

When the customer SPA or backend triggers an EDA-flow (Zählpunkt activation, energy-data request, participation-factor change), eda-xp generates the outbound XML and POSTs to PONTON. Outbound process codes used:

| Process | Trigger |
|---------|---------|
| `EC_REQ_ONL` | Zählpunkt activation |
| `EC_REQ_OFF` | Zählpunkt deactivation |
| `EC_REQ_ENE` | Energy-data request |
| `EC_PRTFACT_CHANGE` (MessageCode `ANFORDERUNG_CPF`) | Change participation factor |
| `EC_REQ_LST` | Request participant list |
| `EC_PODLIST` | Request per-grid-operator metering-point list |

`EC_PRTFACT_CHANGE` is only accepted by the grid operator during business hours (Mon–Fri 09:00–17:00) and the change takes effect the following day. The XML helper sets `ProcessDate = today + 1` and `DateActivate = today + 1` by convention. Triggering outside that window returns an `ABLEHNUNG_CPF` with response code 82 (`Prozessdatum falsch`).

## XXE hardening

`scala.xml.XML.loadString` on attacker-controlled XML without XXE defenses is unsafe: scala-xml versions < 2.2 do not have secure-by-default parsing. eda-xp uses **scala-xml 2.3.0**, which is secure by default (DOCTYPE declarations disallowed), so inbound XML parsing is XXE-hardened.

This protection applies anywhere in eda-xp that consumes inbound XML; keep scala-xml at ≥ 2.2 to retain it.

## Database

Owns conversation state in the `eda` schema. Tables (approximate):

| Table | Holds |
|-------|-------|
| `message` | stored EbMsMessage by conversation id |
| `protocol_state` | per-(tenant, protocol, conversation-id) progress |

The `MessageStorage` actor reads prior state on merge. If the conversation-id does not match a prior outbound (e.g. a partner-initiated message), the merge produces an enriched message with empty prior context — the backend handler then deals with the unenriched case.

## MQTT output

eda-xp publishes on two topic families:

### EDA workflow events (cleartext JSON)

```
eda/<tenant>/protocol/<process_lower>
eda/response/<tenant>/protocol/<process_lower>
```

`<tenant>` is the EEG's `community_id`. `<process_lower>` is the process code lowercased: `cm_rev_sp`, `ec_req_onl`, `cm_rev_imp`, `cr_req_pt`, etc.

Payload is `EbMsMessage` serialized as JSON via circe — cleartext, no encryption.

### CR_MSG (energy data — gzip + Base64)

```
eda/response/<tenant>/protocol/cr_msg
eda/response/<tenant>/protocol/inverter_msg
```

`cr_msg` (network-operator data response — contains per-meter, per-slot energy values) is the **only** topic that receives special encoding: the JSON is gzip-compressed then Base64-encoded (see below). It is **not** encrypted. `inverter_msg` (PV inverter telemetry) is cleartext.

Subscribers: [energystore (v1)](energystore.md) and [energystore-v2](energystore-v2.md).

### CR_MSG payload encoding

`cr_msg` payloads are gzip-compressed and Base64-encoded before publishing. This is the canonical description of the wire format — other pages link here instead of repeating it.

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
| Scope | the `cr_msg` topic only; every other `eda/response/*/protocol/*` topic is published as cleartext JSON (`case _ => value`) |

Subscribers reverse the pipeline: Base64-decode → gunzip → JSON parse.

!!! warning "No encryption — despite the topic naming"
    There is **no encryption** anywhere in eda-xp. In `MqttSystem.scala`, `eventToMqttMessage` carries the comment `// Todo: Encrypt and compress here`, but only the gzip + Base64 path for `CR_MSG` is implemented — there are zero references to AES, `Cipher`, or any encryption primitive in the source. Earlier revisions of this page described an AES-256-CBC envelope with a pre-shared key/IV; that was never implemented. Treat the wire format as gzip + Base64 obfuscation only, not a confidentiality control.

#### Architecture status

Encryption of cluster-internal MQTT traffic remains a TODO (see the source comment above), not an implemented Defense-in-Depth measure. The long-term design of this layer (implement, keep as-is, or replace) is tracked in an internal architecture decision record in the platform repository.

The [energystore-v2](energystore-v2.md) decrypt module is env-configurable; because eda-xp publishes `cr_msg` as plain gzip+Base64, no key material is required to consume it.

## Config

Configuration is layered: `reference.conf` (defaults) ← `application.conf` (mounted at `/conf`) ← environment overrides.

| Key / variable | Purpose |
|----------------|---------|
| `app.interface.mode` | `SIMU` or `PROD` |
| `app.kepserver.url` | outbound Ponton X/P endpoint |
| `app.server.host` / `app.server.port` | HTTP server (port 6090) |
| `app.grpc.host` / `app.grpc.port` | gRPC server (port 9093) |
| `epmsmail.mail.*` | per-domain IMAP/SMTP mailbox config + poll interval |
| `epmsmail.mqtt.host/port/sub-topic/qos/consumer-id` | MQTT broker connection |
| `epmsmail.mqtt.topics.energyTopic/cmTopic/cpTopic/errorTopic` | response topic prefixes |
| `epmsmail.admin.*` | admin notification SMTP |
| `slick.pgsql.local.db.*` | PostgreSQL connection (for `eda` schema) |
| `flyway.*` | DB migration settings |

Common environment overrides: `SMTP_ADMIN_SERVER_HOST` / `SMTP_ADMIN_SERVER_PORT`, `EMAIL_ADMIN_USER`, `EMAIL_ADMIN_PWD`, `TZ`, `JAVA_OPTS`.

Two volumes are mounted: `/conf` (holds `application.conf`) and `/storage/prod`.

## Build and image

- Build: sbt with `sbt-native-packager`
- Docker image name `eegfaktura-kep`; base `eclipse-temurin:17-jre`; binary `xpadapter`; main class `XpAdapter`
- Runtime defaults: `JAVA_OPTS=-Xmx4g`, `TZ=Europe/Vienna`
- Key libraries: scalaxb 1.12.0 (per-namespace XML binding), Courier 3.0.1 (mail), Slick 3.5.1 + slick-pg, Flyway 10.20.0, Circe 0.14.3, scala-xml 2.3.0 (XXE-safe by default), PostgreSQL JDBC 42.7.3, Logback
- Final image is substantially larger than the Go services (JRE + scalaxb-generated classes for every supported EDA protocol version)

## When PONTON is not deployed

In environments without a real PONTON link (dev / mock instances), an [eda-mock](eda-mock.md) placeholder stands in for the PONTON side and the customer SPA's "Messages" menu is hidden. Note the currently deployed `eda-mock` is a `traefik/whoami` placeholder rather than a functional EDA stub — see that page for details.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full inbound pipeline + gap points
- [services/eda-mock](eda-mock.md) — stub for dev
- [services/backend](backend.md) — MQTT consumer side
