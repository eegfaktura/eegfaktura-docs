# Notification e-mail flow

How member- and admin-facing notification e-mails (for example metering-point
**activation** and **completion**) are rendered and delivered.

!!! note "Not the same as EDA-over-e-mail"
    This flow is **separate** from `eda-xp`'s EDA-over-e-mail transport (PONTON-MAIL) used for
    market messages. `eda-xp` happens to host both, but they are different code paths — see
    [Messaging](messaging.md) and [services/eda-xp](../services/eda-xp.md).

## Chain

```mermaid
flowchart LR
    BE[eegfaktura-backend<br/>renders HTML template]
    XP[eda-xp<br/>SendMailService gRPC :9093]
    MB[SMTP relay<br/>prod: eegfaktura-mailbrake:25<br/>compose: eegfaktura-postfix]
    INBOX[(recipient inbox)]
    BE -->|gRPC :9093| XP -->|SMTP :25| MB -->|relay| INBOX
```

1. **Render** — the backend fills an HTML template with the EEG / participant / metering-point
   data. Templates live at `/opt/storage/public/<tenant>/templates/` with a global fallback
   `/opt/storage/public/templates/`.
2. **gRPC hand-off** — the backend has **no direct SMTP path**. It calls the `SendMailService`
   (proto package `at.energydash`) at `services.mail-server`
   (`VFEEG_BACKEND_SERVICES_MAIL_SERVER`), which in the cluster resolves to
   `eegfaktura-xp-adapter-grpc:9093`. The gRPC `Sender` field carries only the **tenant id**,
   not an e-mail address.
3. **SMTP relay** — `eda-xp` relays the message via SMTP to the mail relay:
   **`eegfaktura-mailbrake`** (Postfix) in production, `eegfaktura-postfix` in the
   docker-compose stack.
4. **Delivery** — sent as `From: no-reply@eegfaktura.at`, `Cc:` the EEG's own office address
   (`eeg.Email`), `To:` the member.

## Who sends what

| Sender | Path | Examples |
|---|---|---|
| **eegfaktura-backend** | renders template → gRPC `SendMailService` → relay | metering-point **activation** ("Aktivierung im Serviceportal"), **completion** ("Dein Zählpunkt ist aktiv") |
| billing | its **own** SMTP config (`MAIL_*` env) → relay | run-completion, billing documents |
| keycloak | its own SMTP → relay | password reset, e-mail verification |

Only the backend uses the gRPC hop through `eda-xp`; billing and keycloak talk SMTP to the
relay directly.

## Templates & conventions

- Per-tenant templates under `/opt/storage/public/<tenant>/templates/`; if a **specific file**
  is missing (not just the whole directory), the backend falls back to the global
  `/opt/storage/public/templates/`.
- Member-facing mails use the informal **"du"** salutation.
- Shared signature / footer: EEG description → address → phone / e-mail / website (each behind a
  `Valid` guard) → "versandt durch" → logo capped at `max-height: 90px`.
- `null.String` template fields must be rendered via `.String` with a `Valid` guard — printing
  the value directly emits the raw struct (`{{value true}}`).

## Related

- [services/backend](../services/backend.md) — renders and dispatches member notifications
- [services/eda-xp](../services/eda-xp.md) — hosts the `SendMailService` gRPC endpoint
- [services/mailpit](../services/mailpit.md) — the SMTP relay component
