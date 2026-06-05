# Glossary

Domain terms used throughout the documentation.

## Domain

**Abrechnungsperiode** — Billing period. Configurable per EEG: `MONTHLY`, `QUARTERLY`, `BIANNUAL`, or `YEARLY`. Once chosen it is changed only in coordination with the platform operator.

**Abrechnungs-Workflow** — Three-phase per-period process: preview creation, preview updates, final billing. After the final-billing action the run is immutable.

**Allocation model** — How surplus production is shared among members. `DYNAMIC` weights by per-period consumption; `STATIC` uses fixed shares.

**Approval** — Per-member document acknowledgment flag (`ZUSTIMMUNG_ECON`, `ABSCHLUSS_ECON`). Reflects the network operator's acceptance of the member's participation.

**Community ID** — The EC-ID assigned by the network operator (`AT00300...`). Stored in `base.eeg.community_id`. Also called **EC-ID** or **ecId** in the customer SPA.

**EEG** — Erneuerbare-Energie-Gemeinschaft, an Austrian renewable energy community. One database row in `base.eeg`. One eegfaktura instance can host multiple EEGs (multi-tenant).

**Energiedaten-XLSX** — Energy-data spreadsheet export used by admins to verify data before final billing. Worksheet "Energiedaten" contains one row per metering point × quarter-hour with OBIS values.

**EVU** — Energieversorgungsunternehmen, the regular electricity supplier (distinct from the EEG). Surplus that cannot be allocated within the EEG goes to the EVU.

**Mitglied** — Member of an EEG. Stored as `base.participant`. A member has at least one Zählpunkt.

**Mitgliedsbeitrag** — Annual member fee, a tariff of type `EEG`. Fixed amount per person.

**Network operator** — Netzbetreiber. Operates the physical grid and the smart meters. Communicates with eegfaktura via the EDA protocol through PONTON.

**Ortsgebiet** — Geographical scope of the EEG: `LOKAL` or `REGIONAL`. Determined by the network operator, immutable.

**PONTON** — Adapter system that bridges between EDA messages and the network operators. Acts as the external HTTP endpoint that `eda-xp` integrates with. Not deployed in dev / mock instances.

**Process History** — Append-only log of every EDA conversation. Stored in `base.processhistory`. Surfaced in the customer SPA as the "Prozess-History" view.

**rcNumber** — Invoicing number (`RC...` for production, `TE...` for test). Stored in `base.eeg.rc_number`. Internally treated as the **tenant** identifier in some legacy paths.

**Stammdaten** — Master data: participants, metering points, EEG settings, tariffs. Owned by the `backend` service in the `base` schema.

**Stammdaten-XLSX-Import** — Bulk-import template (`/attachments/15` in the public docs). Brings new members and their metering points into eegfaktura in one shot. **Update-by-import is not supported** — known metering points must not appear in the file.

**Tarif** — Tariff. Three types:
- `EEG` — annual member fee, fixed per person
- `VZP` — consumer Zählpunkt tariff, applied as ct/kWh × G.03 per billing period
- `EZP` — producer Zählpunkt tariff, applied as ct/kWh × (G.01T − P.01T) per billing period

**Teilnahmefaktor** (participation factor) — Percentage (0..100) per metering point per EEG. Determines the share of the metering point's consumption / production attributed to this EEG. A metering point can be in up to 5 EEGs; the sum of participation factors must not exceed 100. Default 100% for metering points activated before 2024-04-08. Changes are only allowed Mon-Fri 09:00–17:00 and take effect the next day.

**Tenant** — In eegfaktura the tenant is the EEG. The string lives in `base.eeg.tenant` and on the user's Keycloak attribute `tenant`. In some deployments tenant equals `community_id`; in others it is a separate ID.

**Verteilungsmodell** — see Allocation model.

**Zählpunkt (ZP)** — Metering point. Direction is `CONSUMPTION` or `GENERATION`. Identified by an AT-prefixed string assigned by the network operator. Lifecycle states: `NEW` → `ACTIVATED` → `ACTIVE`.

## Protocol

**EDA** — Energie Daten Austausch Austria. Asynchronous XML messaging protocol for grid-operator communication.

**EbMsMessage** — Inbound EDA message structure after scalaxb parsing. Wraps the XML payload with conversation metadata.

**EC_REQ_***/**CM_REV_*** etc. — EDA process codes. See [Messaging](../architecture/messaging.md) for the catalogue of codes used by eegfaktura.

**messageCodeVersion** — EDA process version (e.g. `02.30`). Drives handler selection in `MessageHelper`. Distinct from the schema-version XML attribute.

**OBIS** — Object Identification System. Standardized codes for energy-meter readings. Format `<medium>-<channel>:<group>.<attribute>.<storage>` followed by a vendor extension. eegfaktura uses a small canonical subset documented in [OBIS Codes](obis-codes.md).

**G.01 / G.02 / G.03 / P.01** — eegfaktura's OBIS extensions. Roughly: G.01 = measured total, G.02 = theoretically allocatable share, G.03 = actually covered share, P.01 = surplus. Both consumer and producer Zählpunkte have a G.01 with different semantics.

**T-Variant** (e.g. `G.01T`) — value × participation factor / 100. Mandatory for billing periods starting 2024-04-08 or later.

## Roles

**EEG_ADMIN** — Keycloak group. Full admin for the user's own tenant. Required for billing API calls.

**EEG_OWNER** — Keycloak group. Reserved for future reporting use. No code path currently checks it.

**EEG_USER** — Keycloak group. Regular member. Sees only their own participant data in the customer SPA.

**vfeeg-superuser** — Keycloak group. Cross-tenant maintenance role. Required for admin-web login.

## Components

**admin-backend** — Scala/Pekko backend for the maintenance UI. Separate from the customer-side `backend`. Originally named `registration-backend`.

**admin-web** — React-Admin SPA for VFEEG operators. Cross-tenant maintenance UI.

**backend** — Go service. Owns master data and EDA inbound dispatch on the consumer side.

**billing** — Java/Spring service. Generates billing documents, owns the `billingj` schema.

**eda-xp** — Scala/Pekko EDA gateway. Speaks HTTP with PONTON, MQTT with the rest of the platform. Sometimes named `eda-xp-connector` or `xp-adapter` in legacy configs.

**eda-mock** — Scala stub of the EDA gateway for dev/test environments without a real PONTON link.

**energystore** — Go + Badger service. Stores per-period energy time-series and answers report queries.

**filestore** — Python service. Stores generated documents and serves member-side downloads.

**web** — React SPA for community members. Customer-facing UI.

## Infrastructure

**Argo CD** — GitOps tool that reconciles Kubernetes manifests from a Git repository. Manages all eegfaktura service workloads.

**Badger** — Embedded KV store used by energystore. Persists to a local volume.

**Bootstrap chart** — Helm chart `eegfaktura-bootstrap`. Holds the one-shot Jobs that seed schema, realm, users, sample data. Helm-managed, not Argo-managed.

**Mosquitto** — Eclipse Mosquitto MQTT broker. The message bus for EDA inbound and energy-data ingestion.

**PVC** — PersistentVolumeClaim. Stateful storage for PostgreSQL, energystore (Badger), filestore (blobs), Mosquitto (queue persistence).

## Related

- [OBIS Codes](obis-codes.md) — canonical OBIS table and storage mapping
- [Architecture / Service Overview](../architecture/overview.md) — service topology
