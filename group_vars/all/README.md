Place shared variables here. For secrets, use Ansible Vault to create `vault.yml` and encrypt it:

```
ansible-vault create group_vars/all/vault.yml
```

Example (vault.yml contents before encryption):

```
---
bws_access_token: "bws_xxx"
```

