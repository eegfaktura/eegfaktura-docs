# filestore

Python service. Stores generated documents (mainly billing PDFs) and serves member-side downloads.

## At a glance

| | |
|---|---|
| Language | Python |
| Framework | Flask or FastAPI (depending on revision) |
| State | PostgreSQL schema `filestore` + blob storage (PVC or S3) |
| Auth | PyJWT |
| Consumed by | web (member download links), billing (writes generated docs) |

## Responsibilities

- Accept uploads from billing (generated PDFs)
- Serve downloads to authenticated members
- Track file metadata: owner, EEG, content-type, size, created-at

## Storage model

The `filestore` schema in PostgreSQL holds per-file metadata. Blobs are stored either:

- on a PVC (filesystem path referenced from the metadata row), or
- in an S3-compatible object store (deployments where S3 is configured)

In the historically deployed configurations the PVC path is the default. S3 support exists in the code but is not the default.

## Auth

Filestore verifies JWTs with PyJWT. Two operational caveats:

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

| Path | Method | Auth | Purpose |
|------|--------|------|---------|
| `/upload` | POST | EEG_ADMIN (or service-account) | upload a generated document |
| `/files/<id>` | GET | owner or EEG_ADMIN | download |
| `/files` | GET | authenticated | list files for the caller |
| `/health` | GET | none | k8s probe |

Routes vary by revision; check the deployed service for the authoritative list.

## Database

Schema `filestore`. Tables (approximate):

| Table | Holds |
|-------|-------|
| `file` | metadata: id, owner, tenant, mime, size, path-or-S3-key |
| `audit` | optional audit log of access |

Migrations are run by the Python migration runner on startup.

## Config

| Variable | Purpose |
|----------|---------|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | PG connection |
| `KEYCLOAK_URL`, `KEYCLOAK_REALM` | issuer |
| `JWT_PUBKEY_PATH` | path to PyJWT-compatible public key PEM |
| `STORAGE_PATH` | PVC mount where blobs are stored (when not using S3) |
| `S3_*` | S3 endpoint / bucket / credentials (when using S3) |

## Build and image

- Source: `eegfaktura-filestore`
- Build: Docker (Python slim base)

## Operational notes

- The PVC-storage variant is RWO; filestore is a single-replica deployment in that mode.
- Wipe-replay destroys uploaded blobs along with the PVC. Long-lived deployments should snapshot the PVC, not rely on the bootstrap to re-create files.
- Source lineage drift: the production-deployed filestore historically lagged the public-clone source by years. When recovering or upgrading, verify the running image's `.git` config to see which upstream produced it, rather than assuming the public mirror.

## Related

- [Architecture / Authentication](../architecture/auth.md) — PyJWT caveats
- [services/billing](billing.md) — main producer of files
- [services/web](web.md) — consumer
