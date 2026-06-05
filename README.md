# eegfaktura-docs

Developer and maintainer documentation for the **eegfaktura** software suite — an open source platform for billing and energy management in Austrian energy communities (EEG / Energiegemeinschaften).

This documentation covers:

- **Architecture** — service topology, authentication, data flows
- **Services** — per-service responsibilities, APIs, configuration, image provenance
- **Operations** — deployment pipeline, wipe-replay procedure
- **Reference** — glossary, OBIS codes, EDA terminology

## Reading the docs

- **Browsable site:** https://gemeinstrom.github.io/eegfaktura-docs/ (GitHub Pages)
- **Markdown source:** [`docs/`](docs/) in this repository

## Audience

Primary: **developers and maintainers** working on the eegfaktura services or operating a deployment.

Not a user manual for end users (members of an energy community) — see the official [docs.eegfaktura.at](https://docs.eegfaktura.at) for workflow / process documentation.

## Source code

The eegfaktura suite is composed of multiple services. Source repositories:

| Service | Upstream |
|---------|----------|
| `eegfaktura-backend` | https://github.com/eegfaktura/eegfaktura-backend (Go) |
| `eegfaktura-billing` | https://github.com/eegfaktura/eegfaktura-billing (Java/Spring) |
| `eegfaktura-energystore` | https://github.com/eegfaktura/eegfaktura-energystore (Go) |
| `eegfaktura-filestore` | https://github.com/eegfaktura/eegfaktura-filestore (Python) |
| `eegfaktura-eda-xp` | https://github.com/eegfaktura/eegfaktura-eda-xp (Scala) |
| `eegfaktura-web` | https://github.com/eegfaktura/eegfaktura-web (React) |

## Contributing

PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

Documentation under [CC BY-SA 4.0](LICENSE). The eegfaktura software itself is licensed under AGPL-3.0 — see each service repository for details.
