# eda-xp

Scala / Pekko service that bridges PONTON (the external EDA adapter) with the rest of the platform. Accepts inbound EDA XML over HTTP, parses it via scalaxb, enriches with stored conversation state, and republishes on MQTT for the backend to consume.

## At a glance

| | |
|---|---|
| Language | Scala |
| Framework | Pekko (formerly Akka) HTTP + actors |
| Inbound | HTTP multipart upload from PONTON |
| Outbound | MQTT publish to `eda/<tenant>/protocol/<process_lower>` |
| State | PostgreSQL `eda` schema (conversation state) |
| XML parsing | scalaxb (full per-namespace generation) |

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

## Outbound

When the customer SPA or backend triggers an EDA-flow (Zählpunkt activation, energy-data request, participation-factor change), eda-xp generates the outbound XML and POSTs to PONTON. Outbound process codes used:

| Process | Trigger |
|---------|---------|
| `EC_REQ_ONL` | Zählpunkt activation |
| `EC_REQ_OFF` | Zählpunkt deactivation |
| `EC_REQ_ENE` | Energy-data request |
| `EC_REQ_PRZ` | Change participation factor |
| `EC_REQ_LST` | Request participant list |

## XXE hardening

`scala.xml.XML.loadString` on attacker-controlled XML without XXE defenses is unsafe. scala-xml versions < 2.2 do not have secure-by-default parsing. Upgrade to scala-xml ≥ 2.2 (secure by default) **or** use a custom `XMLLoader` with `disallow-doctype-decl` set.

This applies anywhere in eda-xp that consumes inbound XML.

## Database

Owns conversation state in the `eda` schema. Tables (approximate):

| Table | Holds |
|-------|-------|
| `message` | stored EbMsMessage by conversation id |
| `protocol_state` | per-(tenant, protocol, conversation-id) progress |

The `MessageStorage` actor reads prior state on merge. If the conversation-id does not match a prior outbound (e.g. a partner-initiated message), the merge produces an enriched message with empty prior context — the backend handler then deals with the unenriched case.

## MQTT topic

```
eda/<tenant>/protocol/<process_lower>
```

`<tenant>` is the EEG's `community_id`. `<process_lower>` is the process code lowercased: `cm_rev_sp`, `ec_req_onl`, `cm_rev_imp`, etc.

Payload is `EbMsMessage` serialized as JSON via circe.

## Config

| Variable | Purpose |
|----------|---------|
| `DB_*` | PG connection (for `eda` schema) |
| `MQTT_*` | broker connection |
| `PONTON_*` | inbound + outbound PONTON endpoint config |
| JVM heap | tune per environment |

## Build and image

- Source: an `eegfaktura-eda-comm` repo (or `energycash-eda-comm` historically)
- Build: sbt + JDK 17 + sbt 1.7.x; `JavaAppPackaging` produces the runtime image
- Final image size is substantial (Scala runtime + scalaxb-generated classes); ~580 MiB is typical

## When PONTON is not deployed

In environments without a real PONTON link (dev / mock instances), the [eda-mock](eda-mock.md) service stubs the PONTON side and the customer SPA's "Messages" menu is hidden.

## Related

- [Architecture / Messaging](../architecture/messaging.md) — full inbound pipeline + gap points
- [services/eda-mock](eda-mock.md) — stub for dev
- [services/backend](backend.md) — MQTT consumer side
