# Security policy

## Reporting a vulnerability

Report vulnerabilities in this role privately, through [GitHub's private vulnerability reporting](https://github.com/libre-devops/ansible-role-azure-arc/security/advisories/new). Do not open a public issue.

Vulnerabilities in the Azure Connected Machine agent itself go to the [Microsoft Security Response Center](https://msrc.microsoft.com/report).

## What the role handles that is sensitive

- **Onboarding credentials.** The service principal secret, certificate or access token reaches the server only in a file in a new root-only temporary directory, which the role deletes straight after `azcmagent connect` or `azcmagent disconnect`, whether the command worked or not. They are never on a command line, so they never show in a process list or shell history. The tasks that handle them use `no_log`, and the secret and token are marked `no_log` in the argument specification.
- **The service principal.** Give it the Azure Connected Machine Onboarding role on the resource groups it onboards to, and nothing else. Microsoft recommends a service principal dedicated to onboarding, and certificates over secrets.
- **Signing keys.** The role ships Microsoft's public keys for packages.microsoft.com and imports them only when the fingerprint matches. `just check-keys` checks the bundled files in CI.
