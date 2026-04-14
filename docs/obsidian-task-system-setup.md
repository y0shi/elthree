# Obsidian + Claude Task Management System

## Git-Backed, Cross-Platform Personal Task Management

A markdown-based personal task management system using an Obsidian vault as the UI layer, git as the sync/versioning backend, and Claude (Desktop, Code, Mobile) as an active task manager. Every task is a plain `.md` file with YAML frontmatter — human-readable, LLM-parseable, and fully git-trackable.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   GitHub/GHE Private Repo                       │
│                     (source of truth)                           │
└──────────┬──────────────────┬──────────────────┬────────────────┘
           │ git pull/push    │ git pull/push    │ GitHub MCP
           │                  │                  │ (remote)
    ┌──────▼──────┐    ┌─────▼──────┐    ┌──────▼───────────────┐
    │   Claude    │    │  Claude    │    │   Claude Mobile      │
    │   Desktop   │    │   Code    │    │                      │
    │             │    │           │    │  GitHub remote MCP   │
    │ MCP:        │    │ Direct FS │    │  reads/writes files  │
    │ filesystem  │    │ + git CLI │    │  via commits         │
    │ server      │    │           │    │                      │
    │             │    │ CLAUDE.md │    │  OR: Cowork Dispatch │
    │ mcp-obsidian│    │ for ctx   │    │  to Desktop          │
    └──────┬──────┘    └─────┬─────┘    └──────────────────────┘
           │                 │
    ┌──────▼─────────────────▼────────────────────────────────┐
    │              Local Filesystem: ~/vault/                  │
    │                                                         │
    │  Obsidian (UI) ←→ Obsidian Git plugin (auto pull/push) │
    │                    OR fswatch + launchd (auto-commit)   │
    └─────────────────────────────────────────────────────────┘
```

**Data flow**: All changes (whether from Obsidian, Claude Desktop, or Claude Code) hit the local filesystem first, then auto-commit and push to the git remote. Claude Mobile reads/writes via GitHub's remote MCP server, creating commits directly in the repo. Local machines pull those changes automatically.

---

## 1. Vault Structure

```
~/vault/                              # Git repo root / Obsidian vault
├── .obsidian/                        # Obsidian config (gitignored selectively)
├── CLAUDE.md                         # Claude Code instructions (persistent)
├── Inbox.md                          # Quick-capture scratchpad
├── Tasks/                            # One .md file per task (TaskNotes pattern)
│   ├── TASK-001_example-task.md
│   ├── TASK-002_another-task.md
│   └── ...
├── Projects/                         # Project notes with inline tasks
│   ├── addp-infrastructure.md
│   ├── bedrock-compliance.md
│   └── ...
├── Areas/                            # Ongoing responsibility areas
│   ├── modelops.md
│   ├── architecture.md
│   └── ...
├── Archive/                          # Completed tasks moved here
│   └── 2026-Q2/
│       ├── TASK-001_example-task.md
│       └── ...
├── Templates/                        # Task and project templates
│   ├── task.md
│   └── project.md
├── Reviews/                          # Weekly review snapshots
│   └── 2026-W16.md
└── .scripts/                         # Auto-commit and maintenance scripts
    ├── auto-commit.sh
    └── maintenance.sh
```

---

## 2. Task File Format

Every task in `Tasks/` uses YAML frontmatter + a markdown body. This format is parseable by Claude, queryable by Obsidian Tasks plugin, and produces clean git diffs.

### Single Task File: `Tasks/TASK-042_dataiku-adr.md`

```markdown
---
id: "TASK-042"
title: "Finalize Dataiku consolidation ADR"
status: todo          # todo | in-progress | waiting | review | done
priority: high        # critical | high | medium | low
created: 2026-04-13
due: 2026-04-18
scheduled: 2026-04-14
project: "[[Projects/addp-infrastructure]]"
area: platform-engineering
tags:
  - dataiku
  - adr
  - infrastructure
waiting_on: ""
blocked_by: []
source: "RTC meeting 2026-04-11"
---

# Finalize Dataiku consolidation ADR

## Objective
Complete the Architecture Decision Record for reducing 16 EKS clusters to 2.

## Acceptance Criteria
- [ ] Gather current cluster inventory from all accounts
- [ ] Draft cost comparison (current vs proposed)
- [ ] Review with Alex King
- [ ] Present to Nick Maly at RTC

## Notes
- Related Rally feature: F271912
- Current state: 16 clusters across dev/staging/prod per business unit
- Target state: 2 shared clusters with namespace isolation

## Log
- 2026-04-13: Task created from RTC action item
```

### Frontmatter Field Reference

| Field        | Type     | Values / Format                                      | Required |
|-------------|----------|------------------------------------------------------|----------|
| `id`        | string   | `TASK-NNN` (zero-padded 3+ digits)                   | yes      |
| `title`     | string   | Human-readable task name                              | yes      |
| `status`    | string   | `todo`, `in-progress`, `waiting`, `review`, `done`    | yes      |
| `priority`  | string   | `critical`, `high`, `medium`, `low`                   | yes      |
| `created`   | date     | `YYYY-MM-DD`                                          | yes      |
| `due`       | date     | `YYYY-MM-DD`                                          | no       |
| `scheduled` | date     | `YYYY-MM-DD` (when to start working on it)            | no       |
| `project`   | wikilink | `"[[Projects/project-name]]"`                         | no       |
| `area`      | string   | Responsibility area slug                              | no       |
| `tags`      | list     | Freeform tag list                                     | no       |
| `waiting_on`| string   | Person or event being waited on                       | no       |
| `blocked_by`| list     | List of `TASK-NNN` IDs                                | no       |
| `source`    | string   | Where the task originated                             | no       |

### Project Notes with Inline Tasks: `Projects/addp-infrastructure.md`

For quick, lightweight tasks that don't need their own file, use Obsidian Tasks inline syntax inside project notes:

```markdown
---
project: ADDP Infrastructure
area: platform-engineering
status: active
tags:
  - dataiku
  - eks
  - bedrock
---

# ADDP Infrastructure

## Active Tasks
- [ ] Check EKS version EOL dates across all accounts 📅 2026-04-15 ⏫
- [ ] Verify Bedrock model deprecation timeline 📅 2026-04-20 🔼
- [x] Submit cluster inventory spreadsheet ✅ 2026-04-10

## Reference
- Rally features: F271912, F148743
- Slack channel: #cads-devops
```

### Obsidian Tasks Queries

Create dashboard notes that pull tasks from across the vault:

````markdown
## Due This Week
```tasks
filter by function task.file.property('status') === 'todo' || task.file.property('status') === 'in-progress'
filter by function task.file.property('due') !== null
due before next week
not done
sort by due
sort by priority
```

## High Priority (from TaskNotes files)
```tasks
path includes Tasks/
filter by function task.file.property('priority') === 'high' || task.file.property('priority') === 'critical'
filter by function task.file.property('status') !== 'done'
not done
sort by due
```

## By Area
```tasks
path includes Tasks/
not done
group by function task.file.property('area')
sort by due
```
````

---

## 3. Templates

### `Templates/task.md`

```markdown
---
id: "TASK-{{id}}"
title: "{{title}}"
status: todo
priority: medium
created: {{date:YYYY-MM-DD}}
due: 
scheduled: 
project: 
area: 
tags: []
waiting_on: ""
blocked_by: []
source: ""
---

# {{title}}

## Objective


## Acceptance Criteria
- [ ] 

## Notes


## Log
- {{date:YYYY-MM-DD}}: Task created
```

### `Templates/project.md`

```markdown
---
project: "{{title}}"
area: 
status: active
tags: []
---

# {{title}}

## Overview


## Active Tasks
- [ ] 

## Reference

## Decisions

```

---

## 4. Git Setup

### Initialize the Vault Repo

```bash
cd ~/vault

git init
git remote add origin git@github.com:YOUR_USER/vault-tasks.git  # or GHE

# Create .gitignore
cat > .gitignore << 'EOF'
# Obsidian workspace (changes constantly, not useful to track)
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# Obsidian cache
.obsidian/cache

# OS files
.DS_Store
Thumbs.db

# Keep all other .obsidian config (themes, plugins, hotkeys)
EOF

git add -A
git commit -m "Initial vault setup"
git push -u origin main
```

### Option A: Obsidian Git Plugin (Recommended)

Install the **Obsidian Git** community plugin. Recommended settings:

| Setting                        | Value          |
|-------------------------------|----------------|
| Auto pull interval            | 5 minutes      |
| Auto push interval            | 5 minutes      |
| Auto commit interval          | 5 minutes      |
| Commit message                | `vault: {{numFiles}} files changed ({{date}})` |
| Pull on startup               | enabled        |
| Push on backup                | enabled        |
| Disable notifications         | enabled        |

This handles auto-commit, push, and pull entirely within Obsidian. No external scripts needed.

### Option B: fswatch + launchd (For Non-Obsidian Edits)

If Claude Desktop or Claude Code edits files while Obsidian is closed, use fswatch to catch those changes.

**`~/.scripts/vault-auto-commit.sh`**:

```bash
#!/bin/bash
VAULT_DIR="$HOME/vault"
cd "$VAULT_DIR" || exit 1

# Pull first to avoid conflicts
git pull --rebase --autostash origin main 2>/dev/null

/opt/homebrew/bin/fswatch -0 --recursive \
    --exclude '\.git/' \
    --exclude '\.obsidian/workspace' \
    --exclude '\.DS_Store' \
    "$VAULT_DIR" | while read -d "" event; do
    sleep 3  # debounce
    cd "$VAULT_DIR"
    git add -A
    if ! git diff-index --quiet HEAD 2>/dev/null; then
        CHANGED=$(git diff --cached --name-only | head -5 | tr '\n' ', ')
        git commit -m "auto: ${CHANGED%,}"
        git push origin main 2>/dev/null
    fi
done
```

```bash
chmod +x ~/.scripts/vault-auto-commit.sh
brew install fswatch  # if not already installed
```

**`~/Library/LaunchAgents/com.vault.autocommit.plist`**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.vault.autocommit</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>~/.scripts/vault-auto-commit.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/vault-autocommit.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/vault-autocommit-err.log</string>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.vault.autocommit.plist
```

### Git History as Task Changelog

```bash
# What changed this week
git log --oneline --since="1 week ago" -- "Tasks/*.md"

# Full diff of a specific task
git log -p -- "Tasks/TASK-042_dataiku-adr.md"

# Who/what changed status fields
git log -p --all -S 'status:' -- "Tasks/*.md"
```

---

## 5. Claude Desktop Configuration

### MCP Filesystem Server

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "vault": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/josh.reddick/vault"
      ]
    }
  }
}
```

This gives Claude Desktop `read_file`, `write_file`, `edit_file`, `create_directory`, `list_directory`, `move_file`, `search_files`, and `get_file_info` tools — all sandboxed to the vault directory.

### Alternative: mcp-obsidian (Richer Integration)

If you want Claude to use Obsidian's API (search by tags, access metadata cache):

```json
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": ["-y", "mcp-obsidian"],
      "env": {
        "OBSIDIAN_VAULT_PATH": "/Users/josh.reddick/vault"
      }
    }
  }
}
```

### Usage in Claude Desktop

Once configured, you can say things like:

- "Create a new task: review Bedrock inference profile compliance, high priority, due Friday"
- "What are my open high-priority tasks?"
- "Mark TASK-042 as done and add a log entry"
- "Move all done tasks to the archive"
- "Show me tasks blocked by TASK-038"
- "Create a task from today's RTC action items: [paste notes]"

Claude will read/write the actual markdown files via MCP.

---

## 6. Claude Code Configuration

### Option A: Launch From Vault Directory (Simplest)

```bash
cd ~/vault
claude
```

Claude Code has direct filesystem access to the working directory. The `CLAUDE.md` at the vault root is automatically loaded and persists across sessions and context compaction.

### Option B: MCP Server for Cross-Project Access

If you want vault access from within other project directories:

```bash
# Add vault filesystem MCP at user scope (available everywhere)
claude mcp add vault-tasks --scope user -- \
  npx -y @modelcontextprotocol/server-filesystem ~/vault

# Verify
claude mcp list
```

### CLAUDE.md

The `CLAUDE.md` file (provided separately) tells Claude Code exactly how to interact with the vault: file format conventions, ID generation, status lifecycle, archival rules, and common operations.

### Usage in Claude Code

```bash
cd ~/vault
claude

# Then interact naturally:
> "Create a task for the GenAI data engineering evaluation kickoff with MOPPETS team, due next Wednesday"
> "What's overdue?"
> "Bulk update: set all tasks tagged 'bedrock' to area 'platform-engineering'"
> "Run my weekly review — summarize open tasks by project, flag anything overdue"
```

---

## 7. Claude Mobile Configuration

Mobile lacks local filesystem access. Two strategies for git-backed access:

### Strategy A: GitHub Remote MCP Connector (Read/Write via Commits)

1. Go to **claude.ai → Settings → Connectors → Add Custom Connector**
2. Add GitHub MCP: `https://api.githubcopilot.com/mcp/`
3. Authenticate with a GitHub PAT that has repo access
4. This syncs to the Claude mobile app automatically

Claude on mobile can then:
- Read task files with `get_file_contents`
- Update tasks with `create_or_update_file` (each update = a git commit)
- List files with `get_repository_content`

On your desktop, Obsidian Git plugin (or fswatch) picks up those changes on the next auto-pull.

**Usage on mobile**:
- "Show me my open tasks from the vault repo"
- "Mark TASK-042 as done in the repo"
- "Add a new task to Tasks/ for reviewing the FleetMate security findings"

### Strategy B: Cowork Dispatch (Delegates to Desktop)

If Claude Desktop is running on your Mac, you can use **Cowork Dispatch** from mobile to send tasks to Desktop, which has full MCP filesystem access. You get push notifications when work is complete. Best for complex multi-step operations.

### Strategy C: Claude Memory as Lightweight Bridge

Regardless of which strategy you use, tell Claude to remember your current priorities:

> "Remember: my current sprint focus is Dataiku consolidation and Bedrock compliance. Top tasks: TASK-042, TASK-045, TASK-047."

This gives mobile Claude enough context for planning conversations even without direct file access.

---

## 8. Obsidian Plugin Stack

Install these community plugins for the best experience:

| Plugin             | Purpose                                                      |
|-------------------|--------------------------------------------------------------|
| **Obsidian Git**  | Auto-commit, push, pull on interval                          |
| **Tasks**         | Query tasks across vault, checkbox management, due dates     |
| **Dataview**      | Advanced queries on frontmatter (alternative to Tasks)       |
| **Templater**     | Template expansion with dynamic dates and auto-ID generation |
| **Calendar**      | Visual calendar view of tasks by due date                    |
| **Kanban**        | Board view grouped by status                                 |
| **Quick Add**     | Fast task capture with configurable templates                |

### Templater Auto-ID Script

To auto-generate the next `TASK-NNN` ID when creating a new task via Templater:

```javascript
// Templater user script: ~/vault/.obsidian/scripts/next-task-id.js
function nextTaskId() {
    const fs = require('fs');
    const path = require('path');
    const tasksDir = path.join(app.vault.adapter.basePath, 'Tasks');
    
    let maxId = 0;
    if (fs.existsSync(tasksDir)) {
        const files = fs.readdirSync(tasksDir);
        for (const file of files) {
            const match = file.match(/^TASK-(\d+)/);
            if (match) {
                const num = parseInt(match[1], 10);
                if (num > maxId) maxId = num;
            }
        }
    }
    
    return String(maxId + 1).padStart(3, '0');
}
module.exports = nextTaskId;
```

---

## 9. Daily Workflow

### Morning (Any Platform)
1. Claude pulls current task state
2. Review overdue items, re-schedule or escalate
3. Identify top 3 tasks for the day

### During Work (Desktop / Code)
- Claude creates tasks from meeting notes, Slack threads, email action items
- Claude updates status as work progresses
- Inline tasks in project notes for quick items
- TaskNotes files for anything needing tracking/accountability

### End of Day (Any Platform)
- Claude marks completed items as `done`
- Claude logs progress in task `## Log` sections
- Git auto-commits capture the full history

### Weekly Review (Friday)
- Claude generates `Reviews/2026-W16.md` summarizing the week
- Archive completed tasks: move `done` tasks to `Archive/2026-Q2/`
- Review `waiting` tasks — follow up or convert back to `todo`
- Scan `Inbox.md` — process into proper tasks or delete

---

## 10. Maintenance

### Archive Script

Run periodically (or have Claude do it during weekly review):

```bash
#!/bin/bash
cd ~/vault
QUARTER=$(date +%Y-Q$(( ($(date +%-m) - 1) / 3 + 1 )))
mkdir -p "Archive/$QUARTER"

# Find tasks with status: done in frontmatter
grep -rl '^status: done' Tasks/ | while read f; do
    mv "$f" "Archive/$QUARTER/"
done

git add -A
git commit -m "archive: moved completed tasks to $QUARTER"
git push origin main
```

### Conflict Resolution

Git conflicts are rare with one-file-per-task (different files change independently). If they occur (e.g., simultaneous mobile + desktop edits to the same task):

1. Obsidian Git plugin shows conflict markers
2. Resolve in Obsidian or any text editor
3. The YAML frontmatter format makes conflicts easy to read — it's always a field-level disagreement

---

## Quick Start Checklist

- [ ] Create vault directory: `mkdir ~/vault && cd ~/vault && git init`
- [ ] Create directory structure: `mkdir -p Tasks Projects Areas Archive Templates Reviews .scripts`
- [ ] Copy `CLAUDE.md` to vault root
- [ ] Copy task and project templates to `Templates/`
- [ ] Initialize git remote and push
- [ ] Install Obsidian and open vault
- [ ] Install plugins: Obsidian Git, Tasks, Templater, Dataview
- [ ] Configure Obsidian Git (5-min auto intervals)
- [ ] Configure Claude Desktop MCP (`claude_desktop_config.json`)
- [ ] Configure Claude Code MCP (`claude mcp add vault-tasks --scope user ...`)
- [ ] Set up GitHub MCP connector for mobile (Settings → Connectors)
- [ ] Create your first task and verify the round-trip: create in Claude → see in Obsidian → auto-commit → visible on GitHub → readable from mobile
