# bitwarden-setup Role

Purpose: Prepare environment for secret retrieval using Bitwarden (CLI or Secrets Manager) before other roles run.

## Responsibilities
- Ensure Bitwarden token is available (manual entry into `vault.yml`).
- (Future) Perform login/unlock and export required secrets as Ansible facts.
- Provide foundation for dynamic secret resolution (API keys, PATs) used by archival & network roles.

## Current Behavior
- Reads `bws_access_token` from encrypted `group_vars/all/vault.yml`.
- Placeholder for extended logic (no live CLI calls yet).

## Variables
| Variable | Location | Description |
|----------|----------|-------------|
| `bws_access_token` | `vault.yml` | Bitwarden access or service token. |
| (future) `bws_org_id` | `vault.yml` | Organization identifier for shared secrets. |
| (future) `bws_secrets` | vars | Mapping of logical names to Bitwarden item IDs. |

## Planned Enhancements
- Automatic login & unlock workflow.
- Fetch secret items and register as facts (`set_fact`).
- Per-role secret dependency resolution (e.g., Cloudflare, Tailscale keys).

## Usage
Included early in the playbook so subsequent roles can rely on secrets having been staged.

## Scope
No modification of broader vault organization; isolated to secret preparation for this Ansible subrepo.
