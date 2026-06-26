# keycloak

OIDC identity provider for the entire platform. One realm, four clients, four groups.

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-keycloak:latest` |
| Realm | `EEGFaktura` |
| Token signing | RS256 |
| State | PostgreSQL database `keycloak` |

In the local docker-compose stack Keycloak runs the custom `ghcr.io/eegfaktura/eegfaktura-keycloak:latest` image, started with `start --optimized --import-realm`. The realm (`EEGFaktura`) is imported from `./keycloak/import`. It publishes the main port `8080` and the management port `9181` (`KC_HTTP_MANAGEMENT_PORT`), and runs with `KC_PROXY=edge` and `KC_HOSTNAME_STRICT=false` for local use.

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
| `KC_HOSTNAME` | public hostname |
| `KC_HTTP_ENABLED` | `true` when behind a TLS-terminating ingress |
| `KC_PROXY` | `edge` when behind an ingress |
| JVM heap | tune; defaults are large |

## Build and image

The local docker-compose stack uses a custom image, `ghcr.io/eegfaktura/eegfaktura-keycloak:latest`, with the realm and any themes / SPIs baked in. Production may use a different image (e.g. an upstream Keycloak base with custom themes / SPIs layered in a downstream build).

## Operational notes

- Realm re-import overwrites the realm. Any user-side changes (extra users, manually-set attributes) are wiped. The `kc-users` Job re-creates the bootstrap users; everything else must be re-applied externally.
- On Keycloak key rotation, the [billing-cert-rotator](billing-cert-rotator.md) must run before billing reads the new key, otherwise tokens fail verification.
- The Keycloak pod's restart loop on a fresh deployment is usually attributable to the database not being ready. The `initContainer` pattern on KC-dependent Jobs (`curl /realms/master`) is the right wait probe; the KC pod itself can take a minute or two to be reachable after PG comes up.

## Related

- [Architecture / Authentication](../architecture/auth.md) — the full auth contract
- [services/billing-cert-rotator](billing-cert-rotator.md) — JWT cert refresh
