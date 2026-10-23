# Architecture

System-level views of the eegfaktura platform.

- **[Service Overview](overview.md)** — service topology, request flow, language stack per service
- **[Authentication](auth.md)** — JWT structure, Keycloak realm, per-service auth checks
- **[Databases](databases.md)** — PostgreSQL schemas per service, access matrix
- **[Messaging](messaging.md)** — MQTT topics, EDA inbound pipeline
- **[Notification e-mail](email.md)** — who sends which mail, and how

## TL;DR

eegfaktura is a microservice platform with three tiers:

1. **Backend services** (Go + Java + Scala + Python) — domain logic, REST APIs
2. **Frontend services** (React) — customer + admin SPAs
3. **Platform services** (PostgreSQL, Keycloak, Mosquitto MQTT, Postfix mail relay) — infrastructure dependencies

Authentication is centralized via Keycloak (OIDC), state lives in PostgreSQL (per-service schema; exception: energystore v1 keeps energy data in BadgerDB), and EDA messages from the network operators enter through eda-xp and reach the other services via MQTT.

See the per-area pages for details.
