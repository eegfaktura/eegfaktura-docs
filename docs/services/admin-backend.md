# admin-backend

Scala / Pekko backend that serves the VFEEG maintenance UI ([admin-web](admin-web.md)). Cross-tenant maintenance API used by superusers, distinct from the customer-side `backend`.

Historically named `registration-backend`.

## At a glance

| | |
|---|---|
| Language | Scala 2.13.9 |
| Framework | Apache Pekko 1.2.1 + Pekko HTTP 1.2.0 (migrated from Akka) |
| Audience | operators with the `superuser` realm role |
| Config style | HOCON (`-Dconfig.file=...`) |
| Auth | Nimbus JOSE+JWT verify (RS256 only, `exp` required, `iss` checked), `aud` array must contain `at.ourproject.vfeeg.admin`; realm role `superuser` on every route |
| Persistence | Slick 3.5.1 + HikariCP (PostgreSQL) |
| HTTP bind | `0.0.0.0:8085` |

## Responsibilities

- Cross-tenant participant / EEG lookup and editing
- EEG onboarding / registration flow for new EEGs
- Logins (Keycloak accounts) of an EEG: created at registration, added later, listed (several logins per EEG since 1.1.0)
- Operational management tools for the VFEEG team that bypass tenant scoping

The service talks gRPC to the customer-side `backend` (`RegisterEegService` and `AdminEegService`, both through the client config `register.RegisterEegService`, port 9092) and to `eda-xp` (`RegisterPontonService`, port 9093).

admin-web is the authoritative client.

## Endpoints

All routes sit under three prefixes (`vfeeg`, `eeg`, `admin`) and require the `superuser` realm role (see [Auth](#auth)).

| Route | Purpose |
|---|---|
| `GET /vfeeg/eeg` | all EEGs (Slick read) |
| `GET /vfeeg/participants?tenant=<id>` | members of an EEG with their metering points (Slick read) |
| `GET /vfeeg/operators` | grid operators (Slick read) |
| `POST /eeg/register` | register a new EEG: gRPC to backend, Ponton registration at eda-xp for online EEGs, Keycloak logins |
| `POST /eeg/ponton` | set the Ponton config of an existing EEG; HTTP is always 200, the outcome is `status` / `message` in the body |
| `POST /eeg/account?tenant=<id>` | add a login to an existing EEG |
| `GET /eeg/accounts?tenant=<id>` | list the logins of an EEG (name, e-mail, username, enabled, password not yet set) |
| `GET /eeg/addTenant?tenant=<id>&email=<mail>` | add a tenant to an existing Keycloak user |
| `POST /admin/master/update` | metering-point update (`PROCESSSTATUS`, `ACTIVESINCE`, `INACTIVESINCE`) via gRPC `AdminEegService.updateValue` |
| `POST /admin/master/update/participant` | participant update (`PARTICIPANT`), same gRPC call |
| `POST /admin/master/update/eeg` | EEG update (`EEG`), same gRPC call |

The `/admin/master/*` body carries an `updateClass` discriminator, `tenant`, optional `participantId` / `meteringPoint` and a `value` map. The tenant comes from the request body and is not checked against the token: cross-tenant access is what the `superuser` role grants.

The Ponton transport `MAIL` is rejected since 1.0.5 (`/eeg/register` answers 400, `/eeg/ponton` a body status 400).

### Logins per EEG

`POST /eeg/register` takes an optional `accounts` list (`firstname`, `lastname`, `email`). Exactly these get a login, all with the groups `EEG_ADMIN` and `EEG_OWNER`; without the list the contact (`user`) gets the login, as before. The answer reports each account as `created`, `linked` or `failed` with a reason; the EEG is registered even if an account fails. `POST /eeg/account` applies the same rules to an existing EEG.

- An existing account is linked only if its e-mail is verified, and only if it already has `EEG_ADMIN` or no community yet (the groups hold for all its communities). Otherwise it is reported as failed; `/eeg/account` answers 409.
- The tenant is trimmed and must consist of letters, digits, `_` and `-` (else 400). A new account's `tenant` attribute is written as JSON; when a tenant is added, both stored forms (JSON array or one value per tenant) are read and kept.
- New accounts get a random temporary password and the Keycloak mail to set their own (`UPDATE_PASSWORD`). If Keycloak cannot send the mail, this is only logged.
- `GET /eeg/accounts` reads all realm users page by page, because Keycloak cannot search inside the tenant attribute.
- Timeouts: `/eeg/register`, `/eeg/account` and `/eeg/accounts` wait up to 50 s for Keycloak; the other routes use `app.routes.ask-timeout` (10 s).

## Auth

Verifies JWTs with Nimbus JOSE+JWT (9.31). The Keycloak JWKS is fetched at runtime via `RemoteJWKSet` (which caches keys and handles rotation). By default the JWKS URL is derived from the issuer; `KEYCLOAK_AUTHENTICATOR_JWKS_URL` (`keycloakAuthenticator.jwksUrl`) sets a separate fetch URL (e.g. cluster-internal) while the expected issuer stays the public URL.

Checks in `KeycloakJwtAuthenticator`:

- signature algorithm RS256 only (the key selector is bound to `JWSAlgorithm.RS256`)
- `exp` is required and must lie in the future
- `iss` must equal `<keycloakAuthenticator.url>/realms/<keycloakAuthenticator.realm>` (from `KEYCLOAK_URL` / `KEYCLOAK_REALM`, trailing slash stripped), compared case-insensitively
- the `aud` array must contain the expected audience `at.ourproject.vfeeg.admin` (configured via `keycloakAuthenticator.clientId`, overridable with `KEYCLOAK_AUDIENCE`), which the realm injects via the `admin-audience` mapper on the admin client (`realm/EEGFaktura.yaml` in eegfaktura-keycloak)

Authorization: since 1.0.3 every route requires the realm role `superuser` in `realm_access.roles` (`RoleDirectives.requireSuperuser`); otherwise the answer is 403 with `{"error":"superuser role required"}`. There is no additional tenant check. The admin client is a public client with the standard flow, so any realm user can obtain a token with the admin audience; the role check is the boundary. A newly assigned role takes effect only with a fresh token.

## Config — HOCON, not env vars

Configuration is HOCON (Typesafe Config). The image's `CMD` passes `-Dconfig.file=/conf/application.conf` (`/conf` is the exposed volume); the packaged `src/main/resources/application.conf` is the reference.

HOCON sections:

- `keycloak` block: Keycloak URL, realm and the `admin-cli` client (id + secret) used for user management through the Keycloak admin API
- `keycloakAuthenticator` block: URL + realm (they form the expected issuer), the expected audience `at.ourproject.vfeeg.admin`, optional `jwksUrl`
- `app.routes.ask-timeout` (default 10 s)
- `pekko.grpc.client` entries for `RegisterEegService` and `RegisterPontonService`
- `slick.pgsql.vfeeg` DB connection (Slick + HikariCP)

The HOCON file uses `${?ENV}` overrides, so individual values can be supplied via environment variables:

| Env var | Overrides |
|---|---|
| `LOG_LEVEL` | Pekko log level (default `INFO`) |
| `LOG_LEVEL_APP` | level of the application loggers `at.ourproject.*` (default `INFO`; read by `logback.xml`) |
| `KEYCLOAK_URL` | Keycloak base URL |
| `KEYCLOAK_REALM` | Keycloak realm |
| `KEYCLOAK_ADMIN_CLI_AUDIENCE` | admin-cli client id |
| `KEYCLOAK_ADMIN_CLI_SECRET` | admin-cli client secret — **required** since 1.1.0, no default; the service refuses to start without it |
| `KEYCLOAK_AUDIENCE` | expected token audience (`at.ourproject.vfeeg.admin`) |
| `KEYCLOAK_AUTHENTICATOR_JWKS_URL` | optional JWKS fetch URL, separate from the issuer |
| `REGISTER_SERVICE_HOST_EEG` | gRPC host for `RegisterEegService` / `AdminEegService` (port 9092) |
| `REGISTER_SERVICE_HOST_PONTON` | gRPC host for `RegisterPontonService` (port 9093) |
| `DATABASE_HOST` / `DATABASE_PORT` | PostgreSQL host / port |
| `DATABASE_DBNAME` | database name |
| `DATABASE_USER` / `DATABASE_PASSWORD` | database credentials |

The gRPC clients are configured with static host/port and `use-tls=false` — suitable for in-cluster traffic, but note that transport TLS is off.

!!! warning "Override the shipped defaults"
    `application.conf` ships **dev default values** for the database credentials (the `admin-cli` secret has had no default since 1.1.0). These are intended for local development only and **must** be overridden via the environment variables above (or a mounted production HOCON) in any deployed environment.

## Pekko migration

The service was migrated from Akka to Pekko ahead of public-ingress exposure. Background: Akka 2.9+ adopted the BSL license, incompatible with the project's distribution requirements. Pekko is the Apache-licensed continuation.

Practical follow-ups when working on this service:

- All `akka.*` imports → `org.apache.pekko.*`
- akka-grpc-codegen → pekko-grpc equivalent (or proto-build from the existing `.proto` files)
- application.conf top-level prefix `akka` → `pekko`

## Database

Uses Slick 3.5.1 with a HikariCP connection pool and the PostgreSQL driver (42.7.13 since 1.0.6), reading the `base` schema directly (`base.eeg`, `base.participant`, `base.meteringpoint`, `base.gridoperators`) for cross-tenant queries that the customer backend would scope down. Writes do not go through Slick: master-data updates and EEG registration go to the backend over gRPC.

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
