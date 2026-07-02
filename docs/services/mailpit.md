# Mail (Postfix relay)

Outbound mail component for the platform's services (billing notifications, password reset, EDA-mail mode). In the local docker-compose stack this is a **Postfix relay** (`eegfaktura-postfix`) that forwards mail to an external SMTP server.

!!! warning "This page previously described Mailpit — that was incorrect for this stack"
    Earlier revisions of this page described a Mailpit SMTP catcher with a web UI. The docker-compose stack does **not** use Mailpit. It runs `eegfaktura-postfix`, a Postfix **relay** that forwards outbound mail to an external SMTP host. It has **no web UI** and **no exposed ports**. The content below has been corrected to describe the actual mail component.

## At a glance

| | |
|---|---|
| Image (local stack) | custom `ghcr.io/eegfaktura/eegfaktura-postfix:latest` |
| Role | SMTP relay — forwards to an external SMTP host |
| Web UI | none |
| Exposed ports | none (internal to the compose network) |
| Upstream relay | configured via `POSTFIX_RELAY_*` env vars |

## What it does

`eegfaktura-postfix` accepts mail from the other stack services and relays it to a configured upstream SMTP server. It does not capture or store mail for inspection — it delivers onward. In the compose stack, **billing depends on it** (billing won't start until the postfix service is up).

## Configuration

The relay target and credentials are supplied through environment variables (and a mounted secret for the password). Names only — set real values in your own deployment:

| Variable | Purpose |
|----------|---------|
| `POSTFIX_RELAY_HOST` | upstream SMTP host to forward to |
| `POSTFIX_RELAY_PORT` | upstream SMTP port |
| `POSTFIX_RELAY_TLS` | whether to use TLS to the upstream |
| `POSTFIX_RELAY_USER` | SMTP auth username |
| `POSTFIX_RELAY_PASSWORD_FILE` | path to the mounted secret holding the SMTP password |
| `POSTFIX_RELAY_EMAIL` | sender / envelope address |
| `POSTFIX_HOSTNAME` | the relay's own hostname |
| `POSTFIX_MYDOMAIN` | the relay's mail domain |
| `POSTFIX_MYNETWORK` | trusted networks allowed to relay |
| `POSTFIX_DNS_1`, `POSTFIX_DNS_2` | DNS resolvers used for MX lookups |

The SMTP relay password is provided as a Docker secret (`eegfaktura-smtp-password`), not as a plain env value.

## What sends mail through it

| Service | Trigger |
|---------|---------|
| backend | member notifications (metering-point activation/completion) — **rendered in the backend, then sent via gRPC to `eda-xp`'s `SendMailService`**, which relays to this SMTP relay. See [architecture/email](../architecture/email.md). |
| billing | run-completion email, billing-document delivery |
| keycloak | password reset, email verification |
| eda-xp (in MAIL mode) | outbound EDA messages — but real PONTON-MAIL must reach ebutilities, not an arbitrary relay |

!!! note "Production relay is `eegfaktura-mailbrake`"
    In the docker-compose stack the relay is `eegfaktura-postfix`. In production the same role is
    filled by **`eegfaktura-mailbrake`** (Postfix, `:25`). Backend member mails go out as
    `no-reply@eegfaktura.at` (Cc: the EEG office address).

## Production

Production typically uses an external transactional-mail provider rather than running a relay in-stack. The relay in the local stack exists so that mail-sending code paths (billing, Keycloak, EDA-mail) can be exercised end to end against a real upstream SMTP. Do not fabricate provider specifics here — they are deployment-specific and not part of the compose repo.

## Related

- [services/billing](billing.md) — main mail sender; depends on the relay
- [services/keycloak](keycloak.md) — sends reset / verify mail
- [services/eda-xp](eda-xp.md) — MAIL mode uses real SMTP
