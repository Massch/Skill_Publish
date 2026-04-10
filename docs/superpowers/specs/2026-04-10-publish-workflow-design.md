# Publish Workflow Skill — Design Spec
**Date:** 2026-04-10
**Status:** Approved

---

## Overview

A Claude Code skill invoked with `/publish` that lets the user deliberately push a finished project to company Bitbucket and/or publish its documentation to company Confluence. Both destinations are optional and configured per project on first use. The skill is intentional — it never fires automatically.

## Development Project

The skill is developed and maintained in a dedicated project: `C:\Development\Skill_Publish`. This project has its own local git repo (synced to GitHub) so all changes to the skill are version-controlled. The final skill file is deployed from this project to the Claude Code skills directory when ready.

---

## Invocation

The user types `/publish` inside any project folder under `C:\Development\[project-name]`.

---

## Per-Project Configuration

A `.publish-config.json` file is stored in the project root and committed to git. It records which destinations are active for this project and the details needed to reach them.

```json
{
  "bitbucket": {
    "enabled": true,
    "remoteUrl": "https://bitbucket.org/company/api-oliver.git"
  },
  "confluence": {
    "enabled": true,
    "path": "IT Development > Finance > Aldipress"
  }
}
```

**First run:** If `.publish-config.json` does not exist, the skill asks:
1. Should this project push to company Bitbucket? (yes/no) → if yes, ask for the remote URL
2. Should this project publish docs to Confluence? (yes/no) → if yes, ask for the full parent path (e.g. `IT Development > Finance > Aldipress`)

The file is created, committed, and the publish run continues.

**Subsequent runs:** The config is loaded silently; no questions asked.

---

## Docs Folder

Each project may contain a `docs/` folder at the project root. The user places files here freely:
- Generated markdown files (`.md`) created by Claude during development
- External files dropped in manually: PDFs, Word documents, meeting notes, budgets, etc.

The `docs/` folder is pushed to both GitHub and Bitbucket as part of the normal git push. Confluence publishing reads from this same folder.

---

## Step 1 — Git Push

1. **GitHub (`origin`):** Always pushed. Branch: current branch.
2. **Bitbucket:** Only if `bitbucket.enabled` is true in config.
   - If the `bitbucket` remote does not yet exist in the local git repo, add it using the stored `remoteUrl`.
   - Push current branch to `bitbucket` remote.

---

## Step 2 — Confluence Publish

Only runs if `confluence.enabled` is true in config.

### Page hierarchy

```
[confluence.path, e.g. IT Development > Finance > Aldipress]
  └── [Project name]          ← project parent page (created if missing; name = project folder name)
        ├── [doc1.md]         ← child page per .md file
        ├── [doc2.md]
        └── (attachments)     ← non-markdown files attached to parent page
```

The `confluence.path` can be any depth. The skill resolves each level in order using the Atlassian MCP, stopping at the deepest page found, then creating the project parent page under it.

### Markdown files

For each `.md` file in `docs/`:
- If a child page with that name already exists under the project parent → **update** its content.
- If it does not exist → **create** it as a new child page.

The page title is derived from the filename (e.g. `technical-spec.md` → `Technical Spec`).

### Non-markdown files

For each non-`.md` file in `docs/` (PDF, DOCX, XLSX, etc.):
- **Attach** to the project parent page.
- If an attachment with the same filename already exists → replace it.

---

## Step 3 — Summary Report

After all steps complete, the skill prints a short summary:

```
✓ Pushed to GitHub (origin) — branch: main
✓ Pushed to Bitbucket — branch: main
✓ Confluence: updated 2 pages, attached 1 file
    IT Development > Finance > Aldipress > API Oliver
```

If any step fails, it reports the failure clearly and continues with remaining steps rather than stopping entirely.

---

## Error Handling

| Situation | Behavior |
|---|---|
| `docs/` folder does not exist | Skip Confluence step, note in summary |
| `docs/` folder exists but is empty | Skip Confluence step, note in summary |
| Bitbucket auth fails | Report error, continue with Confluence |
| Confluence path not found | Report which level was missing, skip Confluence |
| Confluence MCP not authenticated | Prompt user to authenticate, then retry |
| Bitbucket remote URL wrong | Report error, ask user to correct config |

---

## Out of Scope

- Automatically publishing on every git push (intentionally excluded)
- Publishing to multiple Confluence locations from one project
- Transforming or reformatting document content before publishing
- Jira integration
