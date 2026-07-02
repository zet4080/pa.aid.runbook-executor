| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1351 |

# ARC-1351: Implement `runbook_check_story` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/runbook_check_story.ts` that marks the top-level story checkbox as done (`[x]`) in a runbook file and commits the change — used by `write-completion-summary` after all implementation steps are complete and the completion summary has been written.

## Problem

The `write-completion-summary` skill requires the agent to:
1. Find the story's top-level checkbox in the runbook (`- [ ] {KEY}: {title}`)
2. Flip it to `- [x]`
3. `git add` the runbook
4. `git commit -m "chore({lane}): close {KEY}"`
5. `git push`

This is distinct from checking off individual implementation steps (ARC-1350). The story-level checkbox signals that the entire story is done. Agents frequently confuse step checkboxes with story checkboxes, or forget to commit.

## Acceptance Criteria

**Given** a runbook with a story whose top-level checkbox is `[ ]`,
**When** `runbook_check_story({ runbook_path, story_key, lane, repo_path })` is called,
**Then** the story checkbox is flipped to `[x]`, the file is committed and pushed with message `chore({lane}): close {KEY}`.

---

**Given** the story checkbox is already `[x]`,
**When** `runbook_check_story` is called,
**Then** the tool returns a no-op success: `"Story {KEY} already checked off"`.

---

**Given** the story key is not found in the runbook,
**When** `runbook_check_story` is called,
**Then** the tool returns an error: `"Story {KEY} not found in runbook"`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/runbook_check_story.ts`
- In-place patch of the story-level checkbox (not step-level checkboxes)
- `git add`, `git commit`, `git push` in the repo at `repo_path`
- Commit message: `chore({lane}): close {KEY}`

## Out of Scope

- Checking off individual implementation steps (that is `runbook_check_step`, ARC-1350)
- Verifying that all step checkboxes are done before allowing the story close
- Moving artifacts to `done/` (that is `archive_issue`, ARC-1360)

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `runbook_path` (absolute), `story_key`, `lane`, `repo_path`

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `runbook_check_step` (ARC-1350) — all steps should be checked before story is closed
- `update_issue_status` (ARC-1358) — typically called in the same `write-completion-summary` flow

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1351-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1351-COMPLETION-SUMMARY.md` | TBD |
