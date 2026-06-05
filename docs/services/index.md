# Services

One page per deployed service in a typical eegfaktura instance.

Each service page covers:

- **Responsibility** — what business logic does it own
- **APIs** — exposed endpoints (REST / gRPC / MQTT)
- **Config** — required ConfigMaps and env vars
- **Secrets** — required Kubernetes Secrets
- **Image provenance** — source repository, build pipeline, tagging convention
- **Auth** — which JWT claims it enforces (if any)
- **Database access** — which schemas it reads/writes
- **Inter-service dependencies** — which other services it calls

## Service catalogue

### Backend layer (domain APIs)

- **[backend](backend.md)** (Go) — master data: participants, metering points, EEGs, contracts
- **[billing](billing.md)** (Java/Spring) — billing document generation, tariff application
- **[energystore](energystore.md)** (Go + Badger) — energy data storage and per-period reports
- **[filestore](filestore.md)** (Python) — file storage and download endpoints
- **[eda-xp](eda-xp.md)** (Scala/Pekko) — EDA protocol gateway for network operator communication
- **[eda-mock](eda-mock.md)** (Scala) — EDA mock for development / testing
- **[admin-backend](admin-backend.md)** (Scala/Pekko) — VFEEG superuser API for cross-instance maintenance

### Frontend layer (SPAs)

- **[web](web.md)** (React) — customer-facing web UI for community members
- **[admin-web](admin-web.md)** (React Admin) — VFEEG maintenance UI

### Platform layer (infrastructure)

- **[keycloak](keycloak.md)** — OIDC identity provider
- **[postgres](postgres.md)** — relational database
- **[mosquitto](mosquitto.md)** — MQTT broker
- **[mailpit](mailpit.md)** — SMTP catcher (dev)
- **[billing-cert-rotator](billing-cert-rotator.md)** — JWT signing cert rotation

> **Note:** This index lists the services as currently deployed. Per-service pages cover responsibilities, APIs, configuration, auth, database access, and inter-service dependencies.
