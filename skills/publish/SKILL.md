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

If the push fails for any reason, ask the user: "Bitbucket push failed: [error message]. Would you like to (1) correct the remote URL in `.publish-config.json` and retry, or (2) skip Bitbucket and continue?" Act on their choice. Do NOT stop without asking.

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

**Find the space:**
Use `mcp__claude_ai_Atlassian__getConfluenceSpaces` to list spaces. Match the first segment (e.g. `IT Development`) against the `name` field. Extract the `spaceKey`.

If not found: report "Confluence space not found: [segment]" and skip this step.

**Navigate sub-pages:**
For the **first** remaining segment (e.g. `Finance`), search without an ancestor filter:

CQL: `title = "[segment]" AND space = "[spaceKey]"`

For each **subsequent** segment (e.g. `Aldipress`), add the ancestor from the previous result:

CQL: `title = "[segment]" AND space = "[spaceKey]" AND ancestor = [previous-page-id]`

Use `mcp__claude_ai_Atlassian__searchConfluenceUsingCql` for each query. Take the first result's `id` as the parent for the next segment.

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

- **If found:** First retrieve the current version: call `mcp__claude_ai_Atlassian__getConfluencePage` with the page ID and extract `version.number`. Increment it by 1. Then call `mcp__claude_ai_Atlassian__updateConfluencePage` passing the incremented version number and the markdown content as the body.
- **If not found:** Create the page using `mcp__claude_ai_Atlassian__createConfluencePage` with the project parent as parent.

Track a counter: pages created + pages updated.

### 5e. Attach non-markdown files to the project parent page

For each attachment file in `docs/`:

First check if an attachment with the same filename already exists:
- Call `mcp__claude_ai_Atlassian__fetchAtlassian` GET `/wiki/rest/api/content/[projectPageId]/child/attachment?filename=[filename]`
- If a result is returned, note the existing attachment `id`.

Then upload:
- **If attachment exists (replace):** Use `mcp__claude_ai_Atlassian__fetchAtlassian` PUT `/wiki/rest/api/content/[projectPageId]/child/attachment/[attachment-id]/data` with headers `{"X-Atlassian-Token": "no-check"}` and the file content as multipart/form-data.
- **If attachment does not exist (new):** Use `mcp__claude_ai_Atlassian__fetchAtlassian` POST `/wiki/rest/api/content/[projectPageId]/child/attachment` with headers `{"X-Atlassian-Token": "no-check"}` and the file content as multipart/form-data.

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
    [full path, e.g. IT Development > Finance > Aldipress > [project name]]
  (or): ✗ Confluence — not configured for this project
  (or): ✗ Confluence — [error reason]
──────────────────────────────────────────
```
