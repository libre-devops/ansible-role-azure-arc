<div align="center">
  <a href="https://libredevops.org">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://libredevops.org/assets/libre-devops-white.png">
      <img alt="Libre DevOps" src="https://libredevops.org/assets/libre-devops-black.png" width="300">
    </picture>
  </a>
</div>

# Ansible Role: Azure Arc for Linux servers

Installs and locks down the Azure Connected Machine agent (`azcmagent`) on Linux servers, connects them to Azure Arc and disconnects them, from Microsoft's signed packages.microsoft.com repository. Nothing is needed on the control node beyond `ansible-core`.

[![CI](https://github.com/libre-devops/ansible-role-azure-arc/actions/workflows/ci.yml/badge.svg)](https://github.com/libre-devops/ansible-role-azure-arc/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/libre-devops/ansible-role-azure-arc?sort=semver&label=release)](https://github.com/libre-devops/ansible-role-azure-arc/releases/latest)
[![Galaxy](https://img.shields.io/badge/galaxy-libre__devops.azure__arc-EE0000?logo=ansible&logoColor=white)](https://galaxy.ansible.com/ui/standalone/roles/libre_devops/azure_arc/)
[![Licence](https://img.shields.io/github/license/libre-devops/ansible-role-azure-arc)](./LICENSE)

---

## What it does

| State | What happens |
|---|---|
| `present` (default) | Adds the packages.microsoft.com repository and Microsoft's signing keys, installs `azcmagent`, applies the local agent settings you give, and connects the server to the subscription and resource group in its variables. A server already connected there is left alone. |
| `present` with `azure_arc_connect: false` | The same, without connecting. Use it to lock the agent down before anyone can connect it, or in a golden image. |
| `absent` | Disconnects the server, which deletes its Azure resource (or only resets the agent, with `azure_arc_disconnect_local_only`), then removes the agent. |

Each server's target (tenant, subscription, resource group, region and the rest) is an ordinary variable, so one play onboards servers to as many subscriptions and resource groups as your inventory describes. See [Onboarding targets](#onboarding-targets).

### How it differs from Microsoft's role

Microsoft documents its own role, [`azure.azcollection.azure_arc`](https://learn.microsoft.com/en-us/azure/azure-arc/servers/onboard-ansible-playbooks), which also handles Windows. Choose by how you want to work:

| | `libre_devops.azure_arc` | `azure.azcollection.azure_arc` |
|---|---|---|
| Install | The package manager, from the signed repository or your mirror, with keys checked by fingerprint | Downloads and runs Microsoft's install script |
| Sign-in | Service principal secret or certificate, or an access token, handed to `azcmagent` on the server in a root-only file | An access token fetched on the control node with the Azure SDK |
| Control node needs | `ansible-core` | `azure.azcollection` and its Azure SDK Python packages |
| Local agent lock-down | Mode, extension allowlist and blocklist, guest configuration, remote access, proxy, any other `azcmagent config` property | Proxy |
| Operating systems | Linux | Linux and Windows |

## Contents

- [Requirements](#requirements)
- [Supported platforms](#supported-platforms)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Onboarding targets](#onboarding-targets)
- [Signing in](#signing-in)
- [Role variables](#role-variables)
- [Locking the agent down](#locking-the-agent-down)
- [Repository and signing keys](#repository-and-signing-keys)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Developing](#developing)
- [Releasing](#releasing)
- [Sources](#sources)

## Requirements

| What | Detail |
|---|---|
| Control node | `ansible-core` 2.16 or later. |
| Collections | `community.general`, for SUSE hosts only (`ansible.builtin.package` hands over to `community.general.zypper` there). The `ansible` package already includes it. |
| Managed nodes | A [supported distribution](#supported-platforms) with systemd, and a Python that your `ansible-core` supports. RHEL 8 and SLES 15 ship Python 3.6, which ansible-core 2.17 and later cannot use: see [Python on managed nodes](./docs/using-without-galaxy.md#on-managed-nodes). |
| Privileges | `become: true`. |
| Where | Servers outside Azure. Microsoft does not support Azure Arc on Azure VMs, Azure Stack Hub or Azure Stack Edge, except to [evaluate it on an Azure VM](https://learn.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine). |
| Network | HTTPS to `packages.microsoft.com` (or your mirror) and to the Azure Arc endpoints ([Microsoft Learn: Network requirements](https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements)), directly, through a proxy, a private link scope or an Arc gateway. |
| Azure | The `Microsoft.HybridCompute`, `Microsoft.GuestConfiguration` and `Microsoft.HybridConnectivity` resource providers registered in each subscription, and an identity with the **Azure Connected Machine Onboarding** role (or Contributor) on each target resource group. |

## Supported platforms

Microsoft lists the systems that still get new agent versions in [Microsoft Learn: Connected Machine agent prerequisites](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites#supported-operating-systems). On 8 November 2026 the Linux ones were:

| Family | Distributions (x86-64) | Repository path | Tested in CI |
|---|---|---|---|
| RHEL | RHEL 8, 9 and 10, Rocky 8 and 9, Alma 8 and 9, Oracle Linux 8, 9 and 10 | `rhel/<major>` | RHEL 9 (UBI), Rocky 10 |
| Amazon | Amazon Linux 2023 | `amazonlinux/2023` | No |
| Debian | Debian 13 | `debian/13` | Debian 13 |
| Ubuntu | Ubuntu 22.04, 24.04 and 26.04 | `ubuntu/<version>` | Ubuntu 24.04, Ubuntu 26.04 |
| SUSE | SLES 15 SP7 | `sles/15` | SLES 15 SP7 (BCI) |

Debian 11 and 12, RHEL 7, Oracle Linux 7, Amazon Linux 2, Ubuntu 18.04 and 20.04, and SLES 15 SP6 and earlier stopped getting new agent versions in October 2026: version 1.69 is their last. The role still installs on them, but Microsoft only services that version for 12 months after the final release. Fedora is not supported, and the role stops on it. Arm64 works only on the systems Microsoft marks for it, and not every feature is available there.

The tests run in containers. They install and lock down the agent but cannot connect it: see [Developing](#developing).

## Installation

From Galaxy:

```bash
ansible-galaxy role install libre_devops.azure_arc
```

Or pin a release from GitHub in a `requirements.yml`, as in [`examples/requirements.yml`](./examples/requirements.yml):

```yaml
roles:
  - name: libre_devops.azure_arc
    src: https://github.com/libre-devops/ansible-role-azure-arc.git
    scm: git
    version: 1.0.0
```

Without Galaxy, or to keep a copy inside your organisation, see [Using the role without Galaxy](./docs/using-without-galaxy.md). It covers git, release archives, requirements files that point at your own servers, AWX and Automation Controller, execution environments, and managed nodes with Python 3.6 such as RHEL 8.

## Quick start

```yaml
- name: Connect Linux servers to Azure Arc
  hosts: arc_servers
  become: true
  roles:
    - role: libre_devops.azure_arc
      vars:
        azure_arc_tenant_id: 00000000-0000-0000-0000-000000000000
        azure_arc_subscription_id: 22222222-2222-2222-2222-222222222222
        azure_arc_resource_group: rg-arc-prd-uks
        azure_arc_location: uksouth
        azure_arc_service_principal_id: 11111111-1111-1111-1111-111111111111
        azure_arc_service_principal_secret: "{{ vault_azure_arc_service_principal_secret }}"
```

```bash
ansible-playbook -i inventory.yml site.yml --ask-vault-pass
```

The first run installs the agent, connects the server and sets the `azure_arc_resource_id` fact. Later runs change nothing while the server stays connected where its variables say.

## Onboarding targets

The role decides where each server goes from that server's variables, so the normal Ansible variable sources pick the target: `group_vars` for a group of servers, `host_vars` for one, `vars` on the play or role, or `-e` on the command line. [`examples/inventory`](./examples/inventory) shows one inventory onboarding to two subscriptions, three resource groups and two regions:

```text
examples/inventory/
├── hosts.yml                     # arc_production and arc_development groups
├── group_vars/
│   ├── all/azure_arc.yml         # tenant, service principal, lock-down shared by every server
│   ├── arc_production.yml        # subscription 2222..., rg-arc-prd-uks, uksouth
│   └── arc_development.yml       # subscription 3333..., rg-arc-dev-ukw, ukwest
└── host_vars/
    └── db01.corp.example.yml     # one production server in its own resource group, with its own name
```

```bash
ansible-inventory -i examples/inventory/hosts.yml --host db01.corp.example   # see where a server will go
ansible-playbook -i examples/inventory/hosts.yml examples/playbook.yml --ask-vault-pass
```

Every target variable works per host or per group:

| Target | Variable |
|---|---|
| Tenant | `azure_arc_tenant_id` |
| Subscription | `azure_arc_subscription_id` (the ID, not the name) |
| Resource group | `azure_arc_resource_group` |
| Region | `azure_arc_location` |
| Cloud | `azure_arc_cloud` |
| Resource name | `azure_arc_resource_name` |
| Tags | `azure_arc_tags` |
| Private link scope or Arc gateway | `azure_arc_private_link_scope_id`, `azure_arc_gateway_id` |
| Credentials | `azure_arc_service_principal_id` and the rest, for example a service principal per subscription |

### Servers already connected somewhere else

On each run the role compares the server's current connection (`azcmagent show`) with its target: tenant, subscription, resource group, region, cloud, and the resource name when you set one.

- Same target: nothing changes.
- Different target, `azure_arc_reconnect: false` (the default): the role stops with the current resource ID, so nothing moves by accident.
- Different target, `azure_arc_reconnect: true`: the role disconnects the server and connects it to the new target. Disconnecting deletes the old Azure resource, unless `azure_arc_disconnect_local_only` is true. The new resource has a new resource ID, and extensions, policy assignments and anything else attached to the old resource do not come with it.

The disconnect uses the credentials in the variables, so they must be allowed to delete the old resource. When the old resource is in another tenant, or the credentials cannot reach it, set `azure_arc_disconnect_local_only: true` and delete the old resource in Azure separately.

Tags apply only when the server connects. To change tags afterwards, change them in Azure.

## Signing in

The agent signs in on the server itself. The role writes the sign-in details to a file in a new root-only temporary directory, runs `azcmagent connect --config` (or `disconnect`), and deletes the directory whether the command worked or not. This is the configuration file method Microsoft recommends, so the credentials never appear on a command line, in a process list or in shell history. Microsoft also recommends a service principal used only for onboarding, with as few permissions as possible.

| `azure_arc_auth_method` | Variables | Notes |
|---|---|---|
| `service_principal_secret` (default) | `azure_arc_service_principal_id`, `azure_arc_service_principal_secret`, `azure_arc_tenant_id` | Keep the secret in Ansible Vault or a secret store. |
| `service_principal_certificate` | `azure_arc_service_principal_id`, `azure_arc_service_principal_certificate_file`, `azure_arc_tenant_id` | A PEM or PFX file with the private key and no password, on the control node (it can be vaulted). Microsoft calls certificates the more secure option. |
| `access_token` | `azure_arc_access_token` | A short-lived Azure Resource Manager token, for example from `az account get-access-token --resource https://management.azure.com --query accessToken -o tsv`. |

A service principal limited to one resource group:

```bash
az ad sp create-for-rbac --name sp-arc-onboarding \
  --role "Azure Connected Machine Onboarding" \
  --scopes /subscriptions/22222222-2222-2222-2222-222222222222/resourceGroups/rg-arc-prd-uks
```

Add more `--scopes` for more resource groups. Removing servers (`absent`) deletes their Azure resources, which needs the **Azure Connected Machine Resource Administrator** role as well, unless you use `azure_arc_disconnect_local_only`.

To pass a token without putting it on the command line, read it from the environment:

```yaml
azure_arc_auth_method: access_token
azure_arc_access_token: "{{ lookup('ansible.builtin.env', 'AZURE_ARC_ACCESS_TOKEN') }}"
```

## Role variables

Every variable is validated against [`meta/argument_specs.yml`](./meta/argument_specs.yml) when the role starts. Defaults are in [`defaults/main.yml`](./defaults/main.yml).

### State and package

| Variable | Default | Description |
|---|---|---|
| `azure_arc_state` | `present` | `present` or `absent`. See [What it does](#what-it-does). |
| `azure_arc_connect` | `true` | Connect the server. `false` installs and configures the agent only. |
| `azure_arc_package_state` | `present` | `present` installs if missing. `latest` also upgrades. |
| `azure_arc_version` | `""` | Exact version, as the package manager lists it: `1.69.03579.1568` on apt, `1.69.03579-1568` on dnf and zypper. Empty takes what the repository offers. Cannot be combined with `latest`. |

### Repository

| Variable | Default | Description |
|---|---|---|
| `azure_arc_manage_repository` | `true` | Add the repository and keys. Set `false` when the host already gets `azcmagent` from somewhere else, such as Red Hat Satellite or an internal mirror. |
| `azure_arc_repository_url` | `https://packages.microsoft.com` | Base URL. Point it at a mirror with the same layout if you have one. |
| `azure_arc_repository_path` | `""` | Path under the base URL, such as `rhel/9`. Empty works it out from the host's facts. |
| `azure_arc_remove_repository` | `false` | With `absent`, also remove the repository definition. Off by default because other Microsoft agents (such as Defender for Endpoint) use the same one. |

### Target

| Variable | Default | Description |
|---|---|---|
| `azure_arc_tenant_id` | `""` | Tenant of the subscription. Required to connect. |
| `azure_arc_subscription_id` | `""` | Subscription ID. Required to connect. Use the ID, not the name, so the role can compare it with the current connection. |
| `azure_arc_resource_group` | `""` | Resource group. Required to connect. |
| `azure_arc_location` | `""` | Region, such as `uksouth`. Required to connect. |
| `azure_arc_cloud` | `AzureCloud` | `AzureCloud`, `AzureUSGovernment` or `AzureChinaCloud`. |
| `azure_arc_resource_name` | `""` | Name of the Azure resource. Empty lets the agent use the host name (or the instance ID on AWS). It cannot change without reconnecting. |
| `azure_arc_tags` | `{}` | Tags, applied when the server connects. Names cannot contain commas, equals signs or spaces, and values cannot contain commas or single quotes. |
| `azure_arc_private_link_scope_id` | `""` | Resource ID of an Azure Arc private link scope, for private endpoints. |
| `azure_arc_gateway_id` | `""` | Resource ID of an Azure Arc gateway. |
| `azure_arc_reconnect` | `false` | Move a server connected somewhere else to the target. See [Servers already connected somewhere else](#servers-already-connected-somewhere-else). |

### Sign-in

| Variable | Default | Description |
|---|---|---|
| `azure_arc_auth_method` | `service_principal_secret` | `service_principal_secret`, `service_principal_certificate` or `access_token`. |
| `azure_arc_service_principal_id` | `""` | Application (client) ID of the onboarding service principal. |
| `azure_arc_service_principal_secret` | `""` | Its client secret. |
| `azure_arc_service_principal_certificate_file` | `""` | Path on the control node to its certificate and private key (PEM or PFX, no password). |
| `azure_arc_access_token` | `""` | An Azure Resource Manager access token. |
| `azure_arc_disconnect_local_only` | `false` | When disconnecting, reset the agent without deleting the Azure resource (`azcmagent disconnect --force-local-only`). Needs no credentials. |

A server that was never connected needs no credentials to remove.

### Local agent settings

Unset means "leave it as it is". These have no defaults on purpose.

| Variable | Type | `azcmagent config` property |
|---|---|---|
| `azure_arc_config_mode` | `full` or `monitor` | `config.mode` |
| `azure_arc_extensions_enabled` | bool | `extensions.enabled` |
| `azure_arc_extensions_allowlist` | list of `Publisher/Type` | `extensions.allowlist` (`[]` clears it) |
| `azure_arc_extensions_blocklist` | list of `Publisher/Type` | `extensions.blocklist` (`[]` clears it) |
| `azure_arc_guest_configuration_enabled` | bool | `guestconfiguration.enabled` |
| `azure_arc_incoming_connections_enabled` | bool | `incomingconnections.enabled` |
| `azure_arc_incoming_connections_ports` | list of ports | `incomingconnections.ports` (`[]` clears it) |
| `azure_arc_proxy_url` | string | `proxy.url` (`""` clears it) |
| `azure_arc_proxy_bypass` | list | `proxy.bypass` (`[]` clears it) |
| `azure_arc_config_extra` | dict, default `{}` | Any other property by name, such as `extensions.agent.cpulimit: 10` |

### Facts set by the role

| Fact | Description |
|---|---|
| `azure_arc_resource_id` | The server's Azure resource ID after the role connects it (or finds it connected), for later tasks such as installing extensions. Empty when not connected. |

## Locking the agent down

The agent's local settings are applied with `azcmagent config` on the server, and Microsoft documents them as local only: nobody can change them from Azure, not even an Owner ([Microsoft Learn: Extensions security](https://learn.microsoft.com/en-us/azure/azure-arc/servers/security-extensions)). The role reads the current settings, changes only those that differ, and applies them before connecting, so a server is locked down from its first heartbeat.

Microsoft's advice for locked-down servers:

- Prefer an allowlist to a blocklist, because new extensions are blocked until you allow them.
- Do not allow the Custom Script extension unless you need it, because it runs any command as root.
- Turn guest configuration off unless you need it, because a custom guest configuration policy could change the agent's settings.

Monitoring and Defender for Cloud only:

```yaml
azure_arc_extensions_allowlist:
  - Microsoft.Azure.Monitor/AzureMonitorLinuxAgent
  - Microsoft.Azure.AzureDefenderForServers/MDE.Linux
azure_arc_guest_configuration_enabled: false
azure_arc_incoming_connections_enabled: false
```

The extension manager running, but no extensions allowed:

```yaml
azure_arc_extensions_allowlist:
  - Allow/None
```

### Monitor mode

`azure_arc_config_mode: monitor` is Microsoft's one-setting lock-down: only monitoring and security extensions can be installed, and guest configuration is off. In monitor mode the agent manages the allowlist itself and refuses changes to it, so the role stops if you set `azure_arc_extensions_allowlist`, `azure_arc_extensions_blocklist` or `azure_arc_guest_configuration_enabled` alongside it.

Switching back to `full` keeps the allowlist monitor mode wrote. Set `azure_arc_extensions_allowlist` (or `[]` to clear it) in the same run.

Extensions already installed are not removed by any of these settings. Remove them in Azure.

## Repository and signing keys

The role writes the same repository definitions as Microsoft's `packages-microsoft-prod` package, with the same file names and repository IDs, and the same files as `libre_devops.mde_linux`. A server with both agents, or one set up by hand, ends up with one definition, not two that conflict.

| Package manager | Files written |
|---|---|
| dnf or yum | `/etc/pki/rpm-gpg/microsoft.asc`, `/etc/pki/rpm-gpg/microsoft-2025.asc`, `/etc/yum.repos.d/microsoft-prod.repo` (repository ID `packages-microsoft-com-prod`) |
| zypper | The same keys, and `/etc/zypp/repos.d/microsoft-prod.repo` |
| apt | `/usr/share/keyrings/microsoft-prod.gpg` and `/etc/apt/sources.list.d/microsoft-prod.list` |

Microsoft signs older repositories with its original key and newer ones (RHEL 10, Debian 13, Ubuntu 26.04 and SLES 16) with its 2025 key. The role installs both:

| Key | Fingerprint |
|---|---|
| `microsoft.asc` | `BC52 8686 B50D 79E3 39D3 721C EB3E 94AD BE12 29CF` |
| `microsoft-2025.asc` | `AA86 F75E 427A 19DD 3334 6403 EE4D 7792 F748 182B` |

The keys ship with the role, so nothing is downloaded on trust at run time. On RPM systems `rpm_key` imports each key only if its fingerprint matches, and package and metadata signature checks stay on. `just check-keys` checks the bundled files in CI. The fingerprints are published on [Microsoft Learn: Linux package repository](https://learn.microsoft.com/en-us/linux/packages).

## Examples

### Lock down without connecting, then connect later

```yaml
azure_arc_connect: false
azure_arc_config_mode: monitor
```

Run again with `azure_arc_connect: true` (the default) and the target variables to connect. For golden images, Microsoft's guidance is to connect each server after it is cloned or built from the image, because two agents with the same identity behave inconsistently.

### Through a proxy

```yaml
azure_arc_proxy_url: http://proxy.corp.example:8080
azure_arc_proxy_bypass:
  - Arc
```

The package manager needs its own proxy settings to reach packages.microsoft.com. The role does not change them.

### Through private endpoints

```yaml
azure_arc_private_link_scope_id: /subscriptions/22222222-2222-2222-2222-222222222222/resourceGroups/rg-arc-prd-uks/providers/Microsoft.HybridCompute/privateLinkScopes/pls-arc-prd-uks
```

### Move servers to a new resource group

```yaml
azure_arc_resource_group: rg-arc-prd-uks-new
azure_arc_reconnect: true
```

### Remove the agent

```yaml
azure_arc_state: absent
azure_arc_remove_repository: true   # only if nothing else on the server uses packages.microsoft.com
```

## Troubleshooting

| Symptom | What to check |
|---|---|
| `The module interpreter '/usr/bin/python3' was not found` or `Module result deserialization failed`, on the first task | The host only has Python 3.6 (RHEL 8, SLES 15), which ansible-core 2.17 and later cannot use. Every module fails, not only this role's. See [Python on managed nodes](./docs/using-without-galaxy.md#on-managed-nodes). |
| `azcmagent connect failed with exit code ...` and an `AZCM` code | The message shows the agent's own error. Microsoft explains each code at [aka.ms/arc/azcmerror](https://aka.ms/arc/azcmerror). `AZCM0041` (failed to get an access token) usually means a wrong tenant, client ID or secret, or an expired secret or token. |
| `This server is already connected as ...` | It is connected to a different target. Check its `host_vars` and `group_vars` with `ansible-inventory --host`, or set `azure_arc_reconnect: true` to move it. |
| The role reports the server connected elsewhere when it is not | `azure_arc_subscription_id` is a name rather than the ID, or `azure_arc_location` is spelt differently from the region name Azure uses. |
| `In monitor mode the agent manages ...` | Remove the allowlist, blocklist and guest configuration variables, or use `full` mode with your own allowlist. |
| `cannot modify specific config properties if not in full mode` | The agent was put in monitor mode outside the role. Set `azure_arc_config_mode: full` to manage the lists. |
| Tag rejected | A tag name has a comma, equals sign or space, or a value has a comma or single quote. Set that tag in Azure after connecting. |
| `/etc/cron.d/azcmagent_autoupgrade: No such file or directory` during install | The package's install script needs `/etc/cron.d`, which minimal installs without cron lack. The role creates it before installing, so this only appears when installing by hand. |
| Ansible fails on SLES 15 with a Python version error | The host is on Python 3.6. Install `python311` and set `ansible_python_interpreter: /usr/bin/python3.11`. |
| Network check failures during connect | Run `azcmagent check --location <region>` on the server, and compare with [Microsoft Learn: Network requirements](https://learn.microsoft.com/en-us/azure/azure-arc/servers/network-requirements). |

## Developing

You need [uv](https://docs.astral.sh/uv/), [just](https://github.com/casey/just) and [podman](https://podman.io/). CI runs the same recipes.

| Recipe | What it does |
|---|---|
| `just deps` | Installs the pinned Python tools (`uv.lock`) and the collections in `requirements.yml` into `.ansible/`. |
| `just lint` | `yamllint`, `ansible-lint` on the production profile, and `zizmor` on the workflows. |
| `just test [scenario] [platform]` | Runs a Molecule scenario on one platform, or on all of them. `just test monitor ubuntu2404`. |
| `just test-all` | Every scenario on every platform, one after another. |
| `just converge <scenario> <platform>` | Runs a scenario on one platform and leaves the container running, pass or fail. `just login` opens a shell in it, `just destroy` removes it. |
| `just live <inventory> [args]` | Runs [`examples/playbook.yml`](./examples/playbook.yml) against real servers, connecting them. |
| `just check-keys` | Checks the bundled keys match the fingerprints the role trusts. |
| `just secrets` | Scans the git history with gitleaks. |
| `just platforms`, `just scenarios` | List what can be tested. |

### Molecule scenarios

| Scenario | What it proves |
|---|---|
| `default` | Repository, keys, package and service, an allowlist, guest configuration off, a port list, a proxy and bypass list and an extra property, then a second run with no changes. |
| `monitor` | Monitor mode and remote access off, then a second run with no changes. |
| `remove` | `absent` on a server that was never connected removes the agent and the repository without credentials. |

Every platform runs systemd as PID 1 in a podman container, built from the vendor's image with `sudo`, Python, and the few packages a normal server install has and the minimal images lack. The containers get the `SYS_ADMIN` capability, because `himdsd` keeps it and rootless containers do not grant it by default. Container names start with `arc-`, so tests of the other Libre DevOps roles can run alongside. The containers cannot connect to Azure, and GitHub's hosted runners are Azure VMs, where Microsoft does not support Azure Arc. Test connecting, reconnecting and disconnecting against a lab server outside Azure with `just live`, using a test subscription.

## Releasing

1. Move the Unreleased entries in [`CHANGELOG.md`](./CHANGELOG.md) under the new version.
2. Run `just release 1.2.3`. It tags the commit and pushes the tag.
3. The Release workflow creates the GitHub release and imports the tag into Galaxy, which takes the version from the tag. It needs the `GALAXY_API_KEY` secret, which the libre-devops organisation provides. If the import fails, fix the cause and run the Release workflow by hand with the tag to import it again.

## Sources

Microsoft Learn pages this role is built from, retrieved 8 November 2026.

| Topic | Page |
|---|---|
| Prerequisites and supported systems | [Connected Machine agent prerequisites](https://learn.microsoft.com/en-us/azure/azure-arc/servers/prerequisites) |
| Connect options and sign-in methods | [`azcmagent connect`](https://learn.microsoft.com/en-us/azure/azure-arc/servers/azcmagent-connect) |
| Disconnect | [`azcmagent disconnect`](https://learn.microsoft.com/en-us/azure/azure-arc/servers/azcmagent-disconnect) |
| The `--config` file | [`azcmagent` CLI reference](https://learn.microsoft.com/en-us/azure/azure-arc/servers/azcmagent) |
| Local settings | [`azcmagent config`](https://learn.microsoft.com/en-us/azure/azure-arc/servers/azcmagent-config) |
| Allowlists, blocklists and monitor mode | [Extensions security for Azure Arc-enabled servers](https://learn.microsoft.com/en-us/azure/azure-arc/servers/security-extensions) |
| Package installs and versions | [Manage the Azure Connected Machine agent](https://learn.microsoft.com/en-us/azure/azure-arc/servers/manage-agent) |
| Microsoft's Ansible role | [Connect machines to Azure Arc at scale using Ansible](https://learn.microsoft.com/en-us/azure/azure-arc/servers/onboard-ansible-playbooks) |
| Signing keys | [Linux package repository for Microsoft software](https://learn.microsoft.com/en-us/linux/packages) |

## Licence

[MIT](./LICENSE).
