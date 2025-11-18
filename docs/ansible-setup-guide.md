# Ansible Workstation Bootstrap Guide

Your personal machine automation system for managing dotfiles, dependencies, and secrets across multiple computers (Windows, Linux, macOS, Raspberry Pi).

## Quick Start

```bash
# On a fresh machine:
curl -fsSL https://raw.githubusercontent.com/yourusername/ansible-config/main/bootstrap.sh | bash

# Then enter your Ansible Vault password (you create this once)
# Ansible handles the rest: installs packages, deploys dotfiles, configures BWS
```

---

## Core Concept

Instead of manually configuring each machine, store your setup as code in [[Ansible]] playbooks. Run it once on each machine, and they all stay synchronized.

**Key insight:** You don't configure machines by hand anymore—you commit configuration to Git and replicate it everywhere.

---

## Phase 1: Discovery (Survey Your Current Machines)

### Linux/Mac Discovery

```bash
# Get all installed packages
ansible localhost -m package_facts > linux_packages.json

# Find all your dotfiles
find ~ -maxdepth 1 -name ".*" -type f > dotfiles_list.txt
find ~/.config -type f >> dotfiles_list.txt
```

### Windows Discovery

**Chocolatey:**
```powershell
choco list --local-only > chocolatey_packages.txt
```

**Scoop:**
```powershell
scoop export > scoop_packages.json
```

### Raspberry Pi Discovery

```bash
# From your main machine
echo "raspberrypi ansible_host=192.168.1.100" > /tmp/rpi_inventory
ansible raspberrypi -i /tmp/rpi_inventory -m package_facts > rpi_packages.json
```

---

## Phase 2: Curate Your Package List

Create a spreadsheet (CSV) with columns:
- **Package**: git, python3, docker, etc.
- **Type**: apt/yum, chocolatey, scoop
- **Keep**: Y/N

Remove one-off experiments, keep essential tools:
- `git`, `python3`, `nodejs`, `docker`
- `vim`, `neovim`, `tmux`
- `code` (VS Code)
- `gh` (GitHub CLI)
- `bws` (Bitwarden Secrets Manager)

Export as `curated_packages.csv`

---

## Phase 3: Repository Structure

Create a Git repo with this structure:

```
ansible-config/
├── bootstrap.sh              # One-liner to start setup
├── playbook.yml              # Main playbook
├── group_vars/
│   └── all/
│       └── vault.yml         # ENCRYPTED BWS token & secrets
├── roles/
│   ├── packages/
│   │   └── tasks/main.yml
│   ├── dotfiles/
│   │   └── tasks/main.yml
│   └── bitwarden-setup/      # NEW: Handles BWS config
│       └── tasks/main.yml
├── templates/
│   ├── bashrc.j2
│   ├── gitconfig.j2
│   └── zshrc.j2
├── files/
│   └── (static config files)
└── .gitignore                # Keep sensitive local files out
```

---

## Phase 4: The Bitwarden Setup Integration

### BWS Vault Setup (Encrypted Secrets Storage)

Create `group_vars/all/vault.yml`:

```yaml
# group_vars/all/vault.yml - This file will be ENCRYPTED with Ansible Vault
bws_access_token: "your-actual-bws-token-here"
bws_secrets:
  github_token_id: "your-secret-id-in-bws"
  openai_api_key_id: "another-secret-id"
  database_url_id: "etc"
```

**Encrypt it:**
```bash
ansible-vault encrypt group_vars/all/vault.yml
# Enter a new password (e.g., "my-ansible-vault-password")
```

**Verify encryption:**
```bash
cat group_vars/all/vault.yml
# Output: $ANSIBLE_VAULT;1.1;AES256;filter_default;... (gibberish)
```

### The Bitwarden Setup Role

Create `roles/bitwarden-setup/tasks/main.yml`:

```yaml
---
- name: Install BWS CLI
  package:
    name: bws
    state: present
  become: yes

- name: Create BWS config directory
  file:
    path: "{{ ansible_env.HOME }}/.config/bws"
    state: directory
    mode: '0700'

- name: Deploy BWS access token securely
  copy:
    content: "{{ bws_access_token }}"
    dest: "{{ ansible_env.HOME }}/.config/bws/token"
    mode: '0600'
  no_log: yes  # Don't print the token in logs

- name: Add BWS token to shell environment
  lineinfile:
    path: "{{ ansible_env.HOME }}/.bashrc"
    line: 'export BWS_ACCESS_TOKEN=$(cat ~/.config/bws/token)'
    create: yes

- name: Verify BWS is functional
  shell: "bws secret list"
  environment:
    BWS_ACCESS_TOKEN: "{{ bws_access_token }}"
  register: bws_check
  failed_when: bws_check.rc != 0
```

### Main Playbook Integration

Your main `playbook.yml`:

```yaml
---
- name: Bootstrap my development environment
  hosts: localhost
  connection: local
  vars_files:
    - group_vars/all/vault.yml

  roles:
    - bitwarden-setup      # Deploy BWS token FIRST
    - packages             # Install packages
    - dotfiles             # Deploy config files
    - development-tools    # Any other setup

  tasks:
    - name: Create project .env with BWS secrets
      template:
        src: templates/env.j2
        dest: "~/projects/.env"
        mode: '0600'
      no_log: yes

    - name: Test Bitwarden integration
      shell: |
        export BWS_ACCESS_TOKEN=$(cat ~/.config/bws/token)
        bws secret list
      register: test_result
      failed_when: test_result.rc != 0
      tags: validate
```

---

## Phase 5: The Bootstrap Script

Create `bootstrap.sh`:

```bash
#!/bin/bash
set -e

echo "🚀 Bootstrapping machine..."
echo ""

# Detect OS
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    echo "📦 Detected: Linux"
    sudo apt update && sudo apt install -y ansible git python3-pip
elif [[ "$OSTYPE" == "darwin"* ]]; then
    echo "📦 Detected: macOS"
    brew install ansible git
elif [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "win32" ]]; then
    echo "📦 Detected: Windows"
    choco install -y ansible git
else
    echo "❌ Unsupported OS: $OSTYPE"
    exit 1
fi

echo ""
echo "📥 Cloning configuration repository..."
git clone https://github.com/yourusername/ansible-config.git /tmp/ansible-setup
cd /tmp/ansible-setup

echo ""
echo "⚙️  Running Ansible setup (will prompt for Vault password)..."
echo "💡 Your Ansible Vault password is the same one you set during repo setup"
echo ""

# Run playbook with vault password prompt
ansible-playbook playbook.yml --ask-vault-pass -K

echo ""
echo "✅ Setup complete!"
echo ""
echo "Next steps:"
echo "  1. Verify BWS is working: bws secret list"
echo "  2. Check your environment: echo \$BWS_ACCESS_TOKEN"
echo "  3. Test a secret fetch: source ~/.bashrc && bws secret get <id>"
```

(…remaining sections preserved from original for full reference…)
