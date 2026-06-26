# billing-cert-rotator

!!! note "Separate platform component"
    The cert-rotator is **not** part of the billing service source. It is a platform-level CronJob defined in the deployment repo at `eegfaktura-platform/k8s/82-billing-cert-rotator.yaml`. There is no rotator code, `@Scheduled` task, or CronJob inside the `eegfaktura-billing` repo. The details below are grounded in that manifest.

A small CronJob (plus a one-shot bootstrap `Job`) that fetches Keycloak's current RS256 signing certificate, converts it to the PEM form that [billing](billing.md) expects, refreshes the mounted `ConfigMap` `eegfaktura-jwt-cert`, and triggers a billing rollout.

## At a glance

| | |
|---|---|
| Kind | CronJob (recurring) + one-shot bootstrap `Job` at deploy time |
| Schedule | daily, `30 3 * * *`, `timeZone: Europe/Vienna` |
| Image | `alpine/k8s:1.34.0` (ships `kubectl` + `curl` + `sh`; PEM extraction via `sed`/`grep`, no `jq`) |
| State | none (idempotent; only acts when the cert changed) |

## Why it exists

billing reads its JWT verification key from a file at `/billing/zertifikat-pub.pem`, not from Keycloak's JWKS endpoint at runtime. When Keycloak rotates its signing key (or when the realm is re-imported), this file becomes stale and billing rejects all tokens.

billing-cert-rotator closes that loop by re-deriving the file from Keycloak's current JWKS.

## What it does

The script (`rotate.sh`, mounted from the `billing-cert-rotator-script` ConfigMap) fetches the JWKS, extracts `x5c[0]` of the RS256/`sig` key and wraps it as a PEM. The image has no `jq`, so extraction is done with `sed`/`grep`:

```sh
JWKS=$(curl -sS --max-time 15 "$KC_URL/realms/$REALM/protocol/openid-connect/certs")

X5C=$(printf '%s' "$JWKS" | tr -d '\n\r' \
  | sed 's/},{/}\n{/g' \
  | grep '"alg":"RS256"' \
  | grep '"use":"sig"' \
  | head -1 \
  | sed -n 's/.*"x5c"[[:space:]]*:[[:space:]]*\["\([^"]*\)".*/\1/p')

# fold to 64-char lines, wrap with BEGIN/END CERTIFICATE -> zertifikat-pub.pem
```

Then:

1. Compare the new PEM against the existing `eegfaktura-jwt-cert` ConfigMap. **If unchanged, exit (no-op).**
2. If changed (or the ConfigMap is missing), apply it via `kubectl create configmap --dry-run=client -o yaml | kubectl apply -f -`.
3. Trigger `kubectl rollout restart deploy/eegfaktura-billing` so the new cert is picked up.

The triggering is done by the rotator itself via `rollout restart` (no `Reloader`-style controller is involved in this deployment).

## Config

The env vars the script reads (with the values set in the manifest):

| Variable | Purpose | Default in manifest |
|----------|---------|---------------------|
| `KC_URL` | base URL of Keycloak | `http://eegfaktura-keycloak:8080` |
| `REALM` | realm name | `EEGFaktura` |
| `NS` | namespace where the ConfigMap / deployment live | `eegfaktura` |
| `DEPLOY_NAME` | billing deployment to roll out | `eegfaktura-billing` |
| `CM_NAME` | target ConfigMap name | `eegfaktura-jwt-cert` |

## RBAC

The manifest ships a dedicated `ServiceAccount` (`billing-cert-rotator`) and a namespace-scoped `Role`:

- `get`, `update`, `patch` on the `eegfaktura-jwt-cert` ConfigMap (restricted via `resourceNames`)
- `create` on `configmaps` (for the first-boot bootstrap, before the ConfigMap exists)
- `get`, `patch` on the `eegfaktura-billing` Deployment (restricted via `resourceNames`) to trigger the rollout

This is already the narrowest practical Role; the verbs are scoped to named resources where possible.

## Scheduling

Keycloak's default key rotation interval is typically long (months / disabled). The rotator should run frequently enough to catch:

- explicit operator-triggered key rotation
- realm re-imports during provisioning
- key changes after upgrades

The manifest runs it **daily** at `03:30 Europe/Vienna` (`schedule: "30 3 * * *"`, `timeZone: Europe/Vienna`). If the key has not changed, the run is a no-op. A separate one-shot bootstrap `Job` (with a `wait-for-keycloak` init container and `ttlSecondsAfterFinished: 120`) populates the ConfigMap on first deploy, so the billing pod does not get stuck in `FailedMount`.

## Failure modes

- **JWKS endpoint unreachable** — the Job fails noisily; billing continues with the old (still-valid) cert. Alert on Job failure.
- **Multiple signing keys present** — JWKS may list more than one `sig` key during rotation windows. The script picks the first; if the wrong one is picked, billing rejects fresh tokens. The fix is to prefer the most recently activated key.
- **ConfigMap update but no pod restart** — billing reads the PEM at startup, not on every request. Without a rollout, the new cert is mounted but not loaded.

## Out of scope

- Generating Keycloak's signing key itself (Keycloak owns that)
- Rotating other certs (TLS for ingress is cert-manager's job)
- Multi-realm support

## Related

- [services/billing](billing.md) — the consumer of the PEM file
- [services/keycloak](keycloak.md) — the source of the JWKS
- [Architecture / Authentication](../architecture/auth.md) — billing cert mount details
