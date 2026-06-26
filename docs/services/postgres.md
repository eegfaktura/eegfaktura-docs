# postgres

PostgreSQL cluster. Hosts the main application database (multiple schemas), the Keycloak database, and optionally a PONTON-adapter database.

!!! info "A second Postgres cluster for time-series data"
    With [energystore-v2](energystore-v2.md) in the pilot, there is a separate **`postgres-energy`** Postgres cluster with the **TimescaleDB extension** that holds energy time-series only. The main `postgres` cluster documented on this page continues to hold master data (`base`, `eda`, `billingj`, `filestore`).

    The split is deliberate (ADR-0010 in the platform repo): time-series ingest has fundamentally different access patterns from OLTP master data, and TimescaleDB's hash-partitioning + compression match the time-series workload.

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-postgresql:0.2.0` |
| Topology | typically single instance for dev; managed cluster (CloudNativePG) for production |
| Storage | PVC (RWO) |
| Schemas | `base`, `eda`, `billingj`, `filestore` (in the main DB) |

!!! note "The local stack uses a custom Postgres image"
    The local docker-compose stack runs `ghcr.io/eegfaktura/eegfaktura-postgresql:0.2.0`, **not** an upstream `postgres:*` image. This custom image performs its own initialization internally: on first start it creates the `eegfaktura` application database and the `keycloak` database (via `DB_*` / `KEYCLOAK_DB_*` env vars) and initializes the data directory with locale `de_DE:UTF8` (`POSTGRES_INITDB_ARGS`). Locally it is published on host port `26432` (→ container `5432`). Production may use a different image and topology (see CloudNativePG below).

For the per-service schema layout, see [Architecture / Databases](../architecture/databases.md).

## Logical databases

| Database | Used by | Notes |
|----------|---------|-------|
| `<main-database>` | backend, billing, filestore, eda-xp | name is env-specific |
| `keycloak` | keycloak | managed entirely by Keycloak |
| `pontonxp` | PONTON adapter | only when a real PONTON link is wired |

## Schema ownership

| Schema | Owner | Migration tool |
|--------|-------|----------------|
| `base` | backend | Go-native runner / pg_dump-restore |
| `eda` | backend + eda-xp | shared |
| `billingj` | billing | Flyway |
| `filestore` | filestore | Python migration runner |

A `pg_dump --schema-only` restore conflicts with billing's Flyway. See the "schema-dump vs Flyway" section in [Architecture / Databases](../architecture/databases.md) for the `DROP SCHEMA billingj CASCADE` workaround.

## Extensions

The bootstrap chart enables what each service needs. The most common extension required is `uuid-ossp` (for UUID generation in `base` schema defaults). A missing `uuid-ossp` causes `gen_random_uuid()` or `uuid_generate_v4()` calls in INSERTs to fail at runtime.

The bootstrap Job creates the extension if it does not exist; ensure the bootstrap user has the right privilege.

## Users and privileges

Each application service has its own database user, scoped to its schemas. Connection details come from `Secret`s with consistent names per service:

| Secret | Service |
|--------|---------|
| `backend-db-secret` | backend |
| `billing-db-secret` | billing |
| `filestore-db-secret` | filestore |
| `keycloak-db-secret` | keycloak (own database) |

A separate **bootstrap user** with broader privileges runs the schema-creation Job. It is granted `CREATE` on the database and ownership of the per-service schemas after creation, then handed off to the per-service user.

## Bootstrap

The `db-schema` Job in `eegfaktura-bootstrap`:

1. `initContainer` `pg_isready` + `psql -tAc 'SELECT 1'` against the application database.
2. `DROP SCHEMA IF EXISTS base CASCADE` (idempotent against partial state).
3. Apply the schema dump or migration set.
4. Grant per-schema privileges to the per-service users.

This Job is intentionally idempotent — a failed attempt that left a partially-created schema can be re-run. The `DROP SCHEMA IF EXISTS` step was added explicitly because earlier Job attempts could leave the schema half-created on the first failure, causing all subsequent retries to fail with `relation already exists`.

## Persistence

PostgreSQL uses a RWO PVC. `base.processhistory` and the `eda.*` workflow tables grow with EDA traffic; `base.processhistory` has no retention policy yet — see [eegfaktura-backend issue #105](https://github.com/gemeinstrom/eegfaktura-backend/issues/105) for the partitioning + retention plan. Wipe-replay destroys the PVC along with the namespace.

## postgres-energy (TimescaleDB, energystore-v2 only)

A separate Postgres deployment hosts the TimescaleDB-backed time-series data for [energystore-v2](energystore-v2.md). Architectural choices:

- **Hash-partitioned hypertable** on `tenant_id` for multi-tenant write isolation.
- **Compression policy** on chunks past a configurable age.
- **Continuous aggregates** for the hourly / daily / monthly rollups that settlement queries consume.
- **pgBouncer** in front in production deployments.

The pilot uses a plain Postgres StatefulSet. Production targets **CloudNativePG (CNPG)** with primary + async read-replica + WAL-archive-to-S3. CNPG is referenced in `konzept.md` §4 but not yet deployed (see `feedback_cnpg_not_yet_in_stack`).

## Backups

Backup strategy is deployment-specific. The supported pattern is CloudNativePG with WAL streaming to S3-compatible object storage. Manual `pg_dump` is fine for dev / small instances.

A wipe-replay does not restore from backup — it re-applies the schema dump and the sample-data Job. To recover from a backup after a wipe, restore the PVC (or restore from CNPG WAL) before running `bootstrap`.

## Operational notes

- The `eda` schema is the second-largest after `base` in production instances. `eda.message` (or similar) grows linearly with EDA traffic.
- `base.processhistory` benefits from periodic archival in long-running instances.
- Keycloak's `offline_user_session` grows with idle sessions; periodic cleanup is recommended.

## fsGroup caveat

On some clusters / storage classes, the PostgreSQL container's `fsGroup` setting must match the PVC's mount group, otherwise the PG data directory is not writable. Misconfigured fsGroup is a frequent first-boot failure. The `pre-wipe-check.sh` script in the provisioning pipeline includes an fsGroup audit.

## Related

- [Architecture / Databases](../architecture/databases.md) — schema layout, dump-vs-Flyway, notable tables
- [Operations / Pipeline](../operations/pipeline.md) — bootstrap order
- [services/keycloak](keycloak.md) — uses its own DB
