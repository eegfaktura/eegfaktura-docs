# filestore

Python service. Stores generated documents (mainly billing PDFs) and serves member-side downloads.

## At a glance

| | |
|---|---|
| Language | Python 3.10 |
| Framework | FastAPI 0.95.1 (uvicorn 0.21.1); GraphQL via Strawberry 0.172.0 |
| State | PostgreSQL schema `filestore` (SQLAlchemy 2.0.9 async + asyncpg, Alembic migrations) + blob storage on a PVC |
| Auth | PyJWT 2.12.1 (RS256) |
| Consumed by | web (member download links), billing (writes generated docs) |

## Responsibilities

- Accept uploads from billing (generated PDFs)
- Serve downloads to authenticated members
- Track file metadata: owner, EEG, content-type, size, created-at

## Storage model

The `filestore` schema in PostgreSQL holds per-file metadata. Blobs are stored on a local filesystem (a PVC); there is no S3 support in the code.

On disk, blobs are laid out as:

```
{FILESTORE_LOCAL_BASE_DIR}/{storage_id}/{file_container_id}/{file_id}
```

All three segments are UUIDs — no user-controlled path segments are interpolated.

Allowed upload media types are a hardcoded whitelist: `application/pdf`, `image/jpeg`, `image/png`.

## Auth

Filestore verifies JWTs with PyJWT using RS256. The public key is loaded from a file at import time (the service fails fast if it is missing). JWT decoding validates the `aud` claim against `JWT_AUDIENCE` (default `account`).

The tenant claim in the token is the JSON key `tenant` (singular); the Python model exposes it as `tenants: List[str]` (alias `tenant`). On single-item lookups the service loads the record, then asserts the tenant: `assert_tenant` compares case-insensitively (both sides uppercased) and raises 403 on mismatch.

Two operational caveats:

### PyJWT needs the public key, not the X.509 cert

PyJWT does not accept an X.509 certificate as the verification key. The configured key must be the raw public key (no certificate wrapper).

Extract from a cert:

```sh
openssl x509 -pubkey -noout -in zertifikat-pub.pem > zertifikat-pubkey.pem
```

The resulting `BEGIN PUBLIC KEY` block is what PyJWT consumes.

### PyJWT enforces `aud` even without `audience=`

PyJWT validates the `aud` claim by default. Even if the verification call does not pass an `audience=` parameter, PyJWT will fail tokens whose `aud` is missing or empty. The provisioning realm must configure an audience mapper that emits a valid `aud` for tokens intended to reach filestore.

## Endpoints

The service exposes both a REST API under `/filestore/*` and a GraphQL endpoint at `/graphql` (GraphiQL toggled via `GRAPHIQL_ENABLED`). It listens on `0.0.0.0:5000` (port from `HTTP_PORT`).

Routes vary by revision; check the deployed service for the authoritative list.

## Database

Schema `filestore`, five tables:

| Table | Holds |
|-------|-------|
| `file_categories` | file category definitions |
| `storages` | storage roots |
| `file_containers` | containers grouping files |
| `files` | per-file metadata |
| `file_attributes` | extra per-file attributes |

There is no audit table. (Historically, a migration renamed `community_id` to `tenant`.)

Migrations are run by Alembic (`alembic upgrade head`) on startup, before `python3 main.py` launches uvicorn.

## Config

| Variable | Purpose |
|----------|---------|
| `DB_HOSTNAME`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | PG connection |
| `DB_PASSWORD_FILE` | file the entrypoint reads the DB password from (when `DB_PASSWORD` is unset) |
| `HTTP_PROTOCOL`, `HTTP_HOSTNAME`, `HTTP_PORT` | HTTP server binding (port defaults to 5000) |
| `HTTP_FILE_DL_ENDPOINT` | download endpoint path segment |
| `FILESTORE_LOCAL_BASE_DIR` | PVC mount where blobs are stored (config.py default `/vfeeg-filestore-data`; the Dockerfile sets `/eegfaktura-filestore-data` — note the inconsistency) |
| `FILESTORE_TEMP_DIR` | temp directory for in-progress uploads |
| `FILESTORE_CREATE_UNKNOWN_CATEGORY`, `FILESTORE_CREATE_UNKNOWN_CONTAINER`, `FILESTORE_CREATE_UNKNOWN_STORAGE` | auto-create flags |
| `JWT_KEY_FILE` | path to the PyJWT-compatible RS256 public key PEM (maps to the internal `JWT_PUBLIC_KEY_FILE` setting — note the name mismatch) |
| `JWT_AUDIENCE` | expected `aud` claim (default `account`) |
| `GRAPHIQL_ENABLED` | enable the GraphiQL UI |
| `APP_LOG_LEVEL` | log level |

## Build and image

- Source: `eegfaktura-filestore`
- Build: Docker (`python:3.10-slim-bullseye` base)

## Operational notes

- The PVC is RWO; filestore is a single-replica deployment.
- Deleting the PVC destroys uploaded blobs.
- Source lineage drift: the production-deployed filestore historically lagged the public-clone source by years. When recovering or upgrading, verify the running image's `.git` config to see which upstream produced it, rather than assuming the public mirror.

## Related

- [Architecture / Authentication](../architecture/auth.md) — PyJWT caveats
- [services/billing](billing.md) — main producer of files
- [services/web](web.md) — consumer
