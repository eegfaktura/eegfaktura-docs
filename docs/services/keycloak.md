# keycloak

OIDC identity provider for the entire platform. One realm, three declared clients (plus the built-in `admin-cli`), three groups.

## At a glance

| | |
|---|---|
| Keycloak | 26.7.4 (`quay.io/keycloak/keycloak:26.7.4`, image 1.0.3) |
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-keycloak:v1.0.3` |
| Realm | `EEGFaktura`, declared in `realm/EEGFaktura.yaml` (keycloak-config-cli) |
| Login theme | `eegfaktura-ui` (baked into the image) |
| Token signing | RS256 |
| State | PostgreSQL database `keycloak` |

In the local docker-compose stack Keycloak runs the custom `ghcr.io/eegfaktura/eegfaktura-keycloak:v1.0.3` image, started with `start --optimized --import-realm`. The realm (`EEGFaktura`) is imported from `./keycloak/import`. It publishes the main port `8080` and the management port `9181` (`KC_HTTP_MANAGEMENT_PORT`), and runs with `KC_HOSTNAME_STRICT=false` for local use. Keycloak is reached directly there, so no proxy headers are trusted. The compose file still sets the Keycloak-v1 options `KC_PROXY=edge`, `PROXY_ADDRESS_FORWARDING` and `KC_HOSTNAME_STRICT_BACKCHANNEL`; Keycloak 26 ignores them and only logs "Hostname v1 options … still in use".

For the auth contract (clients, mappers, JWT claims, role behavior), see [Architecture / Authentication](../architecture/auth.md). This page covers the Keycloak realm content.

## Realm content

The declarative source of truth is `realm/EEGFaktura.yaml` in eegfaktura-keycloak (ADR-0009), applied with keycloak-config-cli (create + update, `managed: no-delete`, so undeclared objects such as key providers are never deleted). Environment-specific string fields are templated with config-cli env substitution: `AUTH_URL` (realm `frontendUrl`), `CUSTOMER_HOST` and `ADMIN_HOST` (redirect URIs / web origins). Signing keys are not part of the config; they stay per Keycloak instance.

| Item | Detail |
|------|--------|
| Realm name | `EEGFaktura` |
| Login theme | `eegfaktura-ui` (`themes/eegfaktura-ui/login`, copied into the image); the realm references it as `loginTheme`, the realm YAML does not set it |
| Groups | `EEG_ADMIN`, `EEG_OWNER`, `EEG_USER` (top level; billing consumes them as authorities) |
| Realm roles | `superuser` (cross-tenant VFEEG admin: admin-backend, admin-web, energystore raw-data delete), `Manager`, `user` (plus the Keycloak defaults) |
| Clients | `at.ourproject.vfeeg.app`, `at.ourproject.vfeeg.api`, `at.ourproject.vfeeg.admin`; `admin-cli` is the Keycloak built-in and not declared |

### Mappers per client

| Client | Mapper | Type | Notes |
|--------|--------|------|-------|
| `at.ourproject.vfeeg.app` (public, standard flow) | `tenantAttributeMapper` | oidc-usermodel-attribute-mapper | maps user attribute `tenant` to claim `tenant`; multivalued; aggregate.attrs=true; jsonType.label=JSON |
| `at.ourproject.vfeeg.app` | `Group Mappers` | oidc-group-membership-mapper | claim `access_groups`, multivalued, full.path=true |
| `at.ourproject.vfeeg.api` (confidential, direct grant) | `tenantIds` | oidc-usermodel-attribute-mapper | claim `tenant`, single-valued, jsonType.label=String |
| `at.ourproject.vfeeg.admin` (public, standard flow) | `admin-audience` | oidc-audience-mapper | adds `at.ourproject.vfeeg.admin` to `aud` of the access token (admin-backend requires it) |

The app client deliberately has **no** audience mapper: `aud` stays as resolved by the default `roles` scope. There is no `groups` claim mapper; admin-web and admin-backend read the realm role `superuser` from `realm_access.roles`, not a group claim.

### User profile

`tenant` is not a declared user-profile attribute. The profile sets `unmanagedAttributePolicy: ADMIN_EDIT`, so only admins (admin-backend through the admin API) can see and set it; users cannot change their own tenant. Without this setting a new realm drops `tenant` even when an admin sets it. Each config-cli apply overwrites the realm's user profile with the copy in the YAML (Keycloak 26.7.4 default + policy).

### Brute-force protection and events

Since 1.0.3 the realm config contains:

- **Brute-force protection:** temporary lockout after 10 failures (`failureFactor`), wait growing by 60 s up to 15 min (`maxFailureWaitSeconds: 900`), no permanent lockout. Quick-login check: two failures within 1 s lock for 60 s. Trade-off: anyone who knows a username can lock that account temporarily.
- **Login events:** `eventsEnabled`, limited via `enabledEventTypes` to login/logout, password reset/update, profile, e-mail and credential changes (each with its `_ERROR` variant) and `USER_DISABLED_BY_TEMPORARY_LOCKOUT`; kept 90 days (`eventsExpiration`).
- **Admin events:** enabled without representation details (`adminEventsDetailsEnabled: false`); kept 90 days (`attributes.adminEventsExpiration`).

These apply to Keycloaks set up from this config; the running instances got the same settings separately.

## Database

Keycloak owns its own logical database (typically named `keycloak`) in the shared PostgreSQL cluster. Schema is managed entirely by Keycloak; do not migrate or modify directly.

## User-attribute cache quirk (Keycloak 25)

Keycloak 25 caches `cannot map type for token claim` failures per user. If a `tenant` attribute is provisioned in a format the mapper cannot parse (e.g. unquoted instead of JSON-quoted), the user can be stuck with a permanent token-without-tenant state even after the attribute is fixed.

Recovery:

1. `DELETE` and `INSERT` the `user_attribute` row.
2. Restart the Keycloak pod.

A simple admin-API `PUT` does not always clear the cache.

## Realm-frontend-URL

When Keycloak runs behind an ingress, the realm setting `frontendUrl` must match the public URL, otherwise login redirects target the in-cluster service DNS and the browser cannot follow them. The realm YAML sets it from `AUTH_URL`; it is also the issuer the services match against.

## Config

| Variable | Purpose |
|----------|---------|
| `KC_DB_*` | database connection (the image sets `KC_DB=postgres` and a default `KC_DB_URL`) |
| `KC_HOSTNAME` | public URL incl. scheme, e.g. `https://<auth-domain>` (no `/auth` path) |
| `KC_HTTP_ENABLED` | `true` when TLS terminates at the ingress / reverse proxy (the image sets `true`) |
| `KC_PROXY_HEADERS` | `xforwarded` when behind an ingress / reverse proxy that sets `X-Forwarded-*`; leave unset when Keycloak is reached directly |

The image also sets `KC_HEALTH_ENABLED=true`, `KC_METRICS_ENABLED=true` and `KC_HOSTNAME_STRICT=false`.

The Keycloak-v1 hostname/proxy options (`KC_PROXY`, `PROXY_ADDRESS_FORWARDING`, `KC_HOSTNAME_PATH`, `KC_HOSTNAME_URL`, `KC_HOSTNAME_STRICT_HTTPS`, `KC_HOSTNAME_STRICT_BACKCHANNEL`) have no effect on Keycloak 26 and only trigger the "Hostname v1 options … still in use" warning. Replacements:

| v1 (no effect) | Keycloak 26 |
|---|---|
| `KC_PROXY=edge`, `PROXY_ADDRESS_FORWARDING=true` | `KC_PROXY_HEADERS=xforwarded` + `KC_HTTP_ENABLED=true` |
| `KC_HOSTNAME_PATH=/auth`, `KC_HOSTNAME_URL` | path as part of the `KC_HOSTNAME` URL; eegfaktura serves Keycloak without `/auth` |
| `KC_HOSTNAME_STRICT_HTTPS` | no direct replacement: the scheme is part of the `KC_HOSTNAME` URL |
| `KC_HOSTNAME_STRICT_BACKCHANNEL=true` | `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=false` (default; inverse meaning) |

The `eegfaktura-keycloak` image no longer sets any of the v1 options (since [eegfaktura-keycloak#21](https://github.com/eegfaktura/eegfaktura-keycloak/pull/21)) and deliberately does not set `KC_PROXY_HEADERS` (that would make directly reachable instances trust forwarded headers); deployments behind a proxy set it themselves.

## Build and image

Two-stage `Dockerfile`: the builder stage runs `kc.sh build` on `quay.io/keycloak/keycloak:26.7.4` with health, metrics and `KC_DB=postgres`; the final stage copies the built Keycloak plus `themes/` (the `eegfaktura-ui` login theme) and runs as `USER 1000`. `ENTRYPOINT` is `kc.sh`, `CMD` is `start`, so `args` / `command` replace only the subcommand. The realm is not baked in: it comes from `realm/EEGFaktura.yaml` (config-cli) or, in the docker-compose stack, from the realm import directory.

Images: CI pushes `ghcr.io/vfeeg-development/eegfaktura-keycloak` (used by the deployments); docker-compose uses the public-tier `ghcr.io/eegfaktura/eegfaktura-keycloak`.

## Related

- [Architecture / Authentication](../architecture/auth.md) — the full auth contract
