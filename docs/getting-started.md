# Getting Started (Developer Onboarding)

This page gets a new eegfaktura developer from zero to a running local stack, a
first code change, and an understanding of where that change goes on its way to
production. It is deliberately link-heavy: it points at the canonical source for
each topic instead of duplicating it.

!!! info "Who this is for"
    New members of the VFEEG development team. You do **not** need production
    access to be productive — almost all development happens against the local
    Docker Compose stack.

## 1. Orientation

eegfaktura is a **multi-repo, multi-stack microservice suite**. Two GitHub orgs
are involved:

- **`github.com/eegfaktura/*`** — the application repos (AGPL-3.0) and these docs.
  This is where you contribute code.
- **`github.com/vfeeg-development/*`** — platform/infra tooling
  ([`eegfaktura-platform`](https://github.com/vfeeg-development/eegfaktura-platform))
  and the ArgoCD source of truth
  ([`eegfaktura-gitops`](https://github.com/vfeeg-development/eegfaktura-gitops)).

Read the [Service Overview](architecture/overview.md) first for the big picture,
then the [service page](services/index.md) of whatever you touch.

## 2. Run it locally

The canonical local stack is
**[`eegfaktura-docker-compose`](https://github.com/eegfaktura/eegfaktura-docker-compose#readme)** —
follow its README. In short:

1. Install Docker + Docker Compose.
2. Add the hosts entry for Keycloak (so the token issuer `iss` matches for both
   your browser and the containers):
   ```
   127.0.0.1 eegfaktura-keycloak
   ```
3. `docker compose up`, then create a Keycloak *Manager* user and an EEG as
   described in the README.

!!! note "Local images vs. building from source"
    The Compose stack pulls the **published service images** — you do not need
    to build anything to run the platform. When you *do* build a service from
    source, some services have a code-generation step (e.g. `protoc` /
    `go generate` for the Go services, `sbt` for Scala). The `backend` is a
    normal source-built service like the other eight — see its
    [service page](services/backend.md) and the platform
    [build pipeline](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/build-pipeline.md).

## 3. Work on a single service

Each service lives in its own repo with its own stack and test runner. Pick the
one you are changing and match its existing patterns.

| Service | Stack | Unit tests |
|---|---|---|
| `eegfaktura-backend` | Go (REST/GraphQL) | `go test ./...` |
| `eegfaktura-energystore` | Go | `go test ./...` |
| `eegfaktura-filestore` | Python (FastAPI + Strawberry) | `pytest` |
| `eegfaktura-billing` | Java (Spring Boot) | `./mvnw test` |
| `eegfaktura-eda-xp` | Scala (Pekko) | `sbt test` |
| `eegfaktura-admin-backend` | Scala (Pekko) | `sbt test` |
| `eegfaktura-web` | React 18 + Ionic 7 + Vite | `npm run test.unit` (Vitest), e2e `npm run test.e2e` |
| `eegfaktura-admin` | React (CRA) | `npm test` |

Each service page under [Services](services/index.md) documents its
responsibilities, APIs, configuration and secrets. The frontend talks **only**
to backend services — never directly to a database (see
[Architecture](architecture/index.md)).

### Contribution conventions

- Read [`CONTRIBUTING.md`](https://github.com/eegfaktura/eegfaktura-docs/blob/main/CONTRIBUTING.md)
  in each repo.
- Branch from the repo's **default branch** (for `eegfaktura-backend` that is
  `master`, not the stale `main`).
- **Every change carries a CHANGELOG entry** in the same PR (`[Unreleased]`
  section); the release stamps it to a version.
- Conventional-commit style (`feat(scope): …`, `fix: …`, `docs: …`).

## 4. Zones: Dev / Test / Prod

There are three quality zones. Only **Dev** is a public-cloud cluster with
direct developer access; Test and Prod are on-prem and operator-gated.

| Zone | Where | Access | Data |
|---|---|---|---|
| **Dev** | STACKIT SKE (`eu02`) | direct (kubeconfig) | synthetic / anonymized |
| **Test** | on-prem | operator only | daily refresh from Prod *(deferred)* |
| **Prod** | on-prem, ArgoCD/GitOps | operator only | production data |

Full model:
[`zonen-konzept.md`](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/zonen-konzept.md).
Dev-zone URLs, test users and the daily hibernation window (the cluster sleeps
**22:00–08:00 Vienna**) are in
[`access.md`](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/access.md).

!!! tip "See your branch running in Dev — Preview Deployments"
    Pushing a branch named `preview/**` builds and deploys it on-demand into the
    Dev zone (auto-reset when the branch is deleted). This is how you validate a
    change against a real cluster without touching anyone else's environment.
    See [ADR-0007](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/adr/0007-preview-deployments-unmerged-branches.md).

## 5. Logs & debugging

- **Local (default):** `docker compose logs -f <service>` — enough for almost
  all development.
- **Dev zone:** read-only `kubectl` access via a shared `log-reader` kubeconfig
  (logs, pods, events — no writes, no secrets). Setup and usage in
  [`access.md` → *Read-only Log-Zugriff*](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/access.md).
  Remember the 22:00–08:00 hibernation — no logs while the cluster sleeps.
- **Prod:** read-only via Dynatrace (Grail), maintainer/team-scoped — see
  [`prod-log-review.md`](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/prod-log-review.md).
  Do not reconfigure prod logging.

## 6. Image registries & the release process

The suite uses a **two-tier image registry** ([ADR-0005](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/adr/0005-two-tier-image-registry.md)).
Understanding "what gets built where, and what Prod actually runs" clears up the
most common source of confusion:

- **Dev tier (rolling):** each source repo's CI builds `ghcr.io/vfeeg-development/<image>`
  with rolling tags (`sha-<commit>`, `latest`). The Dev zone runs `latest` and
  auto-rolls on every push to the default branch.
- **Release (pinned):** a release is a **manual, by-digest** promotion — the
  exact `sha-*` image is re-tagged `vX.Y.Z` (`crane tag …`). Nothing is rebuilt;
  the released tag points at the identical digest that was tested.
- **Prod (pinned):** the GitOps repo pins `vX.Y.Z` tags; ArgoCD syncs them. Prod
  never runs `latest`.
- An independent prod **image mirror** exists at `ghcr.io/marki4711/*`.

The end-to-end flow (merge → CI `sha-*` → `crane tag vX.Y.Z` → gitops bump →
ArgoCD) is documented in
[`build-pipeline.md`](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/build-pipeline.md).

## 7. Repo & documentation map

- **Application code + these docs:** `github.com/eegfaktura/*`
  (per-service pages under [Services](services/index.md)).
- **Platform / infra / zones / ADRs / access:**
  [`eegfaktura-platform`](https://github.com/vfeeg-development/eegfaktura-platform)
  — see its
  [`repo-inventory.md`](https://github.com/vfeeg-development/eegfaktura-platform/blob/main/docs/repo-inventory.md)
  for the full repo list.
- **Prod deployment manifests (ArgoCD):**
  [`eegfaktura-gitops`](https://github.com/vfeeg-development/eegfaktura-gitops).

!!! note "Where does a topic belong?"
    These docs cover **how things connect** — architecture, protocols,
    code-level gotchas. Deployment, provisioning, sizing and operational tuning
    live in the platform repo, and are linked from here rather than copied.

## 8. Getting help

Ask the team — bring the service, the zone, and the relevant log excerpt. When
in doubt about whether a change affects Test/Prod, assume it might and ask first.
