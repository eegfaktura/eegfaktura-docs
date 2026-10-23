# Authentication

All eegfaktura services authenticate against a single Keycloak realm. There is no separate auth gateway — Keycloak signs every token (RS256), and each backend is an independent validator that enforces its own role / tenant checks. Services obtain the signing key either by fetching JWKS or from a configured public-key / cert file (see the per-service table below).

## Keycloak realm

The realm is declared in `realm/EEGFaktura.yaml` (eegfaktura-keycloak, applied with keycloak-config-cli).

| Item | Value |
|------|-------|
| Realm name | `EEGFaktura` |
| Token signing | RS256 |
| Login flow | Authorization Code + PKCE (public clients, via `oidc-client-ts` in both SPAs) |
| Realm roles | `superuser` (cross-tenant operator), `Manager`, `user` (plus Keycloak defaults) |
| Brute-force protection | on: temporary lockout after 10 failures, growing up to 15 min, no permanent lockout |
| Events | login/logout, password, profile, e-mail and credential changes, temporary lockouts; admin events without content; 90 days each |
| User profile | `unmanagedAttributePolicy: ADMIN_EDIT` — only admins can see and set the `tenant` attribute |

## Clients

The realm config declares three clients; admin-backend additionally uses Keycloak's built-in `admin-cli`.

| Client ID | Type | Used by |
|-----------|------|---------|
| `at.ourproject.vfeeg.app` | public, PKCE, direct access grants off | customer web SPA |
| `at.ourproject.vfeeg.api` | confidential, direct access grants on | backend and energystore: server-side password exchange for `ProtectApi` (the `api` entry of their `KEYCLOAK_CONFIG` file) |
| `at.ourproject.vfeeg.admin` | public, PKCE, with an `oidc-audience-mapper` that puts `at.ourproject.vfeeg.admin` into `aud` | admin web SPA |
| `admin-cli` | Keycloak built-in, client-credentials grant (not part of `EEGFaktura.yaml`) | admin-backend → Keycloak admin API (`KEYCLOAK_ADMIN_CLI_AUDIENCE`, `KEYCLOAK_ADMIN_CLI_SECRET`) |

## Groups and roles

The realm defines three top-level groups. They are conveyed in the JWT claim `access_groups` by a group-membership mapper (full path, so with a leading slash). Cross-tenant access is the realm **role** `superuser` (claim `realm_access.roles`), not a group.

| Group / role | Meaning | Where enforced |
|-------|---------|----------------|
| `EEG_ADMIN` (group) | Full admin within the user's own tenant(s) | backend `Protect` / `ConditionProtect` / `ProtectApi` (exact `/EEG_ADMIN`), billing `hasRole(EEG_ADMIN)`, web UI |
| `EEG_OWNER` (group) | Reserved (currently no code effect); admin-backend assigns it together with `EEG_ADMIN` to new community logins | — |
| `EEG_USER` (group) | Regular member | backend `ConditionProtect` user branch (exact `/EEG_USER`) |
| `superuser` (realm role) | Cross-tenant operator | skips the tenant check in backend, energystore, billing and filestore; required on all admin-backend routes and on energystore's raw-data delete; admin-web shows its UI only with this role |

## JWT structure

A token issued to a member through `at.ourproject.vfeeg.app` typically looks like:

```json
{
  "preferred_username": "<username>",
  "email": "<member-email>",
  "sub": "<keycloak-user-uuid>",
  "access_groups": ["/EEG_USER"],
  "realm_access": {"roles": ["user", "..."]},
  "tenant": ["<tenant-id>"],
  "iss": "https://<auth-domain>/realms/EEGFaktura",
  "azp": "at.ourproject.vfeeg.app",
  "exp": 1700000000
}
```

Notes:

- Group names appear with a leading slash (`/EEG_ADMIN`). backend and web compare the slash form exactly; billing strips the slash and maps each group to `ROLE_<group>`.
- The tenant claim key is `tenant` (lowercase, singular) and carries a JSON string array. The `at.ourproject.vfeeg.app` mapper has `multivalued=true`, `aggregate.attrs=true`, `jsonType.label=JSON`. Tenant comparison is case-insensitive in every service (backend/energystore upper-case both sides, billing `equalsIgnoreCase`, filestore upper-cases the GEA stem tenant, see below).
- The `at.ourproject.vfeeg.app` client deliberately has **no** audience mapper. backend's claim struct embeds golang-jwt v3 `StandardClaims`, whose `aud` is a plain string: a token whose `aud` is an array fails to parse there (401). energystore uses golang-jwt v4 `RegisteredClaims` and accepts both forms.

## Required mappers on the customer client

`at.ourproject.vfeeg.app` has these mappers in addition to the default scopes (`web-origins`, `roles`, `profile`, `basic`, `email`):

| Mapper | Type | Purpose |
|--------|------|---------|
| `tenantAttributeMapper` | oidc-usermodel-attribute-mapper | maps user attribute `tenant` to claim `tenant`; `multivalued=true`, `aggregate.attrs=true`, `jsonType.label=JSON` |
| `Group Mappers` | oidc-group-membership-mapper | claim `access_groups`, `multivalued=true`, `full.path=true` |

`at.ourproject.vfeeg.api` has its own tenant mapper (`tenantIds`) that is **single-valued** (`multivalued=false`, `jsonType.label=String`). `at.ourproject.vfeeg.admin` has only the audience mapper.

## Per-service auth check

Each backend enforces auth independently.

| Service | Mechanism | iss / aud / exp | Critical input |
|---------|-----------|-----------------|----------------|
| **backend** (Go) | coreos/go-oidc v3 verifier, keys via OIDC discovery/JWKS; Keycloak settings from the `KEYCLOAK_CONFIG` file (default `./keycloak.json`, entries `app` and `api`). `Protect` / `ConditionProtect` / `GQLProtect` middleware | `iss` must match the provider URL; `aud` not checked (`SkipClientIDCheck: true`); `exp` required (go-oidc rejects a missing/expired `exp`); asymmetric algorithms only (go-oidc rejects `none`/HMAC) | tenant from header `tenant` (fallback `X-Tenant`) must be in the JWT `tenant` array unless the token has realm role `superuser`; `access_groups` decides admin (`/EEG_ADMIN`) vs. user (`/EEG_USER`). On `/query` (GraphQL) a token without `EEG_ADMIN` is currently only logged (`gql-admin-check`), not rejected. `azp` is read only for that log line |
| **billing** (Java / Spring) | Auth0 java-jwt, `Algorithm.RSA256` with an X.509 cert read from `JWT_PUBLICKEYFILE` (converted to an `RSAPublicKey`); `anyRequest().hasRole(EEG_ADMIN)` (`/` and `/swagger-ui` permitted) | no `iss`/`aud` check; `exp` checked when present (60 s leeway), not required | requires `EEG_ADMIN` in `access_groups`; `Tenant` HTTP header must be in the JWT `tenant` array (case-insensitive) unless realm role `superuser`; resources additionally compare path/record tenant with the header (`TenantContext.validateTenant`). Profile `dev` disables security |
| **energystore** (Go) | coreos/go-oidc v3 verifier via `KEYCLOAK_CONFIG` (same file format as backend), `ProtectApp` / `GQLProtect` middleware. The golang-jwt v4 `JWTMiddleware` that reads `jwt.pubKeyFile` (`ENERGYSTORE_JWT_PUBKEYFILE`) is not wired to any route | as backend (`SkipClientIDCheck: true`) | header `tenant` (fallback `X-Tenant`) must be in the JWT `tenant` array unless realm role `superuser`; GraphQL resolvers compare their `tenant` argument with the checked header (`tenantctx`) |
| **filestore** (Python) | PyJWT, `algorithms=["RS256"]`; public key from `JWT_KEY_FILE` (env) → internal `JWT_PUBLIC_KEY_FILE` setting, loaded once at start | `aud` must equal `JWT_AUDIENCE` (default `account`); `exp` required (`options={"require": ["exp"]}`); `iss` not checked | requires Keycloak's **public key** in PEM form (not the X.509 cert); `assert_tenant` in each resolver after loading the resource, `superuser` exempt |
| **admin-backend** (Scala / Pekko) | Nimbus JOSE+JWT, RS256 key selector over Keycloak JWKS (`RemoteJWKSet`, URL from `KEYCLOAK_URL`/`KEYCLOAK_REALM` or `KEYCLOAK_AUTHENTICATOR_JWKS_URL`) | `iss` must equal `<KEYCLOAK_URL>/realms/<KEYCLOAK_REALM>` (case-insensitive); `aud` must contain `KEYCLOAK_AUDIENCE` (set to the admin SPA client id, `at.ourproject.vfeeg.admin`); `exp` required | realm role `superuser` on all routes (`/eeg`, `/vfeeg`, `/admin`), otherwise 403 `{"error":"superuser role required"}` |
| **eda-xp** (Scala / Pekko) | no JWT validation | — | talks to the other services via MQTT; its HTTP routes (`/pontonxp/...`, `/admin/upload`) and gRPC admin server carry no bearer check |

### Audience handling

The Go services (backend, energystore) switch the audience check off (`SkipClientIDCheck: true`). Two services enforce an audience: filestore validates `aud` against `JWT_AUDIENCE` (default `account`), and admin-backend requires `aud` to contain `KEYCLOAK_AUDIENCE`, which the admin client's audience mapper supplies (`at.ourproject.vfeeg.admin`). billing checks no audience.

Practical consequence:

- A token from `at.ourproject.vfeeg.admin` carries an `aud` array once more than one audience is present; backend cannot parse such a token (see [JWT structure](#jwt-structure)). Do not add an audience mapper to `at.ourproject.vfeeg.app`.

In practice, separate users for customer-side and admin-side concerns keep the audience requirements clean.

## Machine-to-machine access (HTTP Basic / `ProtectApi`)

Alongside the RS256 bearer path above, two Go services expose a second middleware, `ProtectApi`, for non-interactive / server-to-server callers. Instead of validating a bearer JWT, `ProtectApi` reads an `Authorization: Basic base64(username:password)` header and performs the password exchange against Keycloak server-side (`AuthenticateUserWithPassword`, OIDC `grant_type=password`, client from the `api` entry of `KEYCLOAK_CONFIG`). It verifies the returned ID token and applies the tenant check on its claims, so the caller never handles a JWT directly.

| Service | `ProtectApi` routes | Auth header | Tenant header |
|---------|---------------------|-------------|---------------|
| **backend** (Go) | `GET /master/masterdata`, `POST /master/updatepartfact` (`api/apiController.go`) | `Authorization: Basic …` | `X-Tenant` |
| **energystore** (Go) | `POST /query/rawdata`, `POST /query/{ecid}/metadata` (`rest/energy.go`) | `Authorization: Basic …` | `X-Tenant` |

Notes:

- The Basic credentials are an ordinary Keycloak username/password; the service exchanges them for a token internally. backend additionally requires `/EEG_ADMIN` in `access_groups`; energystore checks only the tenant. Neither applies the `superuser` exemption on this path.
- The exchange uses `at.ourproject.vfeeg.api`, the realm client with Direct Access Grants enabled. The public SPA clients (`…app`, `…admin`) do not allow that grant.
- energystore decodes the Basic value with URL-safe base64 (`base64.URLEncoding`), backend with standard base64.
- Routes guarded by `Protect` (bearer-only) do **not** accept Basic. In particular the participant endpoints — `POST /participant`, `PUT /participant/{id}`, `POST /participant/{id}/confirm`, `PUT /participant/v2/{id}`, `DELETE /participant/v2/{id}` (`api/participantController.go`) — require a bearer JWT with `EEG_ADMIN`; a Basic request returns HTTP 403. There is currently no `ProtectApi` equivalent for participant writes.

## Member-data binding

For endpoints that return "own participant data" (a non-admin member viewing their dashboard, `GET /participant` user branch), the backend matches the JWT's `email` case-insensitively against `base.contactdetail.email` of the participants in the request's tenant.

If no row matches, the member sees an empty UI. Provisioning a new `EEG_USER` therefore requires a Keycloak user **and** a participant in that tenant whose contact detail carries the same e-mail.

## Tenant scoping

- The customer SPA picks the active tenant from the JWT's `tenant` array (last choice kept in `localStorage`); the dropdown lists the communities returned by backend `GET /user/get-user` (all communities for a `superuser`).
- API calls carry the tenant in a header: `tenant` (backend, energystore; `X-Tenant` is accepted as fallback) or `Tenant` (billing). energystore URLs additionally contain the community id (`/eeg/v2/<ecId>/...`), which is not the value checked against the token.
- Every backend rejects requests whose tenant header is not in the JWT's `tenant` array (HTTP 401 or 403 depending on service and route), except for tokens with realm role `superuser`.

The tenant value is `base.eeg.tenant` (the primary key of `base.eeg`, usually the RC number), not the EDA community id (`base.eeg."communityId"`). GEA sub-communities have tenants of the form `GCxxxxxx-00n`; filestore stores their documents under the stem (`GCxxxxxx`) and compares the stem.

## Provisioning a new user

The minimum steps to add a working user, by role (admin-backend does steps 1–2 for new community logins):

### EEG_USER (regular member)

1. Create Keycloak user in group `EEG_USER`.
2. Set user attribute `tenant` to the tenant as JSON (a JSON string `"<tenant-id>"` or a JSON list `["<tenant-id>"]`; admin-backend writes the list form).
3. Ensure the participant exists in that tenant with a contact detail carrying the same e-mail.

### EEG_ADMIN (community admin)

1. Create Keycloak user in group `EEG_ADMIN` (admin-backend also adds `EEG_OWNER`).
2. Set user attribute `tenant` as above.
3. Participant row optional — admin does not have to be a member.

### superuser (cross-tenant)

1. Assign the realm role `superuser` and the group `EEG_ADMIN`. Both are needed: `superuser` skips the tenant checks and opens admin-web/admin-backend, `EEG_ADMIN` is still required by billing's `hasRole` check and backend's admin routes.
2. Set at least one `tenant` value: the customer SPA activates a tenant from the JWT's `tenant` array and stays empty without one. Further communities come from the dropdown.
3. No participant row needed.

## Billing JWT signing cert

The billing service verifies JWTs (Auth0 java-jwt) using an X.509 certificate read from the file named by `JWT_PUBLICKEYFILE` (default `/billing/zertifikat-pub.pem`), which it converts to an `RSAPublicKey` — it does **not** fetch JWKS at runtime. The cert must be Keycloak's current RS256 signing cert. The file is read on every token validation, so a replaced file takes effect without a restart.

## Filestore caveat

Filestore uses PyJWT, which does **not** accept an X.509 certificate as the verification key — it requires the raw public key. Extract with `openssl x509 -pubkey -noout`. The key file is supplied via `JWT_KEY_FILE` (mapped to the internal `JWT_PUBLIC_KEY_FILE` setting) and read once at start. PyJWT also enforces the `aud` claim; filestore validates it against `JWT_AUDIENCE` (default `account`).

## Related

- [Service Overview](overview.md) — overall topology
- [services/keycloak](../services/keycloak.md) — realm import details
- [Glossary](../reference/glossary.md) — EEG / tenant / Zählpunkt terminology
