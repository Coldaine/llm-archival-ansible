# postgresql_server Role

Purpose: Optionally deploy a containerized PostgreSQL instance to centralize structured archival data.

## Status
Optional. Not auto-included in `deploy-all-configs.yml` yet. Add deliberately once storage and backup constraints are defined.

## Responsibilities
- Pull and run `postgres:15-alpine` container.
- Expose port (default 5432) locally or via Tailscale overlay if enabled.
- Initialize schema (placeholder `schema.sql`). Future migrations managed separately.

## Variables (Planned/Gating)
| Variable | Description | Default |
|----------|-------------|---------|
| `enable_postgres` | Controls inclusion of role in playbook | false (future) |
| `postgres_container_name` | Docker container name | `archival-postgres` |
| `postgres_port` | Host port to map | 5432 |
| `postgres_password` | Superuser password (from Vault) | (required) |
| `postgres_data_dir` (future) | Persistent volume path | `/var/lib/archival_pg` |

## Schema Direction (Planned)
Tables:
- `conversations(provider, conversation_id, title, created_at, updated_at)`
- `messages(provider, conversation_id, message_id, role, content_hash, content_text, created_at)`
- `similarity_clusters(cluster_id, provider, conversation_id, message_id_primary, message_id_duplicate)`
- `ingestion_offsets(provider, source_path, last_offset, updated_at)`

Primary Keys & Constraints:
- Composite PK: `(provider, conversation_id, message_id)`.
- Unique content hash index for fast duplicate checks.

## Security Considerations
- Password injected from encrypted `vault.yml`.
- Network exposure limited; pair with Tailscale for secure access.

## Manual Inclusion
Add role invocation manually to playbook until gated variable is merged:
```yaml
- hosts: all
  roles:
    - role: postgresql_server
      tags: [postgres]
```
Run with:
```sh
ansible-playbook -i inventory/hosts playbooks/deploy-all-configs.yml --tags postgres --ask-vault-pass
```

## Backups (Future)
- Logical dumps (`pg_dump`) daily.
- WAL archiving to off-host storage.

## Scope
Does not automatically ingest data yet; serves as target for future direct Python upserts.
