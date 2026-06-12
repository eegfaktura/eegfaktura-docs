# eegfaktura Documentation

Developer and maintainer documentation for the **eegfaktura** software suite — a platform for billing and energy management in Austrian energy communities (EEG / Energiegemeinschaften).

## What is eegfaktura?

A microservice-based platform that:

- Manages **member master data** of energy communities (participants, metering points, contracts)
- Drives the **EDA-protocol lifecycle** of members and metering points: register, activate, deactivate, change participation factor, request energy data, request participant list
- **Imports energy data** from Austrian network operators via the **EDA** (Energie Daten Austausch) protocol — the network operator performs the per-period allocation by Teilnahmefaktor; eegfaktura stores the resulting values
- **Generates billing documents** based on the allocated consumption / production / surplus per period
- Provides **customer-facing UIs** for members and a **maintenance UI** for community administrators

## Documentation map

- **[Architecture](architecture/index.md)** — How services fit together. Service topology, authentication flow, databases, messaging, deployment patterns.
- **[Services](services/index.md)** — One page per deployed service. Responsibilities, APIs, configuration, secrets, image provenance.
- **[Operations](operations/index.md)** — Provisioning, wipe-replay, observability.
- **[Reference](reference/index.md)** — Glossary (OBIS, EDA, EEG terminology), reference tables.

## Quick orientation

If you are…

- **New to the codebase** — start with [Architecture / Service Overview](architecture/overview.md), then drill down to a specific service page.
- **Operating a deployment** — see [Operations / Provisioning Pipeline](operations/pipeline.md).
- **Adding a feature to a single service** — go directly to the service page (`services/<name>.md`).
- **Investigating auth issues** — see [Architecture / Authentication](architecture/auth.md).

## Conventions

- Examples use generic placeholders: `<eeg-domain>` for the public domain, `<tenant-id>` for the EEG community ID, `<namespace>` for the Kubernetes namespace.
- Service-internal DNS uses Kubernetes service-name patterns: `<service>.<namespace>.svc.cluster.local`.
- JWT claims are documented with the exact claim name as emitted by Keycloak.
- These docs focus on **how things connect** — architecture, schemas, protocols, code-level gotchas. Production sizing, resource consumption, and operational tuning live in ADRs (`eegfaktura-platform/docs/adr/`) and per-cluster Helm values, not here.
