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
