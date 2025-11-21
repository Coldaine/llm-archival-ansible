# Ansible Reference

**Ansible** is an **agentless infrastructure automation tool** that runs playbooks (configuration as code) to configure servers, install packages, deploy files, and manage systems.

## Why Ansible for Personal Machines?

- **No agents needed** - Just SSH or local connection
- **YAML syntax** - Easy to read and write
- **Idempotent** - Run the same playbook repeatedly, only changes what needs changing
- **Cross-platform** - Works on Linux, macOS, Windows
- **Batteries included** - 5000+ built-in modules for common tasks

## Core Concepts

### Playbook
A YAML file describing what to configure.

```yaml
---
- name: Configure my laptop
  hosts: localhost
  connection: local
  tasks:
    - name: Install git
      package:
        name: git
        state: present
```

### Roles
Organized collections of tasks, templates, files, and variables.

```
roles/
├── bitwarden-setup/
│   ├── tasks/
│   │   └── main.yml
│   ├── templates/
│   │   └── bashrc.j2
│   └── files/
│       └── config
├── packages/
│   └── tasks/main.yml
└── dotfiles/
    └── tasks/main.yml
```

### Modules
Built-in actions that do specific things. Common ones:

- `package` - Install/remove packages (apt, yum, brew, etc.)
- `copy` - Copy files
- `template` - Deploy templated files (Jinja2 variables)
- `lineinfile` - Add/modify lines in files
- `shell` - Run shell commands
- `file` - Create directories, set permissions
- `user` - Manage user accounts
- `service` - Start/stop services
- `git` - Clone/pull Git repositories

### Variables
Store data to reuse in playbooks.

```yaml
vars:
  python_version: python3
  packages:
    - git
    - docker
    - nodejs

tasks:
  - name: Install packages
    package:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
```

### Templates
Jinja2 templates for dynamic file generation.

```jinja2
# bashrc.j2
export GITHUB_TOKEN={{ github_token }}
export OPENAI_API_KEY={{ openai_api_key }}

{% if ansible_os_family == "Debian" %}
# Debian-specific config
{% endif %}
```

### Vault
Encrypt sensitive data in playbooks.

```bash
# Encrypt file
ansible-vault encrypt group_vars/all/vault.yml

# Run playbook with vault
ansible-playbook playbook.yml --ask-vault-pass
```

## Basic Playbook Structure

```yaml
---
- name: Description of what this playbook does
  hosts: localhost            # Target machines
  connection: local           # How to connect (local, ssh, etc.)
  become: yes                 # Run with sudo
  vars:
    myvar: myvalue
  vars_files:
    - group_vars/all/vault.yml  # Load encrypted variables
  
  roles:
    - role1
    - role2
  
  tasks:
    - name: Do something
      module_name:
        param1: value1
        param2: value2
      when: condition           # Only run if condition is true
      tags: tagname            # Run only with --tags tagname
```

## Running Playbooks

```bash
ansible-playbook playbook.yml
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml -K
ansible-playbook playbook.yml --ask-vault-pass -K
ansible-playbook playbook.yml --check
ansible-playbook playbook.yml --tags "bitwarden"
ansible-playbook playbook.yml -v
```

## localhost (Local Machine)

```yaml
- name: Configure local machine
  hosts: localhost
  connection: local
  become: yes
```

## Useful Modules for Personal Setup

### package - Install system packages
```yaml
- name: Install packages
  package:
    name: "{{ item }}"
    state: present
  become: yes
  loop:
    - git
    - python3
    - docker
```

### copy - Deploy files
```yaml
- name: Copy SSH config
  copy:
    src: files/ssh_config
    dest: ~/.ssh/config
    mode: '0600'
```

### template - Deploy templated files
```yaml
- name: Deploy bashrc with variables
  template:
    src: templates/bashrc.j2
    dest: ~/.bashrc
    mode: '0644'
```

### lineinfile - Add/modify lines
```yaml
- name: Add PATH to bashrc
  lineinfile:
    path: ~/.bashrc
    line: 'export PATH="$PATH:/usr/local/bin"'
    create: yes
```

### file - Create/modify files/directories
```yaml
- name: Create directory
  file:
    path: ~/.config/myapp
    state: directory
    mode: '0755'
```

## Ansible Vault for Secrets

```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
```

## Related

- [[Ansible Workstation Bootstrap Guide]]
- [[Bitwarden Secrets Manager]]
- [[Terraform]]
- [[Dotfiles]]

