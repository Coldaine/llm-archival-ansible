# Ansible Deployment for Claude Configurations
**Created**: 2025-11-17
**Status**: Planning Phase - Part of Master Infrastructure Automation
**Related**: [[Master-Infrastructure-Automation-Plan]] | [[Standards for Claude MD Files]] | [[Kilo Code Deployment Standards]]

## ⚠️ Important Notice

**This document is part of the larger [[Master-Infrastructure-Automation-Plan]] which is currently in planning phase.**

This specific guide focuses on Claude MD and Kilo Code configuration deployment. For the complete automation vision including MCP servers, Bitwarden secrets, and LLM archival, see the Master Plan.

---

## Scope

This playbook deploys:
1. **CLAUDE.md files** - Root and project-specific configurations
2. **Kilo Code settings** - Provider configurations, custom modes, API keys
3. **Sync scripts** - Automated sync back to Obsidian vault

**What this does NOT cover** (see Master Plan):
- MCP server deployment
- Bitwarden secrets integration
- LLM history archival
- Full infrastructure provisioning

---

## Target Infrastructure

Based on current inventory from [[Master-Infrastructure-Automation-Plan]]:

| Hostname | Location | OS | Use | Access |
|----------|----------|----|----|--------|
| **PATRICK-DESKTOP** | Local | Windows 11 | Primary workstation | Local |
| **Icarus** | Local | Windows 11 | Dev machine | Local/Tailscale (future) |
| **Zo Computer** | DigitalOcean | Linux | VPS | `ssh zo` |
| **Raspberry Pi** | Local | Linux | Home server | `ssh pi` |

---

## File Structure (Aligns with Master Plan)

```
E:\Obsidian Vault\
├── Configuration\
│   ├── Master-Configs\
│   │   └── CLAUDE.md                    # Global master configuration
│   ├── Machine-Specific\
│   │   ├── PATRICK-DESKTOP\
│   │   │   ├── CLAUDE.md                # Machine overrides
│   │   │   └── kilo-settings.json
│   │   ├── zo\
│   │   ├── pi\
│   │   └── icarus\
│   └── Templates\
│       ├── CLAUDE.md.j2                 # Ansible Jinja2 template
│       └── kilo-settings.json.j2
├── Operations and Infrastructure\
│   └── Ansible\
│       ├── inventory\
│       │   ├── hosts                    # All machines
│       │   └── group_vars\
│       ├── playbooks\
│       │   ├── deploy-claude-configs.yml    # This playbook
│       │   └── sync-to-obsidian.yml         # Reverse sync
│       └── roles\
│           └── claude_unified\
│               ├── tasks\
│               ├── templates\
│               └── files
```

---

## Playbook: deploy-claude-configs.yml

**Location**: `Operations and Infrastructure/Ansible/playbooks/deploy-claude-configs.yml`

```yaml
---
- name: Deploy Claude and Kilo Code Configurations
  hosts: all
  become: false
  vars:
    obsidian_vault_base: "{{ vault_paths[ansible_os_family] }}"
    claude_config_dir: "{{ ansible_user_dir }}/.claude"
    kilo_config_dir: "{{ kilo_paths[ansible_os_family] }}"

  vars_files:
    - ../group_vars/all/vault.yml  # Encrypted with Ansible Vault

  tasks:
    - name: Display target machine info
      debug:
        msg: "Deploying to {{ inventory_hostname }} ({{ ansible_os_family }})"

    - name: Ensure .claude directory exists
      file:
        path: "{{ claude_config_dir }}"
        state: directory
        mode: '0755'

    - name: Deploy root CLAUDE.md from template
      template:
        src: ../templates/CLAUDE.md.j2
        dest: "{{ claude_config_dir }}/CLAUDE.md"
        backup: yes
        mode: '0644'
      vars:
        hostname: "{{ inventory_hostname }}"
        machine_type: "{{ machine_types[inventory_hostname] }}"

    - name: Deploy Kilo Code settings (if applicable)
      template:
        src: ../templates/kilo-settings.json.j2
        dest: "{{ kilo_config_dir }}/kilo-code-settings.json"
        backup: yes
        mode: '0600'
      when: "'kilo_code_hosts' in group_names"

    - name: Deploy sync script (Windows)
      template:
        src: ../templates/sync-claude-configs.ps1.j2
        dest: "{{ obsidian_vault_base }}\\Configuration\\Claude-MD-Files\\sync-claude-configs.ps1"
        mode: '0755'
      when: ansible_os_family == "Windows"

    - name: Deploy sync script (Linux)
      template:
        src: ../templates/sync-claude-configs.sh.j2
        dest: "{{ obsidian_vault_base }}/Configuration/Claude-MD-Files/sync-claude-configs.sh"
        mode: '0755'
      when: ansible_os_family != "Windows"
```

---

## Inventory Configuration

**File**: `Operations and Infrastructure/Ansible/inventory/hosts`

```ini
[all:vars]
ansible_python_interpreter=/usr/bin/python3

# Windows machines
[windows_workstations]
PATRICK-DESKTOP ansible_connection=local ansible_os_family=Windows
icarus ansible_host=icarus.local ansible_connection=winrm ansible_os_family=Windows

# Linux machines
[linux_servers]
zo ansible_host=ts1.zocomputer.io ansible_user=root ansible_ssh_private_key_file=~/.ssh/zo
pi ansible_host=192.168.1.49 ansible_user=coldaine ansible_ssh_private_key_file=~/.ssh/id_ed25519_pi

# Machines with Kilo Code installed
[kilo_code_hosts]
PATRICK-DESKTOP
icarus
zo

# Machines with Claude Desktop
[claude_desktop_hosts]
PATRICK-DESKTOP
icarus
```

---

## Group Variables

**File**: `group_vars/all/main.yml`

```yaml
---
# Obsidian vault paths by OS family
vault_paths:
  Windows: "E:\\Obsidian Vault"
  Debian: "/opt/obsidian-vault"
  RedHat: "/opt/obsidian-vault"
  Darwin: "/Users/Shared/obsidian-vault"

# Kilo Code config paths by OS family
kilo_paths:
  Windows: "C:\\Users\\{{ ansible_user }}\\.kilocode"
  Debian: "/home/{{ ansible_user }}/.kilocode"
  RedHat: "/home/{{ ansible_user }}/.kilocode"
  Darwin: "/Users/{{ ansible_user }}/.kilocode"

# Machine type classifications
machine_types:
  PATRICK-DESKTOP: "primary_workstation"
  icarus: "dev_machine"
  zo: "linux_server"
  pi: "home_server"
```

---

## Template: CLAUDE.md.j2

**File**: `templates/CLAUDE.md.j2`

```jinja2
# {{ hostname }} - Claude Code Root Configuration

## Meta Information
- **Created**: {{ ansible_date_time.date }}
- **Last Updated**: {{ ansible_date_time.date }}
- **Hostname**: {{ hostname }}
- **Location**: `{{ claude_config_dir }}/CLAUDE.md`
- **Deployment Method**: Ansible (automated)
- **Machine Type**: {{ machine_type }}
- **Sync Status**: Managed by Ansible
- **Sync Location**: `{{ obsidian_vault_base }}/Configuration/Machine-Specific/{{ hostname }}/CLAUDE.md`

## Obsidian Vault Location (MANDATORY)
**This section MUST be present in ALL CLAUDE.md files**

- **Path**: `{{ obsidian_vault_base }}`
- **Purpose**: Primary location for all documentation, research notes, and technical references
- **Format**: Use markdown (.md) format for all documentation

**Folder Structure**:
- `Configuration/` - Claude MD files, Kilo Code settings, system configurations
- `Configuration/Master-Configs/` - Global master configurations
- `Configuration/Machine-Specific/` - Per-machine configurations
- `Operations and Infrastructure/` - Docker, deployment, infrastructure docs
- `Operations and Infrastructure/Ansible/` - Ansible playbooks and roles
- `Operations and Infrastructure/MCP-Servers/` - MCP server inventory
- `Research/` - Experiments, comparisons, investigations
- `Projects/` - Project-specific documentation

**Standards Documentation**:
- See `{{ obsidian_vault_base }}/Configuration/Standards for Claude MD Files.md`
- See `{{ obsidian_vault_base }}/Configuration/Kilo Code Deployment Standards.md`
- See `{{ obsidian_vault_base }}/Operations and Infrastructure/Master-Infrastructure-Automation-Plan.md`

## Platform Information
- **Host OS**: {{ ansible_distribution }} {{ ansible_distribution_version }}
- **Platform**: {{ ansible_system | lower }}
- **OS Family**: {{ ansible_os_family }}
- **Architecture**: {{ ansible_architecture }}
- **Machine Type**: {{ machine_type }}
- **Last Ansible Deployment**: {{ ansible_date_time.iso8601 }}

{% include 'claude-behavioral-guidelines.j2' %}
{% include 'claude-mcp-tools.j2' %}
{% if ansible_os_family == "Windows" %}
{% include 'claude-windows-specific.j2' %}
{% elif ansible_os_family in ["Debian", "RedHat"] %}
{% include 'claude-linux-specific.j2' %}
{% endif %}

## Ansible Management

**This file is managed by Ansible. Local edits will be overwritten on next deployment.**

**To update this configuration**:
1. Edit the template in: `{{ obsidian_vault_base }}/Operations and Infrastructure/Ansible/templates/CLAUDE.md.j2`
2. Or edit machine-specific vars in: `{{ obsidian_vault_base }}/Configuration/Machine-Specific/{{ hostname }}/`
3. Run: `ansible-playbook deploy-claude-configs.yml --limit {{ hostname }}`
4. Configuration will sync back to Obsidian vault automatically

**Deployment Info**:
- **Playbook**: deploy-claude-configs.yml
- **Last Run**: {{ ansible_date_time.iso8601 }}
- **Control Node**: {{ ansible_control_node | default('Unknown') }}
```

---

## Deployment Commands

```bash
cd "E:\\Obsidian Vault\\Operations and Infrastructure\\Ansible"
ansible all -i inventory/hosts -m ping
ansible-playbook -i inventory/hosts playbooks/deploy-claude-configs.yml --check
ansible-playbook -i inventory/hosts playbooks/deploy-claude-configs.yml
ansible-playbook -i inventory/hosts playbooks/deploy-claude-configs.yml --limit PATRICK-DESKTOP
ansible-playbook -i inventory/hosts playbooks/deploy-claude-configs.yml --limit windows_workstations
ansible-playbook -i inventory/hosts playbooks/deploy-claude-configs.yml --ask-vault-pass
```

---

## Integration with Master Plan

Related playbooks, execution order, and Bitwarden integration remain as described in the Master Plan.

---

**Last Updated**: 2025-11-17
**Status**: Planning Phase
**Next Review**: After Master Plan implementation starts
