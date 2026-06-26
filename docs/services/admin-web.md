# admin-web

VFEEG maintenance UI. Cross-tenant administrative tool for the operating team. Talks to [admin-backend](admin-backend.md) and parts of [backend](backend.md).

## At a glance

| | |
|---|---|
| Language | TypeScript 4.9.5 (strict) |
| Framework | React 18.2.0, built with Create React App (react-scripts 5.0.1) |
| UI | Material-UI (MUI) 5.14.0 + @mui/x-data-grid 6.20.4 + @emotion |
| Server | Caddy (port 8080; dev server on 3000) |
| Auth | OIDC, `vfeeg-superuser` group required |
| Backends consumed | admin-backend (primary), backend (some endpoints) |

Historically named `eeg-registration-frontend` or `registration-web`.

## Audience and access

Only users in the `vfeeg-superuser` Keycloak group can log in. The SPA reads the claim from `groups` (note: customer-side reads it from `access_groups` — both claim names carry the same data, populated by two parallel mappers).

In addition, for billing-style operations to succeed, the operator usually needs `EEG_ADMIN` as well (Spring's `hasRole` check, see [services/billing](billing.md)).

## Pages

| Route | Purpose |
|-------|---------|
| `/registration/register-base` | 3-step wizard to create a new EEG |
| `/registration/register-ponton` | EDA / PONTON configuration for an existing EEG |
| `/portal` | Portal Manager — three-column layout for cross-tenant maintenance |

## EEG registration wizard

Three steps:

1. **Allgemein** — name, description, rcNumber, tenant, communityId, settlement interval, legal form, allocation model, area
2. **Betreiber** — first/last name, contact (street, city, zip, phone), email, IBAN, account owner
3. **Kommunikation** — grid operator (from `/vfeeg/operators`), grid name, login (username + password), online flag, PONTON config (`NONE` or `KEP`, with conditional KEP fields)

An XLSX upload button is available top-right; the file is parsed client-side and pre-fills the form. No server-side upload.

## PONTON-only page (`/registration/register-ponton`)

For changing the EDA config of an existing EEG. The `pontonCommType` select here has different options than the wizard (`KEP` or `MAIL`, no `NONE`).

`MAIL` mode adds: domain (default `edanet.at`), username, password, confirm.

`KEP` mode adds: host, port, username, password, confirm.

Typical workflow:

1. Create the EEG via the wizard with `pontonCommType=NONE`.
2. Obtain Klasse-A cert and mail credentials from `ebutilities` (lead time: several weeks).
3. Open `/registration/register-ponton`, switch to `MAIL` mode with the ebutilities credentials.

## Portal Manager

Three-column layout:

| Column | Holds | Source |
|--------|-------|--------|
| sidebar | navigation | — |
| middle | EEG selector — card list with search | `GET /vfeeg/eeg` |
| right top | EEG header strip: name / mode / online / settlement (pencil) / contact / area | inline edit → `POST /admin/master/update/eeg` |
| right middle | Members data grid: first/last name, status, business role, actions | `GET /vfeeg/participants?tenant=<id>` |
| right bottom | Member detail: metering points — id, direction, state, process status (pencil), active / inactive (pencils) | inline edit → `POST /admin/master/update` |

Inline edits open small dialogs and POST to one of three admin endpoints based on the `updateClass` discriminator:

| updateClass | Endpoint | Field |
|-------------|----------|-------|
| `EEG` | `/admin/master/update/eeg` | per-EEG settings |
| `PARTICIPANT`, `BUSINESSROLE` | `/admin/master/update/participant` | member properties |
| `METER`, `PROCESSSTATUS`, `ACTIVESINCE`, `INACTIVESINCE` | `/admin/master/update` | metering-point properties |

## Backend endpoints used

```
EegService:
  GET  /vfeeg/eeg                       → list EEGs
  GET  /vfeeg/participants?tenant=<id>  → members of an EEG
  GET  /vfeeg/operators                 → grid operators (for wizard select)
  GET  /eeg/users                       → users
  POST /eeg/register                    → create EEG
  POST /eeg/ponton                      → set PONTON config

PortalService:
  POST /admin/master/update             → metering-point updates
  POST /admin/master/update/participant → member updates
  POST /admin/master/update/eeg         → EEG updates
```

All routes go to admin-backend.

## Auth caveats

- The SPA reads groups from claim `groups` (not `access_groups`). The Keycloak mapper for this client must emit both names (one mapper per claim).
- Login loop on missing group: if the SPA does not see `vfeeg-superuser` in `groups`, it throws → redirects to login → throws again. Cause is typically a missing `groups` mapper on the admin client.
- The user must have `vfeeg-superuser` **and** be wired to billing-relevant tenants via the `tenant` user attribute (one attribute per accessible tenant, mapper `aggregate.attrs=true`).

## Build and image

- Source: `eegfaktura-admin-web` (historically `eeg-registration-frontend` → `vfeeg-admin-web` → `eegfaktura-admin-web`; the public AGPL fork formerly at that name was archived in 2026-06 — see the `eegfaktura-admin-web-archived-2026-06` tombstone repo)
- Stack: React 18.2.0 + TypeScript 4.9.5 (strict), Create React App (react-scripts 5.0.1). UI is MUI 5.14.0 + @mui/x-data-grid 6.20.4 + @emotion. Forms: react-hook-form 7.45.1 + yup. State: Redux Toolkit 1.9.5 + zustand 5.0.3. Routing: react-router-dom 6.14.1. Also lodash, moment, xlsx (Excel upload), swiper, @maskito, sass.
- Package manager: **npm**. CI runs `npm install` (not `npm ci`) because the committed lockfile is out of sync with `package.json`.
- Build: `npm run build` (CRA → `build/`) → static bundle → Caddy image, served on port **8080**. CI sets **`TSC_COMPILE_ON_ERROR=true`** to bypass pre-existing TypeScript type errors inherited from prod v0.2.15 (the JS still compiles via babel; only the strict ForkTsChecker under `CI=true` would otherwise make them fatal). Context: ~16 files were restored from prod-image sourcemaps and are now committed.
- Image: `ghcr.io/vfeeg-development/eeg-registration-frontend` (legacy name kept).
- Runtime config: fetched at startup from `/config/keycloak-config.json` (Caddy serves it via a templated handler). Uses client_id `at.ourproject.vfeeg.admin`, realm `EEGFaktura`; OIDC via oidc-client-ts 2.2.4 + react-oidc-context 2.2.2 (keycloak-js 22 also present), user store = sessionStorage. Backend endpoint comes from `REACT_APP_ADMIN_SERVER_URL` (default `/admin`).
- Dev proxy (`setupProxy.js`): `/admin` → 8085 (admin-backend), `/api` → 9080 (backend), `/cash` → 9090 (billing).

## Caddy security headers

Same caveat as [services/web](web.md): default Caddy config has historically been empty. Add CSP / HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy.

## AGPL § 13 footer

Required, like every SPA shipping the AGPL software.

## Related

- [services/admin-backend](admin-backend.md) — the API server
- [Architecture / Authentication](../architecture/auth.md) — `groups` vs `access_groups`
- [services/web](web.md) — customer-side counterpart
