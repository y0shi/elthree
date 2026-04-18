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

**Vault directory layout:**
```
<vault_path>/
├── Tasks/       ← task files live here
├── Projects/    ← project notes with inline tasks
├── Areas/       ← ongoing responsibility area notes
├── Archive/     ← completed tasks, organized by quarter
├── Reviews/     ← weekly review snapshots
├── Energy/      ← daily energy level logs (one file per day)
└── Inbox.md     ← quick-capture scratchpad
```

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
cognitive_load:         # 1–6 (Bloom's: 1=remember, 2=understand, 3=apply, 4=analyze, 5=evaluate, 6=create)
effort:                 # 1–10 (1=minutes, 10=days of work)
---
```

**Auto-Classification on Task Creation**

When creating a task, auto-set `cognitive_load` and `effort` from the task title and objective. Allow user override at any time via `/l3 update TASK-NNN set cognitive_load 2` or natural language ("set effort to 3 on TASK-007").

**Cognitive load classification:**

| Value | Label | Keywords / signals |
|-------|-------|-------------------|
| 1 | Remember | send, reply, check, look up, find, confirm |
| 2 | Understand | read, review, summarize, skim, watch |
| 3 | Apply | update, configure, run, calculate, execute |
| 4 | Analyze | debug, compare, investigate, trace, audit |
| 5 | Evaluate | decide, approve, choose, assess, prioritize |
| 6 | Create | design, build, write, architect, implement |

**Effort estimation:**

| Range | Meaning |
|-------|---------|
| 1–2 | Minutes — quick reply, one-liner, checkbox |
| 3–4 | Under an hour — small update, single config change |
| 5–6 | A few hours — meaningful feature, thorough review |
| 7–8 | Half to full day — significant implementation |
| 9–10 | Multi-day — large feature, complex investigation |

If a task is ambiguous, bias toward the lower classification for both fields. Leave both empty (not 0) rather than guessing.

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

1. List all `*.md` files in `Tasks/` (recursive)
2. List all `*.md` files in `Archive/` (recursive)
3. For each filename, extract the leading `TASK-NNN` number using the pattern `TASK-(\d+)`
4. Find the highest number across both lists
5. Increment by 1
6. Zero-pad to 3 digits minimum: `1` → `001`, `42` → `042`, `100` → `100`; IDs ≥ 1000 use their natural digit count (`1000` → `1000`)
7. If no task files exist yet, start at `TASK-001`

**Never reuse an ID**, even if the original task file was deleted.

---

## Status Lifecycle

```
todo → in-progress → review → done → (archived)
todo → waiting → todo  (when unblocked)
any  → waiting         (when waiting_on is populated)
any  → done            (can skip intermediate states)
```

Rules:
- Set `status: waiting` automatically when `waiting_on` is populated
- Set `status: done` when all acceptance criteria are checked off
- Only move tasks to `Archive/` during weekly review or when explicitly asked
- `blocked_by` is metadata only — populating it does not automatically change status; if the user explicitly says they are blocked, also set `status: waiting` and `waiting_on` to the blocking task ID or person

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

1. Read `<vault_path>/Inbox.md`
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

## Energy Log

**Location:** `Energy/YYYY-MM-DD.md` — one file per day, created on first energy entry for that date.

**File format:**
```yaml
---
date: YYYY-MM-DD
level: 3
---
```

**Reading energy:**
- Before any recommendation or quick wins operation, check for `Energy/YYYY-MM-DD.md` using today's date
- If the file exists and has a valid `level` (1–5): use it silently
- If the file does not exist: ask `"What's your energy level today? (1–5)"` — wait for response, then write the file before proceeding

**Writing energy:**
- Write `Energy/YYYY-MM-DD.md` whenever the user states their energy level, whether via the briefing prompt or explicitly ("my energy is 4", "set energy to 2")
- Overwrite if the file already exists — last stated level wins

**Energy-to-cognitive-load compatibility:**

| Energy | Optimal cognitive load range |
|--------|------------------------------|
| 1 (foggy/exhausted) | 1–2 |
| 2 (low) | 1–3 |
| 3 (steady) | 2–4 |
| 4 (focused) | 3–5 |
| 5 (peak) | 4–6 |

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
| "Quick wins", "easy tasks", "what can I knock out fast", "low effort tasks" | Quick Wins operation — see below |

---

## Recommendation Scoring

Used by the energy-aware briefing and quick wins mode. Scores are ephemeral — computed at query time, never written to task files.

**Formula:**

```
score = (urgency × 0.35) + (goal_alignment × 0.30) + (effort_inverse × 0.20) + (energy_match × 0.15)
```

**Component definitions:**

`urgency` — deadline proximity, normalized 0–1:
- Overdue: 1.0
- Due today: 0.9
- Due in 1 day: 0.8
- Due in 2 days: 0.7
- Due in 3–6 days: 0.5
- Due in 7+ days: 0.1
- No due date: 0.3

`goal_alignment` — read the linked project file's `priority` frontmatter field (e.g., `Projects/slug.md`):
- critical: 1.0 / high: 0.75 / medium: 0.5 / low: 0.25
- No project, unreadable project file, or missing priority field: 0.5

`effort_inverse` — `1 - (effort / 10)` — lower effort scores higher

`energy_match` — how well the task's cognitive demand matches current energy:
- `distance = abs(energy_level - cognitive_load)`
- `energy_match = max(0, 1 - (distance / 5))`

**Fallback:** Tasks missing `cognitive_load` or `effort` cannot be scored. Show them after scored tasks, sorted by priority label (critical > high > medium > low), labeled "Top priorities (unscored):".

## Quick Wins

**Trigger:** User says "quick wins", "easy tasks", "what can I knock out fast", "low effort tasks"

**Steps:**
1. Check energy (read `Energy/YYYY-MM-DD.md`; ask if missing)
2. Filter open tasks: `effort ≤ 3` AND `status != done` AND `blocked_by` is empty or all listed IDs have `status: done`
3. Score filtered tasks using Recommendation Scoring formula. Tasks missing `cognitive_load` cannot be fully scored — show them unscored at the bottom of the list, labeled with `[unscored]`
4. Show max 5, sorted by score descending

**Output format:**
```
Quick wins (N tasks, energy: E):
  TASK-007 — Reply to vendor email [effort: 1, load: 1]
  TASK-019 — Update deploy config [effort: 2, load: 3]
  TASK-031 — Review PR from Alex [effort: 3, load: 2]
```

If no tasks qualify: `"No quick wins found. No open tasks with effort ≤ 3 are available (they may be blocked or done)."`

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
4. **Never create task files outside** `Tasks/` — `Inbox.md` is a capture buffer, not a task file home; inline tasks in `Projects/` notes are acceptable
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
- Each log entry begins with `YYYY-MM-DD:` — use this prefix to parse dates when checking for staleness (e.g., weekly review stale detection scans for the most recent `YYYY-MM-DD:` prefix in the `## Log` section)
- YAML: 2-space indentation, never tabs
