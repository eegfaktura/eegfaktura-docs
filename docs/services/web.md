# web

The customer-facing single-page application. Served by Caddy from a built static bundle. Authenticates against Keycloak via PKCE and talks to four different backends.

## At a glance

| | |
|---|---|
| Language | TypeScript |
| Framework | React |
| Server | Caddy |
| Auth | OIDC Authorization Code + PKCE |
| Backends consumed | backend, billing, energystore, filestore |

## Audience

Two role categories meet here:

| Role | Sees |
|------|------|
| `EEG_USER` | own participant data, own energy reports, own documents |
| `EEG_ADMIN` | EEG-wide data: all members, all metering points, billing actions |

A `vfeeg-superuser` sees data for all tenants in their `tenant` claim and can switch between them via a dropdown.

## UI structure

Left navigation:

| Item | Purpose |
|------|---------|
| **Messages** | last EDA messages (only when PONTON is wired; routine energy data is filtered out). Diagnostic. |
| **Dashboard** | energy balance + finances for the current billing period |
| **`<EEG name>`** | EEG master data and settings |
| **Mitglieder** | member list + detail view |
| **Tarife** | member fees, consumer and producer tariffs |

The EEG dropdown in the settings menu lets a multi-tenant user switch between EEGs they belong to (the `tenant` claim is an array).

## Dashboard semantics

The dashboard uses a two-color encoding consistently across charts and bars:

- **EEG (green)** — energy drawn from / delivered to the EEG's own production
- **EVU (purple)** — energy drawn from / delivered to the regular electricity supplier

Tile cards:

- "Mitglieder" — active / inactive as of the reference date
- "Zählpunkte" — active / inactive per billing period, split into consumer / producer
- "Tarifzuordnung", "Produktion und Verteilung", "Finanzen: Eingekauft / Verkauft", "Marge EEG"

`Marge EEG` is roughly `Verkaufseinnahmen - Einkaufskosten`.

## Backend wiring

The customer SPA addresses four backends with distinct conventions:

| Backend | URL prefix | Auth caveat |
|---------|------------|-------------|
| backend (Go) | `/api/eeg/v2/<ecId>/...` | `tenant` claim must contain `ecId` |
| billing | `/cash/...` | `Tenant` header + EEG_ADMIN required |
| energystore | `/energystore/...` | `X-Tenant` header |
| filestore | `/filestore/...` | PyJWT — see service page |

Caddy in the customer-web container does **not** proxy these. The Ingress (typically NGINX with `rewrite-target: /$2`) handles the routing in cluster.

## Runtime configuration

The SPA fetches `/config/keycloak-config.json` at startup; Caddy serves it from `/srv` with template processing (keys `auth-server-url`, `realm`, `resource` — the `resource` is the client id `at.ourproject.vfeeg.app`). In cluster the file is overridden by a mounted ConfigMap.

Backend service URLs are baked in at build time via `VITE_API_SERVER_URL`, `VITE_ENERGY_SERVER_URL`, `VITE_BILLING_SERVER_URL`, and `VITE_FILESTORE_SERVER_URL`.

For local development, `pnpm run dev` starts Vite on port **5173** with proxies `/api`→9080, `/energystore`→9081, `/cash`→9095.

## Auth flow

1. SPA loads from Caddy.
2. Unauthenticated → redirect to Keycloak (`at.ourproject.vfeeg.app` client, PKCE).
3. On callback, the SPA receives the JWT.
4. The SPA parses `tenant` (array), `access_groups`, `email`.
5. `tenant.ecId = base.eeg.community_id` is used to scope API URLs.

If `tenant` is empty (e.g. a `vfeeg-superuser` user attribute was forgotten in Keycloak), the EEG dropdown is empty and the dashboard cannot load.

## `tenant` vs `ecId`

The customer SPA uses `ecId` (= `community_id`) in URLs. The JWT carries `tenant`. They are typically the same value, but this depends on the Keycloak user-attribute mapper. If the `tenant` attribute is set to the legacy `rcNumber` instead of `communityId`, all backend calls return 403.

When provisioning a user, set the `tenant` attribute to the EEG's `community_id`.

## Form caveats from operational experience

- **`InputForm.reset-on-invalid` anti-pattern** — wiping the entire form on an invalid-dirty state via `control._reset()` was a recurring source of data loss. Searches for `_reset` / `reset()` inside form-wrapper components should ask "why".
- **Register-page reset+history race** — calling `reset()` and `history.replace()` synchronously after a fire-and-forget dispatch loses backend failures. Use `dispatch().unwrap()` with try/catch, and run side-effects only on the success branch.

## Build and image

- Source: `eegfaktura-web`
- Stack: React 18.2.0, `@ionic/react` + `@ionic/core` 7.x, Vite 5, TypeScript 5.1.6, React Router 5.3.4, Redux Toolkit, `oidc-client-ts` ^3.0.0 + `react-oidc-context` ^3.0.0, recharts ^2.10.3 (charts), i18next, react-hook-form. Package manager pnpm (CI pins 9.12.1). Node 20.18.0 (`.nvmrc`). Build-time i18n via `vite-plugin-i18next-loader`.
- Build: Vite; produces a static `dist/`. The `build` npm script is `tsc && vite build`, but **CI runs `vite build` only** (skips `tsc`) because there are ~30 pre-existing TS errors; Vite transpiles without type-checking, so the bundle builds while `tsc` would fail.
- Runtime image: Caddy 2.8.4 serving the static bundle on port **8080**, static `dist/` at `/var/www/html/vfeeg-web/`.
- Tag scheme: `sha-<short>` (docker/metadata-action `type=sha,format=short,prefix=sha-`), plus `latest` on the default branch.
- **Build Node version**: pin to `node:20-alpine`. `node:22-alpine` produced a vendor chunk that crashed at boot with `Object.defineProperty called on non-object` inside the `@oozcitak/dom` init graph (xmlbuilder2 dependency). Same source compiles cleanly under Node 20. The deeper root cause has not been isolated; the pin is the surgical workaround.
- **`events` polyfill**: xmlbuilder2 imports the Node built-in `events`. Vite/Rollup since v3 do not polyfill Node built-ins automatically; the `events` npm package must be a direct dependency.

## Caddy security headers

The default Caddy config in some historical builds emits **no** security headers, exposing the SPA to easy XSS. Production deployments should add a `header` block:

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

The SPA must surface a link to the source repository (footer + `/about`) for AGPL § 13 compliance, as do all SPAs that ship the AGPL-licensed software.

## Related

- [Architecture / Authentication](../architecture/auth.md) — full JWT flow
- [services/backend](backend.md) — primary API
- [services/billing](billing.md) — billing actions
- [services/energystore](energystore.md) — energy reports
- [reference/glossary](../reference/glossary.md) — dashboard terms
