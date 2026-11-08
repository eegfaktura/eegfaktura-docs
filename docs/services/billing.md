# billing

Java / Spring service. Generates billing documents per EEG per billing period. Owns the `billingj` schema.

## At a glance

| | |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 4.0.9 + Spring Security 7 (Maven build; since 1.9.0) |
| Reporting | JasperReports 7.0.x (core 7.0.7; PDF invoice / credit-note rendering), Apache POI (XLSX export); mail text per community, sanitized against `mail-html-policy.json` |
| State | PostgreSQL schema `billingj` (Flyway-migrated, `ddl-auto=validate`) |
| Auth | Auth0 java-jwt (RS256), `hasRole(EEG_ADMIN)` |
| Inter-service | reads master data from the backend's SQL view `base.billing_masterdata`; energy values are sent by the caller (web) in the billing request |

## Responsibilities

- Manage `billing_run` state per (EEG, period)
- Compute per-Zählpunkt line items
- Generate documents (invoices, credit notes, reverse-charge credit notes, information statements)
- Track per-run mail status
- Apply tariffs and prefix / numbering schemes per EEG
- One-time joining fee („Beitrittsgebühr“) per member (since 1.6.0)
- Member mail text of the billing mail (`billing-document`), editable per community (since 1.2.0)

Communities of type `P2P` and `EVA` are not billed: `POST /api/billing` refuses them with 400 (since 1.10.0).

## Endpoints

All under `/api/**`, every call needs a bearer token with `EEG_ADMIN` and a `Tenant` header (see [Auth](#auth)). OpenAPI via springdoc (`springdoc.pathsToMatch=/api/**`), off by default since 1.9.0 (`SPRINGDOC_ENABLED=true` turns it on).

| Path | Purpose |
|------|---------|
| `POST /api/billing` | start a billing run (preview or final), asynchronous — see below |
| `GET /api/billingRuns/{id}` | run status (polled by the web), `errorSummary`, mail status / protocol |
| `GET /api/billingRuns/{tenantId}/{clearingPeriodType}/{clearingPeriodIdentifier}` | look up the run of a period |
| `GET /api/billingRuns/tenant/{tenantId}/settlementState` | final periods (runs with documents) and running periods; web passes it to the backend's settlement-interval change |
| `GET /api/billingRuns/{id}/participantAmounts`, `…/billingDocuments`, `…/billingDocuments/xlsx`, `…/billingDocuments/archive` | per-member amounts, documents, XLSX export, ZIP of the PDFs |
| `GET /api/billingRuns/{id}/billingDocuments/sendmail` | mail all documents of the run (see [Mail status](#mail-status)) |
| `DELETE /api/billingRuns/{id}` | delete a run with its preview documents; `409` for a running run or one with final documents |
| `/api/billingConfigs/**` | per-EEG document settings incl. logo / footer image upload |
| `GET /api/billingDocuments/tenant/{tenantId}/joiningFeeState`, `PUT` / `DELETE …/participants/{participantId}/joiningFee` | joining fee state of the members, charge later / paid outside eegfaktura |
| `/api/mailtemplates/**` | member mail text `billing-document` (same schema as the backend: list, get, PUT, DELETE, files, preview, test) |
| `/api/billingDocuments/**`, `/api/billingDocumentFiles/**`, `/api/billingDocumentItems/**`, `/api/billingDocumentNumbers/**`, `/api/fileData/{id}` | read / delete individual records |

## Billing-run state machine

A billing run goes through three phases per period:

1. **Preview creation** — first generation, fills costs per member, produces preview documents
2. **Preview updates** — re-run as often as needed
3. **Final billing** — one-shot per period; preview documents are replaced by definitive invoices; the run becomes **immutable**

Once final, the run cannot be re-run (`POST /api/billing` answers `409`). A final run whose months overlap a final run with documents (e.g. March and the first quarter) is also refused with `409` (since 1.8.1); a final run with an unknown period identifier gets `400`. To "fix" a final run, the only path is a credit note in the next period. `DELETE /api/billingRuns/{id}` refuses runs with final documents (since 1.8.0).

`billing_run.run_status` is a `smallint` mapped by JPA **ordinal** from `BillingRunStatus`: `0 NEW`, `1 DONE`, `2 CANCELLED` (never set, placeholder), `3 RUNNING`, `4 FAILED`. New values may only be appended.

### Asynchronous run (since 1.1.0)

- `POST /api/billing` answers `202 Accepted` with the `billingRunId`; the web polls `GET /api/billingRuns/{id}`.
- A request without energy (`allocations` empty) or without `preview` is refused with `400` before a run is created (since 1.7.0/1.8.0).
- Start is an atomic DB status claim: `RUNNING` only from `NEW` or `FAILED` (safe across replicas). A second start while running gets `409` with the running run's id; `DONE` / `CANCELLED` also get `409`.
- The calculation runs on a small executor (`billingRunExecutor`, 2–4 threads, no queue). When it is full the claim is rolled back and the call gets `503`.
- A preview run ends back at `NEW`, a final run at `DONE`. Any exception sets `FAILED` and persists a short `error_summary` (technical DB errors only generically; details in the server log). A `FAILED` run can be restarted; partial documents of the previous attempt are deleted first. At start, billing sets every run left `RUNNING` by a crash or restart to `FAILED` (since 1.10.0).
- Flyway `V1_15` added `error_summary` and a unique index on `(tenant_id, clearing_period_type, clearing_period_identifier)`.

## Mail status

Each `billing_run` has a `mail_status` field: `sendmail` sets it to `IN PROGRESS`, then `SENT` when mails went out, or `FAILED` when documents existed and no mail went out (since 1.8.0). Only a `FAILED` run can be sent again; every other status refuses a further mail run. The per-recipient result is stored as text in `send_mail_protocol` (`OK`, `FEHLER`, or a warning for rejected address parts on an otherwise delivered mail).

A run with some failed mails stays `SENT`; there is no per-recipient re-send.

Mail path and address handling: see [Architecture / Notification e-mail](../architecture/email.md).

## Auth

```java
.anyRequest().hasRole(UserRoles.EEG_ADMIN)
```

Roles come from the token's `access_groups` claim (group `/EEG_ADMIN` → authority `ROLE_EEG_ADMIN`). The Spring Security configuration accepts only `EEG_ADMIN`; `/` and `/swagger-ui` are permitted. A user with the realm role `superuser` but without the `EEG_ADMIN` group gets 403 — `superuser` only skips the `Tenant` header check (since 1.0.5).

Workaround for `superuser` operators: grant them `EEG_ADMIN` in addition. This is the current convention for VFEEG maintenance accounts (typically `EEG_ADMIN`, `EEG_OWNER`, `superuser`).

A follow-up in this service's source is to accept `superuser` as an alias for `EEG_ADMIN`. Low priority.

Token checks: RS256 signature; `exp` / `nbf` / `iat` are checked when present (60 s leeway), `exp` is not required; no `iss` / `aud` check. The JWT chain applies in every Spring profile (the former `DevSecurityConfig` was removed in 1.7.0).

## JWT cert

billing verifies tokens by reading an **X.509 certificate** PEM (parsed with `CertificateFactory("X.509")` — a bare public-key PEM does not work), taking its `RSAPublicKey` and validating tokens with RS256. It reads the file from the path in `JWT_PUBLICKEYFILE` (default `/billing/zertifikat-pub.pem`), **not** by fetching JWKS at runtime. The image does not bake a PEM.

The file is read on every token validation, so a replaced file takes effect without a restart.

The PEM must be Keycloak's current RS256 signing cert (the `x5c` entry of the realm's `certs` endpoint).

## Tenant header

Billing endpoints require a `Tenant` HTTP header in addition to the JWT. `JwtRequestFilter` checks that the header value is one of the entries of the JWT's `tenant` array (case-insensitive) unless the token has the realm role `superuser`. The resources then compare the tenant of the path, request body or stored record with the header (`TenantContext.validateTenant`, case-insensitive).

## Tariff application

Billing does not fetch energy data. The web builds the request from the energystore report: per metering point an `allocations[]` entry with `allocationKWh` (consumer: the report's `utilization`, producer: `production − allocation`). Only metering points present in `allocations` are billed.

| Item | Applied to | Formula |
|------|-----------|---------|
| energy (consumer) | consumer Zählpunkte → invoice | `allocationKWh` × `tariff_working_fee_per_consumedkwh` (ct/kWh), minus `tariff_freekwh`, tariff discount applied |
| energy (producer) | producer Zählpunkte → credit note / info | `allocationKWh` × `tariff_credit_amount_per_producedkwh` (ct/kWh), tariff discount applied |
| membership fee | member (EEG tariff) | `tariff_participant_fee` per billing run, as its own position |
| metering point fee | Zählpunkte with `tariff_use_metering_point_fee` | `tariff_metering_point_fee` |
| joining fee | member, once: in the first final run with an energy item, when the EEG tariff has `tariff_joining_fee` > 0 and the member has no final document with energy or joining fee yet | 1 × `tariff_joining_fee` on the invoice (own VAT setting, no discount); previews show it without consuming it |

Producers with a VAT id get an `INFO` document, or a reverse-charge credit note (`CREDIT_NOTE_RC`) when the EEG setting "credit notes for all producers" is on; producers without VAT id get a `CREDIT_NOTE`. Documents with a gross amount of 0 are not created.

**Time-variable tariffs (ZVT, since 1.1.0):** for a tariff with `tariff_use_time_tariff`, the caller additionally sends `allocations[].buckets` (`BASE` / `T1` / `T2` kWh sums from energystore) and the `timeWindows` they were computed for. Billing prices each bucket separately (`BASE` with the base price, `T1` / `T2` with the window price; labels `Tarif: Basis` / `Tarif: <Name> (HH:MM - HH:MM)`); buckets with 0 kWh produce no position. Fail-loud guards abort the run (`FAILED`): time tariff without buckets, sent windows that differ from the current master-data windows, or buckets on a simple tariff. Free kWh apply only to simple tariffs. The window fields come from the backend view; billing has no own ZVT migration.

Every active Zählpunkt must have a tariff assigned: without an active tariff the view delivers no price, the position is 0 and is dropped.

Admins typically verify completeness via the **Energiedaten-XLSX export** before triggering the final-billing step.

## Database

Owns schema `billingj`. Key tables:

| Table | Holds |
|-------|-------|
| `billing_run` | one row per (EEG, period); `run_status`, `error_summary`, `mail_status`, `send_mail_protocol` |
| `billing_document` | invoices, credit notes, information statements |
| `billing_document_item` | per-Zählpunkt (and per-member) line items |
| `billing_document_file`, `file_data` | generated PDFs and uploaded images (`oid` blobs) |
| `billing_document_number` | document number sequences per tenant |
| `billing_config` | per-EEG document settings |
| `billing_mail_template`, `billing_mail_template_file` | member mail text `billing-document` and its images |
| `billing_joining_fee_record` | joining fee charged later or marked as paid outside eegfaktura |

Master data is read through the backend's view `base.billing_masterdata` (mapped read-only via `@Subselect`), filtered by `tenant_id` — a cross-schema SQL dependency on the backend.

Migrations are Flyway (`src/main/resources/db/migration`, `V1_0` … `V1_20`). There is no baseline-on-migrate: pre-existing `billingj` tables without Flyway history make the start fail (see [Architecture / Databases](../architecture/databases.md)).

## Document templates

Per-EEG configuration for billing documents (`billingj.billing_config`, one row per tenant):

- prefix and start value for invoice and credit-note numbers
- text blocks before / after line items and terms text, per document type (invoice, credit note, info)
- footer text
- header (logo) and footer image
- number of digits in document number

These are managed in the web UI ("Abrechnungseinstellungen") through `/api/billingConfigs`. Without a row the built-in defaults apply. The PDF layout is `BillingDocumentDefaultTemplate.jrxml`; items are grouped per metering point with a subtotal for blocks of two or more positions.

## Tenant convention

`billing_run.tenant_id` and the master-data filter use the tenant as sent by the web (`base.eeg.tenant` / `base.participant.tenant`), not the RC number. In some deployments `base.eeg.tenant == base.eeg.rc_number`; in others they diverge (e.g. `tenant=<TENANT_ID>`, `rc_number=<RC_NUMBER>`); billing only needs the tenant to match the token's `tenant` claim.

## Config

These are the variables the application actually reads (`application.properties`):

| Variable | Purpose |
|----------|---------|
| `JDBC_DATABASE_URL`, `JDBC_DATABASE_USERNAME`, `JDBC_DATABASE_PASSWORD` | PostgreSQL connection (JDBC; `?currentSchema=billingj` is appended) |
| `JWT_PUBLICKEYFILE` | path to the RS256 X.509 cert PEM (default `/billing/zertifikat-pub.pem`) |
| `MAIL_HOST`, `MAIL_PORT` | SMTP server for sending billing mails |
| `MAIL_USER`, `MAIL_PASSWORD` | SMTP credentials |
| `MAIL_SMTP_AUTH`, `MAIL_SMTP_STARTTLS_ENABLE` | SMTP auth / STARTTLS toggles (defaults `true` / `false`) |
| `MAIL_NO_REPLY_TO` | sender address, also used for the Message-ID domain (default `no-reply@eegfaktura.at`) |
| `HOST_PORT` | HTTP listen port (default `8080`) |
| `SPRINGDOC_ENABLED` | API docs and Swagger UI (default `false`) |
| `LOG_LEVEL`, `LOG_FORMAT`, `DEPLOYMENT_ENVIRONMENT` | log level (default `info`), `json` (default) or `text`, environment name on every log line |

There is **no** `SPRING_DATASOURCE_*` or `KEYCLOAK_*` configuration — the datasource is driven by the `JDBC_DATABASE_*` variables and token validation uses the local PEM file (`JWT_PUBLICKEYFILE`), not a Keycloak issuer URL.

## Build and image

- Source: `eegfaktura-billing`
- Build: multi-stage Docker — builder `maven:3-eclipse-temurin-21`, runtime `eclipse-temurin:21-jre-jammy`; a local build needs JDK 21
- Runtime image installs `fonts-liberation` (fontconfig prefers Liberation Sans so Jasper measures text like the Helvetica it prints)
- Runs as non-root UID/GID `1001`
- Artifact: `/opt/app/eegfaktura-billing.jar`; `ENTRYPOINT java -jar`
- Listens on port `8080` (configurable via `HOST_PORT`)
- Tag scheme: SemVer from `v*` git tags, plus `sha-<short>` and branch tags; CI runs the full test suite (Testcontainers) before building the image

## Related

- [Architecture / Authentication](../architecture/auth.md) — `hasRole` + cert mount
- [Architecture / Databases](../architecture/databases.md) — Flyway, schema layout
- [Architecture / Notification e-mail](../architecture/email.md) — billing's own SMTP path
- [services/energystore](energystore.md) — energy-data source (via the web)
- [reference/obis-codes](../reference/obis-codes.md) — billing-relevant energy values
