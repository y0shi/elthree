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
