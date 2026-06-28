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

Each service's required PostgreSQL extensions must be enabled. The most common one is `uuid-ossp` (for UUID generation in `base` schema defaults). A missing `uuid-ossp` causes `gen_random_uuid()` or `uuid_generate_v4()` calls in INSERTs to fail at runtime.

The bootstrap Job creates the extension if it does not exist; ensure the bootstrap user has the right privilege.

## Users and privileges

Each application service has its own database user, scoped to its schemas. Connection details come from `Secret`s with consistent names per service:

| Secret | Service |
|--------|---------|
| `backend-db-secret` | backend |
| `billing-db-secret` | billing |
| `filestore-db-secret` | filestore |
| `keycloak-db-secret` | keycloak (own database) |

Schema creation requires a higher-privilege user (granted `CREATE` on the database and ownership of the per-service schemas); the per-service users are then used for normal operation.

## Persistence

PostgreSQL uses a RWO PVC. `base.processhistory` and the `eda.*` workflow tables grow with EDA traffic; `base.processhistory` has no retention policy yet (partitioning + retention are planned). Deleting the PVC destroys the database.

## postgres-energy (TimescaleDB, energystore-v2 only)

A separate Postgres deployment hosts the TimescaleDB-backed time-series data for [energystore-v2](energystore-v2.md). Architectural choices:

- **Hash-partitioned hypertable** on `tenant_id` for multi-tenant write isolation.
- **Compression policy** on chunks past a configurable age.
- **Continuous aggregates** for the hourly / daily / monthly rollups that settlement queries consume.
- **pgBouncer** in front in production deployments.

The pilot uses a plain Postgres StatefulSet. Production targets **CloudNativePG (CNPG)** with primary + async read-replica. CNPG is referenced in `konzept.md` §4 but not yet deployed (see `feedback_cnpg_not_yet_in_stack`).

## Operational notes

- The `eda` schema is the second-largest after `base` in production instances. `eda.message` (or similar) grows linearly with EDA traffic.
- `base.processhistory` benefits from periodic archival in long-running instances.
- Keycloak's `offline_user_session` grows with idle sessions; periodic cleanup is recommended.

## fsGroup caveat

On some clusters / storage classes, the PostgreSQL container's `fsGroup` setting must match the PVC's mount group, otherwise the PG data directory is not writable. Misconfigured fsGroup is a frequent first-boot failure.

## Related

- [Architecture / Databases](../architecture/databases.md) — schema layout, dump-vs-Flyway, notable tables
- [services/keycloak](keycloak.md) — uses its own DB
