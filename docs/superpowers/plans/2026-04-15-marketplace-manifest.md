# Marketplace Manifest Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `.claude-plugin/marketplace.json` and `.claude-plugin/plugin.json` so that `/plugin install github:y0shi/elthree` works.

**Architecture:** The Claude Code plugin system treats `github:user/repo` as a marketplace reference. The repo needs a `.claude-plugin/marketplace.json` that declares itself as a single-plugin marketplace with `source: "./"`, plus a `plugin.json` with standard plugin metadata. No code changes — two JSON files.

**Tech Stack:** JSON, git

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| Create | `.claude-plugin/marketplace.json` | Declares the repo as an installable marketplace; lists the elthree plugin with `source: "./"` |
| Create | `.claude-plugin/plugin.json` | Plugin metadata: name, version, author, keywords, license |

---

### Task 1: Create `.claude-plugin/plugin.json`

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Create the directory and write `plugin.json`**

```json
{
  "name": "elthree",
  "description": "Git-backed personal task management for Claude Code. Obsidian is the UI. Claude is the operator. Markdown files are the persistence layer.",
  "version": "1.0.0",
  "author": {
    "name": "Joshua Reddick",
    "url": "https://github.com/y0shi"
  },
  "homepage": "https://github.com/y0shi/elthree",
  "repository": "https://github.com/y0shi/elthree",
  "license": "MIT",
  "keywords": [
    "task-management",
    "productivity",
    "obsidian",
    "gtd",
    "personal-productivity",
    "vault",
    "git",
    "markdown"
  ]
}
```

Write to `.claude-plugin/plugin.json` (create `.claude-plugin/` directory first).

- [ ] **Step 2: Verify JSON is valid**

```bash
cat .claude-plugin/plugin.json | python3 -m json.tool
```

Expected: prints formatted JSON with no errors.

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "feat: add .claude-plugin/plugin.json for marketplace install support"
```

---

### Task 2: Create `.claude-plugin/marketplace.json`

**Files:**
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Write `marketplace.json`**

```json
{
  "name": "elthree",
  "description": "Git-backed personal task management for Claude Code. Obsidian is the UI. Claude is the operator.",
  "owner": {
    "name": "Joshua Reddick",
    "url": "https://github.com/y0shi"
  },
  "plugins": [
    {
      "name": "elthree",
      "description": "Git-backed personal task management for Claude Code. Obsidian is the UI. Claude is the operator. Markdown files are the persistence layer.",
      "version": "1.0.0",
      "source": "./",
      "author": {
        "name": "Joshua Reddick",
        "url": "https://github.com/y0shi"
      }
    }
  ]
}
```

Write to `.claude-plugin/marketplace.json`.

- [ ] **Step 2: Verify JSON is valid**

```bash
cat .claude-plugin/marketplace.json | python3 -m json.tool
```

Expected: prints formatted JSON with no errors.

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat: add .claude-plugin/marketplace.json — enables /plugin install github:y0shi/elthree"
```

---

### Task 3: Push and verify install

- [ ] **Step 1: Push to GitHub**

```bash
git push origin main
```

- [ ] **Step 2: Test the install**

In a Claude Code session (any project directory):

```
/plugin install github:y0shi/elthree
```

Expected: plugin installs without "Marketplace not found" error. Should see elthree's commands (`/l3`, `/l3-setup`) become available.

- [ ] **Step 3: Smoke-test the commands are available**

```
/l3-setup
```

Expected: setup wizard launches. If it does, the install worked end-to-end.
