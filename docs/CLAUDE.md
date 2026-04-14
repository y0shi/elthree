# CLAUDE.md — Task Vault Instructions

You are managing a personal task vault for Josh Reddick. This file defines how you interact with the vault's markdown files. Follow these conventions exactly.

---

## Vault Layout

```
Tasks/          → One .md file per tracked task (TaskNotes pattern)
Projects/       → Project notes with inline Obsidian Tasks
Areas/          → Ongoing responsibility area notes
Archive/YYYY-QN/ → Completed tasks, organized by quarter
Templates/      → task.md and project.md templates
Reviews/        → Weekly review snapshots
Inbox.md        → Quick-capture scratchpad (process into Tasks/ or Projects/)
```

---

## Task File Format

Every file in `Tasks/` uses this structure:

**Filename**: `TASK-NNN_slug-title.md` (e.g., `TASK-042_dataiku-adr.md`)

**YAML Frontmatter** (required fields marked with `*`):

```yaml
---
id: "TASK-NNN"          # * Zero-padded, monotonically increasing
title: "Human title"     # * Same as the H1 heading
status: todo             # * todo | in-progress | waiting | review | done
priority: medium         # * critical | high | medium | low
created: YYYY-MM-DD      # * Date task was created
due:                     # YYYY-MM-DD or empty
scheduled:               # YYYY-MM-DD — when to start work
project:                 # Wikilink: "[[Projects/slug]]"
area:                    # Responsibility area slug
tags: []                 # Freeform list
waiting_on: ""           # Person or event
blocked_by: []           # List of TASK-NNN IDs
source: ""               # Where the task originated
---
```

**Body Structure**:

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

---

## ID Generation

When creating a new task:

1. List all files in `Tasks/` and `Archive/`
2. Find the highest `TASK-NNN` number across both directories
3. Increment by 1, zero-pad to at least 3 digits
4. Never reuse an ID, even if the original task was deleted

---

## Status Lifecycle

```
todo → in-progress → review → done → (archived)
todo → waiting → todo  (when unblocked)
any  → done            (can skip intermediate states)
```

- Set `status: waiting` when `waiting_on` is populated
- Set `status: in-progress` when work begins
- Set `status: done` when all acceptance criteria are met
- Only move to `Archive/` during weekly review or explicit request

---

## Operations Reference

### Creating a Task

1. Generate next ID (see ID Generation above)
2. Create file at `Tasks/TASK-NNN_slug-title.md`
3. Populate all required frontmatter fields
4. Set `created` to today's date
5. Write objective and at least one acceptance criterion
6. Add initial log entry: `- YYYY-MM-DD: Task created`
7. If a source is known (meeting, Slack thread, email), populate `source`

### Updating a Task

1. Read the existing file
2. Update only the changed frontmatter fields
3. If status changes, add a log entry: `- YYYY-MM-DD: Status changed to {new_status}`
4. If meaningful work happened, add a log entry describing it
5. Never remove existing log entries
6. Preserve the body structure — do not rewrite sections you aren't updating

### Completing a Task

1. Set `status: done`
2. Check all acceptance criteria checkboxes (`- [x]`)
3. Add log entry: `- YYYY-MM-DD: Completed`
4. Do NOT move to Archive — that happens during weekly review

### Archiving

Move completed tasks to `Archive/YYYY-QN/` where QN is the current quarter:
- Q1 = Jan–Mar, Q2 = Apr–Jun, Q3 = Jul–Sep, Q4 = Oct–Dec
- Only archive tasks with `status: done`
- Preserve the file exactly as-is (including all log entries)

### Processing Inbox

When asked to process `Inbox.md`:
1. Read the file
2. For each item: create a proper task file in `Tasks/` or add as inline task to the appropriate `Projects/` note
3. Remove processed items from `Inbox.md`
4. Leave unprocessable items with a note about what's missing

---

## Inline Tasks in Project Notes

Files in `Projects/` and `Areas/` may contain inline Obsidian Tasks using GFM checkboxes with emoji metadata:

```markdown
- [ ] Task description 📅 YYYY-MM-DD ⏫
```

Priority emoji mapping:
- `⏫` = highest
- `🔼` = high  
- `🔽` = low
- (no emoji) = medium

When creating inline tasks, use this format. When a project inline task becomes complex enough to need its own acceptance criteria or log, promote it to a full `Tasks/` file and replace the inline item with a wikilink: `- [ ] [[Tasks/TASK-NNN_slug]]`

---

## Weekly Review

When asked to run a weekly review:

1. Create `Reviews/YYYY-WNN.md` with:
   - Summary of completed tasks this week
   - Open tasks grouped by project
   - Overdue items flagged
   - Waiting items with how long they've been waiting
   - Any tasks without a due date that need one
2. Archive all `done` tasks to `Archive/YYYY-QN/`
3. Process `Inbox.md`
4. Report anything that looks stale (in-progress for 2+ weeks with no log entry)

---

## Querying Tasks

When asked about task state, read files directly. Common queries:

| Request                    | Action                                                      |
|---------------------------|-------------------------------------------------------------|
| "What's overdue?"         | Find tasks where `due < today` and `status != done`         |
| "What's high priority?"   | Filter `priority: high` or `priority: critical`, not done   |
| "Tasks for project X"     | Filter by `project` field containing X                      |
| "What am I waiting on?"   | Filter `status: waiting`, report `waiting_on` values        |
| "What did I finish?"      | Filter `status: done` in `Tasks/` (not yet archived)        |
| "Show me task TASK-042"   | Read `Tasks/TASK-042_*.md` directly                         |

---

## Creating Tasks from External Input

When given meeting notes, Slack messages, or other context:

1. Extract actionable items (things with a clear next step)
2. Create one task file per actionable item
3. Set `source` to describe the origin (e.g., "RTC meeting 2026-04-11", "Slack #cads-modelops")
4. Set reasonable defaults: `priority: medium`, `status: todo`
5. If a deadline was mentioned, set `due`
6. If the task clearly belongs to a known project, set `project`
7. Summarize what you created so Josh can verify

---

## Formatting Rules

- All dates: `YYYY-MM-DD` (ISO 8601)
- Frontmatter strings with special characters: quote them (`"[[Projects/x]]"`)
- Tags: lowercase, hyphenated (`data-engineering`, not `DataEngineering`)
- File slugs: lowercase, hyphenated, max 50 chars
- Log entries: most recent at the bottom (chronological order)
- Never use tabs in YAML — use 2-space indentation
- Wikilinks in frontmatter must be quoted: `project: "[[Projects/slug]]"`

---

## What NOT To Do

- Never delete a task file — change status to `done` and archive it
- Never overwrite log entries — only append
- Never change an `id` or `created` date after creation
- Never create tasks outside of `Tasks/`, `Projects/`, or `Inbox.md`
- Never modify files in `Archive/` unless explicitly asked
- Never modify `.obsidian/` configuration
- If unsure about a field value, leave it empty rather than guessing
