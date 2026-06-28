# Databases

eegfaktura runs against **one or two PostgreSQL clusters**, depending on whether [energystore-v2](../services/energystore-v2.md) is deployed:

- **`postgres`** — master data plus Keycloak's own logical database, optional Ponton adapter. Always present. In the local stack this is a single custom PostgreSQL image (`eegfaktura-postgresql`) that provisions both an `eegfaktura` application database and a separate `keycloak` database.
- **`postgres-energy`** — a separate, optional second cluster (TimescaleDB) for energystore-v2. Pilot only — keep in mind it lives in a separate repo and is planned for a later production cutover.

For the v1 stack only the first cluster exists; energystore v1 stores time-series in an **embedded BadgerDB KV store on a persistent volume**, not in PostgreSQL.

## Logical layout

```
postgres cluster (custom image)
├── eegfaktura database
│   ├── base       — backend (Go): participants, EEGs, metering points, contracts, tariffs
│   ├── eda        — backend (Go) + eda-xp: EDA message storage, process history
│   ├── billingj   — billing (Java): billing runs, line items, generated documents
│   └── filestore  — filestore (Python): file metadata (blobs on local filesystem)
└── keycloak database — Keycloak identity data (separate logical DB)

postgres-energy cluster (energystore-v2 only; TimescaleDB)
└── energystore
    ├── energy_data         — hypertable, hash-partitioned on tenant_id
    └── counterpoint_meta   — per-meter metadata
```

The exact main-database name is environment-specific (e.g. `eegfaktura`). Service config uses an env var (`DB_NAME` / equivalent) rather than hard-coding.

## Why a separate cluster for time-series

The split is captured in ADR-0010 (`eegfaktura-platform`). Conceptually:

- **Access patterns** are different: time-series ingest is high-volume write of small rows with no joins; OLTP master data has lower volume with rich joins. Sharing one cluster forces trade-offs that fit neither workload.
- **Operational independence**: a long-running aggregate query on time-series data should not impact master-data OLTP latency, and vice versa.

The `energy_data` table is a TimescaleDB **hypertable** with hash partitioning on `tenant_id`. New rows for a given tenant fall in one bucket; bucket count is chosen so that concurrent tenant writes spread across buckets. See the [postgres-energy section in the postgres service page](../services/postgres.md#postgres-energy-timescaledb-energystore-v2-only) for the architectural decisions; concrete tuning lives in the per-cluster Helm overlay.

## Schema ownership

Each schema is owned by exactly one service. Other services do not write to it.

| Schema | Owner | Notes |
|--------|-------|-------|
| `base` | backend | Largest and most central schema. Master data. Auto-migrated on startup (golang-migrate). |
| `eda` | backend + eda-xp | backend writes process history; eda-xp writes inbound message storage. eda-xp manages its own tables with Slick + Flyway. |
| `billingj` | billing | Flyway-migrated (`ddl-auto=validate`). Conflict with full-DB dumps — see "schema dumps" below. |
| `filestore` | filestore | File metadata only (Alembic). Five tables: `file_categories`, `storages`, `file_containers`, `files`, `file_attributes`. Blobs live on the local filesystem, not in the database. |
| `keycloak` (separate DB) | Keycloak | Keycloak's own logical database; managed entirely by Keycloak, no application access. |

energystore does **not** own a PostgreSQL schema. It reads master data via the backend's REST API (or directly from `base` for some queries, depending on deployment) and stores energy time-series in its own BadgerDB KV store on a PVC.

admin-backend uses Slick and ships **no** Flyway/migrations in-repo.

## Connection model

Each service has its own database user with privileges scoped to its schema. Connection details are provided as Kubernetes Secrets (or, in the local stack, as docker-compose environment variables).

| Service | Schema scope | Migration tool |
|---------|--------------|----------------|
| backend | `base` (write), `eda` (write) | golang-migrate, auto-run on startup |
| billing | `billingj` | Flyway (`ddl-auto=validate`) |
| filestore | `filestore` | Alembic |
| eda-xp | `eda` (write) + own tables | Slick + Flyway 10.20.0 |
| admin-backend | `base` (read/write via Slick) | none (no in-repo migrations) |

## Schema dumps vs. Flyway

Some deployments initialize the schema from a `pg_dump --schema-only` snapshot taken from a reference instance. This works for `base`, `eda`, and `filestore`, but conflicts with billing's Flyway. When billing starts against a database that already contains `billingj.*` tables (because they were in the dump), Flyway tries to re-create them and fails.

Workaround: `DROP SCHEMA billingj CASCADE` before the billing pod starts the first time, then let Flyway create the schema fresh.

## Notable tables

### `base.participant`

The member master record. Joined into nearly every customer-facing view.

| Column | Purpose |
|--------|---------|
| `id` | UUID primary key |
| `tenant` | EEG `communityId` (string) — the scoping field |
| `email` | matched against JWT `email` for member-self lookups |
| `contactdetail` | nested JSON/relation with phone, address, IBAN |
| `business_role` | participant kind (PRIVATE / BUSINESS) |

### `base.metering_point` (Zählpunkt)

| Column | Purpose |
|--------|---------|
| `id` | UUID |
| `participant_id` | FK to participant |
| `meteringpoint_id` | the AT-prefixed ID issued by the network operator |
| `direction` | `CONSUMPTION` or `GENERATION` |
| `participation_factor` | percentage (0..100), default 100 |
| `status` | `NEW`, `ACTIVATED`, `ACTIVE`, `REGISTERED` |

### `base.eeg`

| Column | Purpose |
|--------|---------|
| `tenant` | the scoping identifier (matches `participant.tenant`) |
| `community_id` | the EC-ID issued by the network operator (`AT00300...`) |
| `rc_number` | invoicing number (`RC...` or `TE...`) |
| `allocation_model` | `DYNAMIC` or `STATIC` |
| `billing_period` | `MONTHLY` / `QUARTERLY` / `BIANNUAL` / `YEARLY` |

Important: `tenant` and `community_id` may or may not be equal. In some legacy setups `tenant` is a free-form ID and `community_id` is the EDA-issued EC-ID. The customer SPA always passes `community_id` (as `ecId`) on the URL, so user-attribute `tenant` must equal `community_id` for the SPA to function.

### `base.processhistory`

Append-only log of every interaction (UI action, EDA inbound, billing event). This table grows continuously with platform activity; queries must be filtered (do not `SELECT *` without a `date`/`tenant` predicate).

### `eda.*`

Holds raw and merged EDA message state, keyed by conversation ID. Used both for inbound merging in eda-xp and for the message history view in the customer SPA.

### `billingj.*`

| Table | Purpose |
|-------|---------|
| `billing_run` | the per-period run with `status` and `mail_status` |
| `billing_document` | individual generated invoices / credit notes |
| `line_item` | per-Zählpunkt line items |

`billing_run.mail_status` is a one-shot flag — once a billing run has been emailed, the UI hides the re-send button. To re-send, update the column to `NULL` directly; the next mail action then targets **all** recipients again.

## Energy data — energystore + BadgerDB

Energy time-series are not in PostgreSQL. **energystore** uses an embedded BadgerDB KV store on a PVC (container path `/opt/rawdata`; production overlay `/opt/energy/rawdata`).

Storage shape:

- One bucket per EEG tenant.
- Each bucket has consumer and producer matrices, indexed by metering-point and period (quarter-hour slot).
- Per-period slots: `consumer[0..2]` (consumption, G.02, G.03) and `producer[0..1]` (production, P.01).

The energy data shape is documented in detail in [reference/obis-codes](../reference/obis-codes.md). See also [services/energystore](../services/energystore.md) for the on-disk format and replication caveats.

## Related

- [services/backend](../services/backend.md) — `base` and `eda` schema details
- [services/billing](../services/billing.md) — Flyway migrations, billing-run state
- [services/energystore](../services/energystore.md) — BadgerDB storage format
- [services/postgres](../services/postgres.md) — cluster setup, user roles
- [reference/obis-codes](../reference/obis-codes.md) — energy-data semantics
