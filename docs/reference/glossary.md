# Glossary

Domain terms used throughout the documentation.

## Domain

**Abrechnungsperiode** — Billing period. Configurable per EEG (`base.eeg."settlementInterval"`): `MONTHLY`, `QUARTER`, `BIANNUAL`, or `ANNUAL`. Read-only in the customer SPA; a superuser can change it in admin-web (Portal Manager).

**Abrechnungs-Workflow** — Three-phase per-period process: preview creation, preview updates, final billing. After the final-billing action the run is immutable.

**Allocation model** — How the network operator shares the community's production among the consumers (`base.eeg."allocationMode"`, read-only in the customer SPA). `DYNAMIC` weights by consumption per quarter hour; `STATIC` uses fixed shares. eegfaktura does not compute the allocation; it stores the operator's values.

**Approval** — Steps of the online registration process `EC_REQ_ONL`: `ZUSTIMMUNG_ECON` (the member consented in the grid operator's portal → metering point process state `APPROVED`) and `ABSCHLUSS_ECON` (the metering point is part of the EEG → process state `ACTIVE`).

**Community ID** — The EC-ID assigned by the network operator (`AT…`). Stored in `base.eeg."communityId"`. Also called **EC-ID** or **ecId** in the customer SPA, where it scopes the energystore URLs.

**EEG** — Erneuerbare-Energie-Gemeinschaft, an Austrian renewable energy community. One database row in `base.eeg`. One eegfaktura instance can host multiple EEGs (multi-tenant).

**Energiedaten-XLSX** — Energy-data spreadsheet export used by admins to verify data before final billing. Produced by energystore (`excel/EnergyExport.go`): a sheet "Summary" and a sheet "Energiedaten" with header rows per metering point (id, name, direction, period) and one row per quarter hour, three value columns per consumer and two per producer; cells are coloured by data quality.

**EVU** — Energieversorgungsunternehmen, the regular electricity supplier (distinct from the EEG). Surplus that cannot be allocated within the EEG goes to the EVU.

**Mitglied** — Member of an EEG. Stored as `base.participant`, with its Zählpunkte in `base.meteringpoint`. A member without an active Zählpunkt counts as inactive on the dashboard.

**Mitgliedsbeitrag** — Member fee, a tariff of type `EEG`. A fixed amount that billing adds as its own position on every billing run of the member.

**Network operator** — Netzbetreiber. Operates the physical grid and the smart meters. Communicates with eegfaktura via the EDA protocol through PONTON.

**Ortsgebiet** — Geographical scope / community form of the EEG (`base.eeg.area`): `LOCAL`, `REGIONAL`, `BEG` or `GEA`. Set when the EEG is registered (admin-web wizard); the customer SPA does not edit it.

**PONTON** — The Ponton X/P messenger (KEP) that exchanges EDA messages with the market partners. External HTTP endpoint that `eda-xp` integrates with; not deployed in the Dev zone (see eda-mock) or the local stack.

**Process History** — Log of the EDA messages the backend receives over MQTT (one row per message, including the echo of its own requests). Stored in `base.processhistory`; UI actions are not logged there. Surfaced in the customer SPA as the "Prozess-History" view.

**rcNumber** — Market-partner number of the EEG (e.g. `RC...`, `GC...`, `CC...`; test data uses `TE...`). Stored in `base.eeg."rcNumber"`. The customer SPA sends it to energystore as the `X-Tenant` header, and energystore uses it as its storage key — so where it differs from the tenant, the user's `tenant` claim must also contain it (see [services/web](../services/web.md#tenant-vs-ecid)).

**Stammdaten** — Master data: participants, metering points, EEG settings, tariffs. Owned by the `backend` service in the `base` schema.

**Stammdaten-XLSX-Import** — Bulk-import template (`/attachments/15` in the public docs). Brings new members and their metering points into eegfaktura in one shot, and can append metering points to existing members. **Update-by-import is not supported** — existing member data is never changed, and a metering point the member already has fails that row. See [Master Data Import](masterdata-import.md).

**Tarif** — Tariff. Four types in the data model (`model/tariff.go`, `base.tariff.type`):
- `EEG` — member fee, a fixed amount per member and billing run
- `VZP` — consumer Zählpunkt tariff: ct/kWh × the kWh the web sends (report `utilization` = G.03) per billing period
- `EZP` — producer Zählpunkt tariff: ct/kWh × (report `production − allocation` = G.01 − P.01) per billing period
- `AKONTO` — present in the data model and the SPA's types; billing has no code for it

`VZP` and `EZP` tariffs can be simple or time-based, see **ZVT**.

**Teilnahmefaktor** (participation factor) — Percentage per metering point per EEG (the web form accepts more than 0 up to 100; new metering points and the Excel import default to 100). Determines the share of the metering point's consumption / production attributed to this EEG. A metering point can be in up to 5 EEGs; the sum of participation factors must not exceed 100 (the grid operator rejects with code 183 „Summe der gemeldeten Aufteilungsschlüssel übersteigt 100%“). Default 100% for metering points activated before 2024-04-08. Changes (EDA process `EC_PRTFACT_CHANGE`) are only accepted by the grid operator between 09:00 and 17:00 Austrian time and take effect the next day; outside the window the grid operator rejects with code 82 „Prozessdatum falsch“. Since web 1.8.0 the form shows the window and warns before sending outside it (`src/util/PartFactWindow.util.ts`, hour check only; sending stays possible).

**Tenant** — In eegfaktura the tenant is the EEG. The string is the primary key `base.eeg.tenant` and appears on the user's Keycloak attribute `tenant` (JWT claim `tenant`, a string array, compared case-insensitively). Usually it equals the `rcNumber`; GEA tenants carry a suffix (`GCxxxxxx-00n`) and then differ from it.

**Verteilungsmodell** — see Allocation model.

**Zählpunkt (ZP)** — Metering point. Direction is `CONSUMPTION` or `GENERATION`. Identified by a 33-character string starting with `AT`, assigned by the network operator (the web form checks the length). Process state in the online registration flow: `NEW` → `INIT` → `PENDING` → `APPROVED` → `ACTIVE`; further states include `INACTIVE` (also the result of a revocation), `REJECTED`, `INVALID`, `ARCHIVED` (`model/participant.go`). `REVOKED` exists as a constant but is not set by the revocation flow.

**ZVT** — Zeitvariabler Tarif (time-of-use tariff), since web 1.8.0. A `VZP`/`EZP` tariff in mode "Zeitbasiert" has a base price plus up to two named time windows (15-minute raster, may cross midnight) with their own price. For billing, energystore returns per-window sums (`buckets`) per metering point, and the SPA forwards them with the window definitions to billing.

## Protocol

**EDA** — The Austrian energy-industry data exchange: asynchronous XML messaging (ebUtilities schemas) between grid operators and market participants such as EEGs.

**EbMsMessage** — eda-xp's JSON message model on MQTT, in both directions (message code, conversation id, sender / receiver, `ecId`, meter and energy data). eda-xp maps it from and to the EDA XML; it does not carry the XML itself.

**EC_REQ_***/**CM_REV_*** etc. — EDA process codes. See [Messaging](../architecture/messaging.md) for the catalogue of codes used by eegfaktura.

**messageCodeVersion** — Version label on an outbound request (e.g. `02.40`), stamped by the backend from its config `eda-process-versions.<MESSAGE_CODE>`. It is the schema set version, not the process version, and selects the XML builder in the message class's `getVersion()`; eda-xp sends it as Ponton header `MessageVersion`. Inbound documents are dispatched by their XML namespace instead.

**OBIS** — Object Identification System. Standardized codes for energy-meter readings. eegfaktura sees them in the abbreviated `A-B:C.D.E` form followed by a token (`1-1:1.9.0 G.01`). eegfaktura uses a small canonical subset documented in [OBIS Codes](obis-codes.md).

**G.01 / G.02 / G.03 / P.01** — The EDA tokens for the EEG quantities. Roughly: G.01 = measured total, G.02 = theoretically allocatable share, G.03 = actually covered share, P.01 = surplus. Both consumer and producer Zählpunkte have a G.01 with different semantics.

**T-Variant** (e.g. `G.01T`) — value × participation factor / 100. Mandatory for billing periods starting 2024-04-08 or later.

## Roles

**EEG_ADMIN** — Keycloak group (JWT claim `access_groups`). Full admin for the user's own tenant. Required for billing API calls.

**EEG_OWNER** — Keycloak group. No code path checks it; admin-backend assigns it together with `EEG_ADMIN` to new community logins.

**EEG_USER** — Keycloak group. Regular member. Sees only their own participant data in the customer SPA.

**superuser** — Keycloak **realm role** (`realm_access.roles`), not a group. Cross-tenant maintenance role: backend, energystore, billing and filestore skip the tenant check for it, and the customer SPA lists all EEGs. Required for admin-web login and the admin-backend routes.

## Components

**admin-backend** — Scala/Pekko backend for the maintenance UI. Separate from the customer-side `backend`. Originally named `registration-backend`.

**admin-web** — React 18 + MUI SPA (Create React App) for VFEEG operators (`eegfaktura-admin`). Cross-tenant maintenance UI.

**backend** — Go service. Owns master data and EDA inbound dispatch on the consumer side.

**billing** — Java/Spring service. Generates billing documents, owns the `billingj` schema.

**eda-xp** — Scala/Pekko EDA gateway. Speaks HTTP with PONTON (or mail with tenant mailboxes), MQTT with backend and energystore, and serves gRPC (member mails, Ponton registration). Sometimes named `eda-xp-connector` or `xp-adapter` in legacy configs.

**eda-mock** — Dev-zone placeholder for the PONTON/KEP side: two `traefik/whoami` HTTP echo containers that answer eda-xp's outbound calls with 200; no code of its own, no MQTT, no grid-operator responses.

**energystore** — Go + Badger service. Stores the quarter-hour energy values delivered by the network operator and answers report queries.

**filestore** — Python service. Stores member documents (contracts uploaded in the customer SPA) and serves authenticated downloads. Billing PDFs stay in billing's own schema.

**web** — React SPA for community members. Customer-facing UI.

## Infrastructure

**Badger** — Embedded KV store used by energystore. Persists to a local volume.

**Mosquitto** — Eclipse Mosquitto MQTT broker. The message bus for outbound EDA requests, inbound EDA messages and energy-data ingestion.


## Related

- [OBIS Codes](obis-codes.md) — canonical OBIS table and storage mapping
- [Architecture / Service Overview](../architecture/overview.md) — service topology
