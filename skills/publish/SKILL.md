---
name: publish
description: Publish a finished project to company Bitbucket and/or Confluence documentation. Run /publish from the project folder when you are satisfied with your work.
---

# Publish Workflow

Deliberately publish the current project to company destinations (Bitbucket and/or Confluence). Both are optional and per-project. This skill never fires automatically — it only runs when you invoke it.

## Before Starting

- The Atlassian MCP must be authenticated (the `mcp__claude_ai_Atlassian__*` tools must work — verify with `mcp__claude_ai_Atlassian__atlassianUserInfo`).
- The Bitbucket MCP must be configured (`bitbucket` server in `~/.claude.json` via `claude mcp add bitbucket --scope user`). If missing, offer to add it.
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
   - If yes: "What is the Bitbucket remote URL? (e.g. https://bitbucket.org/ampconnect/api-oliver.git)"
   - **Load saved credentials** (needed for authenticated API calls): Use the Read tool on `~/.claude/publish-credentials.json`.
     - If it exists, parse and use the saved `username` and `apiToken` silently — do NOT ask the user.
     - If it does not exist, credentials will be requested later if needed (e.g. repo creation).
   - **Check if the repository exists.** Parse workspace and slug from the URL. Run:
     - If credentials are available: `curl -s -o /dev/null -w "%{http_code}" https://api.bitbucket.org/2.0/repositories/{workspace}/{slug} -u "{username}:{apiToken}"`
     - If no credentials yet: `curl -s -o /dev/null -w "%{http_code}" https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}`
   - If the response is `404`: the repo does not exist. Ask: "The repository does not exist yet. Should I create it? (yes/no)"
     If yes:
     - **Ensure credentials are available:** If not already loaded, ask:
       - "What is your Bitbucket username?"
       - "What is your Bitbucket API token?"
       Save to `~/.claude/publish-credentials.json`: `{"username": "...", "apiToken": "..."}` — so future projects never need to ask.
     - **Ask about the Bitbucket project:** "Should this repo be created under the 'Masschelein' project? (yes/no, or type a different project name)"
       - If yes or a name is given: look up the project key:
         `curl -s "https://api.bitbucket.org/2.0/workspaces/{workspace}/projects/?q=name+%3D+%22{project-name}%22" -u "{username}:{apiToken}"`
         Extract the `key` field from the first result. If not found, warn the user and create without a project.
       - If no: create without project assignment.
     - **Create the repository:**
       With project: `curl -s -X POST https://api.bitbucket.org/2.0/repositories/{workspace}/{slug} -u "{username}:{apiToken}" -H "Content-Type: application/json" -d '{"scm":"git","is_private":true,"project":{"key":"{project-key}"}}'`
       Without project: `curl -s -X POST https://api.bitbucket.org/2.0/repositories/{workspace}/{slug} -u "{username}:{apiToken}" -H "Content-Type: application/json" -d '{"scm":"git","is_private":true}'`
     Confirm success (response should include `"scm": "git"`) before proceeding.
2. "Should this project publish its docs/ folder to Confluence? (yes/no)"
   - If yes: "What is the full Confluence parent path? Enter all levels separated by ' > ' — e.g. IT Development > Finance > Aldipress"

Write `.publish-config.json` at the project root:

```json
{
  "bitbucket": {
    "enabled": true,
    "remoteUrl": "https://bitbucket.org/ampconnect/project.git"
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
Use `mcp__claude_ai_Atlassian__getConfluenceSpaces` to list spaces. Match the first segment (e.g. `IT Development`) against the `name` field.

Capture **both** values from the matching space:
- `spaceId` — the numeric `id` field (e.g. `189005833`) — required for `createConfluencePage` calls
- `spaceKey` — the string `key` field (e.g. `DEV`) — used in CQL queries

If not found: report "Confluence space not found: [segment]" and skip this step.

**Navigate sub-pages:**
For the **first** remaining segment (e.g. `Finance`), search without an ancestor filter:

CQL: `title = "[segment]" AND space = "[spaceKey]"`

For each **subsequent** segment (e.g. `Aldipress`), add the ancestor from the previous result:

CQL: `title = "[segment]" AND space = "[spaceKey]" AND ancestor = [previous-page-id]`

Use `mcp__claude_ai_Atlassian__searchConfluenceUsingCql` for each query. Take the first result's `id` as the parent for the next segment.

**If a segment is not found:** do NOT skip — **create it** instead:
Use `mcp__claude_ai_Atlassian__createConfluencePage` with:
- `spaceId` = the numeric space ID captured above
- parent ID = the page ID resolved so far (use the space homepage ID for the first missing segment — find it by searching `ancestor = null AND space = "[spaceKey]" AND title = "[space-name]"` or omit ancestor to create at root)
- title = the missing segment name
- body = (minimal placeholder, e.g. `[segment name]`)

Continue navigating or creating until all segments in the path are resolved.

Store the final resolved page ID as `parentPageId`.

### 5c. Find or create the project parent page

Search for the project page:
CQL: `title = "[project-name]" AND space = "[spaceKey]" AND ancestor = [parentPageId]`

- **If found:** Store its `id` as `projectPageId`.
- **If not found:** Create it:
  Use `mcp__claude_ai_Atlassian__createConfluencePage` with:
  - parent ID = `parentPageId`
  - `spaceId` = the numeric space ID (NOT the string `spaceKey`)
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

> **Large file warning:** The Read tool has a limit of ~10,000 tokens per call (~300 lines / ~30 KB). If the file is large, read it in chunks using `offset` and `limit` parameters. If the full content would exceed ~30 KB, publish a **condensed summary page** instead — include section headings, tables, key decisions, and add a prominent note: *"Full implementation details are in the [GitHub repository](origin-url)."* Summaries are more useful to Confluence readers than raw implementation plans anyway.

**Check if page exists:**
CQL: `title = "[page-title]" AND space = "[spaceKey]" AND ancestor = [projectPageId]`

- **If found:** First retrieve the current version: call `mcp__claude_ai_Atlassian__getConfluencePage` with the page ID and extract `version.number`. Increment it by 1. Then call `mcp__claude_ai_Atlassian__updateConfluencePage` passing the incremented version number and the markdown content as the body.
- **If not found:** Create the page using `mcp__claude_ai_Atlassian__createConfluencePage` with the project parent as parent.

Track a counter: pages created + pages updated.

### 5e. Attach non-markdown files to the project parent page

> **Important:** `mcp__claude_ai_Atlassian__fetchAtlassian` only accepts Atlassian Resource Identifiers (ARIs) — it cannot be used for arbitrary REST API paths. Attachment upload via REST API is **not supported** through this MCP tool.

For each attachment file in `docs/`, note in the summary:
`"⚠ [filename] — manual upload required (attachment API not supported via MCP)"`

Direct the user to upload attachments manually to the project page in Confluence after the publish run completes.

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
