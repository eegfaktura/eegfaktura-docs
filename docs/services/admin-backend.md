# admin-backend

Scala / Pekko backend that serves the VFEEG maintenance UI ([admin-web](admin-web.md)). Cross-tenant maintenance API used by superusers, distinct from the customer-side `backend`.

Historically named `registration-backend`.

## At a glance

| | |
|---|---|
| Language | Scala |
| Framework | Pekko HTTP (migrated from Akka) |
| Audience | `vfeeg-superuser` operators |
| Config style | HOCON (`-Dconfig.file=...`) |
| Auth | JWT verify, `aud` array must contain `at.ourproject.vfeeg.admin` |
| State | reads from `base` schema directly (some operations) |

## Responsibilities

- Cross-tenant participant lookup and editing
- Onboarding flow for new EEGs
- Operational tools for the VFEEG team that bypass tenant scoping

The exact endpoint set depends on the deployed revision; admin-web is the authoritative client.

## Auth

Verifies JWTs via Pekko-HTTP's JWT directive. The `aud` array must contain `at.ourproject.vfeeg.admin`, which the Keycloak realm injects via an audience mapper on the admin client.

The admin-web SPA itself requires the `vfeeg-superuser` group for login. Reaching admin-backend from anywhere else (e.g. customer-web with a re-resolved token) would require the same audience.

## Config — HOCON, not env vars

The `v0.2.12` release and the surrounding Pekko-migration generations require HOCON-mode configuration with `-Dconfig.file=/path/to/application.conf`. The compose-style JSON env-var configuration is half-deprecated and not the recommended path.

Provide a `ConfigMap` with the HOCON file and a JVM arg:

```yaml
env:
  - name: JAVA_OPTS
    value: "-Dconfig.file=/etc/admin-backend/application.conf"
volumeMounts:
  - name: app-conf
    mountPath: /etc/admin-backend
volumes:
  - name: app-conf
    configMap:
      name: admin-backend-config
```

Required HOCON sections (typical):

- `keycloak` block with realm + issuer URL
- `audience-mapper` config naming `at.ourproject.vfeeg.admin`
- DB connection (Slick / Flyway-style)
- Pekko-HTTP server bind config

## Pekko migration

The service was migrated from Akka to Pekko ahead of public-ingress exposure. Background: Akka 2.9+ adopted the BSL license, incompatible with the project's distribution requirements. Pekko is the Apache-licensed continuation.

Practical follow-ups when working on this service:

- All `akka.*` imports → `org.apache.pekko.*`
- akka-grpc-codegen → pekko-grpc equivalent (or proto-build from the existing `.proto` files)
- application.conf top-level prefix `akka` → `pekko`

## Database

Reads `base` schema directly for some operations (cross-tenant participant queries that the customer backend would scope down). Also has its own Slick + Flyway layer for admin-side tables (onboarding state, etc.).

A `MigrationRunner` runs Flyway on startup.

## Build and image

- Source: an `eegfaktura-admin-backend` or similar repo (historically `registration-backend`)
- Build: sbt + JDK 17, multi-stage Docker, `JavaAppPackaging`

## Pre-public-ingress gating

Before exposing admin-backend at a public ingress, two checks are mandatory:

1. **Audience mapper is correctly wired**, so that customer-token holders cannot reach admin endpoints by chance.
2. **AGPL § 13 footer** is present in the SPA referencing the admin-backend source.

Both apply equally to admin-web.

## Related

- [services/admin-web](admin-web.md) — the SPA client
- [Architecture / Authentication](../architecture/auth.md) — audience mapper details
- [services/backend](backend.md) — customer-side counterpart
