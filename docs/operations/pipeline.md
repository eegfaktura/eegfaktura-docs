# Provisioning Pipeline

A single shell entry-point, `provision-instance.sh`, provisions and operates an eegfaktura instance end-to-end. It also drives the wipe-replay procedure (full namespace teardown + reprovision in one shot, used to clear test data between runs).

The pipeline lives in the platform repo under `scripts/`.

## Entry points

| Script | Role |
|--------|------|
| `pre-wipe-check.sh <instance>` | 4 static validations before a wipe; exits 1 on FAIL |
| `provision-instance.sh <mode> <instance>` | unified entry point; modes below |

## `provision-instance.sh` modes

| Mode | Effect |
|------|--------|
| `validate <instance>` | parse instance file, run schema check, `helm template` renders cleanly |
| `render <instance>` | print rendered Helm manifests to stdout |
| `bootstrap <instance>` | full provision: restore/generate secrets, ensure services ready, `helm install`, wait for effective state |
| `smoke <instance>` | 5-check end-to-end test |
| `wipe-prep <instance>` | neutralize Argo + `helm uninstall` in preparation for a namespace delete |
| `teardown <instance>` | `helm uninstall` only; leaves Argo active (use for non-wipe cleanup) |

## Environment overrides

| Variable | Default | Purpose |
|----------|---------|---------|
| `KUBECONFIG` | `~/.kube/<cluster>.kubeconfig`, else default | which cluster to act on |
| `PROVISION_BACKUP_DIR` | instance-specific | secrets backup source for `bootstrap` |
| `PROVISION_WAIT_TIMEOUT` | `600` | seconds for `wait_effective` |
| `PROVISION_DRY_RUN=1` | unset | use `helm --dry-run`, skip `wait_effective` |
| `PROVISION_TEARDOWN_NS=1` | unset | `teardown` also deletes the namespace (interactive confirm) |
| `PROVISION_TEARDOWN_SECRETS=1` | unset | `teardown` deletes the local secrets cache |

## Wipe-replay procedure

The full procedure to wipe an instance and bring it back up clean:

```sh
./scripts/pre-wipe-check.sh <instance>
./scripts/provision-instance.sh wipe-prep <instance>
kubectl delete ns <namespace>                       # user confirmation mandatory
./scripts/provision-instance.sh bootstrap <instance>
./scripts/provision-instance.sh smoke <instance>
```

Typical end-to-end time: 10–12 minutes.

### Per-step

#### 1. `pre-wipe-check`

Static, non-destructive validations:

1. **Image-tag rotation** — every service's pinned tag exists in the registry and is the expected hash. Prevents wiping into a known-broken image.
2. **Schema-apply test** — the database schema dump applies cleanly against an ephemeral PG. Catches schema drift before it bricks the bootstrap Job.
3. **fsGroup audit** — PVC fsGroup settings match the expected UID. Mismatch causes permission errors on first mount.
4. **Job name length** — generated bootstrap-Job names are within the 63-character Kubernetes limit. Long instance names can break this silently.

Exit 1 on any FAIL. Do not proceed.

#### 2. `wipe-prep`

Neutralizes Argo so the namespace delete does not trigger a reconcile race:

1. Set the root Argo Application's `syncPolicy=null`.
2. Set the ApplicationSet's `syncPolicy=null`.
3. Set each child App's `syncPolicy=null`.
4. `helm uninstall` the bootstrap chart.

After this step Argo will not re-apply manifests, and the bootstrap chart's Helm-managed resources are gone. The namespace itself still exists, with the service workloads still running (managed by Argo until step 3 below removes them with the namespace).

#### 3. `kubectl delete ns <namespace>`

**This step is destructive and must always be done by the operator, not by the pipeline.** It removes all workloads, all PVCs (and therefore all stored energy data), and all secrets in the namespace.

If the namespace gets stuck terminating because of a finalizer, investigate before patching the finalizer out — it usually indicates an external resource (PVC backed by a still-attached volume, a leftover CR) that should be cleaned up properly.

#### 4. `bootstrap`

Brings the instance back up:

1. Restore secrets from the backup directory (or generate fresh).
2. `ensure_services_ready` — reactivate Argo, then `kubectl wait` for PostgreSQL and Keycloak readiness. Without Argo, the service workloads would not be there to wait for.
3. `helm install` the bootstrap chart with the instance's values. The chart's Jobs run in Helm-hook weight order:
   - `db-schema` (-10) — `initContainer` waits for PG, then DROP+CREATE the application schema.
   - `kc-realm-config` (-5) — `initContainer` waits for KC, then import the realm.
   - `kc-users` (0) — create users, set group memberships, set tenant attributes.
   - `db-sample-data` (5) — `initContainer` waits for PG, then insert sample participants, EEG, metering points, tariffs.
   - `mqtt-sample-energy` (10) — publish sample energy data via MQTT.
4. `wait_effective` — poll until critical post-conditions hold (DB rows present, KC users reachable via admin API).

#### 5. `smoke`

End-to-end probes:

1. Keycloak HTTPS endpoint reachable from outside the cluster.
2. Customer web SPA reachable; index page returns 200.
3. KC admin API responds to a token-fetch + user-list.
4. Backend `/eeg/v2/<ecId>/...` returns sample data.
5. Backend ready (probe endpoint).

A PASS means the instance is end-to-end functional.

## Instance files

Each instance is a single YAML under `instances/`. The pipeline reads it for all modes.

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

keycloak:
  ...
bootstrapUsers:
  - username: <username>
    groups: [EEG_USER]
    tenant: <tenant-id>
sampleData:
  participants:
    - email: <email>
      tenant: <tenant-id>
      ...
sampleEnergy:
  ...
```

## Adding a new instance

```sh
cp instances/template.yaml instances/<new-instance>.yaml
$EDITOR instances/<new-instance>.yaml          # fill in placeholders
./scripts/provision-instance.sh validate <new-instance>
./scripts/provision-instance.sh bootstrap <new-instance>
./scripts/provision-instance.sh smoke <new-instance>
```

Note: the service charts (PostgreSQL, Keycloak, backend, ...) are currently namespace-aware but instance-specific in some pinned values. Multi-instance support on a single cluster is on the roadmap; for now a "new instance" typically means a fresh namespace on a dedicated cluster.

## Troubleshooting

### `wipe-prep` succeeded but `bootstrap` fails on "no PG"

Argo did not re-enable in time. The pipeline's `ensure_services_ready` should handle this — if it does not, manually re-enable the root Argo Application's `syncPolicy` and re-run `bootstrap`.

### `db-schema` Job fails with `relation "..." already exists`

A previous Job attempt partially created the schema. The current chart version drops the schema first; if you see this on a current chart, the Job did not get the drop step — confirm the bootstrap chart version is the expected one.

### `kc-users` Job creates the user but groups do not stick

The Keycloak group name in `bootstrapUsers[*].groups` must match an existing realm group exactly. `OWNER` vs `EEG_OWNER`, `USER` vs `EEG_USER` is a frequent silent-failure source — Keycloak's admin API returns 204 for "group not found" in some versions.

### Smoke fails on backend ready

The backend depends on Mosquitto being subscribable, not just running. Check Mosquitto pod logs for the subscription connection from the backend; a TCP-reachable broker that has not accepted the subscription will still show backend probes failing.

## Related

- [Architecture / Deployment](../architecture/deployment.md) — Helm + Argo pattern background
- [services/keycloak](../services/keycloak.md) — bootstrap of realm, users, mappers
- [services/postgres](../services/postgres.md) — bootstrap of schemas, sample data
