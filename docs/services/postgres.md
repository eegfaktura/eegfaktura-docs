# postgres

PostgreSQL cluster. Hosts the main application database (multiple schemas), the Keycloak database, and optionally a PONTON-adapter database.

## At a glance

| | |
|---|---|
| Image | PostgreSQL (e.g. `postgres:18-alpine` in current deployments) |
| Topology | typically single instance for dev; managed cluster (CloudNativePG) for production |
| Storage | PVC (RWO) |
| Schemas | `base`, `eda`, `billingj`, `filestore` (in the main DB) |

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

## PVC and sizing

PostgreSQL uses a RWO PVC. In production this PVC is large (450 GiB in historical instances), dominated by `base.processhistory` (tens of GB / hundreds of millions of rows over the years).

Wipe-replay destroys the PVC along with the namespace.

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
