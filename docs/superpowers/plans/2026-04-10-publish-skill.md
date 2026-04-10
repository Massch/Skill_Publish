# Publish Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a `/publish` Claude Code skill that deliberately pushes a project to company Bitbucket and/or publishes its `docs/` folder to Confluence.

**Architecture:** A single SKILL.md file contains all the instructions Claude follows when `/publish` is invoked. Per-project config is stored in `.publish-config.json` at each project root. The skill lives in a dedicated `Skill_Publish` project and is registered as a Claude Code plugin via settings.json.

**Tech Stack:** Claude Code skill (Markdown), Atlassian MCP (`mcp__claude_ai_Atlassian__*`), Git (bash), JSON config file.

---

## File Structure

```
C:\Development\Skill_Publish\
├── skills\
│   └── publish\
│       └── SKILL.md          ← the skill Claude reads when /publish is invoked
└── docs\
    └── superpowers\
        ├── specs\
        │   └── 2026-04-10-publish-workflow-design.md   (moved from C:\Development\docs\)
        └── plans\
            └── 2026-04-10-publish-skill.md             (this file, moved from C:\Development\docs\)
```

Per project that uses the skill (e.g. `C:\Development\API-oliver\`):
```
.publish-config.json    ← created on first /publish run, committed to git
docs\                   ← user places .md files and other files here
```

---

## Task 1: Initialize the Skill_Publish project

**Files:**
- Create: `C:\Development\Skill_Publish\` (directory)
- Create: `C:\Development\Skill_Publish\skills\publish\` (directory)
- Create: `C:\Development\Skill_Publish\docs\superpowers\specs\` (directory)
- Create: `C:\Development\Skill_Publish\docs\superpowers\plans\` (directory)

- [ ] **Step 1: Create the project directory and folder structure**

```bash
mkdir -p /c/Development/Skill_Publish/skills/publish
mkdir -p /c/Development/Skill_Publish/docs/superpowers/specs
mkdir -p /c/Development/Skill_Publish/docs/superpowers/plans
```

- [ ] **Step 2: Initialize git**

```bash
cd /c/Development/Skill_Publish
git init
```

Expected: `Initialized empty Git repository in C:/Development/Skill_Publish/.git/`

- [ ] **Step 3: Move the design spec and this plan into the project**

```bash
mv /c/Development/docs/superpowers/specs/2026-04-10-publish-workflow-design.md \
   /c/Development/Skill_Publish/docs/superpowers/specs/

mv /c/Development/docs/superpowers/plans/2026-04-10-publish-skill.md \
   /c/Development/Skill_Publish/docs/superpowers/plans/
```

- [ ] **Step 4: Create a placeholder SKILL.md so git has something to commit**

Write `C:\Development\Skill_Publish\skills\publish\SKILL.md`:
```markdown
# Publish Skill — placeholder
```

- [ ] **Step 5: Initial commit**

```bash
cd /c/Development/Skill_Publish
git add .
git commit -m "chore: initialize Skill_Publish project"
```

Expected output includes: `1 file changed` or similar, no errors.

---

## Task 2: Write the complete SKILL.md

**Files:**
- Modify: `C:\Development\Skill_Publish\skills\publish\SKILL.md`

This is the core deliverable. Replace the placeholder with the full skill content below.

- [ ] **Step 1: Write SKILL.md with the complete content**

Write `C:\Development\Skill_Publish\skills\publish\SKILL.md` with this exact content:

```markdown
---
name: publish
description: Publish a finished project to company Bitbucket and/or Confluence documentation. Run /publish from the project folder when you are satisfied with your work.
---

# Publish Workflow

Deliberately publish the current project to company destinations (Bitbucket and/or Confluence). Both are optional and per-project. This skill never fires automatically — it only runs when you invoke it.

## Before Starting

- The Atlassian MCP must be authenticated (the `mcp__claude_ai_Atlassian__*` tools must work).
- The project must have a local git repo with `origin` pointing to GitHub.
- Optionally: a `docs/` folder in the project root containing files to publish to Confluence.

## Checklist

- [ ] Load or create `.publish-config.json`
- [ ] Push to GitHub (always)
- [ ] Push to Bitbucket (if enabled)
- [ ] Publish docs to Confluence (if enabled)
- [ ] Print summary

---

## Step 1: Identify the project

- Project root = current working directory (where you invoked `/publish`)
- Project name = the folder name (e.g. `API-oliver` from `C:\Development\API-oliver`)
- Current branch: run `git branch --show-current`

## Step 2: Load or create config

Use the Read tool to check if `.publish-config.json` exists at the project root.

### If it does NOT exist (first run for this project):

Ask the user these questions **one at a time**:

1. "Should this project push to your company Bitbucket? (yes/no)"
   - If yes: "What is the Bitbucket remote URL? (e.g. https://bitbucket.org/company/api-oliver.git)"
2. "Should this project publish its docs/ folder to Confluence? (yes/no)"
   - If yes: "What is the full Confluence parent path? Enter all levels separated by ' > ' — e.g. IT Development > Finance > Aldipress"

Write `.publish-config.json` at the project root:

```json
{
  "bitbucket": {
    "enabled": true,
    "remoteUrl": "https://bitbucket.org/company/project.git"
  },
  "confluence": {
    "enabled": true,
    "path": "IT Development > Finance > Aldipress"
  }
}
```

Replace `true`/`false` and the values with what the user answered. If a destination is disabled, its other fields don't matter (set remoteUrl/path to null).

Commit the config:
```bash
git add .publish-config.json
git commit -m "chore: add publish config"
```

### If it already exists:

Read it with the Read tool. Parse the JSON. Announce: "Config loaded. Publishing **[project name]**..."

---

## Step 3: Push to GitHub

```bash
git push origin [current-branch]
```

Record result (success or error message) for the summary.

---

## Step 4: Push to Bitbucket

Only run this step if `bitbucket.enabled` is `true`.

Check existing remotes:
```bash
git remote -v
```

If no remote named `bitbucket` appears:
```bash
git remote add bitbucket [bitbucket.remoteUrl]
```

Push:
```bash
git push bitbucket [current-branch]
```

If this fails due to a wrong URL, ask the user: "The Bitbucket push failed — would you like to correct the remote URL in `.publish-config.json`?" If yes, update the file, re-add the remote, and retry. If it fails for any other reason (auth, network), note the error for the summary and continue. Do NOT stop.

---

## Step 5: Publish to Confluence

Only run this step if `confluence.enabled` is `true`.

### 5a. Check docs folder

Use Glob (`docs/**/*`) to list all files in `docs/`. If no files are found:
- Note "docs/ folder missing or empty" in summary and skip this step entirely.

Classify files:
- **Markdown files**: files ending in `.md`
- **Attachment files**: everything else (PDF, DOCX, XLSX, PNG, etc.)

### 5b. Resolve the Confluence path

Parse `confluence.path` by splitting on ` > `.
Example: `"IT Development > Finance > Aldipress"` → `["IT Development", "Finance", "Aldipress"]`

**Get cloud ID:**
Use `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` — look for the Confluence product in the results and extract the `cloudId`.

**Find the space:**
Use `mcp__claude_ai_Atlassian__getConfluenceSpaces` to list spaces. Match the first segment (e.g. `IT Development`) against the `name` field. Extract the `spaceKey`.

If not found: report "Confluence space not found: [segment]" and skip this step.

**Navigate sub-pages:**
For each remaining segment (e.g. `Finance`, then `Aldipress`), search for it under the previous parent:

CQL: `title = "[segment]" AND space = "[spaceKey]" AND ancestor = [previous-page-id]`

Use `mcp__claude_ai_Atlassian__searchConfluenceUsingCql` with that query. Take the first result's `id` as the parent for the next segment.

If not found at any level: report "Confluence path not found at: [missing segment]" and skip.

Store the final resolved page ID as `parentPageId`.

### 5c. Find or create the project parent page

Search for the project page:
CQL: `title = "[project-name]" AND space = "[spaceKey]" AND ancestor = [parentPageId]`

- **If found:** Store its `id` as `projectPageId`.
- **If not found:** Create it:
  Use `mcp__claude_ai_Atlassian__createConfluencePage` with:
  - parent ID = `parentPageId`
  - space key = `spaceKey`
  - title = project name (e.g. `API-oliver`)
  - body = `Documentation for [project name].`

  Store the new page's `id` as `projectPageId`.

### 5d. Publish each markdown file

For each `.md` file in `docs/`:

**Derive the page title:**
- Remove `.md` extension
- Replace `-` and `_` with spaces
- Apply title case
- Examples: `technical-spec.md` → `Technical Spec`, `meeting_notes.md` → `Meeting Notes`

**Read the file content** using the Read tool.

**Check if page exists:**
CQL: `title = "[page-title]" AND space = "[spaceKey]" AND ancestor = [projectPageId]`

- **If found:** Update the page using `mcp__claude_ai_Atlassian__updateConfluencePage` — pass the markdown content as the body.
- **If not found:** Create the page using `mcp__claude_ai_Atlassian__createConfluencePage` — parent = `projectPageId`, title = derived title, body = markdown content.

Track a counter: pages created + pages updated.

### 5e. Attach non-markdown files to the project parent page

For each attachment file in `docs/`:

Use `mcp__claude_ai_Atlassian__fetchAtlassian` with:
- Method: `POST`
- Path: `/wiki/rest/api/content/[projectPageId]/child/attachment`
- Headers: `{"X-Atlassian-Token": "no-check"}`
- Body: the file content as multipart/form-data

If the upload fails (binary files may not be supported by the MCP tool), note in summary:
`"⚠ [filename] — could not attach, manual upload required"`

---

## Step 6: Print the summary

Always print this at the end, regardless of individual failures:

```
──────────────────────────────────────────
Publish complete: [project name]
──────────────────────────────────────────
✓ GitHub (origin)  — branch: [branch]
✓ Bitbucket        — branch: [branch]
  (or): ✗ Bitbucket — not configured for this project
  (or): ✗ Bitbucket — error: [error message]

✓ Confluence — [N] page(s) created, [N] updated, [N] file(s) attached
    [full path, e.g. IT Development > Finance > Aldipress > API-oliver]
  (or): — not configured for this project
  (or): ✗ Confluence — [error reason]
──────────────────────────────────────────
```
```

- [ ] **Step 2: Commit the completed SKILL.md**

```bash
cd /c/Development/Skill_Publish
git add skills/publish/SKILL.md
git commit -m "feat: write publish skill"
```

---

## Task 3: Register the skill as a Claude Code plugin

This makes `/publish` available in all Claude Code sessions.

- [ ] **Step 1: Invoke the update-config skill**

Use the `update-config` skill (invoke via the Skill tool with `skill: "update-config"`).

Tell it: "I want to register a local plugin at `C:\Development\Skill_Publish` so its skills are available as slash commands."

The update-config skill will edit `C:\Users\masschelein\.claude\settings.json` with the correct plugin entry. Follow its instructions.

- [ ] **Step 2: Verify the skill appears**

Restart Claude Code (or reload the session). Run `/publish` in any project folder. You should see the skill load, not an error.

Expected: Claude begins asking the first-run config questions.

- [ ] **Step 3: Commit any settings notes**

If the update-config skill created any files inside `Skill_Publish`, commit them:
```bash
cd /c/Development/Skill_Publish
git status
# If anything changed:
git add .
git commit -m "chore: register plugin in settings"
```

---

## Task 4: Test — first-run flow (GitHub push only)

**Goal:** Verify the skill correctly runs the first-time setup and pushes to GitHub.

This test uses the existing `API-oliver` project since it already has a git repo and GitHub remote.

- [ ] **Step 1: Confirm API-oliver has no publish config yet**

```bash
ls /c/Development/API-oliver/.publish-config.json
```

Expected: `No such file or directory`

If the file exists: delete it for a clean test:
```bash
rm /c/Development/API-oliver/.publish-config.json
```

- [ ] **Step 2: Change working directory to API-oliver**

In Claude Code, set the working directory to `C:\Development\API-oliver` (or open it as the project).

- [ ] **Step 3: Invoke the skill**

Type `/publish`. The skill should:
1. Detect no config exists
2. Ask: "Should this project push to your company Bitbucket?"
3. Answer: **no**
4. Ask: "Should this project publish docs to Confluence?"
5. Answer: **no**

- [ ] **Step 4: Verify the config file was created**

```bash
cat /c/Development/API-oliver/.publish-config.json
```

Expected:
```json
{
  "bitbucket": {
    "enabled": false,
    "remoteUrl": null
  },
  "confluence": {
    "enabled": false,
    "path": null
  }
}
```

- [ ] **Step 5: Verify the config was committed**

```bash
cd /c/Development/API-oliver
git log --oneline -3
```

Expected: most recent commit message contains "publish config".

- [ ] **Step 6: Verify GitHub push happened**

```bash
cd /c/Development/API-oliver
git log --oneline origin/[branch] -1
```

Expected: the commit hash matches the local HEAD.

- [ ] **Step 7: Verify summary output**

The skill should have printed something like:
```
✓ GitHub (origin)  — branch: main
  Bitbucket        — not configured for this project
  Confluence       — not configured for this project
```

---

## Task 5: Test — Bitbucket push

**Goal:** Verify the Bitbucket remote is added and the push works.

Note: You need a real Bitbucket repo for this test. If you don't have one yet, create an empty repo in your company Bitbucket workspace first.

- [ ] **Step 1: Delete the existing publish config from Task 4**

```bash
rm /c/Development/API-oliver/.publish-config.json
cd /c/Development/API-oliver
git add .publish-config.json
git commit -m "chore: remove publish config for re-test"
```

- [ ] **Step 2: Invoke `/publish` again and answer yes to Bitbucket**

When asked "Should this project push to your company Bitbucket?": answer **yes**
When asked for the URL: enter the real Bitbucket URL.
When asked about Confluence: answer **no**.

- [ ] **Step 3: Verify the bitbucket remote was added**

```bash
cd /c/Development/API-oliver
git remote -v
```

Expected: a `bitbucket` entry appears with the correct URL.

- [ ] **Step 4: Verify the Bitbucket push succeeded**

Check in the Bitbucket web UI that the branch and commits appear.

- [ ] **Step 5: Run `/publish` a second time (subsequent run)**

This time the config already exists. The skill should NOT ask any questions — it should load silently and push directly.

Expected in summary: both GitHub and Bitbucket show ✓.

---

## Task 6: Test — Confluence publish

**Goal:** Verify docs are published to Confluence correctly.

- [ ] **Step 1: Create a test docs folder in API-oliver**

```bash
mkdir -p /c/Development/API-oliver/docs
```

- [ ] **Step 2: Create two test markdown files**

Write `C:\Development\API-oliver\docs\overview.md`:
```markdown
# API Oliver Overview

This is the overview document for API Oliver.
Test content for Confluence publishing.
```

Write `C:\Development\API-oliver\docs\technical-spec.md`:
```markdown
# Technical Spec

## Endpoints

- GET /status
- POST /data
```

- [ ] **Step 3: Update the publish config to enable Confluence**

Read `.publish-config.json`, change `confluence.enabled` to `true` and set the correct path.

Ask the user: "What is the correct Confluence parent path for API-oliver? (e.g. IT Development > Press)"

Update the file with their answer, then commit:
```bash
cd /c/Development/API-oliver
git add docs/ .publish-config.json
git commit -m "test: add docs and enable Confluence"
```

- [ ] **Step 4: Invoke `/publish`**

The skill should load config silently and proceed to Confluence publishing.

- [ ] **Step 5: Verify in Confluence**

Check the Confluence space:
- A page named `API-oliver` should appear under the configured path
- Under it: `Overview` and `Technical Spec` child pages

- [ ] **Step 6: Run `/publish` again**

Modify one of the markdown files slightly, then run `/publish`. Verify in Confluence that the page was **updated** (not duplicated).

---

## Task 7: Push Skill_Publish to GitHub

- [ ] **Step 1: Create a GitHub repository**

Go to github.com and create a new private repository named `Skill_Publish`.

- [ ] **Step 2: Add the remote and push**

```bash
cd /c/Development/Skill_Publish
git remote add origin https://github.com/[your-username]/Skill_Publish.git
git push -u origin main
```

Replace `[your-username]` with your actual GitHub username.

Expected: push succeeds, repo is visible on GitHub.
