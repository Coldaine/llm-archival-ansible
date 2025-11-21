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
1. bitwarden-setup
2. tailscale (optional)
3. cloudflare_tunnel (optional)
4. claude_unified / mcp_servers / llm_archival / postgresql_server

## Python Extractors
- Place extractor scripts in `LLM_Parsing_Project/prototypes/`
- Ansible will invoke them automatically during archival

## PostgreSQL
- Optional: enable on infrastructure servers for central storage
	- Add `enable_postgres: true` in host/group vars (future gating variable)
	- Provides containerized `postgres:15-alpine` with schema bootstrap

Not yet invoked automatically in `deploy-all-configs.yml`—add deliberately once a target host is prepared (disk, backup policy). A future gating variable (`enable_postgres`) will control inclusion.

## Network (Optional)
Tailscale & Cloudflare Tunnel are disabled by default.

Enable by editing `group_vars/all/network.yml`:
```yaml
enable_tailscale: true
enable_cloudflare_tunnel: true
tailscale_auth_key: "tskey-auth-..."   # In vault.yml (encrypted)
cloudflare_tunnel_hostname: "tunnel.example.com"
cloudflare_tunnel_services:
	- "localhost:3456"
	- "localhost:8080"
```
Then run:
```sh
ansible-playbook -i inventory/hosts playbooks/deploy-all-configs.yml --ask-vault-pass
```

Notes:
- Cloudflare tunnel will only expose explicitly listed services; omit to keep tunnel dormant.
- Ensure `tailscale_auth_key` and `cloudflare_api_token` are present in encrypted `vault.yml` before enabling.

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

