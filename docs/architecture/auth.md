# Authentication

All eegfaktura services authenticate against a single Keycloak realm. There is no separate auth gateway — Keycloak signs every token (RS256), and each backend is an independent RS256 validator that enforces its own role / tenant checks. Services obtain the signing key either by fetching JWKS or from a configured public-key / cert file (see the per-service table below).

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
- The tenant claim key is `tenant` (lowercase, singular) and carries a JSON string array. A single-tenant user has one entry; a `vfeeg-superuser` has all tenants they can access. The mapper has `multivalued=true` and `aggregate.attrs=true`. Tenant comparison is case-insensitive across services.
- `aud` is resolved by Keycloak. The Go backends rely on their OIDC library (coreos/go-oidc) for audience handling and do not parse `aud` themselves; admin-backend explicitly requires `at.ourproject.vfeeg.admin` and filestore validates against `JWT_AUDIENCE` (default `account`).

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
| **backend** (Go) | RS256 verify via coreos/go-oidc v3 (issuer-side), `ConditionProtect` / `AdminOnly` middleware | authorization from `realm_access.roles` (admin/user/superuser); `tenant` claim (string array, case-insensitive) matched against route's tenant scope. `azp` is mapped to an internal field; `access_groups` is parsed but unused for authz |
| **billing** (Java / Spring) | RS256 via Auth0 java-jwt, reading an X.509 cert from `JWT_PUBLICKEYFILE` (converted to an `RSAPublicKey`), `hasRole(EEG_ADMIN)` | requires `EEG_ADMIN` in groups; `Tenant` HTTP header validated. No baked-in PEM — restart needed on cert change |
| **energystore** (Go) | RS256 via coreos/go-oidc + golang-jwt v4; key from `ENERGYSTORE_JWT_PUBKEYFILE`, Keycloak config via `KEYCLOAK_CONFIG` | `X-Tenant` header must be in JWT `tenant` array |
| **filestore** (Python) | RS256 via PyJWT; public key from `JWT_KEY_FILE` (env) → internal `JWT_PUBLIC_KEY_FILE` setting | requires Keycloak's **public key** in PEM form (not the X.509 cert); validates `aud` against `JWT_AUDIENCE` (default `account`) |
| **admin-backend** (Scala / Pekko) | Nimbus JOSE+JWT, Keycloak JWKS via `RemoteJWKSet` | `aud` must contain `at.ourproject.vfeeg.admin` |

### Audience handling

The customer-side Go backends do not parse `aud` themselves — audience handling is left to their OIDC library (coreos/go-oidc), which accepts both a single string and an array. Two services do enforce a specific audience explicitly: filestore validates `aud` against `JWT_AUDIENCE` (default `account`), and admin-backend requires `aud` to contain `at.ourproject.vfeeg.admin`.

Practical consequence:

- An admin user that needs both customer-side and admin-side access needs careful audience-mapper configuration so that the admin-side audience is present where admin-backend expects it.

In practice, separate users for customer-side and admin-side concerns keep the audience requirements clean.

## Machine-to-machine access (HTTP Basic / `ProtectApi`)

Alongside the RS256 bearer path above, two Go services expose a second middleware, `ProtectApi`, for non-interactive / server-to-server callers. Instead of validating a bearer JWT, `ProtectApi` reads an `Authorization: Basic base64(username:password)` header and performs the password exchange against Keycloak server-side (`AuthenticateUserWithPassword`, OIDC `grant_type=password`). It then applies the same tenant check on the resulting claims, so the caller never handles a JWT directly.

| Service | `ProtectApi` routes | Auth header | Tenant header |
|---------|---------------------|-------------|---------------|
| **backend** (Go) | `GET /master/masterdata`, `POST /master/updatepartfact` (`api/apiController.go`) | `Authorization: Basic …` | `X-Tenant` |
| **energystore** (Go) | `POST /query/rawdata`, `POST /query/{ecid}/metadata` (`rest/energy.go`) | `Authorization: Basic …` | `X-Tenant` |

Notes:

- The Basic credentials are an ordinary Keycloak username/password; the service exchanges them for a token internally. The user still needs the appropriate group / tenant for the operation.
- This path depends on the Keycloak client used for the server-side exchange permitting the Resource Owner Password Credentials grant ("Direct Access Grants"). Public SPA clients (`…app`, `…admin`) commonly have it disabled, in which case the operator must designate a client that allows the grant.
- Routes guarded by `Protect` (bearer-only) do **not** accept Basic. In particular the participant endpoints — `POST /participant`, `PUT /participant/{id}`, `POST /participant/{id}/confirm`, `DELETE /participant/v2/{id}` (`api/participantController.go`) — require a bearer JWT with `EEG_ADMIN`; a Basic request returns HTTP 403. There is currently no `ProtectApi` equivalent for participant writes.

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

The billing service verifies JWTs (Auth0 java-jwt) using an X.509 certificate read from the file named by `JWT_PUBLICKEYFILE`, which it converts to an `RSAPublicKey` — it does **not** fetch JWKS at runtime, and there is no baked-in PEM. The cert must be Keycloak's current RS256 signing cert, and billing must be restarted when the cert changes.

## Filestore caveat

Filestore uses PyJWT, which does **not** accept an X.509 certificate as the verification key — it requires the raw public key. Extract with `openssl x509 -pubkey -noout`. The key file is supplied via `JWT_KEY_FILE` (mapped to the internal `JWT_PUBLIC_KEY_FILE` setting). PyJWT also enforces the `aud` claim; filestore validates it against `JWT_AUDIENCE` (default `account`).

## Related

- [Service Overview](overview.md) — overall topology
- [services/keycloak](../services/keycloak.md) — realm import details
- [Glossary](../reference/glossary.md) — EEG / tenant / Zählpunkt terminology
