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
| `archival_base_dir` | `group_vars/all/archival.yml` | Base directory for Obsidian Vault per OS family. |
| `archival_scripts_dir` | `group_vars/all/archival.yml` | Scripts installation directory per OS family. |
| `archival_extractor_dir` | `group_vars/all/archival.yml` | Python extractor prototypes directory per OS family. |
| `archival_interval_hours` | `group_vars/all/archival.yml` | Archival cadence in hours (default: 6). |
| `archival_python_executable` | `group_vars/all/archival.yml` | Python interpreter path (default: "python"). |
| `computed_scripts_dir` | `group_vars/all/archival.yml` | Auto-computed scripts path for current OS. |
| `computed_extractor_dir` | `group_vars/all/archival.yml` | Auto-computed extractor path for current OS. |
| `computed_base_dir` | `group_vars/all/archival.yml` | Auto-computed base directory for current OS. |

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
