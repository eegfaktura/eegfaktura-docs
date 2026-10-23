# web

The customer-facing single-page application. Served by Caddy from a built static bundle. Authenticates against Keycloak via PKCE and talks to four different backends.

## At a glance

| | |
|---|---|
| Language | TypeScript |
| Framework | React 18 + Ionic 7, built with Vite 6 |
| Server | Caddy |
| Auth | OIDC Authorization Code + PKCE |
| Backends consumed | backend, billing, energystore, filestore |

## Audience

Two role categories meet here:

| Role | Sees |
|------|------|
| `EEG_USER` | own participant data, own energy reports, own documents |
| `EEG_ADMIN` | EEG-wide data: all members, all metering points, billing actions |

The SPA reads the groups from the `access_groups` claim (`/EEG_ADMIN`, `/EEG_USER`, `/EEG_OWNER`). A user with the realm role `superuser` passes the tenant checks of backend, energystore, billing and filestore and gets **all** EEGs in the community dropdown (`GET /api/eeg/user/get-user`); every other user gets the EEGs in their `tenant` claim.

## UI structure

Left navigation:

| Item | Purpose |
|------|---------|
| **Messages** (`/page/notifications`) | notifications from the backend (`GET /api/eeg/notifications/{start}`), e.g. EDA process outcomes and import reports; reloaded on every visit and on community switch |
| **Dashboard** (`/page/dashboard`) | energy balance + finances for the current billing period |
| **`<EEG name>`** (`/page/eeg`) | EEG master data and settings |
| **Mitglieder** (`/page/participants`, default route) | member list + detail view |
| **Tarife** (`/page/rates`) | member fees, consumer and producer tariffs |
| **Prozesse** (`/page/processes`) | EDA processes; shown only to `EEG_ADMIN` and only for online (EDA-connected) EEGs |

The "Auswahl Gemeinschaft" dropdown in the user popover lets a multi-tenant user switch between EEGs (the `tenant` claim is an array). The selected tenant is kept in `localStorage` (`tenant`).

## Dashboard semantics

The dashboard uses a two-color encoding consistently across charts and bars:

- **EEG (green)** — energy drawn from / delivered to the EEG's own production
- **EVU (purple)** — energy drawn from / delivered to the regular electricity supplier

Tile cards:

- "Übersicht EEG":
    - "Mitglieder" — active / inactive (plus "in Anmeldung", if any), counted from the current state of each member's metering points: active = at least one active metering point; inactive = all left, archived, rejected or moved to another member
    - "Zählpunkte" — count split into "Abnehmer" (consumer) / "Einspeiser" (producer); the "inaktive" figure is a hard-coded `0`
    - "Tarife" — number of metering points per tariff
- "Übersicht Stromproduktion und Verteilung" — "Produziert", "Verteilt", "EVU", each with the previous period for comparison
- "Energiemengen" — "Eingekauft" / "Verkauft"

## Backend wiring

The customer SPA addresses four backends with distinct conventions (prefixes are the `VITE_*` values from `.env.production`):

| Backend | URL prefix | Tenant header | Auth caveat |
|---------|------------|---------------|-------------|
| backend (Go) | `/api/...` (e.g. `/api/eeg`, `/api/participant`, `/api/meteringpoint/...`) | `tenant` = selected tenant | header must be in the `tenant` claim (case-insensitive), unless realm role `superuser` |
| billing | `/cash/api/...` | `tenant` = selected tenant | every request requires `EEG_ADMIN` |
| energystore | `/energystore/eeg/v2/<ecId>/...`, `/energystore/query` | `X-Tenant` = the EEG's **rcNumber** | header must be in the `tenant` claim, unless `superuser` |
| filestore | `/filestore/graphql`, `/filestore/filestore/<fileId>` | `tenant` | PyJWT — see service page |

Caddy in the customer-web container does **not** proxy these. The Ingress (typically NGINX with `rewrite-target: /$2`) handles the routing in cluster.

## Billing run (since 1.8.0)

- The run starts asynchronously: `POST /cash/api/billing` answers `202` (or `409` if a run is already active) with the run id; the SPA polls `GET /cash/api/billingRuns/{id}` every 2 s until `DONE`/`NEW` (preview) or `FAILED`, and shows the run's persisted `errorSummary` on failure. Opening the billing panel while a run is `RUNNING` attaches to that run.
- Time-of-use tariffs (ZVT): for tariffs in mode "Zeitbasiert" the SPA sends the tariff time windows with the energystore report request, receives per-window sums (`buckets`) per metering point, and forwards buckets plus the windows echoed by energystore to billing. Simple tariffs never carry buckets.

## Runtime configuration

The SPA fetches `/config/keycloak-config.json` at startup; Caddy serves it from `/srv` with template processing (keys under `app`: `auth-server-url`, `realm`, `resource` — the `resource` is the client id, default `at.ourproject.vfeeg.app`). The template fills them from the container env vars `KEYCLOAK_URL`, `KEYCLOAK_REALM` and `KEYCLOAK_CLIENT_ID` (local defaults when unset); `src/main.tsx` builds the OIDC authority as `<auth-server-url>/realms/<realm>/`.

Backend service URLs are baked in at build time via `VITE_API_SERVER_URL`, `VITE_ENERGY_SERVER_URL`, `VITE_BILLING_SERVER_URL`, and `VITE_FILESTORE_SERVER_URL` (`.env.production`: `/api`, `/energystore`, `/cash`, `/filestore`).

For local development, `pnpm run dev` starts Vite on port **5173** with proxies `/api`→9080, `/energystore`→9081, `/cash`→9095.

## Auth flow

1. SPA loads from Caddy.
2. Unauthenticated → redirect to Keycloak (`at.ourproject.vfeeg.app` client, PKCE).
3. On callback, the SPA receives the JWT.
4. The SPA parses `tenant` (array), `access_groups`, `realm_access.roles`, `email`.
5. The selected tenant goes into the `tenant` header. After the EEG is loaded, `ecId` (= `base.eeg."communityId"`) scopes the energystore URLs and `rcNr` (= `base.eeg."rcNumber"`) is sent to energystore as `X-Tenant`.

If `tenant` is empty for a non-superuser (the user attribute was forgotten in Keycloak), the EEG dropdown is empty and the dashboard cannot load.

## `tenant` vs `ecId`

The customer SPA uses three identifiers: the selected **tenant** (from the JWT `tenant` claim, matching `base.eeg.tenant`) for backend, billing and filestore; `ecId` (= `"communityId"`) in the energystore URLs; and `rcNr` (= `"rcNumber"`) as the energystore `X-Tenant` header, which energystore also uses as its storage key.

The trap: where an EEG's tenant differs from its rcNumber (e.g. GEA tenants `GCxxxxxx-00n` with rcNumber `GCxxxxxx`), the claim holds the tenant but not the rcNumber. Backend calls succeed, every energystore call is rejected (403 on REST, 401 on GraphQL), and the dashboard stays empty. Superusers do not see it, because they bypass the tenant check. For such an EEG the user's `tenant` attribute must also contain the rcNumber.

## Form caveats from operational experience

- **`InputForm.reset-on-invalid` anti-pattern** — wiping the entire form on an invalid-dirty state via `control._reset()` was a recurring source of data loss. Searches for `_reset` / `reset()` inside form-wrapper components should ask "why".
- **Register-page reset+history race** — calling `reset()` and `history.replace()` synchronously after a fire-and-forget dispatch loses backend failures. Use `dispatch().unwrap()` with try/catch, and run side-effects only on the success branch.

## Build and image

- Source: `eegfaktura-web`
- Stack (resolved versions from `pnpm-lock.yaml`): React 18.3 (`^18.2.0`), `@ionic/react` + `@ionic/core` 7.x, Vite 6 with `@vitejs/plugin-react` 4.7 and `@vitejs/plugin-legacy` 6.1, TypeScript 5.6, React Router 5.3.4, Redux Toolkit 2, `oidc-client-ts` ^3.0.0 + `react-oidc-context` ^3.0.0, recharts 2.x (charts), i18next, react-hook-form, xlsx, xmlbuilder2. Tests: vitest 3 + Testing Library (jsdom). Build-time i18n via `vite-plugin-i18next-loader` (`locales/`).
- Package manager: **pnpm only**, version pinned in `package.json` (`packageManager: pnpm@9.12.1`); CI installs with `--frozen-lockfile`. Node 20.18.0 (`.nvmrc`, used by CI via `setup-node`).
- Build: `pnpm run build` = `tsc && vite build`, produces a static `dist/`. Since 1.8.0 the release image is built this way, so a type error stops the release.
- Runtime image: `caddy:2.8.4` serving the static bundle on port **8080** from `/var/www/html/vfeeg-web/`, running as the non-root user `app` (UID/GID 1000). The Dockerfile only copies the prebuilt `dist/`; the SPA is built on the CI runner.
- Tag scheme (docker/metadata-action): `latest` on the default branch, branch/PR refs, semver tags, and `sha-<short>`.
- **Build Node version**: keep Node 20. A Node 22 build produced a vendor chunk that crashed at boot with `Object.defineProperty called on non-object` inside the `@oozcitak/dom` init graph (xmlbuilder2 dependency). The deeper root cause has not been isolated.

## Caddy security headers

The `caddy.conf` shipped in the image sets **no** security headers. A `header` block would look like this:

```caddy
header {
  Content-Security-Policy "..."
  Strict-Transport-Security "max-age=31536000; includeSubDomains"
  X-Frame-Options "DENY"
  X-Content-Type-Options "nosniff"
  Referrer-Policy "strict-origin-when-cross-origin"
  Permissions-Policy "..."
}
```

The exact CSP depends on which third-party assets the SPA loads.

## AGPL § 13 footer

Since 1.8.0 the Info/Impressum footer names the licence (AGPL-3.0) and links the source code of all components (github.com/eegfaktura), as AGPL-3.0 § 13 asks for a network service.

## Related

- [Architecture / Authentication](../architecture/auth.md) — full JWT flow
- [services/backend](backend.md) — primary API
- [services/billing](billing.md) — billing actions
- [services/energystore](energystore.md) — energy reports
- [reference/glossary](../reference/glossary.md) — dashboard terms
