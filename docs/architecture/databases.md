# Databases

eegfaktura runs against **one or two PostgreSQL clusters**, depending on whether [energystore-v2](../services/energystore-v2.md) is deployed:

- **`postgres`** — master data plus Keycloak's own logical database (in production also a `pontonxp` database of the external Ponton X/P messenger, which no eegfaktura component uses). Always present. In the local stack this is a single custom PostgreSQL image (`eegfaktura-postgresql`) that provisions both an `eegfaktura` application database and a separate `keycloak` database.
- **`postgres-energy`** — a separate, optional second cluster (TimescaleDB) for energystore-v2. energystore-v2 lives in a separate repo and is not rolled out yet; it is not part of the production manifests.

For the v1 stack only the first cluster exists; energystore v1 stores time-series in an **embedded BadgerDB KV store on a persistent volume**, not in PostgreSQL.

## Logical layout

```
postgres cluster (custom image)
├── eegfaktura database
│   ├── base       — backend (Go): EEGs, participants, metering points, tariffs, process history
│   ├── eda        — eda-xp (Scala): mail/Ponton tenant config, inbox/outbox, conversations
│   ├── billingj   — billing (Java): billing config, runs, documents + items, generated PDFs
│   └── filestore  — filestore (Python): file metadata (blobs on local filesystem)
└── keycloak database — Keycloak identity data (separate logical DB)

postgres-energy cluster (energystore-v2 only; TimescaleDB)
└── energystore-v2 database (default schema)
    ├── energy_data         — hypertable on ts, space-partitioned by hash of tenant_id
    ├── counterpoint_meta   — per-meter metadata
    ├── energy_hourly / energy_daily — continuous aggregates
    ├── mqtt_dlq            — dead-letter queue for MQTT messages
    └── schema_migrations   — applied migration files
```

The exact main-database name is environment-specific (e.g. `eegfaktura`). Each service takes it from its own config (filestore `DB_DATABASE`, billing `JDBC_DATABASE_URL`, backend `database.dbname` in its config / `VFEEG_BACKEND_DATABASE_DBNAME`, admin-backend `DATABASE_DBNAME`, eda-xp `slick.pgsql.local.db.url`) rather than hard-coding.

## Why a separate cluster for time-series

Conceptually:

- **Access patterns** are different: time-series ingest is high-volume write of small rows with no joins; OLTP master data has lower volume with rich joins. Sharing one cluster forces trade-offs that fit neither workload.
- **Operational independence**: a long-running aggregate query on time-series data should not impact master-data OLTP latency, and vice versa.

The `energy_data` table (one row per `tenant_id, ec_id, metering_point, meter_code, ts`) is a TimescaleDB **hypertable** on `ts` (7-day chunks) with a hash space partition on `tenant_id` (16 partitions). Rows of a given tenant fall in one partition; the partition count spreads concurrent tenant writes. Chunks older than 30 days are compressed (segmented by `tenant_id, ec_id, metering_point, meter_code`). Migrations are embedded SQL files (`migrations/0001_init.sql`, `0002_mqtt_dlq.sql`) applied by the service's own runner (`internal/migrate`, recorded in `schema_migrations`, serialised by an advisory lock). See the [postgres-energy section in the postgres service page](../services/postgres.md#postgres-energy-timescaledb-energystore-v2-only) for the architectural decisions.

## Schema ownership

Each schema is owned by exactly one service. Other services do not write to it.

| Schema | Owner | Notes |
|--------|-------|-------|
| `base` | backend | Largest and most central schema. Master data and EDA process history. Auto-migrated on startup (golang-migrate). Read by billing (view `base.billing_masterdata`), admin-backend and eda-xp (`base.eeg`, to fill a missing `ecId`). |
| `eda` | eda-xp | Tables `tenantconfig`, `inbox`, `outbox`, `conversation`. Slick for access, Flyway for migrations (`defaultSchema("eda")`, `baselineOnMigrate`). backend does not use this schema. |
| `billingj` | billing | Flyway-migrated (`ddl-auto=validate`). Pre-existing tables without Flyway history stop the start — see [below](#flyway-and-pre-existing-billingj-tables). |
| `filestore` | filestore | File metadata only (Alembic). Five tables: `file_categories`, `storages`, `file_containers`, `files`, `file_attributes`. Blobs live on the local filesystem, not in the database. |
| `keycloak` (separate DB) | Keycloak | Keycloak's own logical database; managed entirely by Keycloak, no application access. |

energystore (v1) does **not** use PostgreSQL at all: it has no database driver and stores energy time-series and metering-point metadata in its own BadgerDB KV store on a PVC.

admin-backend uses Slick and ships **no** Flyway/migrations in-repo. It only reads `base.eeg`, `base.participant`, `base.meteringpoint` and `base.gridoperators`; registering an EEG goes to the backend via gRPC (`RegisterEegService`), not through SQL.

eda-xp additionally keeps a Pekko Persistence journal and snapshots on local disk (LevelDB, `storage/prod/journal`, used by `PrepareMessageActor`); this is not in PostgreSQL.

## Connection model

In the GitOps manifests the database credentials come from a Kubernetes Secret (host and some user names from ConfigMaps); in the local stack they are docker-compose environment variables. The code does not assume per-schema database users: in the local stack (`eegfaktura-docker-compose`) all application services connect as the same application user.

| Service | Schema scope | Migration tool |
|---------|--------------|----------------|
| backend | `base` (write) | golang-migrate v4 (embedded `migrations/*.sql`, version table in `base`), auto-run on startup; migrations are generated with Atlas from `schema.sql` (`atlas migrate diff`) |
| billing | `billingj` (write), `base.billing_masterdata` (read) | Flyway via Spring Boot (`db/migration/V1_*`, `spring.flyway.default-schema=billingj`, `ddl-auto=validate`) |
| filestore | `filestore` | Alembic, `alembic upgrade head` in the entrypoint |
| eda-xp | `eda` (write), `base.eeg` (read) | Flyway 10.20.0 (`db/migration`, `defaultSchema("eda")`, `baselineOnMigrate`), run by `MigrationRunner` at startup; Slick 3.5.1 for access |
| admin-backend | `base` (read via Slick) | none (no in-repo migrations) |

## Flyway and pre-existing `billingj` tables

billing's Flyway has no baseline-on-migrate: if `billingj.*` tables exist without Flyway's history table (e.g. from a schema dump), billing fails at startup instead of adopting them.

## Notable tables

Column names in `base` mix snake_case and quoted camelCase (e.g. `"rcNumber"`, `participant_id`); the authoritative definition is the backend's `schema.sql`.

### `base.participant`

The member master record. Joined into nearly every customer-facing view.

| Column | Purpose |
|--------|---------|
| `id` | UUID primary key |
| `tenant` | the EEG's `tenant` — the scoping field |
| `"participantNumber"`, `firstname`, `lastname` | member number and name |
| `"businessRole"` | participant kind (`EEG_PRIVATE` / `EEG_BUSINESS`) |
| `status` | a `ProcessStatusType` value (`model/participant.go`); the backend sets `PENDING` on `POST /participant` and `ACTIVE` on confirm (column default `NEW`) |
| `"tariffId"` | participant tariff |

Contact data lives in child tables keyed by `participant_id`: `base.contactdetail` (`email`, `phone`), `base.address` (`type` `RESIDENCE` / `BILLING`) and `base.bankaccount` (IBAN, SEPA mandate). For member-self lookups the backend matches `base.contactdetail.email` case-insensitively against the JWT `email` claim.

### `base.meteringpoint` (Zählpunkt)

Primary key `(metering_point_id, tenant, participant_id)`; a partial unique index allows one row with `flag = 1` per `(metering_point_id, tenant)`.

| Column | Purpose |
|--------|---------|
| `metering_point_id` | the AT-prefixed ID issued by the network operator |
| `participant_id` | FK to participant |
| `tenant` | scoping field |
| `direction` | `CONSUMPTION` or `GENERATION` |
| `status` | `StatusType`: `INIT`, `ACTIVE`, `INACTIVE` |
| `process_state`, `last_process_state` | EDA process state, `ProcessStatusType` (`model/participant.go`): `NEW`, `INIT`, `PENDING`, `APPROVED`, `ACTIVE`, `INACTIVE`, `REJECTED`, `REVOKED`, `INVALID`, `ARCHIVED`, `ABORTED`, `RESTORE`, `WARN` |
| `consent_id` | EDA consent id |
| `tariff_id` | metering-point tariff |

The participation factor is versioned in `base.metering_partition_factor` (`"partFact"`, one row per version); the view `base.activeMeteringPartition` returns the latest version per metering point and participant.

### `base.eeg`

| Column | Purpose |
|--------|---------|
| `tenant` | primary key; the scoping identifier compared with the token's `tenant` claim (matches `participant.tenant`) |
| `"communityId"` | the EC-ID issued by the network operator (`AT…`) |
| `"rcNumber"` | RC number (`RC...`, `TE...`, `GC...`, `CC...`) |
| `area` | `LOCAL` / `REGIONAL` / `BEG` / `GEA` |
| `"allocationMode"` | `DYNAMIC` or `STATIC` |
| `"settlementInterval"` | `MONTHLY` / `QUARTER` / `BIANNUAL` / `ANNUAL` |

Important: `tenant`, `"rcNumber"` and `"communityId"` are separate values and may differ (e.g. GEA tenants `GCxxxxxx-00n` share one RC number stem). The customer SPA sends the tenant in a `tenant` header to backend, billing and filestore, but to energystore `communityId` as `ecId` in the URL and `"rcNumber"` as the `X-Tenant` header; do not assume one can be derived from the other (see [services/web](../services/web.md#tenant-vs-ecid)).

### `base.processhistory`

Log of EDA process messages written by the backend: one row per message it receives over MQTT (including the echo of its own requests and energystore's `cr_msg_history`) with `"conversationId"`, `type` (message code), `issuer`, `direction`, `protocol`, `date` and the message as JSON. The column default of `direction` is `OUT`, but the backend writes every row with `IN`; UI actions are not logged here. This table grows continuously with platform activity; queries must be filtered (do not `SELECT *` without a `date`/`tenant` predicate).

Other `base` tables: `tariff` (versioned, PK `(id, version)`, view `base.activeTariff`), `notification`, `gridoperators`. The view `base.billing_masterdata` is the read interface for billing.

### `eda.*`

Owned by eda-xp (Flyway migrations in `src/main/resources/db/migration`):

| Table | Purpose |
|-------|---------|
| `tenantconfig` | per-tenant transport config (`type`, default `MAIL`; mail/IMAP/SMTP settings) |
| `inbox` / `outbox` | received / sent raw messages per tenant (`content` as `bytea`) |
| `conversation` | conversation state as JSON, keyed by conversation id |

The message history view in the customer SPA comes from the backend (`base.processhistory`), not from `eda.*`.

### `billingj.*`

| Table | Purpose |
|-------|---------|
| `billing_config` | per-tenant document texts, number ranges, header/footer images, custom template |
| `billing_run` | the per-period run with `run_status`, `mail_status`, `sepa_status`, `error_summary`; unique per `(tenant_id, clearing_period_type, clearing_period_identifier)` |
| `billing_document` | individual generated invoices / credit notes (issuer and recipient data copied in) |
| `billing_document_item` | line items, per metering point (`metering_point_id`, tariff fields) |
| `billing_document_number` | document number sequences per tenant |
| `billing_document_file`, `file_data` | generated PDFs and images; the bytes are stored in `file_data.data` (large object `oid`) |

`billing_run.mail_status` is a one-shot flag: `sendAllBillingDocuments` refuses to run when it is non-empty, sets it to `IN PROGRESS` and finally to `SENT`. A send always targets **all** documents of the run.

## Energy data — energystore + BadgerDB

Energy time-series are not in PostgreSQL. **energystore** uses an embedded BadgerDB (v4) KV store on a PVC (`persistence.path`; `config-prod.yaml` sets `/opt/energy/rawdata`, the image also declares a volume `/opt/rawdata`).

Storage shape:

- One Badger database per tenant and EC-ID (`<persistence.path>/<tenant>/<ecId>`), opened through a connection pool.
- Buckets `rawdata` (time series) and `metadata` (metering-point metadata under `cpmeta/0`: index and direction per metering point).
- One `rawdata` row per quarter-hour slot (key `CP/YYYY/MM/DD/HH/mm/ss`) holds the values of **all** metering points of the EC in shared consumer and producer arrays, addressed by each metering point's index.
- Per metering point: consumer slots `[0..2]` (consumption G.01, G.02, G.03) and producer slots `[0..1]` (production G.01, P.01), plus parallel quality-of-value arrays.

The energy data shape is documented in detail in [reference/obis-codes](../reference/obis-codes.md). See also [services/energystore](../services/energystore.md) for the on-disk format and replication caveats.

## Related

- [services/backend](../services/backend.md) — `base` schema details
- [services/eda-xp](../services/eda-xp.md) — `eda` schema (owned by eda-xp)
- [services/billing](../services/billing.md) — Flyway migrations, billing-run state
- [services/energystore](../services/energystore.md) — BadgerDB storage format
- [services/postgres](../services/postgres.md) — cluster setup, user roles
- [reference/obis-codes](../reference/obis-codes.md) — energy-data semantics
