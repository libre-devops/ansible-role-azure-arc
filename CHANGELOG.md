# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

### Fixed

- Galaxy import: call Galaxy's import API with the `libre_devops` namespace and the GitHub clone URL, because `ansible-galaxy role import` sends the GitHub organisation name, which Galaxy refuses with HTTP 500. The import now waits for the result and prints Galaxy's log, and the Release workflow can be run by hand to import a tag again.

## 1.0.0 - 2026-11-11

### Added

- First release.
- `present` and `absent` states, and `azure_arc_connect: false` to install and lock down the agent without connecting it.
- packages.microsoft.com repository for the RHEL, Debian, Ubuntu, SUSE and Amazon Linux families, with both Microsoft signing keys bundled and checked by fingerprint. The repository files match `libre_devops.mde_linux`, so the two roles share one definition.
- Connect and disconnect with a service principal secret, a service principal certificate or an access token, passed in a root-only `--config` file that is deleted after use. Nothing is needed on the control node beyond `ansible-core`.
- Per-host targets (tenant, subscription, resource group, region, cloud, name, tags, private link scope, gateway) from ordinary inventory variables, a check that stops when a server is connected somewhere else, and `azure_arc_reconnect` to move it.
- Local agent settings (`azcmagent config`): mode, extension allowlist and blocklist, extension manager, guest configuration, incoming connections and ports, proxy and proxy bypass, plus any other property. Only settings that differ are changed.
- The `azure_arc_resource_id` fact.
- Molecule scenarios on RHEL 9, Rocky 10, Debian 13, Ubuntu 24.04 and 26.04, and SLES 15.
