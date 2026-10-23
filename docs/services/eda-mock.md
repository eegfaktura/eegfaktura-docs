# eda-mock

!!! warning "Placeholder, not a functional EDA mock"
    There is **no** `eda-mock` source code in any eegfaktura component repository; the only `mock` code in eda-xp is test-fixture (`EmailMock`). The deployed `eegfaktura-eda-mock` exists only in the **dev zone** and is defined in the platform repository (`overlays/dev/resources/eda-mock.yaml`, earlier `k8s/71-eda-mock.yaml`); it is not part of the production manifests and not part of the docker-compose stack. It is two `traefik/whoami:v1.10` containers (ports 6060 and 6090) that simply echo HTTP requests. It does **not** publish MQTT messages, stub EDA process flows, or answer like a grid operator. A functional EDA mock is planned but not implemented.

A stand-in for the external Ponton messenger (KEP) in the dev zone, so that `eda-xp` starts and can send outbound messages without a real PONTON link.

## At a glance

| | |
|---|---|
| Role | replaces the Ponton messenger behind `eda-xp` (not `eda-xp` itself) |
| Image | `traefik/whoami:v1.10` (Docker Hub), two containers in one Deployment |
| Ports | 6060 (HTTP target of eda-xp's outbound messages), 6090 (echo only) |
| Inbound | none — it never calls eda-xp |
| Outbound | none — it does not publish to MQTT |
| State | stateless |

## How it is wired

In the dev zone, eda-xp's `application.conf` points `app.kepserver.url` at `http://eegfaktura-eda-mock:6060/ponton/eda/webservice/outbound`. eda-xp POSTs every outbound EDA message (XML) there. whoami answers every request with HTTP 200 and an echo of the request, and eda-xp's HTTP client treats any 200/202 response as a successful send.

## Effect on the flows

Because nothing ever comes back from a grid operator:

- Outbound requests (e.g. `ANFORDERUNG_ECON`, `ANFORDERUNG_CPF`) are sent "successfully" and end there.
- No `ANTWORT_*`, `ZUSTIMMUNG_*`, `ABSCHLUSS_*` or `ABLEHNUNG_*` message arrives, so the backend never moves a metering point past the state the outbound request left it in, and process history shows no grid-operator responses.
- No energy data (`CR_MSG`) arrives. Energy data for dev/test must be seeded directly (e.g. by publishing on MQTT).
- Real EDA error and rejection flows cannot be tested with it.

## Config

None. The whoami containers only take `--port` as an argument.

## Related

- [services/eda-xp](eda-xp.md) — the real gateway
- [Architecture / Messaging](../architecture/messaging.md) — inbound pipeline
- [services/backend](backend.md) — MQTT consumer side
