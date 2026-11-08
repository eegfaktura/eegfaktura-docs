# Notification e-mail flow

How member- and admin-facing notification e-mails (for example metering-point
**activation**, **completion** and **inactive**) are rendered and delivered.

!!! note "Not the same as EDA-over-e-mail"
    This flow is **separate** from `eda-xp`'s EDA-over-e-mail transport (PONTON-MAIL) used for
    market messages. `eda-xp` happens to host both, but they are different code paths — see
    [Messaging](messaging.md) and [services/eda-xp](../services/eda-xp.md).

## Chain

```mermaid
flowchart LR
    BE[eegfaktura-backend<br/>renders mail text]
    XP[eda-xp<br/>SendMailService gRPC :9093]
    MB[SMTP server<br/>eda-xp's own config<br/>epmsmail.admin.javaxmail]
    INBOX[(recipient inbox)]
    BE -->|gRPC :9093| XP -->|SMTP, STARTTLS + auth| MB -->|deliver| INBOX
```

1. **Render** — the backend fills the mail text with the EEG / participant / metering-point
   data. The text is the community's saved text (`base.mail_template`, edited by EEG admins in web
   under „E-Mail-Texte“, since backend 1.8.0) or the built-in default embedded in the binary
   (`public/templates`). Hand-made templates on the data volume (`file-content.templates`) are
   ignored since backend 1.9.0. Mails from the EDA handlers are queued and sent one at a time by
   a background worker (since 1.14.0); each send waits up to 30 s for the mail server.
2. **gRPC hand-off** — the backend has **no direct SMTP path**. It calls the `SendMailService`
   (proto package `at.energydash`) at the config key `services.mail-server` (backend
   `config.yaml`; the env override keeps the hyphen, `VFEEG_BACKEND_SERVICES_MAIL-SERVER`, because
   the key replacer only maps `.`), which points at eda-xp's gRPC port 9093. The gRPC `Sender`
   field carries the **tenant id** (member mails) or `<RC number>@eegfaktura.at` (ZP list mail);
   eda-xp **ignores** it.
3. **SMTP** — `eda-xp` sends the message itself over SMTP with the transport in
   `epmsmail.admin.javaxmail` (STARTTLS + auth; host/port/user via `SMTP_ADMIN_SERVER_HOST`,
   `SMTP_ADMIN_SERVER_PORT`, `EMAIL_ADMIN_USER`, `EMAIL_ADMIN_PWD`). In the GitOps manifests this
   is an external transactional mail service, **not** the in-cluster relay that billing uses.
4. **Delivery** — sent as `From:` `epmsmail.admin.from` (default `no-reply@eegfaktura.at`, env
   `EMAIL_ADMIN_FROM`), `Cc:` the EEG's own office address (`eeg.Email`), `To:` the member.

## Who sends what

| Sender | Path | Examples |
|---|---|---|
| **eegfaktura-backend** | saved text or built-in default → gRPC `SendMailService` → eda-xp SMTP | metering-point **activation** ("Aktivierung im Serviceportal"), **completion** ("Dein Zählpunkt ist aktiv"), **inactive** ("Dein Zählpunkt ist nicht mehr Teil der Energiegemeinschaft"); ZP list Excel to the EEG office address (`SendMailWithAttachment`) |
| billing | saved text (`billing-document`) or built-in default → its **own** `JavaMailSender`/SMTP (`MAIL_*` env) → in-cluster relay (`eegfaktura-mailbrake` in the GitOps manifests, `eegfaktura-postfix` in docker-compose) | billing documents (PDF attached) |
| energystore | gRPC `ExcelAdminService.SendExcel` on eda-xp's port 9093 — **currently broken**: eda-xp does not implement `ExcelAdminService` | energy Excel report to the requesting user (`POST /eeg/{ecid}/excel/export/{year}/{month}`) |
| keycloak | its own SMTP (configured on the instance; not in `realm/EEGFaktura.yaml`) | password reset, e-mail verification, the "set your password" mail that admin-backend triggers for new accounts (`executeActionsEmail(UPDATE_PASSWORD)`) |

backend and energystore use the gRPC hop through `eda-xp` (energystore's currently fails, see below); billing and keycloak talk SMTP
directly.

!!! note "energystore's Excel mail has no server"
    eda-xp's gRPC server registers only `SendMailService` and `RegisterPontonService`; it does
    not implement `ExcelAdminService`, so energystore's `SendExcel` call fails.

backend (`consent-request`, `zp-active`, `zp-inactive`) and billing (`billing-document`, since
billing 1.2.0) keep the editable texts in their own tables (`base.mail_template`,
`billingj.billing_mail_template`) with the same schema and API (`/eeg/mailtemplates`,
`/api/mailtemplates`) and the same policy file `mail-html-policy.json` (see
[Templates & conventions](#templates-conventions) below). billing sends with the `MAIL_NO_REPLY_TO`
sender and `Cc:` the issuer, as multipart/alternative with a plain-text part and
`Auto-Submitted: auto-generated`.

## Recipient address handling

backend, eda-xp, billing and web share **one address rule** (implemented per language, not via a
shared library):
after trimming outer unicode whitespace — including the non-breaking spaces `U+00A0` / `U+202F`
/ `U+2007` — per `;`-separated part, a valid address is `local@domain.tld` with an ASCII local
part (`[a-zA-Z0-9._%+-]`), a TLD of at least two letters, and **no** TLD allowlist. A value that
is empty after trimming means "no e-mail" — not an error, no send. Canonical stored form:
`;`-joined without spaces.

- **Write paths (backend, server-side — not only the web form):** the rule is enforced on every
  path that persists a member or EEG e-mail — participant create / update / partial-update
  (`contact.email`), the EEG office address (`eeg.Email`), and the Excel master-data import.
  Invalid addresses are rejected (API) or imported without e-mail plus an import-log entry
  (Excel); a value that trims to empty is stored as `NULL` so the send-path `Valid` guard stays
  meaningful.
- **Send paths:** both backend senders (`SendMail`, `SendMailWithAttachment`) normalise `to`/`cc`
  and send the **normalised** value; one invalid part fails the backend send. `eda-xp` splits on `;`, delivers to the valid recipients,
  and returns the refused ones in the additive proto field `SendMailReply.rejectedRecipients`
  (backward compatible) so the backend can raise an admin notification instead of dropping them
  silently; an address list with no valid recipient fails the request. billing normalises and
  validates before building the `MimeMessage` — a rejected part on an **otherwise delivered**
  mail (e.g. an invalid office `Cc:`) is a *warning* in the run protocol, **not** a `FEHLER`
  (a delivered invoice must not trigger a manual re-send → duplicate invoices).
- **Admin visibility:** failed sends surface through the existing `N_TYPE_ERROR` notification
  system (web `/page/notifications`); the member detail pane shows an **"E-Mail ungültig"** badge
  for stored addresses that fail the rule, so tenant admins can spot and correct them.

## Templates & conventions

- A saved text is never run through a template engine: it is sanitised server-side against
  `mail-html-policy.json` (identical copy in backend and billing), and placeholders are replaced
  by escaped values only.
- Limits for the member mails of the backend: at most 3 images, no PDF attachment; links only to
  `https:` / `mailto:`. billing attaches the document PDF and allows no further PDF.
- Every member mail of the backend ends with a fixed, non-editable footer
  („versandt durch EEG Faktura, ein Service des VFEEG“, since 1.10.0); billing appends a fixed
  footer too (since 1.3.0).
- Member-facing mails use the informal **"du"** salutation.

## Related

- [services/backend](../services/backend.md) — renders and dispatches member notifications
- [services/eda-xp](../services/eda-xp.md) — hosts the `SendMailService` gRPC endpoint
- [services/billing](../services/billing.md) — billing document mails
- [services/mailpit](../services/mailpit.md) — the SMTP relay component
