# filestore

Python service. Stores member documents (contracts uploaded through the web UI; the categories are `contract` and `invoice`) with their metadata and serves authenticated downloads.

## At a glance

| | |
|---|---|
| Language | Python 3.13 (image base `python:3.13-slim-bookworm`) |
| Framework | FastAPI 0.138.2 (Starlette 1.3, Pydantic v2 + `pydantic-settings`, uvicorn 0.49.0); GraphQL via Strawberry 0.320.0 |
| State | PostgreSQL schema `filestore` (SQLAlchemy 2.0.51 async + asyncpg 0.31.0, Alembic 1.18.5 migrations) + blobs on a local filesystem |
| Auth | PyJWT 2.15.0 (RS256) |
| Consumed by | web (contract documents of a member: list, upload, download, delete) |

## Responsibilities

- Accept uploads (GraphQL `addFile`, REST `POST /filestore/upload/…`)
- Serve downloads to authenticated callers of the owning tenant
- Track file metadata: tenant, user (participant) id, category, original name, extension, media type, size, created-at

billing does not use filestore: it keeps its generated PDFs in its own schema (`billingj.file_data`).

## Storage model

The `filestore` schema in PostgreSQL holds per-file metadata. Blobs are stored on a local filesystem (a PVC); there is no S3 support in the code.

On disk, blobs are laid out as:

```
{FILESTORE_LOCAL_BASE_DIR}/{storage_id}/{file_container_id}/{file_id}
```

All three segments are UUIDs — no user-controlled path segments are interpolated.

Allowed upload media types are a hardcoded whitelist: `application/pdf`, `image/jpeg`, `image/png`.

### Provisioning (since 1.0.6)

- File categories are a fixed list in code (`allowed_file_categories` in `app/file_metadata.py`: `contract`, `invoice`). A GraphQL upload with any other category is rejected ("Invalid file category"); the REST route only accepts the two enum values. A missing row for one of the two names is created in `file_categories` on first use.
- A tenant's storage and its per-category container are created on its first upload, **after** the tenant assertion. There is one storage per tenant: a new category reuses the existing storage (lowest id). The former `FILESTORE_CREATE_UNKNOWN_*` flags no longer exist.
- GEA sub-community tenants (`GCxxxxxx-00n`) are normalised to their stem (`GCxxxxxx`) for storage and for the tenant check (`base_tenant` in `app/auth.py`), so all sub-communities of a GEA share one storage.
- New directories are created with mode `0o750`; the image runs as non-root user `app` (UID/GID 1000).

## Auth

Filestore verifies JWTs with PyJWT, `algorithms=["RS256"]` only (HS256 and `alg=none` get 401). `exp` is required (`options={"require": ["exp"]}`). The public key is loaded from a file at import time (the service fails fast if it is missing; the key is not baked into the image since 1.0.4). JWT decoding validates the `aud` claim against `JWT_AUDIENCE` (default `account`); `iss` is not checked.

The tenant claim in the token is the JSON key `tenant` (singular); the Python model exposes it as `tenants: List[str]` (alias `tenant`). On single-item lookups the service loads the record, then asserts the tenant; list queries and uploads assert the requested tenant first. `assert_tenant` compares case-insensitively (both sides uppercased, after GEA stem normalisation) and raises 403 on mismatch. The realm role `superuser` (in `realm_access.roles`) skips the tenant check, mirroring the backend.

Two operational caveats:

### PyJWT needs the public key, not the X.509 cert

PyJWT does not accept an X.509 certificate as the verification key. The configured key must be the raw public key (no certificate wrapper).

Extract from a cert:

```sh
openssl x509 -pubkey -noout -in zertifikat-pub.pem > zertifikat-pubkey.pem
```

The resulting `BEGIN PUBLIC KEY` block is what PyJWT consumes.

### Tokens need a matching `aud`

`get_claims` passes `audience=settings.JWT_AUDIENCE` to `jwt.decode`, so a token without `aud`, or whose `aud` does not contain that value, gets 401. Tokens intended to reach filestore must carry the configured audience.

## Endpoints

The service exposes both a REST API under `/{HTTP_FILE_DL_ENDPOINT}` (default `/filestore`) and a GraphQL endpoint at `/graphql` (GraphiQL toggled via `GRAPHIQL_ENABLED`; multipart uploads explicitly enabled). It listens on `0.0.0.0` (hardcoded in `main.py`), port `HTTP_PORT` (default 5000). Every route requires a bearer token.

REST (`app/rest/routers/filestore.py`):

| Route | Purpose |
|-------|---------|
| `GET /filestore/` | always 400 ("No parameter given") |
| `GET /filestore/{file_id}` | download a file (tenant asserted after loading the metadata) |
| `POST /filestore/upload/{file_category}/{tenant}/{user_id}` | multipart upload; `file_category` is `contract` or `invoice` |

GraphQL (`app/graphql/schemas/`, camelCase field names):

- Queries: `files(tenant, attributes, category, userId, limit, offset)`, `invoices(tenant, userId, …)`, `contracts(tenant, userId, …)`, `rcContracts(tenant, …)`, `file(id)`, `invoice(id)`, `contract(id)`
- Mutations: `addFile(file, name, fileCategory, tenant, attributes, userId)`, `deleteFile(fileId)`, `readFile(file)` (echoes file name and content type; no storage, no tenant check)

`addFile` rejects attributes that use one of the default keys (`orig_file_name`, `file_extension`, `media_type`, `file_bytes`) before anything is written. Download URIs in responses are built from `HTTP_FILE_DL_BASE_URI`.

## Database

Schema `filestore`, five tables:

| Table | Holds |
|-------|-------|
| `file_categories` | file category definitions; fixed list `contract`, `invoice` (since 1.0.6 any other category is rejected) |
| `storages` | storage roots; one per community, created on its first upload |
| `file_containers` | containers grouping files; one per community and category, created on first use |
| `files` | per-file metadata (`tenant`, `user_id`, `name`, `file_container_id`, `created_at`) |
| `file_attributes` | key/value attributes per file (PK `file_id`, `key`); the service writes `orig_file_name`, `file_extension`, `media_type`, `file_bytes` |

There is no audit table. Two Alembic revisions exist: `e7e81ce21ac0` (initial layout, tenant columns as `varchar(8)` named `community_id`) and `c40127f5c680` (renames `community_id` to `tenant`). The Alembic version table is `filestore.alembic_version`.

Migrations are run by Alembic (`alembic upgrade head`) on startup in `entrypoint.sh`, before `python3 main.py` launches uvicorn.

## Config

| Variable | Purpose |
|----------|---------|
| `DB_HOSTNAME`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | PG connection |
| `DB_PASSWORD_FILE` | file the entrypoint reads the DB password from (when `DB_PASSWORD` is unset) |
| `HTTP_PROTOCOL`, `HTTP_HOSTNAME`, `HTTP_PORT` | build the default download base URI (`{protocol}://{hostname}:{port}`); `HTTP_PORT` (default 5000) is also the listen port, the bind address is always `0.0.0.0` |
| `HTTP_FILE_DL_ENDPOINT` | REST route prefix and download path segment (default `filestore`) |
| `HTTP_FILE_DL_BASE_URI` | download base URI returned to clients (default `{HTTP_BASE_URI}/{HTTP_FILE_DL_ENDPOINT}`) |
| `FILESTORE_LOCAL_BASE_DIR` | directory where blobs are stored (config.py default `/vfeeg-filestore-data`; the Dockerfile sets `/eegfaktura-filestore-data` — note the inconsistency) |
| `FILESTORE_TEMP_DIR` | temp directory for in-progress uploads (default `{FILESTORE_LOCAL_BASE_DIR}/tmp`) |
| `JWT_KEY_FILE` | path to the PyJWT-compatible RS256 public key PEM (default `jwt_pub_key.pem`). It is the default of the settings field `JWT_PUBLIC_KEY_FILE`; pydantic-settings also reads `JWT_PUBLIC_KEY_FILE` directly, which then takes precedence |
| `JWT_AUDIENCE` | expected `aud` claim (default `account`) |
| `GRAPHIQL_ENABLED` | enable the GraphiQL UI (`true` only) |
| `APP_LOG_LEVEL` | read into settings, but currently unused (the uvicorn `log_level` argument is commented out) |

## Build and image

- Source: `eegfaktura-filestore`
- Build: Docker (`python:3.13-slim-bookworm` base, runs as non-root `app`, UID/GID 1000); `requirements.txt` pins all dependencies including transitive ones
- CI (`pr-checks.yml`): byte-compile and import; Alembic `upgrade head` + `alembic check`, pytest (`tests/`, FastAPI TestClient against a migrated PostgreSQL; covers 401/403 cases, categories, GEA stem, superuser) and an application start; ruff lint (`F`, `E7`, `B`); a check that every requirement is pinned. `security-scan.yml` runs Gitleaks, Trivy and OSV-Scanner.

## Source lineage caveat

The production-deployed filestore historically lagged the public-clone source by years. When recovering or upgrading, verify the running image's `.git` config to see which upstream produced it, rather than assuming the public mirror.

## Related

- [Architecture / Authentication](../architecture/auth.md) — PyJWT caveats
- [services/billing](billing.md) — main producer of files
- [services/web](web.md) — consumer
