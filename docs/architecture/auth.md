# Authentication

All eegfaktura services authenticate against a single Keycloak realm. There is no separate auth gateway — each backend independently verifies the JWT against Keycloak's JWKS and enforces its own role / tenant checks.

## Keycloak realm

| Item | Value |
|------|-------|
| Realm name | `EEGFaktura` |
| Token signing | RS256 |
| Login flow | Authorization Code + PKCE (public clients) |

## Clients

The realm contains four clients, each serving a distinct consumer.

| Client ID | Type | Used by |
|-----------|------|---------|
| `at.ourproject.vfeeg.app` | public, PKCE | customer web SPA |
| `at.ourproject.vfeeg.api` | confidential | backend-side token introspection (where used) |
| `at.ourproject.vfeeg.admin` | public, PKCE | admin web SPA |
| `admin-cli` | service account | backend → Keycloak admin API calls |

## Groups

The realm defines four top-level groups. They are conveyed in the JWT via group-membership mappers.

| Group | Meaning | Where enforced |
|-------|---------|----------------|
| `EEG_ADMIN` | Full admin within the user's own tenant | backend middleware, billing `hasRole` |
| `EEG_OWNER` | Reserved for reporting (currently no code effect) | — |
| `EEG_USER` | Regular member | implicit (anything not admin) |
| `vfeeg-superuser` | Cross-tenant maintenance role | backend middleware, admin-web SPA, admin-backend |

## JWT structure

A token issued to a member typically looks like:

```json
{
  "preferred_username": "<username>",
  "email": "<member-email>",
  "sub": "<keycloak-user-uuid>",
  "access_groups": ["/EEG_USER"],
  "groups": ["/EEG_USER"],
  "tenant": ["<tenant-id>"],
  "iss": "https://<auth-domain>/realms/EEGFaktura",
  "aud": "at.ourproject.vfeeg.app",
  "exp": 1700000000
}
```

Notes:

- Group names appear with a leading slash (`/EEG_ADMIN`). Backends accept both forms.
- Two claim names carry the same group list: `access_groups` (Go backends, `PlatformClaims` struct) and `groups` (admin-web SPA). Two `oidc-group-membership-mapper` instances populate both, with identical values.
- `tenant` is a JSON array. A single-tenant user has one entry; a `vfeeg-superuser` has all tenants they can access. The mapper has `multivalued=true` and `aggregate.attrs=true`.
- `aud` is a string. Customer Go backends parse it as `string`, not array — see the "audience" caveat below.

## Required mappers on the customer client

`at.ourproject.vfeeg.app` needs these mappers in addition to the default scopes:

| Mapper | Type | Purpose |
|--------|------|---------|
| `tenantAttributeMapper` | oidc-usermodel-attribute-mapper | maps user attribute `tenant` to claim `tenant`; `multivalued=true`, `aggregate.attrs=true`, `jsonType.label=JSON` |
| `access_groups` mapper | oidc-group-membership-mapper | claim `access_groups`, `multivalued=true`, `full.path=true` |
| `groups` mapper | oidc-group-membership-mapper | claim `groups`, identical to above (consumed by admin-web) |

## Per-service auth check

Each backend enforces auth independently.

| Service | Mechanism | Critical input |
|---------|-----------|----------------|
| **backend** (Go) | JWT verify against JWKS, `ConditionProtect` / `AdminOnly` middleware | `access_groups` for role, `tenant` claim matched against route's tenant scope |
| **billing** (Java / Spring) | Spring Security JWT, `hasRole(EEG_ADMIN)` | requires `EEG_ADMIN` in groups; `Tenant` HTTP header validated |
| **energystore** (Go) | JWT verify, custom middleware | `X-Tenant` header must be in JWT `tenant` array |
| **filestore** (Python) | PyJWT verify | requires Keycloak's **public key** in PEM form (not the X.509 cert) |
| **admin-backend** (Scala / Pekko) | Pekko-HTTP JWT directive | `aud` array must contain `at.ourproject.vfeeg.admin` |

### "audience" caveat

The customer-side Go backends declare `aud` as `string`. If a user is granted client roles on a foreign client, Keycloak resolves additional audiences and `aud` becomes an array, breaking JSON parsing. Practical consequences:

- A customer user must not hold client roles on other clients.
- An admin user that needs both customer-side and admin-side access needs careful audience-mapper configuration.

In practice, separate users for customer-side and admin-side concerns avoid the issue.

## Member-data binding

For endpoints that return "own participant data" (a non-admin member viewing their dashboard), the backend matches the JWT's `email` or `preferred_username` against `base.participant.email` / `base.participant.id` in the database.

If no row matches, the member sees an empty UI. Provisioning a new `EEG_USER` therefore requires a Keycloak user **and** a `base.participant` row with the same email.

## Tenant scoping

- The customer SPA derives the active tenant from the JWT's `tenant` array and exposes a dropdown if more than one is present.
- API calls carry the tenant either as a path segment (`/eeg/v2/<ecId>/...`), an `X-Tenant` header (energystore), or a `Tenant` header (billing).
- Every backend rejects requests whose tenant header / path is not in the JWT's `tenant` array (typically HTTP 403).

The tenant value is the EEG's `communityId` (also called `ecId`), as stored in `base.eeg.community_id`. Provisioning must ensure the Keycloak user attribute matches that exact value.

## Provisioning a new user

The minimum steps to add a working user, by role:

### EEG_USER (regular member)

1. Create Keycloak user with `groups: ["EEG_USER"]`.
2. Set user attribute `tenant = '"<tenant-id>"'` (JSON-quoted).
3. Ensure `base.participant` row exists with matching `email` and `tenant = <tenant-id>`.

### EEG_ADMIN (community admin)

1. Create Keycloak user with `groups: ["EEG_ADMIN"]` (`EEG_OWNER` optional).
2. Set user attribute `tenant = '"<tenant-id>"'`.
3. `base.participant` row optional — admin does not have to be a member.

### vfeeg-superuser (cross-tenant)

1. Create Keycloak user with `groups: ["vfeeg-superuser", "EEG_ADMIN"]`. Both are needed: `vfeeg-superuser` for admin-web, `EEG_ADMIN` for billing's `hasRole` check.
2. Set one `tenant` user-attribute row per accessible tenant. With `aggregate.attrs=true` on the mapper, the JWT receives all of them as an array.
3. No `base.participant` row needed.

## Billing JWT signing cert

The billing service verifies JWTs using a PEM file at `/billing/zertifikat-pub.pem`, **not** by fetching JWKS at runtime. The cert must be Keycloak's current RS256 signing cert.

Rotation procedure:

1. Fetch the cert from Keycloak's JWKS endpoint and convert to PEM (full procedure: see [services/billing-cert-rotator](../services/billing-cert-rotator.md)).
2. Update the `ConfigMap` mounted into the billing pod at the path above.
3. Restart billing.

The `billing-cert-rotator` service automates this on a schedule.

## Filestore caveat

Filestore uses PyJWT, which does **not** accept an X.509 certificate as the verification key — it requires the raw public key. Extract with `openssl x509 -pubkey -noout`. PyJWT also enforces the `aud` claim even without an explicit `audience=` parameter.

## Related

- [Service Overview](overview.md) — overall topology
- [services/keycloak](../services/keycloak.md) — realm import details
- [Glossary](../reference/glossary.md) — EEG / tenant / Zählpunkt terminology
