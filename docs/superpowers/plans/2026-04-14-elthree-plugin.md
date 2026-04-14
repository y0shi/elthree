# Elthree (L3) Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that provides git-backed personal task management via Obsidian as the UI layer, installable with `/plugin install github:y0shi/elthree`.

**Architecture:** Three markdown files form the plugin — a `SKILL.md` encoding all vault knowledge (the brain), plus two command files (`l3.md` and `l3-setup.md`) that become `/l3` and `/l3-setup` slash commands. A `~/.l3/config.md` written at setup time makes the vault location-independent. No executable code — everything is prompt content loaded by Claude Code at invocation time.

**Tech Stack:** Claude Code plugin system (markdown files), YAML frontmatter for skills/config, git for vault sync, Obsidian as optional UI layer.

---

## File Map

| File | Purpose |
|------|---------|
| `README.md` | User-facing docs: what it is, install, usage |
| `LICENSE` | MIT license |
| `skills/l3-vault/SKILL.md` | Vault brain: all format rules, operations, query patterns, safety constraints |
| `commands/l3.md` | `/l3` slash command: smart orchestrator for all vault operations |
| `commands/l3-setup.md` | `/l3-setup` slash command: one-time vault initialization wizard |

---

## Task 1: Scaffold Repository Structure

**Files:**
- Create: `LICENSE`
- Create: `skills/l3-vault/` (directory)
- Create: `commands/` (directory)

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p skills/l3-vault
mkdir -p commands
```

Run from the repo root (`/Users/y0shi/workspace/y0shi/elthree`).

- [ ] **Step 2: Write LICENSE**

Create `LICENSE` with MIT license content:

```
MIT License

Copyright (c) 2026 Joshua Reddick

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Verify structure**

```bash
ls -R skills/ commands/
```

Expected output:
```
commands/:
skills/:
skills/l3-vault/:
```

- [ ] **Step 4: Commit**

```bash
git add LICENSE skills/ commands/
git commit -m "chore: scaffold plugin directory structure"
```

---

## Task 2: Write `skills/l3-vault/SKILL.md`

This is the brain of the plugin. It encodes every rule Claude needs to manage vault files. The `description` field is used for semantic matching — when the user says anything task-related, this skill should activate.

**Files:**
- Create: `skills/l3-vault/SKILL.md`

- [ ] **Step 1: Verify spec coverage checklist before writing**

The SKILL.md must contain:
- [ ] Frontmatter with `name` and `description` trigger conditions
- [ ] Config lookup instructions (read `~/.l3/config.md` first)
- [ ] Task file format: filename pattern, YAML frontmatter fields, body structure
- [ ] ID generation algorithm (scan Tasks/ + Archive/, find max, increment, zero-pad)
- [ ] Status lifecycle rules
- [ ] Create operation (7 steps)
- [ ] Update operation (5 rules)
- [ ] Complete operation (3 steps)
- [ ] Archive operation with quarter mapping
- [ ] Process Inbox operation
- [ ] Weekly Review operation
- [ ] Query patterns table
- [ ] Inline task format (emoji priority)
- [ ] Safety rules (7 rules)
- [ ] Formatting rules

- [ ] **Step 2: Write the skill file**

Create `skills/l3-vault/SKILL.md` with this exact content:

````markdown
---
name: l3-vault
description: Use for any task management or personal productivity operation. Activate when the user mentions tasks, to-dos, work items, projects, weekly review, inbox processing, or any reference to the vault, L3, or elthree. Also activate when the user invokes /l3 or /l3-setup.
---

# L3 Vault Skill

You are operating on a personal task vault — a directory of markdown files in a git repository. This skill defines every rule for reading, writing, and managing vault content. Follow it exactly.

## Config Lookup

**Before any vault operation**, read `~/.l3/config.md` to get the vault path.

If `~/.l3/config.md` does not exist: stop immediately and say:
> "Vault not configured. Run `/l3-setup` to get started."

Config file format:
```yaml
---
vault_path: ~/vault
git_remote: git@github.com:user/vault-tasks.git
created: YYYY-MM-DD
---
```

If `vault_path` directory does not exist on disk: stop and say:
> "Vault directory not found at `<path>`. Re-run `/l3-setup`."

All file paths in this skill are relative to `vault_path`.

---

## Task File Format

**Location:** `Tasks/` directory inside the vault

**Filename pattern:** `TASK-NNN_slug-title.md`

Examples:
- `Tasks/TASK-001_initial-setup.md`
- `Tasks/TASK-042_bedrock-compliance.md`
- `Tasks/TASK-100_q2-planning.md`

Slug rules: lowercase, spaces become hyphens, strip special characters, max 50 chars.

**YAML Frontmatter:**

Required fields are marked with `*`. Write all fields, leave optional ones empty rather than omitting them.

```yaml
---
id: "TASK-NNN"          # * Zero-padded, monotonically increasing — never reuse
title: "Human title"    # * Same as the H1 heading in the body
status: todo            # * todo | in-progress | waiting | review | done
priority: medium        # * critical | high | medium | low
created: YYYY-MM-DD     # * Date task was created — never change after creation
due:                    # YYYY-MM-DD or empty
scheduled:              # YYYY-MM-DD — when to start work
project:                # Wikilink, quoted: "[[Projects/slug]]"
area:                   # Responsibility area slug (e.g., platform-engineering)
tags: []                # Freeform list, lowercase-hyphenated
waiting_on: ""          # Person or event being waited on
blocked_by: []          # List of TASK-NNN IDs blocking this task
source: ""              # Where the task originated (e.g., "RTC meeting 2026-04-14")
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
- YYYY-MM-DD: Task created
```

---

## ID Generation

When creating a new task:

1. List all `*.md` files in `Tasks/` (flat, not recursive)
2. List all `*.md` files in `Archive/` (recursive, including subdirectories)
3. For each filename, extract the leading `TASK-NNN` number using the pattern `TASK-(\d+)`
4. Find the highest number across both lists
5. Increment by 1
6. Zero-pad to at least 3 digits: `1` → `001`, `42` → `042`, `100` → `100`, `1000` → `1000`
7. If no task files exist yet, start at `TASK-001`

**Never reuse an ID**, even if the original task file was deleted.

---

## Status Lifecycle

```
todo → in-progress → review → done → (archived)
todo → waiting → todo  (when unblocked)
any  → done            (can skip intermediate states)
```

Rules:
- Set `status: waiting` automatically when `waiting_on` is populated
- Set `status: done` when all acceptance criteria are checked off
- Only move tasks to `Archive/` during weekly review or when explicitly asked

---

## Operations

### Create

1. Generate the next task ID (see ID Generation above)
2. Derive slug from title: lowercase, replace spaces/special chars with hyphens, strip leading/trailing hyphens, max 50 chars
3. Create file at `Tasks/TASK-NNN_slug.md`
4. Populate all required frontmatter fields; leave optional fields empty
5. Set `created` to today's date (YYYY-MM-DD)
6. Write the objective section and at least one acceptance criterion
7. Add initial log entry: `- YYYY-MM-DD: Task created`
8. If a source is known (meeting name, Slack channel, email subject), populate `source`

### Update

1. Read the existing file first — never update blindly
2. Modify only the specified frontmatter fields
3. If `status` changes, append to the Log section: `- YYYY-MM-DD: Status changed to {new_status}`
4. If meaningful work happened (not just a field update), append a descriptive log entry
5. Never edit or remove existing log entries — only append new ones at the bottom
6. Preserve existing body structure — do not rewrite sections you are not changing

### Complete

1. Set `status: done`
2. Mark all acceptance criteria checkboxes as checked: `- [x]`
3. Append to Log: `- YYYY-MM-DD: Completed`
4. Do NOT move to Archive — archiving happens during weekly review or when explicitly asked

### Archive

Move `done` tasks from `Tasks/` to `Archive/YYYY-QN/` using this quarter mapping:
- Q1 = January–March
- Q2 = April–June
- Q3 = July–September
- Q4 = October–December

Example: a task completed in May 2026 goes to `Archive/2026-Q2/`.

Rules:
- Only archive tasks with `status: done`
- Move the file exactly as-is — do not modify its content
- Create the `Archive/YYYY-QN/` directory if it doesn't exist

### Process Inbox

1. Read `Inbox.md`
2. For each item, decide:
   - **Needs tracking** (has a clear owner, deadline, or acceptance criteria): create a task file in `Tasks/` using the Create operation
   - **Small project action** (single-step, belongs to a known project): add as inline task to the relevant `Projects/` note
3. Remove each processed item from `Inbox.md`
4. Leave unprocessable items in place with a comment explaining what's missing: `<!-- needs clarification: who owns this? -->`
5. Report what was created

### Weekly Review

1. Determine the current ISO week number. Format: `YYYY-WNN` (zero-pad the week, e.g., `2026-W16`)
2. Create `Reviews/YYYY-WNN.md` with these sections:

```markdown
# Week Review: YYYY-WNN

## Completed This Week
[Tasks that moved to status: done this week, identified by log entries dated this week]

## Open Tasks by Project
[Group all non-done tasks by their project field. Tasks with no project go under "Unassigned".]

## Overdue
[Tasks where due < today and status ≠ done. Include task ID, title, due date, days overdue.]

## Waiting
[Tasks with status: waiting. Include task ID, title, waiting_on value, and how many days they've been waiting (today - the last log entry date that set status to waiting).]

## Missing Due Dates
[Open tasks with empty due field that appear time-sensitive based on their title or content.]

## Stale In-Progress
[Tasks with status: in-progress that have no log entry in the past 14 days.]
```

3. Archive all `done` tasks in `Tasks/` to `Archive/YYYY-QN/` (see Archive operation)
4. Process `Inbox.md` (see Process Inbox operation)
5. Report a summary: N tasks completed, N archived, N overdue, N waiting

---

## Query Patterns

Read actual file frontmatter — do not guess or estimate task state.

| Natural language | What to do |
|----------------|------------|
| "What's overdue?" | Find files in `Tasks/` where `due < today` and `status != done` |
| "What's high priority?" | Find files where `priority: high` or `priority: critical` and `status != done` |
| "Tasks for project X" | Find files where `project` field contains X |
| "What am I waiting on?" | Find files where `status: waiting`, report each `waiting_on` value and how long |
| "What did I finish?" | Find files in `Tasks/` where `status: done` (not yet archived) |
| "Show TASK-042" | Read `Tasks/TASK-042_*.md` — glob the number, filename suffix may vary |
| "What's blocked?" | Find files with non-empty `blocked_by` list and `status != done` |
| "What's scheduled today?" | Find files where `scheduled == today` |

---

## Inline Tasks

Files in `Projects/` and `Areas/` may use GFM checkboxes with emoji priority metadata:

```markdown
- [ ] Task description 📅 YYYY-MM-DD ⏫
- [ ] Another task 📅 YYYY-MM-DD 🔼
- [ ] Low priority thing 🔽
- [ ] Medium priority (no emoji)
```

Priority emoji:
- `⏫` = critical/highest
- `🔼` = high
- (no emoji) = medium
- `🔽` = low

**When to promote to a full task file:** When the item needs its own acceptance criteria, log history, due date tracking, or is taking more than a day. After promoting, replace the inline checkbox with a wikilink:

```markdown
- [ ] [[Tasks/TASK-NNN_slug]]
```

---

## Safety Rules

Follow these without exception:

1. **Never delete a task file** — change `status` to `done` and archive it instead
2. **Never edit or remove log entries** — only append new ones at the bottom of the Log section
3. **Never change `id` or `created`** after the file is first created
4. **Never create task files outside** `Tasks/`, `Projects/`, or `Inbox.md`
5. **Never modify files in `Archive/`** unless explicitly asked by the user
6. **Never touch `.obsidian/`** configuration files
7. **If unsure about a field value** — leave it empty rather than guessing

---

## Formatting Rules

- All dates: `YYYY-MM-DD` (ISO 8601 — no slashes, no shortcuts)
- Frontmatter strings containing special characters (brackets, colons): always quote them
- Wikilinks in frontmatter must be quoted: `project: "[[Projects/slug]]"`
- Tags: lowercase, hyphenated — `data-engineering`, not `DataEngineering` or `data_engineering`
- File slugs: lowercase, hyphenated, max 50 chars
- Log entries: chronological order, most recent at the bottom
- YAML: 2-space indentation, never tabs
````

- [ ] **Step 3: Verify YAML frontmatter is valid**

```bash
python3 -c "
import yaml, re

with open('skills/l3-vault/SKILL.md') as f:
    content = f.read()

# Extract frontmatter between first two --- delimiters
match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if not match:
    print('ERROR: No frontmatter found')
    exit(1)

data = yaml.safe_load(match.group(1))
print('Frontmatter fields:', list(data.keys()))
assert 'name' in data, 'Missing: name'
assert 'description' in data, 'Missing: description'
print('OK')
"
```

Expected output:
```
Frontmatter fields: ['name', 'description']
OK
```

- [ ] **Step 4: Verify all spec sections are present**

```bash
python3 -c "
with open('skills/l3-vault/SKILL.md') as f:
    content = f.read()

required = [
    'Config Lookup',
    'Task File Format',
    'ID Generation',
    'Status Lifecycle',
    '### Create',
    '### Update',
    '### Complete',
    '### Archive',
    '### Process Inbox',
    '### Weekly Review',
    'Query Patterns',
    'Inline Tasks',
    'Safety Rules',
    'Formatting Rules',
]

for section in required:
    if section not in content:
        print(f'MISSING: {section}')
    else:
        print(f'OK: {section}')
"
```

Expected: all lines print `OK`.

- [ ] **Step 5: Commit**

```bash
git add skills/l3-vault/SKILL.md
git commit -m "feat: add l3-vault skill with vault format, operations, and safety rules"
```

---

## Task 3: Write `commands/l3-setup.md`

This becomes the `/l3-setup` slash command. Claude reads it in full when the user invokes `/l3-setup` and follows the steps exactly. The templates are embedded directly — no external files.

**Files:**
- Create: `commands/l3-setup.md`

- [ ] **Step 1: Verify spec coverage checklist before writing**

The command must contain:
- [ ] Step 1: Check for existing config, offer reconfigure/cancel
- [ ] Step 2: Ask two questions (vault path, git remote)
- [ ] Step 3: Create vault directory structure with all directories listed
- [ ] Step 3: Embedded `Templates/task.md` content (Templater syntax)
- [ ] Step 3: Embedded `Templates/project.md` content
- [ ] Step 3: Embedded `.gitignore` content
- [ ] Step 3: Embedded `Inbox.md` initial content
- [ ] Step 3: Re-run safety (skip existing directories, overwrite Templates/)
- [ ] Step 4: `git init`, `git add -A`, `git commit`, optional remote add + push
- [ ] Step 5: Write `~/.l3/config.md`
- [ ] Step 6: Print manual steps checklist (Obsidian, plugins, MCP configs)

- [ ] **Step 2: Write the command file**

Create `commands/l3-setup.md` with this exact content:

````markdown
# /l3-setup — Vault Initialization Wizard

You are running the L3 vault setup wizard. Follow these steps in order. Be conversational but efficient — this is a guided setup, not a Q&A session.

---

## Step 1: Check for Existing Config

Read `~/.l3/config.md`.

**If it exists**, show the current settings:
```
Current vault config:
  vault_path: <value>
  git_remote: <value or "(none)">
  created: <value>
```
Then ask: "Your vault is already configured. Do you want to reconfigure it, or cancel?"
- If cancel: stop here, say "Setup cancelled. Your existing vault is unchanged."
- If reconfigure: continue to Step 2

**If it does not exist**, continue directly to Step 2.

---

## Step 2: Gather Configuration

Ask these two questions together in a single message:

> 1. Where should your vault live? (Press Enter for default: `~/vault`)
> 2. Do you have a Git remote URL? (SSH or HTTPS — optional, press Enter to skip)

Accept whatever the user provides. Expand `~` to the full home directory path.
- If vault path is blank: use `~/vault`
- If git remote is blank: skip git remote setup

---

## Step 3: Create Vault Structure

Create the following structure at the specified vault path. **If a directory already exists, skip it** — never overwrite existing `Tasks/`, `Projects/`, `Areas/`, `Archive/`, `Reviews/` contents or `Inbox.md`.

Only `Templates/task.md`, `Templates/project.md`, and `.gitignore` are always written (overwrite if present).

```
<vault_path>/
├── Tasks/
├── Projects/
├── Areas/
├── Archive/
├── Templates/
│   ├── task.md         ← always write
│   └── project.md      ← always write
├── Reviews/
├── Inbox.md            ← only create if missing
└── .gitignore          ← always write
```

### `Templates/task.md`

Write this content exactly (the `{{...}}` syntax is expanded by Obsidian's Templater plugin at note-creation time):

```markdown
---
id: "TASK-{{tp.user.nextTaskId()}}"
title: "{{title}}"
status: todo
priority: medium
created: {{tp.date.now("YYYY-MM-DD")}}
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
- {{tp.date.now("YYYY-MM-DD")}}: Task created
```

### `Templates/project.md`

```markdown
---
name: "{{title}}"
area:
status: active
created: {{tp.date.now("YYYY-MM-DD")}}
tags: []
---

# {{title}}

## Goal


## Active Tasks
- [ ] 

## Notes


## Decisions

```

### `.gitignore`

```
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.DS_Store
Thumbs.db
```

### `Inbox.md` (only if file does not exist)

```markdown
# Inbox

Add quick-capture items here. Run `/l3 process inbox` to file them into Tasks/.

```

---

## Step 4: Initialize Git

Run these commands in the vault directory:

```bash
git init
git add -A
git commit -m "Initial vault setup"
```

If a git remote was provided in Step 2:
```bash
git remote add origin <url>
git push -u origin main
```

If any git command fails, report the specific error and continue — the vault files are usable without git. Do not abort the setup.

---

## Step 5: Write Config File

Create `~/.l3/` directory if it doesn't exist. Write `~/.l3/config.md`:

```markdown
---
vault_path: <full expanded vault path>
git_remote: <remote URL, or empty string if not provided>
created: <today YYYY-MM-DD>
---
```

Example:
```markdown
---
vault_path: /Users/josh/vault
git_remote: git@github.com:josh/vault-tasks.git
created: 2026-04-14
---
```

---

## Step 6: Print Manual Steps Checklist

Tell the user:

> Your vault is ready at `<vault_path>`. A few manual steps remain:

```
Next steps (do these yourself):

[ ] Install Obsidian: https://obsidian.md
[ ] Open your vault in Obsidian: File → Open Folder → <vault_path>
[ ] Install community plugins (Settings → Community plugins → Browse):
    - Obsidian Git
    - Tasks
    - Templater
    - Dataview
[ ] Configure Obsidian Git:
    - Auto-commit interval: 5 minutes
    - Pull on startup: enabled
    - Push after commit: enabled

Optional (for Claude Desktop):
[ ] Add filesystem MCP server to ~/Library/Application Support/Claude/claude_desktop_config.json:
    {
      "mcpServers": {
        "vault": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "<vault_path>"]
        }
      }
    }

Optional (for Claude Mobile — requires vault pushed to GitHub):
[ ] Go to claude.ai → Settings → Connectors → Add GitHub MCP connector
    Authenticate with a GitHub PAT that has repo read/write access
```

Then say: "You're all set. Run `/l3` anytime to manage your tasks."
````

- [ ] **Step 3: Verify all spec sections present**

```bash
python3 -c "
with open('commands/l3-setup.md') as f:
    content = f.read()

required = [
    'Step 1',
    'Step 2',
    'Step 3',
    'Step 4',
    'Step 5',
    'Step 6',
    'Templates/task.md',
    'Templates/project.md',
    '.gitignore',
    'Inbox.md',
    'git init',
    'git remote add origin',
    '~/.l3/config.md',
    'obsidian.md',
    'Obsidian Git',
    'Templater',
]
for item in required:
    status = 'OK' if item in content else 'MISSING'
    print(f'{status}: {item}')
"
```

Expected: all lines print `OK`.

- [ ] **Step 4: Commit**

```bash
git add commands/l3-setup.md
git commit -m "feat: add /l3-setup vault initialization wizard command"
```

---

## Task 4: Write `commands/l3.md`

This becomes the `/l3` slash command — the main interface for all vault operations. It reads the config, classifies intent, applies l3-vault skill operations, and reports results.

**Files:**
- Create: `commands/l3.md`

- [ ] **Step 1: Verify spec coverage checklist before writing**

The command must contain:
- [ ] Step 1: Load config (with both error cases)
- [ ] Step 2: No-argument morning briefing (overdue, today's scheduled, top 3 by priority)
- [ ] Step 3: Intent classification table (create, update, query, review, archive, inbox, bulk)
- [ ] Step 4: Map intents to l3-vault skill operations
- [ ] Step 5: Confirmation before destructive/bulk actions
- [ ] Step 6: Report what was done after every write
- [ ] Error handling table (4 cases)

- [ ] **Step 2: Write the command file**

Create `commands/l3.md` with this exact content:

```markdown
# /l3 — L3 Task Manager

You are the L3 task management interface. Use the `l3-vault` skill for all vault operations. This command is the single entry point for everything.

---

## Step 1: Load Config

Read `~/.l3/config.md`.

**If missing:** Stop and say:
> "Vault not configured. Run `/l3-setup` to get started."

**If `vault_path` directory does not exist on disk:** Stop and say:
> "Vault directory not found at `<path>`. Re-run `/l3-setup`."

---

## Step 2: Handle No-Argument Case

If `/l3` was invoked with no arguments (just `/l3` with nothing after it), show a morning briefing:

**Format:**
```
Good morning. Here's your task status:

Overdue (N):
  TASK-NNN — Title (due YYYY-MM-DD, N days ago)
  ...

Scheduled today (N):
  TASK-NNN — Title
  ...

Top priorities (open):
  1. TASK-NNN — Title [critical]
  2. TASK-NNN — Title [high]
  3. TASK-NNN — Title [high]
```

If nothing is overdue, say "Nothing overdue." If nothing is scheduled today, say "Nothing scheduled today." Show top 3 open tasks by priority (critical > high > medium > low). If there are fewer than 3 open tasks total, show all of them.

---

## Step 3: Classify Intent

Map the user's input to one of these intents:

| Intent | Example inputs |
|--------|---------------|
| `create` | "create a task for X", "add a task: Y", "new task: Z", "track this: ..." |
| `update` | "mark TASK-042 as done", "set priority to high on TASK-007", "TASK-007 is now in-progress", "update TASK-042: ..." |
| `query` | "what's overdue?", "show high priority tasks", "what am I waiting on?", "show TASK-042", "what's blocked?" |
| `review` | "weekly review", "run my review", "end of week review" |
| `archive` | "archive done tasks", "archive TASK-042", "move completed tasks to archive" |
| `inbox` | "process inbox", "what's in my inbox?", "clear the inbox" |
| `bulk` | "bulk update: set all X tasks to Y", "mark all bedrock-tagged tasks as done" |

**If intent is ambiguous:** Ask exactly one clarifying question before acting. Do not act on ambiguous input. Do not guess.

---

## Step 4: Apply l3-vault Skill Operations

Route to the correct l3-vault skill operation:

| Intent | Skill operation |
|--------|----------------|
| `create` | Create |
| `update` (not marking done) | Update |
| `update` (marking done / completing) | Complete |
| `query` | Query Patterns — read actual frontmatter, never estimate |
| `review` | Weekly Review |
| `archive` | Archive |
| `inbox` | Process Inbox |
| `bulk` | Update applied to each matching task (confirm first — see Step 5) |

---

## Step 5: Confirm Before Destructive or Bulk Actions

Before executing any of these, summarize what will happen and ask "Proceed?":

- Archiving 2 or more tasks at once
- Any bulk update affecting more than 1 task
- Processing inbox (modifies Inbox.md)
- Weekly review (archives tasks, modifies multiple files)

Single-task creates, updates, and completes do not need confirmation.

---

## Step 6: Report What Was Done

After every write operation, always report:
- What was created, updated, completed, or archived
- Task ID and title
- New status if it changed
- Due date if relevant

**Examples:**

Create: `Created TASK-007: Review Bedrock compliance (high priority, due 2026-04-18)`

Complete: `Completed TASK-042: Finalize Dataiku ADR — all criteria checked off`

Bulk: `Updated 3 tasks: set area to platform-engineering on TASK-031, TASK-038, TASK-044`

Archive: `Archived 5 tasks to Archive/2026-Q2/`

---

## Error Handling

| Condition | Response |
|-----------|----------|
| `~/.l3/config.md` missing | "Vault not configured. Run `/l3-setup` to get started." |
| `vault_path` directory not found | "Vault directory not found at `<path>`. Re-run `/l3-setup`." |
| Task ID not found (e.g., TASK-999) | "TASK-999 not found. Did you mean: [list 3 tasks with similar titles or nearby IDs]?" |
| Intent is ambiguous | Ask one clarifying question. State what you understood and what's unclear. |
```

- [ ] **Step 3: Verify all spec sections present**

```bash
python3 -c "
with open('commands/l3.md') as f:
    content = f.read()

required = [
    'Step 1',
    'Step 2',
    'Step 3',
    'Step 4',
    'Step 5',
    'Step 6',
    'morning briefing',
    'Intent',
    'create',
    'update',
    'query',
    'review',
    'archive',
    'inbox',
    'bulk',
    'Error Handling',
    'config.md missing',
    'vault_path',
    'Task ID not found',
]
for item in required:
    status = 'OK' if item in content else 'MISSING'
    print(f'{status}: {item}')
"
```

Expected: all lines print `OK`.

- [ ] **Step 4: Commit**

```bash
git add commands/l3.md
git commit -m "feat: add /l3 vault orchestration command with routing and error handling"
```

---

## Task 5: Write `README.md`

User-facing documentation. Covers what it is, how to install, how to use the two commands, and how the system works.

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

Create `README.md` with this exact content:

````markdown
# Elthree (L3)

A Claude Code plugin for git-backed personal task management. Obsidian is the UI. Claude is the operator. Markdown files are the persistence layer.

---

## Install

```
/plugin install github:y0shi/elthree
```

---

## Setup (once)

```
/l3-setup
```

This initializes your vault directory, creates the file structure, sets up git, and gives you a checklist of manual steps for Obsidian and MCP configuration.

---

## Usage

```
/l3
```

No arguments: shows a morning briefing — overdue items, today's scheduled tasks, top 3 open tasks by priority.

```
/l3 create a task for reviewing Bedrock compliance, high priority, due Friday
/l3 what's overdue?
/l3 mark TASK-042 as done
/l3 weekly review
/l3 process inbox
/l3 show everything blocked by TASK-038
/l3 bulk update: set all bedrock-tagged tasks to area platform-engineering
```

---

## How It Works

**Vault** (`~/vault` by default): A local directory of markdown files tracked in git. One `.md` file per task, plus project notes, area notes, a reviews folder, and an inbox.

**Config** (`~/.l3/config.md`): Written by `/l3-setup`. Stores your vault path so `/l3` works from any directory.

**Task files** live in `Tasks/` with YAML frontmatter:

```
Tasks/TASK-042_bedrock-compliance.md
```

```yaml
---
id: "TASK-042"
title: "Review Bedrock compliance"
status: in-progress
priority: high
created: 2026-04-14
due: 2026-04-18
project: "[[Projects/platform-engineering]]"
area: platform-engineering
tags: [bedrock, compliance]
---
```

**Git sync**: The Obsidian Git plugin auto-commits on a 5-minute interval. Changes from any Claude client land in the same repo and sync across devices.

**Mobile**: Add the GitHub MCP connector in claude.ai → Settings → Connectors. Claude Mobile reads and writes your vault via GitHub commits.

---

## Task States

```
todo → in-progress → review → done → archived
todo → waiting → todo (when unblocked)
```

---

## Vault Structure

```
~/vault/
├── Tasks/           # One .md file per tracked task
├── Projects/        # Project notes with inline tasks
├── Areas/           # Ongoing responsibility area notes
├── Archive/YYYY-QN/ # Completed tasks, organized by quarter
├── Templates/       # task.md and project.md for Obsidian Templater
├── Reviews/         # Weekly review snapshots
└── Inbox.md         # Quick-capture scratchpad
```

---

## Recommended Obsidian Plugins

| Plugin | Purpose |
|--------|---------|
| Obsidian Git | Auto-commit, push, pull on interval |
| Tasks | Query tasks across the vault by frontmatter |
| Templater | Template expansion with auto-generated task IDs |
| Dataview | Advanced frontmatter queries and dashboards |

---

## Requirements

- Claude Code with plugin support
- Git
- A GitHub account (for mobile sync — optional)
- Obsidian (for the UI layer — optional but recommended)
````

- [ ] **Step 2: Verify README structure**

```bash
python3 -c "
with open('README.md') as f:
    content = f.read()

required = [
    '/plugin install',
    '/l3-setup',
    '/l3',
    'morning briefing',
    'vault',
    '~/.l3/config.md',
    'YAML frontmatter',
    'Obsidian Git',
    'Templater',
    'Tasks/',
    'Archive/',
]
for item in required:
    status = 'OK' if item in content else 'MISSING'
    print(f'{status}: {item}')
"
```

Expected: all lines print `OK`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README with install, usage, and vault structure overview"
```

---

## Task 6: Spec Coverage Self-Review

Check the final deliverable against the spec. No subagent needed — do this inline.

**Files:**
- Read: `docs/superpowers/specs/2026-04-13-elthree-plugin-design.md`
- Read: all plugin files

- [ ] **Step 1: Verify repo structure matches spec**

```bash
find . -not -path './.git/*' -not -path './docs/*' -not -path './.claude/*' -not -name '.DS_Store' | sort
```

Expected output should include:
```
./LICENSE
./README.md
./commands/l3-setup.md
./commands/l3.md
./skills/l3-vault/SKILL.md
```

- [ ] **Step 2: Cross-check spec requirements**

Verify each spec section has coverage:

| Spec Section | Coverage |
|-------------|---------|
| Config file at `~/.l3/config.md` with `vault_path`, `git_remote`, `created` | `skills/l3-vault/SKILL.md` Config Lookup; `commands/l3-setup.md` Step 5 |
| Task filename pattern `TASK-NNN_slug-title.md` | `skills/l3-vault/SKILL.md` Task File Format |
| All 13 YAML frontmatter fields | `skills/l3-vault/SKILL.md` Task File Format |
| ID generation: scan Tasks/ + Archive/, find max, increment, zero-pad | `skills/l3-vault/SKILL.md` ID Generation |
| Status lifecycle with all valid transitions | `skills/l3-vault/SKILL.md` Status Lifecycle |
| Create, Update, Complete, Archive, Process Inbox, Weekly Review operations | `skills/l3-vault/SKILL.md` Operations |
| All 7 query patterns | `skills/l3-vault/SKILL.md` Query Patterns |
| Inline tasks with emoji priority in Projects/Areas | `skills/l3-vault/SKILL.md` Inline Tasks |
| 7 safety rules | `skills/l3-vault/SKILL.md` Safety Rules |
| `/l3` with no args → morning briefing | `commands/l3.md` Step 2 |
| `/l3` routing logic with 7 intents | `commands/l3.md` Steps 3–4 |
| `/l3` error handling (4 cases) | `commands/l3.md` Error Handling |
| `/l3-setup` 6-step flow | `commands/l3-setup.md` Steps 1–6 |
| `/l3-setup` embedded templates | `commands/l3-setup.md` Step 3 |
| `/l3-setup` re-run safety | `commands/l3-setup.md` Step 3 |
| Manual steps checklist (Obsidian, plugins, MCP) | `commands/l3-setup.md` Step 6 |

- [ ] **Step 3: Check for consistency — IDs, terms, paths used across files**

Verify these are consistent:
- Config path is `~/.l3/config.md` in both the skill and both commands
- Status values (`todo`, `in-progress`, `waiting`, `review`, `done`) match between skill and commands
- Archive quarter format (`YYYY-QN`) is used consistently
- The `/l3` command references `l3-vault` skill by name

```bash
grep -r "config.md" skills/ commands/ | grep -v "~/.l3/config.md"
```

Expected: no output (all config references should use `~/.l3/config.md`).

```bash
grep -r "YYYY-QN\|Archive/" skills/ commands/ | head -20
```

Expected: all archive references use the same `YYYY-QN` format.

- [ ] **Step 4: Final commit**

```bash
git add -A
git status  # verify only expected files staged
git commit -m "chore: complete elthree plugin v1.0 — l3-vault skill, /l3 and /l3-setup commands"
```

---

## Out of Scope (do not implement)

Per spec:
- Obsidian plugin installation automation
- Claude Desktop MCP auto-configuration
- Multi-vault support
- Team/shared vaults
- Web UI or dashboard
