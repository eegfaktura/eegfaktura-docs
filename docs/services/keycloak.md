# keycloak

OIDC identity provider for the entire platform. One realm, four clients, four groups.

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-keycloak:latest` |
| Realm | `EEGFaktura` |
| Token signing | RS256 |
| State | PostgreSQL database `keycloak` |

In the local docker-compose stack Keycloak runs the custom `ghcr.io/eegfaktura/eegfaktura-keycloak:latest` image, started with `start --optimized --import-realm`. The realm (`EEGFaktura`) is imported from `./keycloak/import`. It publishes the main port `8080` and the management port `9181` (`KC_HTTP_MANAGEMENT_PORT`), and runs with `KC_HOSTNAME_STRICT=false` for local use. Keycloak is reached directly there, so no proxy headers are trusted. The compose file still sets the Keycloak-v1 options `KC_PROXY=edge`, `PROXY_ADDRESS_FORWARDING` and `KC_HOSTNAME_STRICT_BACKCHANNEL`; Keycloak 26 ignores them and only logs "Hostname v1 options … still in use".

For the auth contract (clients, mappers, JWT claims, role behavior), see [Architecture / Authentication](../architecture/auth.md). This page covers the Keycloak realm content.

## Realm content

| Item | Detail |
|------|--------|
| Realm name | `EEGFaktura` |
| Login theme | (configurable; the platform ships a default) |
| Groups | `EEG_ADMIN`, `EEG_OWNER`, `EEG_USER`, `vfeeg-superuser` |
| Clients | `at.ourproject.vfeeg.app`, `at.ourproject.vfeeg.api`, `at.ourproject.vfeeg.admin`, `admin-cli` |

### Required mappers on `at.ourproject.vfeeg.app`

| Mapper | Type | Notes |
|--------|------|-------|
| `tenantAttributeMapper` | oidc-usermodel-attribute-mapper | maps user attribute `tenant` to claim `tenant`; multivalued; aggregate.attrs=true; jsonType.label=JSON |
| `access_groups` | oidc-group-membership-mapper | claim `access_groups`, multivalued, full.path=true |
| `groups` | oidc-group-membership-mapper | claim `groups`, identical to above |

The two group mappers are required because customer-side Go backends read `access_groups` while admin-web reads `groups`. Omitting either breaks one of the SPAs.

## Database

Keycloak owns its own logical database (typically named `keycloak`) in the shared PostgreSQL cluster. Schema is managed entirely by Keycloak; do not migrate or modify directly.

The largest table is `offline_user_session` in long-running instances.

## User-attribute cache quirk (Keycloak 25)

Keycloak 25 caches `cannot map type for token claim` failures per user. If a `tenant` attribute is provisioned in a format the mapper cannot parse (e.g. unquoted instead of JSON-quoted), the user can be stuck with a permanent token-without-tenant state even after the attribute is fixed.

Recovery:

1. `DELETE` and `INSERT` the `user_attribute` row.
2. Restart the Keycloak pod.

A simple admin-API `PUT` does not always clear the cache.

## Realm-frontend-URL

When Keycloak runs behind an ingress, the realm setting `frontendUrl` must match the public URL, otherwise login redirects target the in-cluster service DNS and the browser cannot follow them.

## Config

| Variable | Purpose |
|----------|---------|
| `KC_DB_*` | database connection |
| `KC_HOSTNAME` | public URL incl. scheme, e.g. `https://<auth-domain>` (no `/auth` path) |
| `KC_HTTP_ENABLED` | `true` when TLS terminates at the ingress / reverse proxy |
| `KC_PROXY_HEADERS` | `xforwarded` when behind an ingress / reverse proxy that sets `X-Forwarded-*`; leave unset when Keycloak is reached directly |
| JVM heap | tune; defaults are large |

The Keycloak-v1 hostname/proxy options (`KC_PROXY`, `PROXY_ADDRESS_FORWARDING`, `KC_HOSTNAME_PATH`, `KC_HOSTNAME_URL`, `KC_HOSTNAME_STRICT_HTTPS`, `KC_HOSTNAME_STRICT_BACKCHANNEL`) have no effect on Keycloak 26 and only trigger the "Hostname v1 options … still in use" warning. Replacements:

| v1 (no effect) | Keycloak 26 |
|---|---|
| `KC_PROXY=edge`, `PROXY_ADDRESS_FORWARDING=true` | `KC_PROXY_HEADERS=xforwarded` + `KC_HTTP_ENABLED=true` |
| `KC_HOSTNAME_PATH=/auth`, `KC_HOSTNAME_URL` | path as part of the `KC_HOSTNAME` URL; eegfaktura serves Keycloak without `/auth` |
| `KC_HOSTNAME_STRICT_HTTPS` | no direct replacement: the scheme is part of the `KC_HOSTNAME` URL |
| `KC_HOSTNAME_STRICT_BACKCHANNEL=true` | `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=false` (default; inverse meaning) |

The `eegfaktura-keycloak` image no longer sets any of the v1 options (since [eegfaktura-keycloak#21](https://github.com/eegfaktura/eegfaktura-keycloak/pull/21)) and deliberately does not set `KC_PROXY_HEADERS` (that would make directly reachable instances trust forwarded headers); deployments behind a proxy set it themselves.

## Build and image

The local docker-compose stack uses a custom image, `ghcr.io/eegfaktura/eegfaktura-keycloak:latest`, with the realm and any themes / SPIs baked in. Production may use a different image (e.g. an upstream Keycloak base with custom themes / SPIs layered in a downstream build).

## Related

- [Architecture / Authentication](../architecture/auth.md) — the full auth contract
