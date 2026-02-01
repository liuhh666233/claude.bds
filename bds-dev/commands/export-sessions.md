---
name: export-sessions
description: Export Claude Code conversation sessions to Markdown format with auto-generated and custom tags
---

# Export Sessions Command

Export Claude Code conversation history to well-organized Markdown files with metadata tags.

## Prerequisites

This command requires Python 3.10+. On NixOS, use:
```bash
nix-shell -p python312 --run "python3 script.py"
```

## Data Sources

Claude Code stores conversation data in two locations:

1. **Global history index**: `~/.claude/history.jsonl`
   - Each line: `{"display": "...", "timestamp": 1234567890, "project": "/path/to/project", "sessionId": "uuid"}`
   - Used for listing sessions and getting metadata

2. **Project-level sessions**: `~/.claude/projects/{encoded-path}/{sessionId}.jsonl`
   - Directory name encoding rules:
     - `/` becomes `-` (e.g., `/home/user` -> `-home-user`)
     - `.` becomes `-` (e.g., `claude.bds` -> `claude-bds`)
   - Contains full conversation records with message types: `user`, `assistant`, `summary`, `file-history-snapshot`

## Workflow

### Step 1: Scan and Display ALL Projects

First, list ALL projects with their session counts so the user can see the full picture:

```bash
for dir in ~/.claude/projects/*/; do
  name=$(basename "$dir")
  count=$(ls "$dir"*.jsonl 2>/dev/null | grep -v "agent-" | wc -l)
  if [ "$count" -gt 0 ]; then
    echo "$count|$name"
  fi
done 2>/dev/null | sort -t'|' -k1 -rn
```

**IMPORTANT**: Output the FULL list as text BEFORE asking the user to select, like this:

```
Found 15 projects with Claude Code sessions:

 #  | Sessions | Project
----|----------|----------------------------------
 1  |    56    | intraday-alpha-research
 2  |    13    | reporting
 3  |    13    | all-weather-portfolio
 4  |    12    | research-incubator
 5  |    10    | fastpy
 6  |     9    | research-toolkit
 7  |     6    | data-pipeline-monitor
 8  |     5    | wonder-pkgs
 9  |     5    | excalibur-cli
10  |     5    | claude.bds
11  |     4    | simulated-trading
...
```

Then use AskUserQuestion with common options. The user can select "Other" to input specific project numbers:

```
Options:
1. All projects (export everything)
2. Top 5 by session count
3. Last 7 days only
4. [Other - let user input project numbers like "1,3,5" or "1-5"]
```

### Step 2: Select Sessions by Time Range

After selecting projects, ask how to filter sessions:

```
Options:
1. All sessions (may be many)
2. Last 7 days
3. Last 30 days
4. Last 5 sessions per project
```

For the selected time range, query `~/.claude/history.jsonl`:

```python
import json
from datetime import datetime

# Filter by timestamp (milliseconds)
seven_days_ago = (datetime.now().timestamp() - 7*24*60*60) * 1000

sessions = []
with open('~/.claude/history.jsonl') as f:
    for line in f:
        data = json.loads(line)
        if data['timestamp'] >= seven_days_ago:
            if data['project'] in selected_projects:
                sessions.append(data)
```

Display the filtered sessions:

```
Found 12 sessions in selected time range:

Project: reporting (3 sessions)
  1. [2026-01-31 20:11] "Please refactor the reporting code..."
  2. [2026-01-30 15:42] "Change the table background color..."
  3. [2026-01-29 10:15] "Read the pnl.py file and explain..."

Project: claude.bds (2 sessions)
  4. [2026-02-01 08:27] "Export sessions plugin design..."
  5. [2026-02-01 07:50] "Create new skill plugin..."
```

### Step 3: Configure Export Options

Ask for export configuration with sensible defaults:

```
Options for Export Path:
1. /var/lib/wonder/warehouse/database/lxb/claude-sessions/ (last used)
2. ./claude-exports/ (current directory)
3. ~/Documents/claude-exports/
4. [Other - custom path]

Options for Output Detail:
1. Truncate large outputs (recommended) - keeps files smaller
2. Full output - includes complete tool results

Custom Tags (optional):
- Let user input comma-separated tags like "work, finance, q4-project"
- These will be prefixed with "custom/" in the output
```

### Step 4: Process Each Session

#### 4.1 Find the Session File

The project path to directory mapping is tricky. Use this approach:

```python
def find_project_dir(project_path):
    """Find the encoded directory for a project path"""
    projects_dir = Path("~/.claude/projects").expanduser()

    # Try encoding: replace / with - and . with -
    encoded = project_path.replace('/', '-').replace('.', '-')
    if (projects_dir / encoded).exists():
        return projects_dir / encoded

    # Fallback: search by project name
    project_name = Path(project_path).name.replace('.', '-')
    for d in projects_dir.iterdir():
        if d.is_dir() and project_name in d.name:
            return d

    return None
```

#### 4.2 Filter Valid Sessions

**IMPORTANT**: Skip sessions that only contain metadata (no actual conversation):

```python
def has_conversation(session_file):
    """Check if session has actual user/assistant messages"""
    with open(session_file) as f:
        for line in f:
            data = json.loads(line)
            if data.get('type') in ('user', 'assistant'):
                msg = data.get('message', {})
                content = msg.get('content')
                # Check for actual content, not just tool results
                if isinstance(content, str) and content.strip():
                    return True
                if isinstance(content, list):
                    for block in content:
                        if block.get('type') == 'text':
                            return True
    return False
```

Sessions with only `summary` and `file-history-snapshot` entries should be skipped with a note.

#### 4.3 Auto-detect Tags

Analyze the conversation content to generate tags:

**Language Detection** - scan file paths and commands:
| Pattern | Tag |
|---------|-----|
| `.py`, `python`, `pip` | `auto/python` |
| `.ts`, `.tsx` | `auto/typescript` |
| `.js`, `.jsx` | `auto/javascript` |
| `.nix`, `nix build`, `nix develop` | `auto/nix` |
| `.rs`, `cargo` | `auto/rust` |
| `.go` | `auto/go` |
| `.md` | `auto/markdown` |

**Task Type Detection** - scan user messages for keywords:
| Keywords | Tag |
|----------|-----|
| fix, bug, error, issue, broken | `auto/bug-fix` |
| add, implement, create, new, build | `auto/new-feature` |
| refactor, clean, reorganize, restructure | `auto/refactoring` |
| test, unittest, pytest, spec | `auto/testing` |
| doc, readme, comment, explain | `auto/documentation` |
| review, analyze, understand | `auto/code-review` |

**Tech Stack Detection** - scan for framework/library patterns:
| Pattern | Tag |
|---------|-----|
| streamlit | `auto/streamlit` |
| react, nextjs | `auto/react` |
| pytorch, torch | `auto/pytorch` |
| pandas, numpy | `auto/data-science` |
| fastapi, flask, django | `auto/web-backend` |
| docker, kubernetes | `auto/devops` |

#### 4.4 Generate Markdown Content

Create the markdown file with this structure:

```markdown
---
title: "[Session title from summary or first user message truncated to 50 chars]"
date: YYYY-MM-DD HH:MM
end_date: YYYY-MM-DD HH:MM
project: /path/to/project
session_id: uuid-here
git_branch: main
tags:
  - auto/python
  - auto/refactoring
  - custom/user-provided-tag
---

# Session: [Title]

**Project**: `/path/to/project`
**Date**: YYYY-MM-DD HH:MM - HH:MM
**Git Branch**: main

---

## Conversation

### User (HH:MM)

[User message content - preserve markdown formatting]

---

### Assistant (HH:MM)

[Assistant text response]

#### Tool: [ToolName]

**[Tool-specific details, e.g., File: /path/to/file, or Command: git status]**

<details>
<summary>Output (click to expand)</summary>

```
[Tool output, truncated if too long]
```

</details>

---

[Continue for all messages...]
```

### Step 5: Handle Message Content

#### User Messages

If `message.content` is a string, output it directly.

If `message.content` is an array with `tool_result` items, format as:

```markdown
### Tool Result (HH:MM)

<details>
<summary>Output</summary>

```
[tool result content]
```

</details>
```

#### Assistant Messages

Process `message.content` array:

- **`text` blocks**: Output the text directly
- **`tool_use` blocks**: Format based on tool type:

  | Tool | Format |
  |------|--------|
  | Bash | `**Command**: \`command\`` |
  | Read | `**File**: \`file_path\`` |
  | Edit | `**File**: \`file_path\`` |
  | Write | `**File**: \`file_path\`` |
  | Grep | `**Pattern**: \`pattern\``, `**Path**: \`path\`` |
  | Glob | `**Pattern**: \`pattern\`` |
  | Task | `**Description**: description` |

- **`thinking` blocks**: If user chose to include them, wrap in `<details>` tag

### Step 6: Truncate Large Outputs

For tool outputs longer than 100 lines:

```
[first 80 lines]

... [N lines omitted] ...

[last 20 lines]
```

### Step 7: Write Files and Generate Index

1. Create the export directory if it doesn't exist
2. For each session, write: `{YYYY-MM-DD}_{project-name}_{session-id-first-6-chars}.md`
3. Create/update `index.md` with session table and tag summary

### Step 8: Report Results

Provide a clear summary:

```
Export Complete!

Location: /path/to/exports/
Sessions exported: 5
Sessions skipped: 2 (no conversation content)

Files created:
  - 2026-01-31_reporting_c74f37.md (python, refactoring) - 45KB
  - 2026-01-30_reporting_f7ef9f.md (python, bug-fix) - 12KB
  - index.md (updated)

Note: Please review exported files for any sensitive information before sharing.
```

## Error Handling

- **Corrupted JSONL lines**: Skip and log a warning, continue with next line
- **Missing session files**: Report which sessions couldn't be found
- **Empty sessions**: Skip sessions with only metadata (summary/file-history-snapshot)
- **Permission errors**: Report and suggest checking file permissions

## Implementation Notes

### Python 3 Requirement

The export script uses f-strings and pathlib, requiring Python 3.6+. On NixOS:

```bash
nix-shell -p python312 --run "python3 /tmp/export_sessions.py '$sessions_json'"
```

### Path Encoding Gotchas

- `/home/lxb/github/claude.bds` encodes to `-home-lxb-github-claude-bds` (not `-home-lxb-github-claude.bds`)
- Always verify the encoded path exists before reading

### Session Validation

Before exporting, verify the session has actual content:
- Sessions from `/resume` commands often only contain `file-history-snapshot` entries
- Sessions should have at least one `user` or `assistant` message with text content

### Performance Tips

- For large exports (>50 sessions), consider running in background
- Tool outputs can be very large; truncation is recommended
- The index file should be regenerated to include new exports
