# Infrastructure Ansible Subrepo

# LLM Archival Ansible Automation

This repository automates LLM chat history archival, parsing, and infrastructure setup across Windows and Linux machines.

## Features
- Application-aware backup and extraction (Claude, Copilot, Goose, etc.)
- Orchestrates Python extractor prototypes
- Schedules archival (Windows Task Scheduler, Linux cron)
- Optional PostgreSQL deployment for central storage
- Security-first: secrets via Ansible Vault, sanitization patterns
- Optional network stack: Tailscale mesh VPN & Cloudflare Tunnel (disabled by default)

## Architecture Overview (Current State)
The playbook coordinates a linear set of roles:
1. `bitwarden-setup` seeds secret access (future: dynamic fetch)
2. Optional network hardening (`tailscale`, `cloudflare_tunnel`)
3. Application / tooling roles (`claude_unified`, `mcp_servers`) – placeholders for future config templating
4. `llm_archival` invokes extractor prototypes and sets up scheduling
5. (Optional / not yet auto-run) `postgresql_server` can provide a central store

Data Flow (today): Source application files -> Python extractor scripts (inline) -> Parsed artifacts left on host (future: direct DB insert).

Security Layers: Secrets isolated in Vault, optional network isolation (mesh VPN + tunnel), planned sanitizer role for outbound payload scrubbing.

## Planned Enhancements (Tracking)
| Area | Planned Change | Status |
|------|----------------|--------|
| Deduplication | Hybrid: message-level uniqueness + similarity consolidation in PostgreSQL | Pending design |
| Direct Ingestion | Move extractor output from filesystem to immediate `INSERT ... ON CONFLICT` | Pending |
| Sanitization | Add `security_sanitizer` role (regex filters for API keys, PII) | Pending |
| Role Docs | Per-role README for clarity | In progress |
| Network Gating | Variables to toggle tunnel service definitions | Partial |
| Scheduling Runner | External orchestrator instead of inline Python | Future |

## Dedup Strategy (Planned)
Current implementation performs straightforward extraction without persistence-layer dedup. Planned approach:
- Primary Keys: (`provider`, `conversation_id`, `message_id`) for strict uniqueness.
- Content Hash: SHA-256 of normalized message text for quick duplicate detection.
- Similarity Window: Periodic job (e.g., weekly) clusters near-duplicates using cosine similarity (threshold ~0.95) for consolidation.
- File Growth Handling: Append-only source logs produce evolving files; only new message boundaries processed each run (offset tracking table).

Nothing here alters non-Ansible vault organization—scope remains confined to this subrepo.

## Role Index
| Role | Purpose | Optional |
|------|---------|----------|
| `bitwarden-setup` | Seeds Bitwarden token / env prerequisites | Required if using Bitwarden |
| `tailscale` | Establishes mesh VPN for secure host connectivity | Yes |
| `cloudflare_tunnel` | Exposes selected internal services via authenticated tunnel | Yes |
| `claude_unified` | Placeholder for unified Claude configuration templating | Yes (future active) |
| `mcp_servers` | Placeholder for MCP server configuration | Yes (future active) |
| `llm_archival` | Sets up extractors + scheduling for LLM backups | Core |
| `postgresql_server` | Deploys containerized PostgreSQL for central storage | Yes |


## Getting Started
1. Clone the repo:
	```sh
	git clone https://github.com/Coldaine/llm-archival-ansible.git
	cd llm-archival-ansible
	```
2. Install Ansible and required collections:
	```sh
	ansible-galaxy collection install -r collections/requirements.yml
	```
3. Create and encrypt your secrets file:
	```sh
	ansible-vault create group_vars/all/vault.yml
	# Add secrets (see vault.yml.sample)
	```
4. Run playbooks:
	```sh
	ansible-playbook -i inventory/hosts playbooks/deploy-all-configs.yml --limit PATRICK-DESKTOP
	```

## Secrets
- Store secrets in `group_vars/all/vault.yml` (encrypted with Ansible Vault)
- Never commit real secrets; `.gitignore` protects this file
- Use `vault.yml.sample` as a starting template, then:
	```sh
	cp group_vars/all/vault.yml.sample group_vars/all/vault.yml
	ansible-vault encrypt group_vars/all/vault.yml
	```
	Re-run playbooks with `--ask-vault-pass` or configure a vault password file outside the repo.

### Bitwarden Integration
- `bitwarden-setup` role seeds BWS token/environment (extend with lookups when ready).
- Future enhancement: dynamic secret fetch (GitHub PAT, API keys) before other roles.

### Order of Operations
The `deploy-all-configs.yml` playbook runs roles in this order:
1. bitwarden-setup (required)
2. claude_unified (placeholder, future)
3. mcp_servers (placeholder, future)
4. llm_archival (core functionality)
5. tailscale (optional, controlled by `enable_tailscale`)
6. cloudflare_tunnel (optional, controlled by `enable_cloudflare_tunnel`)

**Note:** `postgresql_server` role is NOT auto-included. See PostgreSQL section below for manual setup.

## Python Extractors
- Place extractor scripts in `LLM_Parsing_Project/prototypes/`
- Ansible will invoke them automatically during archival

## PostgreSQL (Manual Setup Only)
The `postgresql_server` role provides optional centralized storage but is **NOT automatically run** by `deploy-all-configs.yml`.

**To enable PostgreSQL:**

1. Add the role manually to your playbook or create a separate playbook:
```yaml
# playbooks/deploy-postgres.yml
---
- name: Deploy PostgreSQL Server
  hosts: infrastructure_servers
  become: false
  roles:
    - postgresql_server
```

2. Run the PostgreSQL deployment:
```sh
ansible-playbook -i inventory/hosts playbooks/deploy-postgres.yml --ask-vault-pass
```

**Requirements:**
- Target host must have Docker installed
- Ensure `vault_postgres_password` is set in encrypted `vault.yml`
- Plan for disk space (default: `/mnt/postgres_data`)
- Establish backup policy before production use

**Future:** A gating variable (`enable_postgres`) will allow conditional inclusion in the main playbook.

## Network (Optional)
Tailscale & Cloudflare Tunnel are disabled by default.

**Step 1: Add secrets to encrypted vault.yml:**
```yaml
# In group_vars/all/vault.yml (encrypted with ansible-vault)
tailscale_auth_key: "tskey-auth-ACTUAL_KEY_HERE"
cloudflare_api_token: "CLOUDFLARE_API_TOKEN_HERE"
```

**Step 2: Enable features in network.yml:**
```yaml
# In group_vars/all/network.yml (unencrypted, safe to commit)
enable_tailscale: true
enable_cloudflare_tunnel: true
cloudflare_tunnel_hostname: "tunnel.example.com"
cloudflare_tunnel_services:
	- "localhost:3456"
	- "localhost:8080"
```

**Step 3: Deploy:**
```sh
ansible-playbook -i inventory/hosts playbooks/deploy-all-configs.yml --ask-vault-pass
```

**Important:**
- Never put auth keys or API tokens in `network.yml` - they belong only in encrypted `vault.yml`
- Cloudflare tunnel will only expose explicitly listed services; omit to keep tunnel dormant

## Scheduling
- Windows: `win_scheduled_task` for 6h archival cadence
- Linux: cron job (`cron.yml`) every 6 hours
- Extractor tasks run inline (swap to dedicated runner later for scale)

## Extending
- Add new tools by appending paths to `tool_paths.yml` and updating extractor scripts.
- Introduce sanitization patterns in a dedicated role (`security_sanitizer`) later.

## Change Log (Documentation Only)
- 2025-11-18: Added architecture overview, planned dedup section, role index, PostgreSQL invocation note.

## License
MIT
