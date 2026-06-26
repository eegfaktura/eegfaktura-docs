# Deployment

eegfaktura is deployed on Kubernetes. The pattern combines **Argo CD** (for application workloads) with a one-shot **Helm-managed bootstrap chart** (for data initialization). Cluster-level resources (DNS, TLS, ingress, secrets) are provided by the surrounding platform.

## Layout

```
┌──────────────────────────────────────────────────────┐
│  Kubernetes cluster                                  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │ Namespace: <instance>                          │  │
│  │                                                │  │
│  │   Service workloads (Deployments, StatefulSets)│  │
│  │   ├── keycloak  postgres  mosquitto  mailpit   │  │
│  │   ├── backend  billing  energystore  filestore │  │
│  │   ├── eda-xp  eda-mock  admin-backend          │  │
│  │   ├── web  admin-web                           │  │
│  │   └── billing-cert-rotator (CronJob)           │  │
│  │                                                │  │
│  │   Bootstrap Jobs (Helm-managed)                │  │
│  │   ├── db-schema                                │  │
│  │   ├── kc-realm-config                          │  │
│  │   ├── kc-users                                 │  │
│  │   ├── db-sample-data                           │  │
│  │   └── mqtt-sample-energy                       │  │
│  │                                                │  │
│  │   Ingresses + Certificates + Secrets           │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Argo CD (separate namespace, often `argocd`)        │
│  ApplicationSet → Apps → service charts              │
└──────────────────────────────────────────────────────┘
```

!!! note "Mail component"
    In the Kubernetes manifests the `mailpit` workload is an **in-cluster SMTP mock** (image `axllent/mailpit`) used for billing-mail testing; its Service is named `eegfaktura-postfix` so the billing image finds its mail target without changes. The **local docker-compose** stack instead runs a real **Postfix relay** (`eegfaktura-postfix`, forwarding to an external SMTP relay via `POSTFIX_RELAY_*`). Production is expected to use an external transactional-mail provider rather than mailpit.

!!! note "`billing-cert-rotator`"
    `billing-cert-rotator` is a Kubernetes **CronJob** (plus a one-shot bootstrap Job) defined in the platform repo, not part of the billing service source. It periodically refreshes Keycloak's RS256 signing certificate into the `eegfaktura-jwt-cert` ConfigMap and triggers a billing rollout, because the billing backend reads the cert from a file instead of OIDC discovery.

## Helm charts

Each service has its own chart (or shares a chart with similar services). Charts are co-located with the platform repo, not in a separate chart repo.

| Chart | Type | Owner |
|-------|------|-------|
| `eegfaktura-<service>` | service workload (Deployment + Service + Ingress) | per-service |
| `eegfaktura-bootstrap` | one-shot data initialization (Jobs) | platform |
| `eegfaktura-common` | shared library | platform |

Service charts are managed by Argo CD. The bootstrap chart is managed by Helm directly (released by the provisioning pipeline) — it is intentionally **not** under Argo's control because its Jobs must run once at provision time and not be reconciled.

## Why hybrid Helm + Argo

Pure Argo + Helm hooks is awkward for data initialization:

- Argo treats Helm hooks as separate sync waves, not as Job lifecycle.
- A failed bootstrap Job needs idempotent retry, which Argo's "sync" semantics complicate.
- Schema seeding has cross-service dependencies (PostgreSQL ready → keycloak ready → users → sample data) that map naturally to Helm hook weights but not to Argo sync waves.

The bootstrap chart uses Helm hook weights `-10`, `-5`, `0`, `5`, `10` to order Jobs. Each Job has an `initContainer` that waits for its prerequisite (PostgreSQL ready with the right user/DB, or Keycloak's `/realms/master` reachable). On a partial-state retry, each Job is idempotent — for example, the schema Job drops and recreates the `base` schema, the user-creation Job upserts.

## Provisioning pipeline

A single shell entry-point orchestrates everything:

```
./scripts/provision-instance.sh <mode> <instance-name>
```

Modes:

| Mode | Effect |
|------|--------|
| `validate` | parse instance file, run `helm template` |
| `render` | print rendered manifests to stdout |
| `bootstrap` | restore/generate secrets, ensure Argo + PG + KC ready, `helm install`, wait for effective state |
| `smoke` | end-to-end check (KC reachable, web reachable, KC admin API, DB sample data, backend ready) |
| `wipe-prep` | neutralize Argo (root app + ApplicationSet + child apps `syncPolicy=null`) + `helm uninstall` |
| `teardown` | `helm uninstall` only — leaves Argo active |

Wipe-replay procedure (4 commands):

```sh
./scripts/pre-wipe-check.sh <instance>
./scripts/provision-instance.sh wipe-prep <instance>
kubectl delete ns <namespace>                       # explicit user confirmation
./scripts/provision-instance.sh bootstrap <instance>
./scripts/provision-instance.sh smoke <instance>
```

See [Operations / Pipeline](../operations/pipeline.md) for the full procedure.

## Instance files

Each instance (one EEG / one deployment target) is described by a single YAML file:

```yaml
_instance:
  name: <instance-name>
  namespace: <namespace>
  cluster: <cluster-name>
  domain:
    base: <eeg-domain>
    customer: faktura.<eeg-domain>
    admin: admin.<eeg-domain>
    auth: auth.<eeg-domain>
  tenant: <tenant-id>
  certIssuer: <cert-issuer>
  secretsSource: <secrets-source>

# Flat under here: helm values for eegfaktura-bootstrap chart
keycloak:
  ...
bootstrapUsers:
  - username: ...
    groups: [...]
    tenant: ...
sampleData:
  ...
sampleEnergy:
  ...
```

The provisioning pipeline reads this file and feeds the relevant sections to the appropriate chart.

## Cross-cutting platform

| Concern | Tool |
|---------|------|
| Ingress | NGINX Ingress Controller or equivalent |
| TLS | cert-manager + Let's Encrypt (HTTP-01 or DNS-01) |
| DNS | external — typically a DNS provider integrated via cert-manager DNS-01 |
| Secrets | provisioning pipeline reads from a backup directory; Vault / External Secrets is the planned upgrade |
| Logs / metrics | not part of the application stack; cluster-level |

## Image provenance

Images are pulled from a container registry (typically GHCR or an org-internal registry). Each service repo has its own CI pipeline that publishes images on push to `master`, tagged with the commit SHA. Pinned image tags (not `:latest`) are the intended production policy.

The image tag of each service in an instance is set in the per-service Helm values, not in the chart itself. This decouples chart releases from image releases.

!!! note "Local dev uses `:latest`"
    The local **docker-compose** stack is development-only and pulls `:latest` tags for most services (a few are pinned, e.g. `eegfaktura-postgresql:0.2.0`). Do not treat the compose tags as the production pinning policy.

## Robustness layers

The provisioning pipeline applies several layers of defensive measures, learned from wipe-replay iterations:

| Layer | Mechanism |
|-------|-----------|
| `pre-wipe-check` | 4 static checks (image-tag rotation, schema apply test, fsGroup audit, Job name length) |
| `wipe-prep` | Argo neutralization in the correct order |
| `ensure_services_ready` | reactivate Argo, `kubectl wait` for PG and KC |
| `initContainer wait-for-pg` | `pg_isready` + `SELECT 1` as the application user |
| `initContainer wait-for-kc` | curl `/realms/master` loop |
| Helm hooks | deterministic ordering, no hook-phase cache |
| db-schema idempotency | `DROP SCHEMA IF EXISTS base CASCADE` before `CREATE` |
| `wait_effective` | poll for DB tables + KC users |
| smoke checks | 5 endpoint / DB / backend probes |

## Related

- [Operations / Pipeline](../operations/pipeline.md) — full wipe-replay procedure
- [services/postgres](../services/postgres.md) — DB cluster details
- [services/keycloak](../services/keycloak.md) — realm and bootstrap
