# Infrastructure Ansible Subrepo

This folder contains the Ansible project for automating workstation and server setup across Windows and Linux machines.

Contents:
- inventory/: Hosts and groups
- playbooks/: Orchestration playbooks
- roles/: Modular roles (bitwarden-setup, llm_archival, mcp_servers, claude_unified)
- templates/: Jinja2 templates
- group_vars/: Shared and encrypted variables
- docs/: Reference docs moved from Configuration/

Repo usage:
- Initialize: git init; add and commit
- Optional: add as git submodule from the parent repository
- Run playbooks from this folder with your chosen inventory

Security:
- Do NOT commit real secrets. Use Ansible Vault for `group_vars/all/vault.yml`.
- Prefer Bitwarden Secrets Manager for runtime secret retrieval.
