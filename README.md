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

## Scheduling
- Windows: `win_scheduled_task` for 6h archival cadence
- Linux: cron job (`cron.yml`) every 6 hours
- Extractor tasks run inline (swap to dedicated runner later for scale)

## Extending
- Add new tools by appending paths to `tool_paths.yml` and updating extractor scripts.
- Introduce sanitization patterns in a dedicated role (`security_sanitizer`) later.

## License
MIT
