# Elthree (L3) Plugin Design Spec

**Date:** 2026-04-13
**Status:** Approved

---

## Overview

Elthree (L3) is an installable Claude Code plugin that turns any machine into a git-backed personal task management system using Obsidian as the UI layer. Users install it once from a GitHub repo, run `/l3-setup` to initialize their vault, then use `/l3` for all ongoing task operations — from anywhere, without needing to be in the vault directory.

The vault (a local directory of markdown files in a git repo) is the persistence layer. Claude is the operator.

---

## Target User

Anyone — from Obsidian power users to people who have never used a CLI tool. Setup must be guided and approachable. Operations must work via natural language as well as slash commands.

---

## Installation

Users install by pointing the Claude Code plugin system at the GitHub repo:

```
/plugin install github:y0shi/elthree
```

---

## Repository Structure

The `elthree` repo serves directly as the plugin package:

```
elthree/
├── README.md              # User-facing docs: what it is, how to install, how to use
├── LICENSE
├── skills/
│   └── l3-vault/
│       └── SKILL.md       # All vault knowledge: format, operations, config location
└── commands/
    ├── l3.md              # /l3 — smart orchestrator for all ongoing operations
    └── l3-setup.md        # /l3-setup — one-time vault initialization wizard
```

---

## Config File

Written by `/l3-setup`, read by the skill from any working directory:

**Path:** `~/.l3/config.md`

**Contents:**

```markdown
---
vault_path: ~/vault
git_remote: git@github.com:user/vault-tasks.git
created: YYYY-MM-DD
---
```

This is what makes the vault location-independent — Claude always knows where to look regardless of the current working directory.

---

## Component 1: `l3-vault` Skill (`skills/l3-vault/SKILL.md`)

The brain of the plugin. Activated by its description when the user's intent matches task management or vault operations. Encodes all vault knowledge.

### Config Lookup

On any vault operation, read `~/.l3/config.md` first to get the vault path. If the file doesn't exist, prompt the user to run `/l3-setup` before proceeding.

### Task File Format

**Filename:** `TASK-NNN_slug-title.md` (e.g., `TASK-042_bedrock-compliance.md`)

**YAML Frontmatter:**

```yaml
---
id: "TASK-NNN"          # Zero-padded, monotonically increasing — never reuse
title: "Human title"    # Same as the H1 heading
status: todo            # todo | in-progress | waiting | review | done
priority: medium        # critical | high | medium | low
created: YYYY-MM-DD     # Date task was created — never change after creation
due:                    # YYYY-MM-DD or empty
scheduled:              # YYYY-MM-DD — when to start work
project:                # Wikilink: "[[Projects/slug]]"
area:                   # Responsibility area slug
tags: []                # Freeform list, lowercase-hyphenated
waiting_on: ""          # Person or event
blocked_by: []          # List of TASK-NNN IDs
source: ""              # Where the task originated
---
```

**Body Structure:**

```markdown
# Task Title

## Objective
What this task accomplishes.

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Notes
Context, references, links.

## Log
- YYYY-MM-DD: Entry describing what happened
```

### ID Generation

1. List all files in `Tasks/` and `Archive/` (recursively)
2. Find the highest `TASK-NNN` number across both
3. Increment by 1, zero-pad to at least 3 digits
4. Never reuse an ID, even if the original task was deleted

### Status Lifecycle

```
todo → in-progress → review → done → (archived)
todo → waiting → todo  (when unblocked)
any  → done            (can skip intermediate states)
```

- Set `status: waiting` when `waiting_on` is populated
- Set `status: done` when all acceptance criteria are met
- Only move to `Archive/` during weekly review or explicit request

### Operations

**Create:** Generate next ID → write `Tasks/TASK-NNN_slug.md` with required frontmatter → write objective and at least one acceptance criterion → add initial log entry → populate `source` if known

**Update:** Read existing file → change only specified fields → add log entry for status changes or meaningful progress → never remove existing log entries → preserve body structure

**Complete:** Set `status: done` → check all acceptance criteria checkboxes → add `- YYYY-MM-DD: Completed` log entry → do NOT archive (happens during weekly review)

**Archive:** Move `done` tasks to `Archive/YYYY-QN/` (Q1=Jan–Mar, Q2=Apr–Jun, Q3=Jul–Sep, Q4=Oct–Dec) → preserve file exactly as-is

**Process Inbox:** Read `Inbox.md` → create task files or inline tasks for each actionable item → remove processed items → leave unprocessable items with a note

**Weekly Review:** Create `Reviews/YYYY-WNN.md` with completed tasks summary, open tasks by project, overdue items flagged, waiting items with age, tasks missing due dates → archive all `done` tasks → process `Inbox.md` → flag anything in-progress for 2+ weeks with no log entry

**Query patterns:**

| Natural language | Action |
|-----------------|--------|
| "What's overdue?" | `due < today` and `status != done` |
| "What's high priority?" | `priority: high or critical`, not done |
| "Tasks for project X" | Filter by `project` field |
| "What am I waiting on?" | `status: waiting`, report `waiting_on` values |
| "What did I finish?" | `status: done` in `Tasks/` (not yet archived) |
| "Show TASK-042" | Read `Tasks/TASK-042_*.md` |

### Inline Tasks (Project/Area Notes)

Files in `Projects/` and `Areas/` may use GFM checkboxes with emoji priority:

```markdown
- [ ] Task description 📅 YYYY-MM-DD ⏫
```

Priority: `⏫` = highest, `🔼` = high, `🔽` = low, none = medium

Promote to a full task file when the item needs its own acceptance criteria or log.

### Safety Rules

- Never delete a task file — change status to `done` and archive
- Never overwrite log entries — only append
- Never change `id` or `created` after creation
- Never create tasks outside `Tasks/`, `Projects/`, or `Inbox.md`
- Never modify files in `Archive/` unless explicitly asked
- If unsure about a field value, leave it empty rather than guessing

### Trigger Conditions

This skill activates when:
- The user invokes `/l3` or `/l3-setup`
- Natural language involves task management ("create a task", "what's on my plate", "mark that done", "weekly review", "what's overdue")
- Context references the vault, L3, or elthree

---

## Component 2: `/l3` Command (`commands/l3.md`)

Single entry point for all ongoing vault operations. Accepts free-form input.

### Usage Examples

```
/l3 create a task for reviewing Bedrock compliance, high priority, due Friday
/l3 what's overdue?
/l3 mark TASK-042 as done
/l3 weekly review
/l3 process inbox
/l3 show everything blocked by TASK-038
/l3 bulk update: set all bedrock-tagged tasks to area platform-engineering
/l3
```

### Routing Logic

1. Read `~/.l3/config.md` — if missing, abort with prompt to run `/l3-setup`
2. Classify intent: `create` / `update` / `query` / `review` / `archive` / `inbox` / `bulk`
3. Apply `l3-vault` skill operation rules for that intent
4. Confirm before executing destructive or bulk actions
5. Summarize what was done after every write operation

### No-Argument Behavior

`/l3` with no arguments shows a morning briefing: overdue items, today's scheduled tasks, top 3 open tasks by priority.

### Error Handling

| Condition | Response |
|-----------|----------|
| `~/.l3/config.md` missing | "Vault not configured. Run `/l3-setup` to get started." |
| Vault path doesn't exist | "Vault directory not found at `<path>`. Re-run `/l3-setup`." |
| Task ID not found | Suggest fuzzy matches by title |
| Ambiguous intent | Ask one clarifying question before acting |

---

## Component 3: `/l3-setup` Command (`commands/l3-setup.md`)

One-time vault initialization. Safe to re-run.

### Flow

**Step 1 — Check for existing config**
If `~/.l3/config.md` exists, show current settings and ask: reconfigure, or cancel.

**Step 2 — Ask two key questions**
- Where should the vault live? (default: `~/vault`)
- Git remote URL? (optional — can skip and add later)

**Step 3 — Create vault structure**

```
<vault_path>/
├── Tasks/
├── Projects/
├── Areas/
├── Archive/
├── Templates/
│   ├── task.md
│   └── project.md
├── Reviews/
├── Inbox.md
└── .gitignore
```

Templates are embedded in the command and written during setup. `.gitignore` excludes `.obsidian/workspace.json`, `.obsidian/workspace-mobile.json`, `.obsidian/cache`, `.DS_Store`.

**Step 4 — Initialize git**

```bash
git init
git add -A
git commit -m "Initial vault setup"
# If remote provided:
git remote add origin <url>
git push -u origin main
```

**Step 5 — Write `~/.l3/config.md`**

Stores vault path, git remote (if provided), creation date.

**Step 6 — Print manual steps checklist**

Claude cannot automate Obsidian installation or plugin management. Setup ends with a printed checklist:

```
Next steps (manual):
[ ] Install Obsidian: https://obsidian.md
[ ] Open your vault directory in Obsidian
[ ] Install community plugins: Obsidian Git, Tasks, Templater, Dataview
[ ] Configure Obsidian Git: 5-minute auto-commit interval, pull on startup
[ ] (Claude Desktop) Add filesystem MCP server to claude_desktop_config.json:
    vault path: <vault_path>
[ ] (Mobile) Add GitHub MCP connector in claude.ai Settings → Connectors
```

### Re-run Safety

Never overwrites existing `Tasks/`, `Projects/`, `Areas/`, `Archive/`, or `Reviews/` directories or their contents. Only creates missing directories and overwrites `Templates/` and config.

---

## Data Flow

```
User: /l3 create a task for X
  → l3.md command reads ~/.l3/config.md → gets vault path
  → applies l3-vault skill: generate ID, write Tasks/TASK-NNN_x.md
  → reports: "Created TASK-007: X"

User: /l3 weekly review
  → l3.md routes to review operation
  → l3-vault skill: reads all Tasks/*.md, Archives done tasks,
    processes Inbox.md, writes Reviews/2026-W16.md
  → reports summary

Obsidian Git plugin auto-commits → pushes to GitHub
Claude Mobile reads via GitHub MCP → creates commits directly
Local machines pull on next auto-sync
```

---

## Out of Scope

- Obsidian plugin installation automation (not possible from CLI)
- Claude Desktop MCP auto-configuration (requires manual JSON edit)
- Multi-vault support (single vault per config for now)
- Team/shared vaults (personal use only)
- Web UI or dashboard beyond Obsidian

---

## Open Questions

None — all design decisions resolved during brainstorming.
