| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1350 |

# ARC-1350: Implement `runbook_check_step` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/runbook_check_step.ts` that marks a specific implementation step checkbox as done (`[x]`) within a story's runbook entry, then commits the change — replacing the agent's manual markdown edit + git commit after each implementation step.

## Problem

The `execute-implementation-plan` skill requires the agent to mark a runbook step checkbox after completing each implementation step:
1. Find the correct story section in the runbook
2. Find the correct step checkbox by step number or description
3. Flip `- [ ]` to `- [x]`
4. `git add` the runbook file
5. `git commit -m "chore({lane}): {KEY} step {N} done"`
6. `git push`

This is repeated after every single implementation step. Agents frequently mis-identify step numbers, corrupt surrounding markdown, or forget to commit between steps.

## Acceptance Criteria

**Given** a runbook with a story section containing numbered step checkboxes,
**When** `runbook_check_step({ runbook_path, story_key, step_number, repo_path, lane })` is called,
**Then** the checkbox for that step number is flipped from `[ ]` to `[x]`, the file is committed and pushed.

---

**Given** the step checkbox is already `[x]`,
**When** `runbook_check_step` is called,
**Then** the tool returns a no-op success: `"Step {N} of {KEY} already checked"`.

---

**Given** the step number does not exist in the story section,
**When** `runbook_check_step` is called,
**Then** the tool returns an error: `"Step {N} not found in story {KEY}"`.

---

**Given** the story key does not exist in the runbook,
**When** `runbook_check_step` is called,
**Then** the tool returns an error: `"Story {KEY} not found in runbook"`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/runbook_check_step.ts`
- In-place patch of the target step checkbox within the named story section
- `git add`, `git commit`, `git push` in the repo at `repo_path`
- Commit message format: `chore({lane}): {KEY} step {step_number} done`

## Out of Scope

- Checking off the story-level checkbox (that is `runbook_check_story`, ARC-1351)
- Creating or modifying step content beyond the checkbox state

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `runbook_path` (absolute), `story_key`, `step_number` (integer), `lane`, `repo_path`

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `runbook_claim_story` (ARC-1349) — story must be claimed before steps are checked off

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1350-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1350-COMPLETION-SUMMARY.md` | TBD |
