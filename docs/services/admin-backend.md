# admin-backend

Scala / Pekko backend that serves the VFEEG maintenance UI ([admin-web](admin-web.md)). Cross-tenant maintenance API used by superusers, distinct from the customer-side `backend`.

Historically named `registration-backend`.

## At a glance

| | |
|---|---|
| Language | Scala 2.13.9 |
| Framework | Apache Pekko 1.2.1 + Pekko HTTP 1.2.0 (migrated from Akka) |
| Audience | `vfeeg-superuser` operators |
| Config style | HOCON (`-Dconfig.file=...`) |
| Auth | Nimbus JOSE+JWT verify, `aud` array must contain `at.ourproject.vfeeg.admin` |
| Persistence | Slick 3.5.1 + HikariCP (PostgreSQL) |
| HTTP bind | `0.0.0.0:8085` |

## Responsibilities

- Cross-tenant participant / EEG lookup and editing
- EEG onboarding / registration flow for new EEGs
- Operational management tools for the VFEEG team that bypass tenant scoping

The service talks gRPC to the customer-side `backend` (`RegisterEegService`, port 9092) and to `eda-xp` (`RegisterPontonService`, port 9093).

The exact endpoint set depends on the deployed revision; admin-web is the authoritative client.

## Auth

Verifies JWTs with Nimbus JOSE+JWT (9.31). The Keycloak JWKS is fetched at runtime via `RemoteJWKSet` (which caches keys and handles rotation). The authenticator enforces that the token `aud` array contains the expected audience `at.ourproject.vfeeg.admin` (configured via `keycloakAuthenticator.clientId`, overridable with `KEYCLOAK_AUDIENCE`), which the Keycloak realm injects via an audience mapper on the admin client (external realm config).

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
- `keycloakAuthenticator` block naming the expected audience `at.ourproject.vfeeg.admin`
- `pekko.grpc.client` entries for `RegisterEegService` and `RegisterPontonService`
- `slick.pgsql.vfeeg` DB connection (Slick + HikariCP)

The HOCON file uses `${?ENV}` overrides, so individual values can be supplied via environment variables:

| Env var | Overrides |
|---|---|
| `LOG_LEVEL` | Pekko log level |
| `KEYCLOAK_URL` | Keycloak base URL |
| `KEYCLOAK_REALM` | Keycloak realm |
| `KEYCLOAK_ADMIN_CLI_AUDIENCE` | admin-cli client id |
| `KEYCLOAK_ADMIN_CLI_SECRET` | admin-cli client secret |
| `KEYCLOAK_AUDIENCE` | expected token audience (`at.ourproject.vfeeg.admin`) |
| `REGISTER_SERVICE_HOST_EEG` | gRPC host for `RegisterEegService` (port 9092) |
| `REGISTER_SERVICE_HOST_PONTON` | gRPC host for `RegisterPontonService` (port 9093) |
| `DATABASE_HOST` / `DATABASE_PORT` | PostgreSQL host / port |
| `DATABASE_DBNAME` | database name |
| `DATABASE_USER` / `DATABASE_PASSWORD` | database credentials |

The gRPC clients are configured with static host/port and `use-tls=false` — suitable for in-cluster traffic, but note that transport TLS is off.

!!! warning "Override the shipped defaults"
    `application.conf` ships **dev default values** for the Keycloak secret and the database credentials. These are intended for local development only and **must** be overridden via the environment variables above (or a mounted production HOCON) in any deployed environment.

## Pekko migration

The service was migrated from Akka to Pekko ahead of public-ingress exposure. Background: Akka 2.9+ adopted the BSL license, incompatible with the project's distribution requirements. Pekko is the Apache-licensed continuation.

Practical follow-ups when working on this service:

- All `akka.*` imports → `org.apache.pekko.*`
- akka-grpc-codegen → pekko-grpc equivalent (or proto-build from the existing `.proto` files)
- application.conf top-level prefix `akka` → `pekko`

## Database

Uses Slick 3.5.1 with a HikariCP connection pool and the PostgreSQL driver, reading the `base` schema directly for cross-tenant participant queries that the customer backend would scope down.

!!! note "No migrations in this repo"
    There is **no Flyway and no `MigrationRunner`** in admin-backend — it opens a Slick connection only. The database schema is managed elsewhere.

## Build and image

- Build: sbt-native-packager (`JavaAppPackaging`), base image `eclipse-temurin:17-jre`
- Image package name: `eeg-registration-backend` (legacy name kept intentionally), pushed to `ghcr.io/vfeeg-development/eeg-registration-backend`
- Main class `at.ourproject.Registration`; Docker `CMD` is `/opt/docker/bin/eegfaktura-registration -Dconfig.file=/conf/application.conf`
- HTTP server binds `0.0.0.0:8085`

!!! note "Pekko version pinning"
    `pekko-discovery` is explicitly pinned to **1.2.1** to match the rest of Pekko. The Pekko gRPC sbt plugin (1.1.1) transitively pulls `pekko-discovery` 1.1.2, and the resulting mixed Pekko versioning crashes at runtime — keep the explicit pin.

## Related

- [services/admin-web](admin-web.md) — the SPA client
- [Architecture / Authentication](../architecture/auth.md) — audience mapper details
- [services/backend](backend.md) — customer-side counterpart
