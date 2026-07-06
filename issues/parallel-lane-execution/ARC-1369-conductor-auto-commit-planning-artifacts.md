| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Parent Epic | ARC-1290 — ARC: Parallel Lane Execution |
| Jira | ARC-1369 |

# ARC-1369: Conductor: auto-commit implementation plans and completion summaries to planning repo

## Goal

After each agent step completes successfully, the conductor should scan the planning repo for new or modified files in `implementation_plans/` and `task-completions/`. Any found are automatically committed and pushed to `main`. This removes all manual `planning_commit` git ceremony from agent workflows — agents write the files, the conductor handles the commit.

## Problem

Agents currently must explicitly call `planning_commit` (or run raw git commands) after writing an implementation plan or completion summary. These commits are fully deterministic: fixed directories, fixed commit-message format, always targeting planning repo `main`. The conductor already auto-commits runbook checkoffs via `checkpointGit.ts`; the same pattern should cover planning artifacts.

## Acceptance Criteria

**Given** An agent step completes successfully and has written a new file under `implementation_plans/`,
**When** The conductor's post-step hook runs,
**Then** The new file is staged, committed with message `docs({lane}): add {filename}`, and pushed to planning repo `main`.

---

**Given** An agent step completes successfully and has written a new file under `task-completions/`,
**When** The conductor's post-step hook runs,
**Then** The new file is committed and pushed to planning repo `main` with an appropriate commit message.

---

**Given** No new or modified files exist in those directories after a step,
**When** The post-step hook runs,
**Then** No commit is made (idempotent no-op).

---

**Given** The auto-commit is implemented,
**When** An agent finishes writing an implementation plan,
**Then** The `write-implementation-plan` and `execute-implementation-plan` skills no longer instruct agents to run git commit commands for these artifacts.

## In Scope

- Post-step success hook in `laneRunner.ts` (or a dedicated `planningArtifactCommitter` module)
- Scan directories: `implementation_plans/` and `task-completions/` in the planning repo
- Detect new or modified files via `git status --porcelain` against planning repo `main`
- Commit message format: `docs({lane}): add {filename}`
- Push to planning repo `main`
- Idempotency: no commit if nothing changed
- Update `write-implementation-plan`, `execute-implementation-plan`, and `write-completion-summary` skills to remove manual git ceremony

## Out of Scope

- Committing files outside `implementation_plans/` and `task-completions/` (runbook checkoffs are handled by `checkpointGit.ts`)
- Committing on step failure
- Deciding the content of the files — agents still own that

## Constraints

- Must reuse or extend the existing auto-commit infrastructure in `checkpointGit.ts` — do not introduce a second git abstraction
- Commit must target planning repo `main`, never the feature branch
- Push failure must be logged as a warning and must not fail the step

## Dependencies

- ARC-1359 (planning_commit tool — reference implementation and existing pattern)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1369-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1369-COMPLETION-SUMMARY.md` | TBD |