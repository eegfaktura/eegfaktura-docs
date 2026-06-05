# eda-mock

A stub of the EDA gateway for environments without a real PONTON link. Lets the rest of the stack run end-to-end (Zählpunkt activation, energy-data flow, participation-factor changes) without involving the network operator.

## At a glance

| | |
|---|---|
| Language | Scala |
| Role | drop-in stand-in for `eda-xp` + PONTON |
| Inbound | none from external (no PONTON) |
| Outbound | MQTT publish to the same topic shape as eda-xp |
| State | typically stateless / minimal |

## When to use

eda-mock replaces `eda-xp` in:

- **Dev environments** — local + cluster-internal flows without external dependencies
- **Sample / demo instances** — to exercise the customer SPA's full flow including process history
- **CI environments** — for end-to-end tests

A real production instance uses `eda-xp` + PONTON instead.

## What it stubs

For each outbound EDA process triggered by the backend (e.g. `EC_REQ_ONL`), eda-mock generates a plausible response message and publishes it back to MQTT on the appropriate inbound topic. The backend's subscription handler consumes it as if it had originated from the real network operator.

Stubbed flows:

| Process | Stubbed response |
|---------|------------------|
| `EC_REQ_ONL` (Zählpunkt activation) | sends `ONLINE_REG_COMPLETION` after a short delay → metering point goes to `ACTIVE` |
| `EC_REQ_OFF` | sends a completion message → metering point goes to inactive |
| `EC_REQ_PRZ` (participation factor) | sends an approval message |
| `EC_REQ_LST` (participant list) | returns a generated list |
| `EC_REQ_ENE` (energy data) | typically not stubbed — sample energy data is provisioned directly via MQTT by the bootstrap chart |

The exact set of stubbed flows depends on the eda-mock revision.

## UI effect

When eda-mock is in use instead of eda-xp, the customer SPA's "Messages" menu may or may not appear, depending on whether the eda-mock publishes message-history entries. Typically the dev experience is:

- Process History shows the activation lifecycle
- Messages menu either shows synthetic messages or is suppressed

## Config

Minimal. The mock needs MQTT to publish and the same EEG / tenant configuration as eda-xp would use, so the messages route to the right `<tenant>` topic slot.

| Variable | Purpose |
|----------|---------|
| `MQTT_*` | broker connection |
| `TENANT` | which tenant to publish under |
| `RESPONSE_DELAY_MS` | how long to wait before publishing the stubbed response |

## Limitations

- No XML schema-version negotiation — the mock produces a fixed shape.
- No conversation-state merge against `eda.*` — the mock publishes synthetic messages without prior context.
- Not a substitute for testing real EDA error / rejection flows.

## Related

- [services/eda-xp](eda-xp.md) — the real gateway
- [Architecture / Messaging](../architecture/messaging.md) — inbound pipeline
- [services/backend](backend.md) — MQTT consumer side
