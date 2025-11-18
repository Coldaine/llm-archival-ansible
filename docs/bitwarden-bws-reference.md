# Bitwarden Secrets Manager (BWS) Reference

**Bitwarden Secrets Manager** is your central secret vault for API keys, tokens, database credentials, and any other sensitive data. Store secrets once in BWS, reference them everywhere.

## Why Use BWS?

- **Single source of truth** - All secrets in one place
- **Centralized access control** - Manage who can access which secrets
- **API access** - Query secrets programmatically (what Ansible does)
- **Audit trail** - See who accessed what secrets and when
- **Organization** - Group secrets by project
- **Encryption** - Zero-knowledge encryption (Bitwarden can't see your secrets)

## Getting Started

### Create a Bitwarden Organization

1. Go to [bitwarden.com](https://bitwarden.com)
2. Create account or log in
3. Create new Organization
4. Get Organization ID (shown in Organization Settings)

### Create Secrets in BWS

1. In Bitwarden web vault, go to **Secrets Manager**
2. Create new Secret with:
   - **Name**: `github-api-token`
   - **Value**: `ghp_xxxxxxxxxxxx...`
   - **Project**: (optional, for organization)

3. Copy the **Secret ID** (UUID, looks like: `550e8400-e29b-41d4-a716-446655440000`)

### Generate Access Token

1. In **Organization** → **Settings** → **Access Tokens**
2. Click **Create Token**
3. Choose **Machine Account** (for automated access)
4. Name it (e.g., `ansible-bootstrap`)
5. Save the token (long string starting with `bws_`)

**⚠️ Important:** Save this token immediately - you won't see it again!

```
BWS_ACCESS_TOKEN=bws_aCfGq8Yt3jK9mN2pQwErTyUiOpAsD...
```

## Storing in Ansible Vault

```bash
cat > group_vars/all/vault.yml << 'EOF'
---
bws_access_token: "bws_aCfGq8Yt3jK9mN2pQwErTyUiOpAsD..."
bws_secrets:
  github_token_id: "550e8400-e29b-41d4-a716-446655440000"
  openai_key_id: "550e8400-e29b-41d4-a716-446655440001"
  database_url_id: "550e8400-e29b-41d4-a716-446655440002"
EOF
ansible-vault encrypt group_vars/all/vault.yml
```

## Using BWS in Ansible

### 1. Deploy the Token (During Bootstrap)

```yaml
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

- name: Deploy BWS access token
  copy:
    content: "{{ bws_access_token }}"
    dest: "{{ ansible_env.HOME }}/.config/bws/token"
    mode: '0600'
  no_log: yes

- name: Add token to shell environment
  lineinfile:
    path: "{{ ansible_env.HOME }}/.bashrc"
    line: 'export BWS_ACCESS_TOKEN=$(cat ~/.config/bws/token)'
```

### 2. Fetch Secrets During Deployment

```yaml
- name: Fetch GitHub token from BWS
  shell: |
    export BWS_ACCESS_TOKEN=$(cat ~/.config/bws/token)
    bws secret get {{ bws_secrets.github_token_id }} --json | jq -r '.value'
  register: github_token_result
  no_log: yes

- name: Store as variable
  set_fact:
    github_token: "{{ github_token_result.stdout }}"
```

### 3. Lookup Alternative

```bash
ansible-galaxy collection install bitwarden.secrets
```

```yaml
- name: Get secret using lookup
  set_fact:
    github_token: "{{ lookup('bitwarden.secrets.lookup', bws_secrets.github_token_id) }}"
```

## Security Best Practices

- Use a Machine Account token with minimal scope
- Keep `vault.yml` encrypted; never commit plaintext secrets
- Rotate tokens periodically and update vault
- Prefer environment injection over hardcoding

## Common Issues

- Ensure `BWS_ACCESS_TOKEN` is set in the environment
- Verify Machine Account access to the right projects
- Use Secret IDs (UUIDs), not names, for retrieval
