# Sift Features in L3 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add energy matching, cognitive load awareness, ephemeral task scoring, and quick wins mode to the elthree skill — entirely file-based, no external dependencies.

**Architecture:** Two markdown files change (`skills/l3-vault/SKILL.md` and `commands/l3.md`). No new code, no new commands beyond the `quick_wins` intent. Energy state lives in `<vault_path>/Energy/YYYY-MM-DD.md`. Scoring is computed in-session and never written back to task files.

**Tech Stack:** Markdown, YAML frontmatter, git. No code files involved — these are Claude skill instruction documents.

**Spec:** `docs/superpowers/specs/2026-04-17-sift-features-in-l3-design.md`

---

## File Map

| File | Change type | Responsibility |
|------|-------------|---------------|
| `skills/l3-vault/SKILL.md` | Modify | Task schema, energy log format, auto-classification rules, quick wins query pattern |
| `commands/l3.md` | Modify | Energy check step, DO NOW briefing output, quick wins intent routing |

---

### Task 1: Add cognitive_load and effort to task frontmatter schema

**Files:**
- Modify: `skills/l3-vault/SKILL.md` (frontmatter block, lines 61–77)

- [ ] **Step 1: Add the two new fields to the YAML frontmatter template**

In `skills/l3-vault/SKILL.md`, find the frontmatter block that ends with:
```yaml
blocked_by: []          # List of TASK-NNN IDs blocking this task
source: ""              # Where the task originated (e.g., "RTC meeting 2026-04-14")
```

Replace it with:
```yaml
blocked_by: []          # List of TASK-NNN IDs blocking this task
source: ""              # Where the task originated (e.g., "RTC meeting 2026-04-14")
cognitive_load:         # 1–6 (Bloom's: 1=remember, 2=understand, 3=apply, 4=analyze, 5=evaluate, 6=create)
effort:                 # 1–10 (1=minutes, 10=days of work)
```

- [ ] **Step 2: Add auto-classification rules after the frontmatter block**

Find the line:
```
**Body Structure:**
```

Insert the following section immediately before it:

```markdown
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

```

- [ ] **Step 3: Verify the edit looks correct**

Read lines 60–120 of `skills/l3-vault/SKILL.md` and confirm:
- Both new fields appear in the frontmatter template with correct inline comments
- The auto-classification section appears between the frontmatter block and `**Body Structure:**`
- No existing content was removed or corrupted

- [ ] **Step 4: Commit**

```bash
git add skills/l3-vault/SKILL.md
git commit -m "feat(skill): add cognitive_load and effort fields to task schema"
```

---

### Task 2: Add Energy Log to vault layout and SKILL.md

**Files:**
- Modify: `skills/l3-vault/SKILL.md` (vault directory layout + new Energy Log section)

- [ ] **Step 1: Add Energy/ to the vault directory layout**

Find the vault directory layout block in `skills/l3-vault/SKILL.md`:
```
<vault_path>/
├── Tasks/       ← task files live here
├── Projects/    ← project notes with inline tasks
├── Areas/       ← ongoing responsibility area notes
├── Archive/     ← completed tasks, organized by quarter
├── Reviews/     ← weekly review snapshots
└── Inbox.md     ← quick-capture scratchpad
```

Replace it with:
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

- [ ] **Step 2: Add Energy Log section to SKILL.md**

Find the line:
```
## Query Patterns
```

Insert the following section immediately before it:

```markdown
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

```

- [ ] **Step 3: Verify the edit looks correct**

Read the vault directory layout and the new Energy Log section in `skills/l3-vault/SKILL.md` and confirm:
- `Energy/` appears in the vault layout with the correct label
- The Energy Log section appears immediately before `## Query Patterns`
- File format, read/write rules, and compatibility table are all present
- No existing content was removed or corrupted

- [ ] **Step 4: Commit**

```bash
git add skills/l3-vault/SKILL.md
git commit -m "feat(skill): add Energy Log directory and read/write rules"
```

---

### Task 3: Add quick wins query pattern and scoring logic to SKILL.md

**Files:**
- Modify: `skills/l3-vault/SKILL.md` (Query Patterns table + new Recommendation Scoring section)

- [ ] **Step 1: Add quick wins row to Query Patterns table**

Find the Query Patterns table:
```
| "What's scheduled today?" | Find files where `scheduled == today` |
```

Add a new row after it:
```
| "Quick wins", "easy tasks", "what can I knock out fast", "low effort tasks" | Quick Wins operation — see below |
```

- [ ] **Step 2: Add Recommendation Scoring and Quick Wins sections after Query Patterns**

Find the line:
```
---

## Inline Tasks
```

Insert the following before it:

```markdown
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
3. Score filtered tasks using Recommendation Scoring formula
4. Show max 5, sorted by score descending

**Output format:**
```
Quick wins (N tasks, energy: E):
  TASK-007 — Reply to vendor email [effort: 1, load: 1]
  TASK-019 — Update deploy config [effort: 2, load: 3]
  TASK-031 — Review PR from Alex [effort: 3, load: 2]
```

If no tasks qualify: `"No quick wins found. All open tasks have effort > 3."`

---

```

- [ ] **Step 3: Verify the edit looks correct**

Read the Query Patterns table and the new sections in `skills/l3-vault/SKILL.md` and confirm:
- Quick wins row appears in the Query Patterns table
- Recommendation Scoring section appears before Quick Wins, and both appear before `## Inline Tasks`
- Formula, all five component definitions, and fallback rule are present
- Quick wins filter conditions (effort ≤ 3, not done, not blocked) are correct

- [ ] **Step 4: Commit**

```bash
git add skills/l3-vault/SKILL.md
git commit -m "feat(skill): add recommendation scoring formula and quick wins operation"
```

---

### Task 4: Update /l3 no-arg briefing for energy-aware DO NOW

**Files:**
- Modify: `commands/l3.md` (Step 2 — no-argument case)

- [ ] **Step 1: Add energy check before the briefing in Step 2**

Find the beginning of Step 2 in `commands/l3.md`:
```
## Step 2: Handle No-Argument Case

If `/l3` was invoked with no arguments (just `/l3` with nothing after it), show a morning briefing:
```

Replace it with:
```markdown
## Step 2: Handle No-Argument Case

If `/l3` was invoked with no arguments (just `/l3` with nothing after it):

**2a. Check energy** — read `Energy/YYYY-MM-DD.md` (today's date, relative to vault path from config):
- If found: use `level` value silently
- If not found: ask `"What's your energy level today? (1–5)"` — wait for response, write `Energy/YYYY-MM-DD.md`, then continue

**2b. Show morning briefing:**
```

- [ ] **Step 2: Update the briefing output format to show DO NOW**

Find the briefing format block:
```
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
```

Replace it with:
```markdown
**Format:**
```
Good morning. Here's your task status:

Overdue (N):
  TASK-NNN — Title (due YYYY-MM-DD, N days ago)
  ...

Scheduled today (N):
  TASK-NNN — Title
  ...

DO NOW (energy: E):
  1. TASK-NNN — Title [effort: N, load: N, score: 0.NN]
  2. TASK-NNN — Title [effort: N, load: N, score: 0.NN]
  3. TASK-NNN — Title [effort: N, load: N, score: 0.NN]

Top priorities (unscored):
  4. TASK-NNN — Title [critical]
```
```

- [ ] **Step 3: Update the prose below the format block**

Find:
```
If nothing is overdue, say "Nothing overdue." If nothing is scheduled today, say "Nothing scheduled today." Show top 3 open tasks by priority (critical > high > medium > low). If there are fewer than 3 open tasks total, show all of them.
```

Replace it with:
```
If nothing is overdue, say "Nothing overdue." If nothing is scheduled today, say "Nothing scheduled today."

For DO NOW: score all open tasks that have both `cognitive_load` and `effort` set using the Recommendation Scoring formula from the l3-vault skill. Show the top 3 by score. If fewer than 3 tasks can be scored, fill remaining slots from unscored tasks sorted by priority label (critical > high > medium > low), shown under "Top priorities (unscored):". If no tasks can be scored at all, show "Top priorities (open):" with the priority-label sort. If there are fewer than 3 open tasks total, show all of them.
```

- [ ] **Step 4: Verify the edit looks correct**

Read Step 2 of `commands/l3.md` and confirm:
- Energy check (2a) appears before the briefing output (2b)
- DO NOW section shows `[effort: N, load: N, score: 0.NN]` format
- "Top priorities (unscored):" appears as the fallback for unscored tasks
- Original overdue and scheduled-today behavior is unchanged

- [ ] **Step 5: Commit**

```bash
git add commands/l3.md
git commit -m "feat(command): add energy check and DO NOW scoring to no-arg briefing"
```

---

### Task 5: Add quick wins intent to /l3 routing

**Files:**
- Modify: `commands/l3.md` (Step 3 intent table + Step 4 routing table)

- [ ] **Step 1: Add quick_wins to the intent classification table**

Find in `commands/l3.md`:
```
| `bulk` | "bulk update: set all X tasks to Y", "mark all bedrock-tagged tasks as done" |
```

Add a new row after it:
```
| `quick_wins` | "quick wins", "easy tasks", "what can I knock out fast", "low effort tasks" |
```

- [ ] **Step 2: Add quick_wins to the routing table**

Find:
```
| `bulk` | Update applied to each matching task (confirm first — see Step 5) |
```

Add a new row after it:
```
| `quick_wins` | Quick Wins operation from l3-vault skill |
```

- [ ] **Step 3: Verify the edit looks correct**

Read Steps 3 and 4 of `commands/l3.md` and confirm:
- `quick_wins` appears in both tables
- Example phrases match the spec ("quick wins", "easy tasks", "what can I knock out fast", "low effort tasks")
- Routing points to "Quick Wins operation from l3-vault skill"
- No other rows were modified

- [ ] **Step 4: Commit**

```bash
git add commands/l3.md
git commit -m "feat(command): add quick_wins intent routing"
```

---

## Verification Checklist

After all tasks complete, spot-check the full integration:

- [ ] Create a new task with `/l3 create a task: review the Bedrock compliance report` — confirm Claude sets `cognitive_load: 2` (understand/review) and `effort: 3–4` in the created file
- [ ] Run `/l3` with no args and no energy file for today — confirm it asks for energy level before showing the briefing
- [ ] Run `/l3` again immediately after — confirm it does NOT ask for energy again (reads today's file)
- [ ] Run `/l3 quick wins` — confirm it shows only tasks with `effort ≤ 3` and not blocked ones
- [ ] Update a task's cognitive_load via `/l3 update TASK-001 set cognitive_load 5` — confirm the frontmatter changes and no log entry is added (field-only update, no status change)
