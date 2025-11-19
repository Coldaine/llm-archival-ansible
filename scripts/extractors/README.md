# LLM Archival Extractor Scripts

This directory contains Python extractor prototypes for parsing LLM conversation data from various tools.

## Overview

These are **stub implementations** demonstrating the expected structure and interface. Replace with your actual extraction logic as needed.

## Available Extractors

### 1. jsonl_parser.py
Parses JSONL (JSON Lines) format conversation logs.

**Usage:**
```bash
python jsonl_parser.py inline
python jsonl_parser.py batch --input /path/to/file.jsonl --output ./parsed
```

**Tools using JSONL:**
- Claude Desktop (some formats)
- Various API-based tools

---

### 2. base64_jsonl_decoder.py
Decodes Base64-encoded content within JSONL files.

**Usage:**
```bash
python base64_jsonl_decoder.py inline
python base64_jsonl_decoder.py batch --input /path/to/encoded.jsonl
```

**Tools with Base64 encoding:**
- Some browser-based LLM extensions
- Tools storing binary/rich content

---

### 3. leveldb_extractor.py
Extracts conversation data from LevelDB databases.

**Requirements:**
```bash
pip install plyvel
```

**Usage:**
```bash
python leveldb_extractor.py inline
python leveldb_extractor.py batch --input /path/to/leveldb/
```

**Tools using LevelDB:**
- Cursor (conversation history)
- Goose (local storage)
- Various Electron-based LLM apps

---

### 4. main_orchestrator.py
Main entry point that coordinates all extractors.

**Usage:**
```bash
# Full extraction
python main_orchestrator.py --output /path/to/output

# Sample mode (first 10 conversations per tool)
python main_orchestrator.py --sample --summary

# With custom config
python main_orchestrator.py --config /path/to/tool_paths.json --summary
```

**What it does:**
1. Discovers installed LLM tools
2. Determines appropriate extractor for each tool
3. Runs extractors in sequence
4. Aggregates results
5. Generates summary reports

---

## Integration with Ansible

The `llm_archival` Ansible role invokes these extractors automatically.

**Configuration:**
- Tool paths: `group_vars/all/tool_paths.yml`
- Extractor location: `group_vars/all/archival.yml` → `archival_extractor_dir`
- Schedule: Set via `archival_interval_hours` (default: 6 hours)

**Ansible invocation:**
```yaml
# roles/llm_archival/tasks/extractors.yml
- name: Run JSONL extractor
  shell: python jsonl_parser.py inline
  args:
    chdir: "{{ computed_extractor_dir }}"
```

---

## Implementing Real Extractors

To replace stubs with actual implementations:

1. **Understand the tool's data format:**
   - Locate where the tool stores conversations
   - Examine file formats (JSONL, SQLite, LevelDB, etc.)
   - Identify conversation/message structure

2. **Parse and normalize:**
   - Extract: provider, conversation_id, message_id, role, content, timestamp
   - Compute content hash (SHA-256 of normalized text)
   - Handle encoding issues (UTF-8, Base64, etc.)

3. **Output format:**
   - JSON files (one per conversation)
   - CSV (flat message table)
   - Direct PostgreSQL INSERT (future)

4. **Error handling:**
   - Log parse errors but continue processing
   - Track offsets for incremental extraction
   - Validate extracted data

5. **Testing:**
   ```bash
   # Test on sample data
   python jsonl_parser.py batch --input ./test_data/ --output ./test_output/

   # Verify output structure
   head test_output/conversation_*.json
   ```

---

## Common Data Paths

**Windows:**
```
%APPDATA%\Claude\projects\
%APPDATA%\Cursor\User\History\
%APPDATA%\Code\User\globalStorage\github.copilot-chat\
%APPDATA%\Roaming\Goose\Local Storage\leveldb\
```

**Linux:**
```
~/.config/claude/
~/.config/cursor/
~/.config/goose/
~/.cache/aider/
```

**macOS:**
```
~/Library/Application Support/Claude/
~/Library/Application Support/Cursor/
~/Library/Application Support/Goose/
```

See `group_vars/all/tool_paths.yml` for full list.

---

## Dependencies

**Python version:**
- Python 3.8 or higher

**Common packages:**
```bash
pip install plyvel        # For LevelDB extraction
pip install xxhash        # For fast content hashing
pip install python-dateutil  # For timestamp parsing
```

---

## Deduplication Strategy

When implementing extractors, consider the planned deduplication approach:

1. **Primary key uniqueness:**
   - Composite key: `(provider, conversation_id, message_id)`
   - Prevents exact duplicates at insert time

2. **Content hash indexing:**
   - SHA-256 hash of normalized message text
   - Fast lookup for duplicate content
   - Handles copy-pasted conversations

3. **Similarity clustering (future):**
   - Periodic job using cosine similarity
   - Groups near-duplicates (threshold ~0.95)
   - Consolidates related messages

4. **Offset tracking:**
   - Track last processed offset per source file
   - Only process new data on subsequent runs
   - Stored in PostgreSQL `ingestion_offsets` table (future)

See `README.md` and `DOCUMENTATION_REVIEW.md` for more details.

---

## Troubleshooting

**Import errors:**
```bash
# Ensure Python can find modules
export PYTHONPATH=/path/to/extractors:$PYTHONPATH
```

**LevelDB errors:**
```bash
# Install system dependencies (Ubuntu/Debian)
sudo apt-get install libleveldb-dev

# Reinstall plyvel
pip uninstall plyvel
pip install --no-binary :all: plyvel
```

**Encoding errors:**
```python
# Always specify UTF-8 encoding
with open(file_path, 'r', encoding='utf-8', errors='replace') as f:
    content = f.read()
```

---

## Future Enhancements

- [ ] Direct PostgreSQL ingestion (bypass file output)
- [ ] Incremental extraction with offset tracking
- [ ] Parallel processing for large datasets
- [ ] Sanitization pre-processing (PII/secrets removal)
- [ ] Support for additional tools (Kilo Code, Jan, etc.)
- [ ] Automatic tool discovery and version detection

---

**Last Updated:** 2025-11-19
**Status:** Stub implementations - ready for custom logic
