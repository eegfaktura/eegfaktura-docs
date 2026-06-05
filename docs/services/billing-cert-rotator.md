# billing-cert-rotator

A small periodic Job (or CronJob) that fetches Keycloak's current RS256 signing certificate, converts it to the PEM form that [billing](billing.md) expects, and refreshes the mounted `ConfigMap`.

## At a glance

| | |
|---|---|
| Kind | CronJob (typical) or one-shot Job at deploy time |
| Schedule | weekly, or aligned with the Keycloak key-rotation cadence |
| Image | small base (alpine + curl + jq + openssl) or a purpose-built image |
| State | none |

## Why it exists

billing reads its JWT verification key from a file at `/billing/zertifikat-pub.pem`, not from Keycloak's JWKS endpoint at runtime. When Keycloak rotates its signing key (or when the realm is re-imported), this file becomes stale and billing rejects all tokens.

billing-cert-rotator closes that loop by re-deriving the file from Keycloak's current JWKS.

## What it does

```sh
curl -s https://<auth-domain>/realms/<realm>/protocol/openid-connect/certs \
  | jq -r '.keys[] | select(.alg=="RS256" and .use=="sig") | .x5c[0]' \
  | sed 's/\(.\{64\}\)/\1\n/g' \
  | sed '1i-----BEGIN CERTIFICATE-----' \
  | sed '$a-----END CERTIFICATE-----' \
  > zertifikat-pub.pem
```

Then:

1. Apply the result as a `ConfigMap` (`eegfaktura-jwt-cert` or similar).
2. Trigger a billing-pod rollout so the new cert is picked up.

The triggering happens by patching a pod annotation (`kubectl rollout restart deployment/billing`) or by relying on a `Reloader`-style controller that watches the ConfigMap.

## Config

| Variable | Purpose |
|----------|---------|
| `KEYCLOAK_URL` | base URL of Keycloak |
| `KEYCLOAK_REALM` | realm name (`EEGFaktura`) |
| `BILLING_NAMESPACE` | namespace where to apply the ConfigMap |
| `BILLING_DEPLOY` | deployment name to roll out (or rely on Reloader) |
| `CERT_CONFIGMAP_NAME` | target ConfigMap name |

## RBAC

The rotator needs:

- `get`, `create`, `update` on `ConfigMaps` in the target namespace
- `patch` on `Deployments` (if it triggers the rollout itself)

Use a dedicated `ServiceAccount` with the narrowest possible Role.

## Scheduling

Keycloak's default key rotation interval is typically long (months / disabled). The rotator should run frequently enough to catch:

- explicit operator-triggered key rotation
- realm re-imports during provisioning
- key changes after upgrades

A weekly cadence is a reasonable default. If the key has not changed, the Job is a no-op.

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
