# Contributing

Thanks for your interest in improving the eegfaktura documentation.

## Reporting issues

Found a factual error, broken link, or outdated section? Please open an issue:
https://github.com/gemeinstrom/eegfaktura-docs/issues

Include the affected page (path under `docs/`) and, if possible, what the correct content should be.

## Pull requests

- Branch from `main`.
- Keep changes focused — one topic per PR.
- Use Conventional Commit messages, e.g. `docs(services): fix energystore topic table`.
- The site must build cleanly before merge:

  ```
  pip install -r requirements.txt
  mkdocs build --strict
  ```

## Scope

This repository contains **developer and maintainer documentation** (architecture, service behaviour, protocols, operational procedures). It is not an end-user manual, and it intentionally does not contain production sizing data, credentials, or instance-specific configuration.

## Licensing

- Documentation is licensed under [CC BY-SA 4.0](LICENSE); by contributing you agree that your contribution is published under the same license.
- The eegfaktura software itself is licensed under AGPL-3.0 — see the respective service repositories.

## Contact

Maintainer contact: eegfaktura@vfeeg.org
