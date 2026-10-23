# Glossary

Domain terms used throughout the documentation.

## Domain

**Abrechnungsperiode** — Billing period. Configurable per EEG (`base.eeg."settlementInterval"`): `MONTHLY`, `QUARTER`, `BIANNUAL`, or `ANNUAL`. Once chosen it is changed only in coordination with the platform operator.

**Abrechnungs-Workflow** — Three-phase per-period process: preview creation, preview updates, final billing. After the final-billing action the run is immutable.

**Allocation model** — How surplus production is shared among members (`base.eeg."allocationMode"`). `DYNAMIC` weights by per-period consumption; `STATIC` uses fixed shares.

**Approval** — Steps of the online registration process `EC_REQ_ONL`: `ZUSTIMMUNG_ECON` (the member consented in the grid operator's portal → metering point status `APPROVED`) and `ABSCHLUSS_ECON` (the metering point is part of the EEG → status `ACTIVE`).

**Community ID** — The EC-ID assigned by the network operator (`AT00300...`, 33 characters). Stored in `base.eeg."communityId"`. Also called **EC-ID** or **ecId** in the customer SPA, where it scopes the energystore URLs.

**EEG** — Erneuerbare-Energie-Gemeinschaft, an Austrian renewable energy community. One database row in `base.eeg`. One eegfaktura instance can host multiple EEGs (multi-tenant).

**Energiedaten-XLSX** — Energy-data spreadsheet export used by admins to verify data before final billing. Worksheet "Energiedaten" contains one row per metering point × quarter-hour with OBIS values.

**EVU** — Energieversorgungsunternehmen, the regular electricity supplier (distinct from the EEG). Surplus that cannot be allocated within the EEG goes to the EVU.

**Mitglied** — Member of an EEG. Stored as `base.participant`. A member has at least one Zählpunkt.

**Mitgliedsbeitrag** — Member fee, a tariff of type `EEG`. Fixed amount per member.

**Network operator** — Netzbetreiber. Operates the physical grid and the smart meters. Communicates with eegfaktura via the EDA protocol through PONTON.

**Ortsgebiet** — Geographical scope / community form of the EEG (`base.eeg.area`): `LOCAL`, `REGIONAL`, `BEG` or `GEA`. Determined by the network operator, immutable.

**PONTON** — Adapter system that bridges between EDA messages and the network operators. Acts as the external HTTP endpoint that `eda-xp` integrates with. Not deployed in dev / mock instances.

**Process History** — Append-only log of every EDA conversation. Stored in `base.processhistory`. Surfaced in the customer SPA as the "Prozess-History" view.

**rcNumber** — Market-partner number of the EEG (`RC...`, `GC...`, `CC...` in production, `TE...` for test). Stored in `base.eeg."rcNumber"`. The customer SPA sends it to energystore as the `X-Tenant` header, and energystore uses it as its storage key — so where it differs from the tenant, the user's `tenant` claim must also contain it (see [services/web](../services/web.md#tenant-vs-ecid)).

**Stammdaten** — Master data: participants, metering points, EEG settings, tariffs. Owned by the `backend` service in the `base` schema.

**Stammdaten-XLSX-Import** — Bulk-import template (`/attachments/15` in the public docs). Brings new members and their metering points into eegfaktura in one shot, and can append metering points to existing members. **Update-by-import is not supported** — existing member data is never changed, and a metering point the member already has fails that row. See [Master Data Import](masterdata-import.md).

**Tarif** — Tariff. Three types:
- `EEG` — annual member fee, fixed per person
- `VZP` — consumer Zählpunkt tariff, applied as ct/kWh × G.03 per billing period
- `EZP` — producer Zählpunkt tariff, applied as ct/kWh × (G.01T − P.01T) per billing period
- `AKONTO` — a fourth type present in the data model (`base.tariff.type`)

`VZP` and `EZP` tariffs can be simple or time-based, see **ZVT**.

**Teilnahmefaktor** (participation factor) — Percentage (0..100) per metering point per EEG. Determines the share of the metering point's consumption / production attributed to this EEG. A metering point can be in up to 5 EEGs; the sum of participation factors must not exceed 100. Default 100% for metering points activated before 2024-04-08. Changes (EDA process `EC_PRTFACT_CHANGE`) are only accepted by the grid operator Mon-Fri 09:00–17:00 Austrian time and take effect the next day; outside the window the grid operator rejects with code 82 „Prozessdatum falsch“. Since web 1.8.0 the form shows the window and warns before sending outside it (`src/util/PartFactWindow.util.ts`, hour check only; sending stays possible).

**Tenant** — In eegfaktura the tenant is the EEG. The string is the primary key `base.eeg.tenant` and appears on the user's Keycloak attribute `tenant` (JWT claim `tenant`, a string array, compared case-insensitively). Usually it equals the `rcNumber`; GEA tenants carry a suffix (`GCxxxxxx-00n`) and then differ from it.

**Verteilungsmodell** — see Allocation model.

**Zählpunkt (ZP)** — Metering point. Direction is `CONSUMPTION` or `GENERATION`. Identified by a 33-character AT-prefixed string assigned by the network operator. Status in the online registration flow: `NEW` → `PENDING` → `APPROVED` → `ACTIVE`; further states include `INACTIVE`, `REJECTED`, `REVOKED`, `INVALID`, `ARCHIVED` (`model/participant.go`).

**ZVT** — Zeitvariabler Tarif (time-of-use tariff), since web 1.8.0. A `VZP`/`EZP` tariff in mode "Zeitbasiert" has a base price plus up to two named time windows (15-minute raster, may cross midnight) with their own price. For billing, energystore returns per-window sums (`buckets`) per metering point, and the SPA forwards them with the window definitions to billing.

## Protocol

**EDA** — Energie Daten Austausch Austria. Asynchronous XML messaging protocol for grid-operator communication.

**EbMsMessage** — Inbound EDA message structure after scalaxb parsing. Wraps the XML payload with conversation metadata.

**EC_REQ_***/**CM_REV_*** etc. — EDA process codes. See [Messaging](../architecture/messaging.md) for the catalogue of codes used by eegfaktura.

**messageCodeVersion** — EDA process version (e.g. `02.30`). Drives handler selection in `MessageHelper`. Distinct from the schema-version XML attribute.

**OBIS** — Object Identification System. Standardized codes for energy-meter readings. Format `<medium>-<channel>:<group>.<attribute>.<storage>` followed by a vendor extension. eegfaktura uses a small canonical subset documented in [OBIS Codes](obis-codes.md).

**G.01 / G.02 / G.03 / P.01** — eegfaktura's OBIS extensions. Roughly: G.01 = measured total, G.02 = theoretically allocatable share, G.03 = actually covered share, P.01 = surplus. Both consumer and producer Zählpunkte have a G.01 with different semantics.

**T-Variant** (e.g. `G.01T`) — value × participation factor / 100. Mandatory for billing periods starting 2024-04-08 or later.

## Roles

**EEG_ADMIN** — Keycloak group (JWT claim `access_groups`). Full admin for the user's own tenant. Required for billing API calls.

**EEG_OWNER** — Keycloak group. Reserved for future reporting use. No code path currently checks it.

**EEG_USER** — Keycloak group. Regular member. Sees only their own participant data in the customer SPA.

**superuser** — Keycloak **realm role** (`realm_access.roles`), not a group. Cross-tenant maintenance role: backend and energystore skip the tenant check for it, and the customer SPA lists all EEGs. Required for admin-web login and the admin-backend routes.

## Components

**admin-backend** — Scala/Pekko backend for the maintenance UI. Separate from the customer-side `backend`. Originally named `registration-backend`.

**admin-web** — React 18 + MUI SPA (Create React App) for VFEEG operators (`eegfaktura-admin`). Cross-tenant maintenance UI.

**backend** — Go service. Owns master data and EDA inbound dispatch on the consumer side.

**billing** — Java/Spring service. Generates billing documents, owns the `billingj` schema.

**eda-xp** — Scala/Pekko EDA gateway. Speaks HTTP with PONTON, MQTT with the rest of the platform. Sometimes named `eda-xp-connector` or `xp-adapter` in legacy configs.

**eda-mock** — Scala stub of the EDA gateway for dev/test environments without a real PONTON link.

**energystore** — Go + Badger service. Stores per-period energy time-series and answers report queries.

**filestore** — Python service. Stores generated documents and serves member-side downloads.

**web** — React SPA for community members. Customer-facing UI.

## Infrastructure

**Badger** — Embedded KV store used by energystore. Persists to a local volume.

**Mosquitto** — Eclipse Mosquitto MQTT broker. The message bus for EDA inbound and energy-data ingestion.

**PVC** — PersistentVolumeClaim. Stateful storage for PostgreSQL, energystore (Badger), filestore (blobs), Mosquitto (queue persistence).

## Related

- [OBIS Codes](obis-codes.md) — canonical OBIS table and storage mapping
- [Architecture / Service Overview](../architecture/overview.md) — service topology
