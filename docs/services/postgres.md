# postgres

PostgreSQL cluster. Hosts the main application database (multiple schemas), the Keycloak database, and optionally a PONTON-adapter database.

!!! info "A second Postgres cluster for time-series data"
    For [energystore-v2](energystore-v2.md) (planned successor, not rolled out yet) there is a separate **`postgres-energy`** Postgres cluster with the **TimescaleDB extension** that holds energy time-series only. The main `postgres` cluster documented on this page continues to hold master data (`base`, `eda`, `billingj`, `filestore`).

    The split is deliberate: time-series ingest has fundamentally different access patterns from OLTP master data, and TimescaleDB's hash-partitioning + compression match the time-series workload.

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-postgresql:0.2.0` |
| Topology | single instance in the local stack; the production Postgres is a separate image of the organisation and not part of the application manifests |
| Storage | PVC (RWO) |
| Schemas | `base`, `eda`, `billingj`, `filestore` (in the main DB) |

!!! note "The local stack uses a custom Postgres image"
    The local docker-compose stack runs `ghcr.io/eegfaktura/eegfaktura-postgresql:0.2.0`, **not** an upstream `postgres:*` image. This custom image performs its own initialization internally: on first start it creates the `eegfaktura` application database and the `keycloak` database (via `DB_*` / `KEYCLOAK_DB_*` env vars) and initializes the data directory with locale `de_DE:UTF8` (`POSTGRES_INITDB_ARGS`). Locally it is published on host port `26432` (→ container `5432`). Production may use a different image and topology (see CloudNativePG below).

For the per-service schema layout, see [Architecture / Databases](../architecture/databases.md).

## Logical databases

| Database | Used by | Notes |
|----------|---------|-------|
| `<main-database>` | backend, billing, filestore, eda-xp, admin-backend | name is env-specific (`eegfaktura` in the local stack) |
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

Each service's required PostgreSQL extensions must be enabled. The most common one is `uuid-ossp`: the `base` schema defaults use `uuid_generate_v4()`. backend's first migration (`database/migration/000001_init-setup.up.sql`) and eda-xp's `migration.sql` run `CREATE EXTENSION IF NOT EXISTS "uuid-ossp"` themselves, so either the database user may create it or it must already exist. filestore uses the built-in `gen_random_uuid()` and needs no extension.

## Users and privileges

The application services share **one** database user on the main database (`eegfaktura` in the local stack, created by the custom image from `DB_USERNAME` / `DB_PASSWORD_FILE`); Keycloak has its own user and database (`KEYCLOAK_DB_*`). In the GitOps manifests the connection details come from two Secrets (names only):

| Secret | Used by |
|--------|---------|
| `eegfaktura-postgresql-eegfaktura-secret` (keys `username`, `password`, `database`) | backend, billing, filestore, eda-xp, admin-backend |
| `eegfaktura-postgresql-keycloak-secret` | keycloak (own database) |

Because all services use the same user, schema separation is a convention of the code (each service migrates and writes its own schema), not a privilege boundary.

## postgres-energy (TimescaleDB, energystore-v2 only)

A separate Postgres deployment hosts the TimescaleDB-backed time-series data for [energystore-v2](energystore-v2.md). Architectural choices:

- **Hash-partitioned hypertable** on `tenant_id` for multi-tenant write isolation.
- **Compression policy** on chunks past a configurable age.
- **Continuous aggregates** for the hourly / daily / monthly rollups that settlement queries consume.
- **pgBouncer** in front in production deployments.

energystore-v2 is not rolled out yet; the design targets **CloudNativePG (CNPG)** with primary + async read-replica. CNPG is not deployed yet.

## Related

- [Architecture / Databases](../architecture/databases.md) — schema layout, dump-vs-Flyway, notable tables
- [services/keycloak](keycloak.md) — uses its own DB
