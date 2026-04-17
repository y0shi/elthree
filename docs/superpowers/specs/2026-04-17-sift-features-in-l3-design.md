# Sift Features in L3 — Design Spec

**Date:** 2026-04-17
**Status:** Approved

---

## Overview

Incorporate energy matching, cognitive load awareness, task prioritization, and quick wins from the [sift](https://github.com/y0shi/sift) skill into elthree's `l3-vault` skill and `/l3` command — keeping everything file-based (no Notion, no external MCPs).

**Approach:** Energy-matched recommendations are computed ephemerally at query time and never written back to task files. The existing `priority` label is preserved as a manual field; scoring is layered on top for the briefing and quick wins queries.

---

## Section 1: Task Schema Changes

Two new optional frontmatter fields added to task files:

```yaml
cognitive_load: 3    # 1–6 (Bloom's taxonomy)
effort: 5            # 1–10 (1=minutes, 10=days)
```

### Auto-Classification on Task Creation

Both fields are inferred from the task title and objective when creating a new task. Both can be overridden via natural language or explicit update.

**Cognitive load classification rules:**

| Level | Label | Keywords / signals |
|-------|-------|-------------------|
| 1 | Remember | send, reply, check, look up, find, confirm |
| 2 | Understand | read, review, summarize, skim, watch |
| 3 | Apply | update, configure, run, calculate, execute |
| 4 | Analyze | debug, compare, investigate, trace, audit |
| 5 | Evaluate | decide, approve, choose, assess, prioritize |
| 6 | Create | design, build, write, architect, implement |

**Effort estimation rules (1–10):**

| Range | Meaning | Signals |
|-------|---------|---------|
| 1–2 | Minutes | quick reply, one-liner, checkbox |
| 3–4 | Under an hour | small update, single config change |
| 5–6 | A few hours | meaningful feature, thorough review |
| 7–8 | Half to full day | significant implementation |
| 9–10 | Multi-day | large feature, complex investigation |

### Backward Compatibility

Old tasks without `cognitive_load` or `effort` still work. They are excluded from energy-matched scoring and fall back to priority-label sort (critical > high > medium > low), shown after scored tasks in recommendations.

---

## Section 2: Energy Log

A new `Energy/` directory inside the vault. One file per day:

```
<vault_path>/Energy/YYYY-MM-DD.md
```

**File format:**
```markdown
---
date: YYYY-MM-DD
level: 3
---
```

**Reading:** When energy is needed, check for `Energy/YYYY-MM-DD.md` (today's date). If it exists, use `level` silently. If not, ask the user.

**Writing:** When the user states their energy level (via briefing prompt or explicitly — e.g., "my energy is 4"), write or overwrite today's file. Last stated level wins — one entry per day.

**Energy-to-cognitive-load compatibility:**

| Energy | Best cognitive load range |
|--------|--------------------------|
| 1 (foggy/exhausted) | 1–2 |
| 2 (low) | 1–3 |
| 3 (steady) | 2–4 |
| 4 (focused) | 3–5 |
| 5 (peak) | 4–6 |

---

## Section 3: Energy-Aware Briefing Flow

The no-arg `/l3` briefing gains a new first step.

### Updated Flow

1. **Check energy** — read `Energy/YYYY-MM-DD.md`
   - If found: use level silently, proceed
   - If not found: ask `"What's your energy level today? (1–5)"` — wait for response, write the file, then proceed

2. **Overdue** — unchanged, always shown regardless of energy

3. **DO NOW** (replaces "Top priorities"):
   - Score all open tasks that have both `cognitive_load` and `effort` set
   - Show max 3, labeled **DO NOW**
   - Tasks missing either field fall back to priority-label sort, appended as "Top priorities (unscored):" if fewer than 3 scored tasks exist

4. **Scheduled today** — unchanged

### Priority Score Formula

```
score = (urgency × 0.35) + (goal_alignment × 0.30) + (effort_inverse × 0.20) + (energy_match × 0.15)
```

**Component definitions:**

- **urgency** — deadline proximity, normalized 0–1:
  - Overdue: 1.0
  - Due today: 0.9
  - Due in 1 day: 0.8
  - Due in 2 days: 0.7
  - Due in 3–6 days: 0.5
  - Due in 7+ days: 0.1
  - No due date: 0.3

- **goal_alignment** — derived from the linked project file's `priority` field (read `Projects/<slug>.md` frontmatter):
  - critical: 1.0 / high: 0.75 / medium: 0.5 / low: 0.25
  - No project, unreadable project file, or missing priority field: 0.5

- **effort_inverse** — `1 - (effort / 10)` (lower effort scores higher)

- **energy_match** — how well current energy matches the task's cognitive demand:
  - `optimal_load = energy_level` (1-indexed, same scale)
  - `distance = abs(optimal_load - cognitive_load)`
  - `energy_match = 1 - (distance / 5)` — clamped to [0, 1]

### Output Format

```
Good morning. Here's your task status:

Overdue (2):
  TASK-003 — Renew SSL cert (due 2026-04-10, 7 days ago)
  TASK-011 — Submit expense report (due 2026-04-15, 2 days ago)

DO NOW (energy: 3):
  1. TASK-019 — Update deploy config [effort: 2, load: 3, score: 0.87]
  2. TASK-042 — Review PR from Alex [effort: 3, load: 2, score: 0.74]
  3. TASK-007 — Reply to vendor email [effort: 1, load: 1, score: 0.68]

Scheduled today (1):
  TASK-055 — Team sync prep
```

---

## Section 4: Quick Wins Mode

A new intent in `/l3`.

**Trigger phrases:** "quick wins", "easy tasks", "what can I knock out fast", "low effort tasks"

**Behavior:**
1. Check energy (same as briefing — read today's file, or ask if missing)
2. Filter: `effort ≤ 3` AND `status != done` AND not blocked (`blocked_by` empty or all blockers done)
3. Score using the same formula as the briefing
4. Show max 5, flat list labeled **QUICK WINS**

**Output format:**
```
Quick wins (3 tasks):
  TASK-007 — Reply to vendor email [effort: 1, load: 1]
  TASK-019 — Update deploy config [effort: 2, load: 3]
  TASK-031 — Review PR from Alex [effort: 3, load: 2]
```

If no tasks qualify: `"No quick wins found. All open tasks have effort > 3."`

---

## Files Changed

| File | Change |
|------|--------|
| `skills/l3-vault/SKILL.md` | Add `cognitive_load` and `effort` to frontmatter schema; add auto-classification rules; add energy log file format and read/write rules; add `quick_wins` query pattern; update briefing scoring logic |
| `commands/l3.md` | Add energy check step to no-arg briefing; add `quick_wins` intent to routing table; update output format for DO NOW section |
| `<vault_path>/Energy/` | New directory — created on first energy log write (not during `/l3-setup`) |

---

## Out of Scope

- Energy log notes/context field (morning/afternoon/evening) — not needed, level alone is sufficient
- Momentum/streak tracking — quick wins mode covers the use case without persistence overhead
- Crisis/planning/low-energy context mode adjustments to the formula — can be added later if needed
- Auto-creating the `Energy/` directory during `/l3-setup` — it creates itself on first use
