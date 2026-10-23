# admin-web

VFEEG maintenance UI. Cross-tenant administrative tool for the operating team. Talks to [admin-backend](admin-backend.md) and, for one ops page, directly to [energystore](energystore.md).

## At a glance

| | |
|---|---|
| Language | TypeScript 4.9.5 (strict) |
| Framework | React 18.2.0, built with Create React App (react-scripts 5.0.1) |
| UI | Material-UI (MUI) 5.14.0 + @mui/x-data-grid 6.20.4 + @mui/x-date-pickers + @emotion |
| Server | Caddy 2.8.4 (port 8080; dev server on 3000) |
| Auth | OIDC, realm role `superuser` required |
| Backends consumed | admin-backend (primary), energystore (raw-data delete only) |

Historically named `eeg-registration-frontend` or `registration-web`.

## Audience and access

Only users with the `superuser` realm role can use the portal. The SPA decodes the access token and reads `realm_access.roles` (`src/services/superuser.ts`); without the role it renders a single "no access" page (`AccessDenied`), which also says that a newly assigned role needs a fresh sign-in. This is a display decision only: the boundary is enforced server-side (admin-backend requires `superuser` on all routes, energystore on the raw-data delete). The token is decoded, not verified, in the browser.

## Pages

| Route | Purpose |
|-------|---------|
| `/registration/register-base` | 3-step wizard to create a new EEG |
| `/registration/register-ponton` | EDA / PONTON configuration for an existing EEG |
| `/registration/accounts` | EEG Accounts — list and add the logins of an EEG |
| `/portal` | Portal Manager — three-column layout for cross-tenant maintenance |
| `/rawdata-delete` | Ops page "Energiedaten löschen" — delete the raw energy data of one metering point in a time range |

## EEG registration wizard

Three steps:

1. **Allgemein** — name, description, rcNumber, tenant, communityId, settlement interval, legal form, allocation model, area
2. **Betreiber** — first/last name, contact (street, city, zip, phone), email, IBAN, account owner, and the list **Logins** (first name, last name, e-mail per row)
3. **Kommunikation** — grid operator (from `/vfeeg/operators`), grid name, login (username + password), online flag, PONTON config (`NONE` = "ignorieren / bereits konfiguriert" or `KEP`; selectable only for online EEGs)

An XLSX upload button sits right of the step-1 heading (`CommonEegProperties`); the file is parsed client-side and pre-fills the form. No server-side upload. Column headers are matched ignoring case, spaces, line breaks and hyphens. From the Excel member list the Logins list is filled with the main account ("Hauptaccount", or the contact if that is empty) and the further accounts 1–3 that have an e-mail; it can be edited.

Exactly the listed logins get an account (see [admin-backend — Logins per EEG](admin-backend.md#logins-per-eeg)); with an empty list the operator (contact) gets the login, as before. After the registration the message names the result per login; a message with a failed login stays until it is closed.

## PONTON-only page (`/registration/register-ponton`)

For changing the EDA config of an existing EEG: RC number, tenant and the `pontonCommType`, which here offers only `KEP` ("Kep Server"). The mail transport (`MAIL`) was removed in 1.1.4; admin-backend rejects it.

admin-backend answers `POST /eeg/ponton` with HTTP 200 and the real outcome as `status` / `message` in the body; the page treats a body status outside 200–299 as an error and shows the backend's message.

## EEG Accounts (`/registration/accounts`)

Choose a community, see its logins (name, e-mail, username, state "aktiv" / "Passwort noch nicht gesetzt" / "deaktiviert") and add one (first name, last name, e-mail). Uses `GET /eeg/accounts` and `POST /eeg/account`. Removing a login is done in Keycloak.

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
| `PARTICIPANT` | `/admin/master/update/participant` | member properties |
| `PROCESSSTATUS`, `ACTIVESINCE`, `INACTIVESINCE` | `/admin/master/update` | metering-point properties |

The business-role editor is commented out in the code; admin-backend's `updateClass` enum has no `BUSINESSROLE` or `METER` value.

## Raw-data delete (`/rawdata-delete`)

Deletes (zeroes) the raw energy data of a single metering point within a time range in energystore v1. Two steps: a dry run shows the affected timesteps and summed kWh, then a double confirmation (retype the metering point) executes. Calls energystore directly, `POST ${REACT_APP_ENERGY_SERVER_URL}/eeg/v2/{ecId}/rawdata/delete` (default `/energystore`, same-origin), with the operator's bearer token and a `tenant` header; energystore requires the `superuser` role on that endpoint. The community field is the long `AT…` community id, not the RC number.

## Backend endpoints used

```
EegService:
  GET  /vfeeg/eeg                       → list EEGs
  GET  /vfeeg/participants?tenant=<id>  → members of an EEG
  GET  /vfeeg/operators                 → grid operators (for wizard select)
  POST /eeg/register                    → create EEG (optional accounts list)
  POST /eeg/ponton                      → set PONTON config
  GET  /eeg/accounts?tenant=<id>        → logins of an EEG
  POST /eeg/account?tenant=<id>         → add a login

PortalService:
  POST /admin/master/update             → metering-point updates
  POST /admin/master/update/participant → member updates
  POST /admin/master/update/eeg         → EEG updates
  POST /energystore/eeg/v2/{ecId}/rawdata/delete → raw-data delete (energystore)
```

All routes except the raw-data delete go to admin-backend.

## Auth caveats

- The SPA reads the role from `realm_access.roles` in the access token; it does not read the `groups` or `access_groups` claim.
- Roles are frozen into the token when it is issued: after assigning `superuser`, the user must sign out and in again.

## Build and image

- Source: `eegfaktura-admin` (historically `eeg-registration-frontend` → `vfeeg-admin-web` → `eegfaktura-admin-web`; the public AGPL fork formerly at that name was archived in 2026-06 — see the archived `gemeinstrom/eegfaktura-admin-web-archived-2026-06` tombstone repo)
- Stack: React 18.2.0 + TypeScript 4.9.5 (strict), Create React App (react-scripts 5.0.1). UI is MUI 5.14.0 + @mui/x-data-grid 6.20.4 + @mui/x-date-pickers (moment adapter, `de` locale) + @emotion. Forms: react-hook-form 7.45.1 + yup. State: Redux Toolkit 1.9.5 + zustand 5.0.3. Routing: react-router-dom 6.30. Also lodash, moment, xlsx (Excel upload), @maskito, sass.
- Package manager: **npm**. CI (`pr-checks.yml`, `rolling-release.yml`) runs `npm ci` on Node 20.18.0.
- Build: `npm run build` (CRA → `build/`) → static bundle → Caddy image, served on port **8080**. Since 1.2.0 CI no longer sets `TSC_COMPILE_ON_ERROR`: type errors fail the build, and the release workflow checks that `build/index.html` exists. Source files were restored from prod-image sourcemaps for 1.0.0 and are committed.
- Image: `ghcr.io/vfeeg-development/eeg-registration-frontend` (legacy name kept), base `caddy:2.8.4`, runs as user `app` (UID 1000).
- Runtime config: fetched at startup from `/config/keycloak-config.json` (Caddy renders it from a template with `KEYCLOAK_REALM`, `KEYCLOAK_URL`, `KEYCLOAK_CLIENT_ID`). Defaults: client_id `at.ourproject.vfeeg.admin`, realm `EEGFaktura`; OIDC via oidc-client-ts 2.2.4 + react-oidc-context 2.2.2 (keycloak-js 22 also present), user store = sessionStorage. Backend endpoints come from `REACT_APP_ADMIN_SERVER_URL` (default `/admin`) and `REACT_APP_ENERGY_SERVER_URL` (default `/energystore`).
- Dev proxy (`setupProxy.js`): `/admin` → 8085 (admin-backend), `/api` → 9080 (backend), `/cash` → 9090 (billing).

## Caddy security headers

Same caveat as [services/web](web.md): default Caddy config has historically been empty. Add CSP / HSTS / X-Frame-Options / X-Content-Type-Options / Referrer-Policy.

## AGPL § 13 footer

Required, like every SPA shipping the AGPL software. admin-web does not render one yet (no footer component in `src/`).

## Related

- [services/admin-backend](admin-backend.md) — the API server
- [Architecture / Authentication](../architecture/auth.md) — claims and roles
- [services/web](web.md) — customer-side counterpart
