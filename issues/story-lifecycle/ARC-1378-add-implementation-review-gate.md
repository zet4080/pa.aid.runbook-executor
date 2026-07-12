| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1378 |

# ARC-1378: Add new human implementation-review gate after self-review passes

## Goal

After agent self-review (local-code-review) passes clean, automatically open the real code delivery PR (the actual branch that ships the code to main, not a side-channel) and pause the lane at a new awaiting_implementation_review checkpoint, using the shared sub-machine from ARC-1375 with artifactKind: 'implementation'. This closes a confirmed gap: today, code changes are never gated by human review before archiving — only agent self-review and the (separate, plan-only) planning-artifact gate exist.

## Acceptance Criteria

Given local-code-review passes clean, when the lane proceeds, then a real PR is opened for the story's code changes and the lane pauses at awaiting_implementation_review (does not proceed directly to completion-summary/archive).

---

Given the implementation-review PR is merged, when resume is triggered, then the lane advances to completion_summary_drafting.

---

Given the implementation-review PR has unresolved comments and is not merged, when resume is triggered, then the shared comment-addressing flow from ARC-1375 runs (agent addresses comments, revises code, commits, pushes) and the checkpoint re-queues.

---

Given this gate did not exist before, when regression tests run against existing lanes, then no story can reach archived state without having passed through awaiting_implementation_review with a merged PR.

## In Scope

- laneRunner.ts wiring of the new gate immediately after the self-review step and before completion-summary-drafting
- new CheckpointQueueEntry shape/fields as needed for artifactKind: 'implementation'

## Out of Scope

- the shared sub-machine mechanism itself (built in ARC-1375)
- self-review state tracking (built in ARC-1377, this story depends on it)

## Dependencies

- ARC-1375
- ARC-1377

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1378-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1378-COMPLETION-SUMMARY.md` | TBD |