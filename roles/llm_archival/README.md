# llm_archival Role

Purpose: Set up scheduled extraction of local LLM/chat application data using Python prototype scripts.

## Responsibilities
- Invoke extractor scripts located under `LLM_Parsing_Project/prototypes/`.
- Configure Windows Task Scheduler or Linux cron (6h cadence by default).
- Provide a central place to later introduce sanitization hooks.

## Variables
| Variable | Location | Description |
|----------|----------|-------------|
| `tool_base_paths` | `group_vars/all/tool_paths.yml` | Root paths for each tool's data store. |
| `archive_interval_hours` (future) | (vars) | Override default 6 hour cadence. |
| `python_exec` (future) | (vars) | Path to Python interpreter if not system default. |

## Planned Additions
- Offset tracking table (when PostgreSQL enabled) to process only new messages.
- Hybrid dedup logic (message primary keys + similarity clustering).
- Sanitization pre-write (security_sanitizer integration).

## Scheduling Details
- Windows: creates a task named `LLMArchival`.
- Linux: cron entry invoking inline shell command.

## Manual Run
```sh
ansible-playbook -i inventory/hosts playbooks/deploy-all-configs.yml --tags llm_archival
```

## Scope
Does not reorganize vault content; strictly automates extraction tasks.

