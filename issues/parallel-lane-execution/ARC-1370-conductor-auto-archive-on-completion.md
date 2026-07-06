| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Parent Epic | ARC-1290 — ARC: Parallel Lane Execution |
| Jira | ARC-1370 |

# ARC-1370: Conductor: auto-update issue status and archive story artifacts on completion

## Goal

When the conductor marks a story as fully complete (all step checkboxes done), it should automatically: update the issue file's frontmatter `Status` to `Completed` and set `Last Updated`; move the issue file, implementation plan, and completion summary to their respective `done/` folders; and append an entry to `done/README.md`. This eliminates the entire `close-issue` skill ceremony from agents.

## Problem

The `close-issue` skill currently asks agents to perform a multi-step file-move and index-update sequence after a story is done. Every action in that sequence is deterministic: fixed source paths, fixed destination paths, fixed frontmatter fields, fixed README format. The conductor knows the story is complete (it just checked off the last step) and has all the path information needed — it can run the archiving atomically without agent involvement.

## Acceptance Criteria

**Given** The conductor checks off the final step of a story,
**When** The story-complete event fires,
**Then** The issue file's `Status` frontmatter field is updated to `Completed` and `Last Updated` is set to today's date (YYYY-MM-DD).

---

**Given** The issue file has been updated,
**When** The archive routine runs,
**Then** The issue file is moved from `issues/{lane}/{KEY}-*.md` to `done/issues/{KEY}-*.md`.

---

**Given** An implementation plan file exists for the story,
**When** The archive routine runs,
**Then** It is moved from `implementation_plans/{lane}/{KEY}-implementation-plan.md` to `done/plans/{KEY}-implementation-plan.md`.

---

**Given** A completion summary exists for the story,
**When** The archive routine runs,
**Then** It is moved from `task-completions/{KEY}-COMPLETION-SUMMARY.md` to `done/completions/{KEY}-COMPLETION-SUMMARY.md`.

---

**Given** One or more expected artifact files do not exist yet (e.g., completion summary not written),
**When** The archive routine runs,
**Then** Archiving is skipped (not failed); conductor logs a warning and waits for the next step cycle.

---

**Given** All files have been moved,
**When** Archive completes,
**Then** `done/README.md` is updated with a new entry for the story and all changes are committed + pushed to planning repo `main`.

---

**Given** The auto-archive is implemented,
**When** A story completes,
**Then** The `close-issue` skill is no longer needed as a post-implementation ceremony step.

## In Scope

- Story-complete hook triggered after all story steps are checked off
- Update issue frontmatter: `Status: Completed`, `Last Updated: <YYYY-MM-DD>`
- Move issue file: `issues/{lane}/{KEY}-*.md` → `done/issues/{KEY}-*.md`
- Move implementation plan: `implementation_plans/{lane}/{KEY}-implementation-plan.md` → `done/plans/{KEY}-implementation-plan.md`
- Move completion summary: `task-completions/{KEY}-COMPLETION-SUMMARY.md` → `done/completions/{KEY}-COMPLETION-SUMMARY.md`
- Append entry to `done/README.md` (KEY, title, lane, completion date)
- Commit + push all changes to planning repo `main`
- Guard: skip archive if completion summary does not yet exist (log warning)
- Update `close-issue` skill to reflect that this ceremony is now conductor-automated

## Out of Scope

- Archiving partial stories (only triggers on full completion)
- Deleting worktrees — that is a separate cleanup concern
- Updating Jira ticket status — out of conductor scope
- Creating the `done/` folder structure — assumed to already exist

## Constraints

- Archive must be idempotent: if destination file already exists, skip with a log message
- Commit must target planning repo `main`, never the feature branch
- The `done/` folder structure (`done/issues/`, `done/plans/`, `done/completions/`) must exist; conductor should create missing subdirectories

## Dependencies

- ARC-1358 (update_issue_status tool — reference implementation for frontmatter updates)
- ARC-1360 (archive_issue tool — reference implementation for file moves and README update)
- ARC-1369 (auto-commit planning artifacts — commit infrastructure reused here)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1370-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1370-COMPLETION-SUMMARY.md` | TBD |