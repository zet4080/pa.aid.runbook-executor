| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Completion | 100% |
| Last Updated | 2026-07-05 |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1360 |

# ARC-1360: Implement `archive_issue` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/archive_issue.ts` that atomically moves all three issue artifacts (issue file, implementation plan, completion summary) to their respective `done/` subdirectories, updates the `done/README.md` completed issues index, and commits everything in one operation — replacing the six-step manual close sequence in the `close-issue` skill.

## Problem

The `close-issue` skill requires the agent to perform six sequential operations:
1. `mkdir -p done/issues/{EPIC}/`
2. `mv issues/{EPIC}/{file} done/issues/{EPIC}/`
3. `mkdir -p done/implementation_plans/{EPIC}/`
4. `mv implementation_plans/{EPIC}/{file} done/implementation_plans/{EPIC}/`
5. `mkdir -p done/task-completions/{EPIC}/`
6. `mv task-completions/{file} done/task-completions/{EPIC}/`
7. Append a row to `done/README.md`
8. `git add`, `git commit`, `git push`

Agents frequently get paths wrong, forget to create subdirectories, skip the README update, or commit only some of the moved files.

## Acceptance Criteria

**Given** an issue key, epic folder name, and planning repo path,
**When** `archive_issue({ repo_path, issue_key, epic })` is called,
**Then** all three artifacts are moved to their `done/` subdirectories, `done/README.md` is updated, and the entire change is committed and pushed in one commit.

---

**Given** the implementation plan file does not exist (optional artifact),
**When** `archive_issue` is called,
**Then** the tool skips the plan move without error and proceeds with the other artifacts.

---

**Given** `done/README.md` does not yet have a table,
**When** `archive_issue` is called,
**Then** the tool creates the table header before appending the new row.

---

**Given** any source file does not exist,
**When** `archive_issue` is called,
**Then** the tool returns an error listing which files are missing before making any changes.

---

**Given** the git push fails,
**When** `archive_issue` is called,
**Then** the local commit (with all moves) is preserved and the tool returns a push-failure error.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/archive_issue.ts`
- Creates `done/{subdir}/{epic}/` directories if they do not exist
- Moves: `issues/{epic}/{KEY}-*.md` → `done/issues/{epic}/`
- Moves: `implementation_plans/{epic}/{KEY}-*.md` → `done/implementation_plans/{epic}/` (if exists)
- Moves: `task-completions/{KEY}-*.md` → `done/task-completions/{epic}/`
- Appends row to `done/README.md`: `| {KEY} | {Title} | {YYYY-MM-DD} | done/issues/{epic}/{file} |`
- `git add`, `git commit -m "docs(close): archive {KEY}"`, `git push`
- Returns: `{ moved: string[], readmeUpdated: boolean, hash: string, pushed: boolean }`

## Out of Scope

- Updating the issue's Completion Tracking section (use `update_issue_status`, ARC-1358)
- Checking off the runbook story checkbox (use `runbook_check_story`, ARC-1351)
- Jira ticket status updates

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$`
- Inputs: `repo_path` (absolute), `issue_key` (e.g. `ARC-1285`), `epic` (e.g. `core-infrastructure`)
- Pre-flight: verify all required source files exist before moving anything

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `update_issue_status` (ARC-1358) — must be called before `archive_issue` to mark Status: Completed
- `runbook_check_story` (ARC-1351) — must be called before or after `archive_issue`

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1360-implementation-plan.md` | ✅ Done |
| Completion summary | `task-completions/ARC-1360-COMPLETION-SUMMARY.md` | ✅ Done |
