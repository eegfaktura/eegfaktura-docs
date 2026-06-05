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
- Build: Vite (or CRA in older revisions); produces a static `dist/`
- Runtime image: Caddy serving the static bundle
- Tag scheme: SemVer

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
