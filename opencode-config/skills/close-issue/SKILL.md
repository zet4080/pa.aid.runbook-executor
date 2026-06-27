---
name: close-issue
description: Use when an issue reaches Status Completed — moves all three artifacts to done/ and updates the index
---

# Close Issue

## Purpose

Archive completed issue artifacts: move issue file, implementation plan, completion summary to done/ folder. Updates index. Enforces atomic move of all three artifacts.

## When to Use

- Issue reaches Status Completed
- All artifacts exist and are finalized
- After completion-summary and issue status updated

---

## Pre-Flight Check

> **Runbook gate:** Before closing an issue, verify the runbook has been updated — the corresponding story checkbox MUST be `[x]` in `docs/plans/runbook-*.md`. If it is still `[ ]`, check it off now and commit to main before archiving.

## Overview

Archive all three artifacts for a completed issue: issue file, implementation plan, completion summary.

**Core principle:** Move all three together. Never rename. Update index last.

## What Gets Moved

| Artifact | From | To |
|----------|------|----|
| Issue | `issues/{EPIC}/` | `done/issues/{EPIC}/` |
| Implementation Plan | `implementation_plans/{EPIC}/` | `done/implementation_plans/{EPIC}/` |
| Completion Summary | `task-completions/{EPIC}/` | `done/task-completions/{EPIC}/` |

Epic folder name must match exactly (e.g., `IP-1692-ERP-Cartridge`).

## Step-by-Step Checklist

- [ ] 1. Update issue `Status: Completed`, `Last Updated: {today}`
- [ ] 2. Confirm completion summary exists at `task-completions/{EPIC}/{TRACKER-ID}-*.md`
- [ ] 3. Move issue file: `mv issues/{EPIC}/{file} done/issues/{EPIC}/{file}`
- [ ] 4. Move implementation plan (if exists): `mv implementation_plans/{EPIC}/{file} done/implementation_plans/{EPIC}/{file}`
- [ ] 5. Move completion summary: `mv task-completions/{EPIC}/{file} done/task-completions/{EPIC}/{file}`
- [ ] 6. Update `done/README.md` — add entry to Completed Issues Index
- [ ] 7. Run final runbook state check — all subtasks for this issue must be `[x]` in the runbook before archiving

## done/README.md Entry Format

```markdown
| {TRACKER-ID} | {Title} | {YYYY-MM-DD} | `done/issues/{EPIC}/{file}` |
```

## What Stays in Active Folders

Never move these:
- Epic overview files
- `BACKLOG.md`
- Domain model files
- Issues with `Status: In Progress` or `Status: Not Started`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Moving only the issue | All 3 artifacts move together |
| Renaming files during move | Never rename — use `mv` not `cp+rename` |
| Wrong epic folder name | Verify exact match including case |
| Forgetting done/README.md | Required — index must stay current |
| Moving In Progress issue | Check status first |
| Archiving with runbook checkbox unchecked | Check off story in `docs/plans/runbook-*.md` and commit to main first |
