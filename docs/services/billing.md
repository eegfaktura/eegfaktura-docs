# billing

Java / Spring service. Generates billing documents per EEG per billing period. Owns the `billingj` schema.

## At a glance

| | |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5.3 + Spring Security (Maven build) |
| Reporting | JasperReports 7.0.2 (PDF invoice / credit-note rendering) |
| State | PostgreSQL schema `billingj` (Flyway-migrated, `ddl-auto=validate`) |
| Auth | Spring Security JWT (RS256), `hasRole(EEG_ADMIN)` |
| Inter-service | reads master data from backend, energy from energystore |

## Responsibilities

- Manage `billing_run` state per (EEG, period)
- Compute per-Zählpunkt line items
- Generate documents (invoices, credit notes, information statements)
- Track per-document mail status
- Apply tariffs and prefix / numbering schemes per EEG

## Billing-run state machine

A billing run goes through three phases per period:

1. **Preview creation** — first generation, fills costs per member, produces preview documents
2. **Preview updates** — re-run as often as needed
3. **Final billing** — one-shot per period; preview documents are replaced by definitive invoices; the run becomes **immutable**

Once final, the run cannot be edited. To "fix" a final run, the only path is a credit note in the next period.

## Mail status

Each `billing_run` has a `mail_status` field. It is a one-shot flag — once a run has been emailed, the UI hides the re-send button.

To re-send, update `mail_status` to `NULL` in the database. The next mail action then targets **all** recipients again (no per-recipient state). This is sometimes used in incident response; document the rationale before applying it.

## Auth

```java
.anyRequest().hasRole(UserRoles.EEG_ADMIN)
```

The Spring Security configuration accepts only `EEG_ADMIN`. A user with `vfeeg-superuser` but no `EEG_ADMIN` gets 403.

Workaround for `vfeeg-superuser` operators: grant them `EEG_ADMIN` in addition. This is the current convention for VFEEG maintenance accounts (typically `EEG_ADMIN`, `EEG_OWNER`, `vfeeg-superuser`).

A follow-up in this service's source is to accept `vfeeg-superuser` as an alias for `EEG_ADMIN`. Low priority.

## JWT cert mount

billing verifies tokens by reading an X.509 certificate / RSA public-key PEM, converting it to an `RSAPublicKey` and validating tokens with RS256. It reads the file from the path in `JWT_PUBLICKEYFILE` (default `/billing/zertifikat-pub.pem`), **not** by fetching JWKS at runtime. The image does not bake a PEM.

The cert is read once and is **not** hot-reloaded; the billing pod must be restarted when the mounted ConfigMap changes.

The PEM must be Keycloak's current RS256 signing cert. Fetch:

```sh
curl -s https://<auth-domain>/realms/<realm>/protocol/openid-connect/certs \
  | jq -r '.keys[] | select(.alg=="RS256" and .use=="sig") | .x5c[0]' \
  | sed 's/\(.\{64\}\)/\1\n/g' \
  | sed '1i-----BEGIN CERTIFICATE-----' \
  | sed '$a-----END CERTIFICATE-----' \
  > zertifikat-pub.pem
```

Mount via `ConfigMap`:

```yaml
volumeMounts:
  - name: jwt-cert
    mountPath: /billing/zertifikat-pub.pem
    subPath: zertifikat-pub.pem
volumes:
  - name: jwt-cert
    configMap:
      name: eegfaktura-jwt-cert
```

On Keycloak re-import or key rotation, the ConfigMap must be regenerated and the billing pod restarted. This loop is automated at the platform level (not in the billing source) by the `billing-cert-rotator` CronJob — see [services/billing-cert-rotator](billing-cert-rotator.md).

## Tenant header

Billing endpoints require a `Tenant` HTTP header in addition to the JWT. The header value is matched against the JWT's claim. The exact match rule is the legacy `tenant == rcNumber` convention — see notes under "Tenant convention" below.

## Tariff application

| Tariff type | Applied to | Formula |
|-------------|-----------|---------|
| `EEG` | members | fixed amount per year |
| `VZP` | consumer Zählpunkte | ct/kWh × G.03 per period |
| `EZP` | producer Zählpunkte | ct/kWh × (G.01T − P.01T) per period |

Energy values come from energystore via REST. Every active Zählpunkt must have a tariff assigned, otherwise the line item is skipped and the final billing is invalid for that member.

Admins typically verify completeness via the **Energiedaten-XLSX export** before triggering the final-billing step.

## Database

Owns schema `billingj`. Key tables:

| Table | Holds |
|-------|-------|
| `billing_run` | one row per (EEG, period); `status`, `mail_status` |
| `billing_document` | invoices, credit notes, information statements |
| `line_item` | per-Zählpunkt line items |

Migrations are Flyway. If the schema is initialized from a full-DB `pg_dump` (e.g. a dump-restore deployment), the dump will contain `billingj.*` tables and Flyway will fail to re-create them. Workaround: `DROP SCHEMA billingj CASCADE` before the billing pod starts the first time. See [Architecture / Databases](../architecture/databases.md) for context.

## Document templates

Per-EEG configuration for billing documents:

- prefix and start value for the document number
- text blocks before / after line items
- footer text
- logo position
- number of digits in document number

These are managed in the admin UI ("Abrechnungseinstellungen") and stored alongside the `base.eeg` row.

## Tenant convention

In some deployments `base.eeg.tenant == base.eeg.rc_number`; in others they diverge (e.g. `tenant=<TENANT_ID>`, `rc_number=<RC_NUMBER>`). Billing's tenant check is sensitive to this. Provisioning should keep `tenant` and `rc_number` aligned where possible.

## Config

These are the variables the application actually reads (`application.properties`):

| Variable | Purpose |
|----------|---------|
| `JDBC_DATABASE_URL`, `JDBC_DATABASE_USERNAME`, `JDBC_DATABASE_PASSWORD` | PostgreSQL connection (JDBC) |
| `JWT_PUBLICKEYFILE` | path to the RS256 public-key / X.509 cert PEM (default `/billing/zertifikat-pub.pem`) |
| `MAIL_HOST`, `MAIL_PORT` | SMTP server for sending billing mails |
| `MAIL_USER`, `MAIL_PASSWORD` | SMTP credentials |
| `MAIL_SMTP_AUTH`, `MAIL_SMTP_STARTTLS_ENABLE` | SMTP auth / STARTTLS toggles |
| `MAIL_NO_REPLY_TO` | no-reply sender address |
| `HOST_PORT` | HTTP listen port (default `8080`) |
| JVM heap (`-Xmx`) | tune per environment; defaults are JVM-default which is large |

There is **no** `SPRING_DATASOURCE_*` or `KEYCLOAK_*` configuration — the datasource is driven by the `JDBC_DATABASE_*` variables and token validation uses the local PEM file (`JWT_PUBLICKEYFILE`), not a Keycloak issuer URL.

In smaller / dev deployments, set `-Xmx512m` or so to keep idle memory reasonable.

## Build and image

- Source: `eegfaktura-billing`
- Build: multi-stage Docker — builder `maven:3-eclipse-temurin-21`, runtime `eclipse-temurin:21`
- Artifact: `/opt/app/eegfaktura-billing.jar`; `ENTRYPOINT java -jar`
- Listens on port `8080` (configurable via `HOST_PORT`)
- Tag scheme: SemVer (Spring side stayed on tagged releases historically)

## Related

- [Architecture / Authentication](../architecture/auth.md) — `hasRole` + cert mount
- [Architecture / Databases](../architecture/databases.md) — Flyway + pg_dump conflict
- [services/billing-cert-rotator](billing-cert-rotator.md) — automated cert refresh
- [services/energystore](energystore.md) — energy-data source
- [reference/obis-codes](../reference/obis-codes.md) — billing-relevant values (G.03, G.01T, P.01T)
