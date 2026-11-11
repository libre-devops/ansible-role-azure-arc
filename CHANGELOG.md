# Changelog

Notable changes to this role. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow [Semantic Versioning](https://semver.org/).

## Unreleased

### Added

- `docs/using-without-galaxy.md`, linked from the README's Installation section: installing the role without Galaxy (git, release archives, requirements files that name their own source), getting its collections with no Galaxy access, keeping a copy on your own git server or artifact store, AWX, Automation Controller and execution environments, and the ansible-core and Python versions it needs.

### Changed

- The README's Requirements and Troubleshooting explain managed nodes with only Python 3.6 (RHEL 8, SLES 15), which ansible-core 2.17 and later cannot manage, and link to the two ways round it.
- The Release workflow imports the release's own tag into Galaxy, rather than the default branch, and can be run by hand with a tag to import that tag again.

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
