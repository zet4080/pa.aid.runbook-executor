| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1358 |

# ARC-1358: Implement `update_issue_status` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/update_issue_status.ts` that patches the `Status`, `Completion`, and `Last Updated` fields in an issue markdown file's Completion Tracking section — replacing fragile manual sed/regex edits by the agent in `write-completion-summary`.

## Problem

The `write-completion-summary` skill requires the agent to update three specific fields in the issue file:
- `Status: Completed`
- `Completion: 100%`
- `Last Updated: {today}`

Agents accomplish this with manual text editing or sed commands that often break on slight formatting variations, leaving stale status fields in issue files.

## Acceptance Criteria

**Given** an issue file with a Completion Tracking table containing `Status`, `Completion`, and `Last Updated` rows,
**When** `update_issue_status({ issue_path, status, completion, date })` is called,
**Then** those three fields are updated in-place and the file is saved.

---

**Given** the issue file does not have a Completion Tracking section,
**When** `update_issue_status` is called,
**Then** the tool returns an error: `"No Completion Tracking section found in {path}"`.

---

**Given** `date` is omitted,
**When** `update_issue_status` is called,
**Then** today's date in `YYYY-MM-DD` format is used automatically.

---

**Given** the issue file does not exist at the given path,
**When** `update_issue_status` is called,
**Then** the tool returns an error with the missing path.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/update_issue_status.ts`
- Parses the Completion Tracking table in the issue markdown
- Updates `Status`, `Completion`, `Last Updated` rows in-place
- Writes the file back — no git operations (commit is handled separately by `planning_commit`)
- Default `date`: today in `YYYY-MM-DD`

## Out of Scope

- Git add/commit/push (use `planning_commit`, ARC-1359)
- Moving the issue file to `done/` (use `archive_issue`, ARC-1360)
- Updating any other fields in the issue frontmatter

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `issue_path` (absolute), `status` (string), `completion` (string, e.g. `"100%"`), `date` (optional, ISO date string)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `planning_commit` (ARC-1359) — call after this tool to commit the change

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1358-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1358-COMPLETION-SUMMARY.md` | TBD |
