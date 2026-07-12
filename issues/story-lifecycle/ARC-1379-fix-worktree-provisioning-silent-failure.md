| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Completion | 100% |
| Last Updated | 2026-07-12 |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1379 |

# ARC-1379: Fix worktree-provisioning silent failure — route to escalation checkpoint

## Goal

Currently, when git worktree provisioning (ARC-1368) throws an uncaught error, the lane fails silently with no state change and no notification. Route this failure explicitly into the awaiting_escalation_checkpoint state instead, so a supervisor is notified and can resume once the underlying issue is fixed.

## Acceptance Criteria

Given worktree provisioning throws an error, when the error occurs, then the story transitions to awaiting_escalation_checkpoint instead of failing silently.

---

Given the story is in awaiting_escalation_checkpoint due to a worktree failure, when the supervisor resumes after fixing the issue, then the exact same worktree-provisioning operation is re-attempted (uniform resume behavior, no special-case routing needed).

## In Scope

- try/catch wrapping around provisionWorktree() calls in laneRunner.ts
- routing caught errors into the escalation-checkpoint queue

## Out of Scope

- changing worktreeGit.ts's own provisioning logic itself

## Dependencies

- ARC-1374

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/story-lifecycle/ARC-1379-implementation-plan.md` | Done |
| Completion summary | `task-completions/ARC-1379-COMPLETION-SUMMARY.md` | Done |