# mailpit

SMTP catcher for non-production environments. Receives all outbound mail from the platform's services (billing notifications, password reset, EDA-mail mode) and serves them via a web UI for inspection.

## At a glance

| | |
|---|---|
| Image | `axllent/mailpit` |
| SMTP port | 1025 (configurable) |
| Web UI port | 8025 (configurable) |
| State | in-memory by default; optional persistent store |
| Auth | none by default |

## When to use

Mailpit replaces a real SMTP relay in:

- **Dev environments** — capture mail without delivering it
- **Test / sample instances** — operators can verify mail rendering and content
- **CI** — automated checks of generated mail

A real production instance uses a real SMTP relay (Postfix, an external SMTP provider, or the EDA-MAIL outbound to ebutilities).

## What sends mail to it

| Service | Trigger |
|---------|---------|
| billing | run-completion email, billing-document delivery |
| keycloak | password reset, email verification |
| eda-xp (in MAIL mode) | outbound EDA messages — but: real PONTON-MAIL must reach ebutilities, not Mailpit |

## SMTP config in clients

Point each client at Mailpit's service DNS:

```
SMTP_HOST=mailpit.<namespace>.svc.cluster.local
SMTP_PORT=1025
SMTP_AUTH=none
SMTP_TLS=none
```

The exact env-var names depend on the client (Spring uses `SPRING_MAIL_*`, Keycloak uses realm-level SMTP settings).

## Inspecting mail

The web UI is reachable on port 8025. In dev / sample instances it is typically exposed through an Ingress (or port-forward). The UI shows the raw RFC822 message, the rendered HTML, attachments, and headers.

Mailpit never delivers anything onwards; mail accumulates until the pod restarts or the store is cleared.

## Operational notes

- For sample / demo instances, no separate cleanup is needed — pod restart wipes the in-memory store.
- For longer-running test instances, consider Mailpit's persistent-store mode (PVC-backed). Otherwise the in-memory buffer grows over time.
- Mailpit is **not** an EDA-MAIL substitute. If you need to test EDA in MAIL mode, you also need an SMTP path to the real ebutilities endpoint or a careful integration mock.

## Related

- [services/billing](billing.md) — main mail sender
- [services/keycloak](keycloak.md) — sends reset / verify mail
- [services/eda-xp](eda-xp.md) — MAIL mode uses real SMTP, not Mailpit
