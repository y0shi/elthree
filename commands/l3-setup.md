# /l3-setup — Vault Initialization Wizard

You are running the L3 vault setup wizard. Follow these steps in order. Be conversational but efficient — this is a guided setup, not a Q&A session.

---

## Step 1: Check for Existing Config

Read `~/.l3/config.md`.

**If it exists**, attempt to read the YAML frontmatter. If the file is not valid YAML, say: "Your config file at `~/.l3/config.md` appears corrupted. Do you want to reinitialize (this will overwrite it) or cancel?" Otherwise, show the current settings:
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

Accept whatever the user provides. If the path contains `~`, expand it to the absolute home directory path before using it in any file operations or git commands (e.g., `~/vault` → `/Users/josh/vault`).
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
git branch -M main
git add -A
git commit -m "Initial vault setup"
```

If a git remote was provided in Step 2:
```bash
git remote add origin <url>
git push -u origin main
```

If `git init`, `git add`, or `git commit` fail, report the error and continue — the vault files are still usable without git.

If `git push` fails: report the error, tell the user the vault is local-only for now, and note they can retry with `git push -u origin main` from the vault directory after fixing the remote URL.

---

## Step 5: Write Config File

Check if `~/.l3` exists. If it exists and is a file (not a directory), stop and say: "Cannot create config directory: `~/.l3` exists as a file. Please remove or rename it, then re-run `/l3-setup`." Otherwise, create the directory if it doesn't exist. Write `~/.l3/config.md`:

```markdown
---
vault_path: <vault_path>
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
[ ] Add filesystem MCP server to your Claude Desktop config file:
      macOS: ~/Library/Application Support/Claude/claude_desktop_config.json
      Windows: %APPDATA%/Claude/claude_desktop_config.json
      Linux: ~/.config/Claude/claude_desktop_config.json
    {
      "mcpServers": {
        "vault": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "<vault_path>"]
        }
      }
    }

Optional (for Claude Mobile — only applicable if you pushed your vault to GitHub above):
[ ] Go to claude.ai → Settings → Connectors → Add GitHub MCP connector
    Authenticate with a GitHub PAT that has repo read/write access
    This lets Claude Mobile read and write your vault via GitHub commits
```

Then say: "You're all set. Run `/l3` anytime to manage your tasks."
