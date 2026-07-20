| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-20 |
| Parent Epic | ARC-1295 — ARC: Checkpoint Management |
| Jira | ARC-1374 |

# ARC-1374: Verify and harden implementation plan review loop (Checkpoint #1)

## Goal

Verify that the existing implementation plan review checkpoint (Checkpoint #1) correctly loops when resumed with unresolved PR comments, rather than proceeding with stale comments. If the loop mechanism is missing or incorrect, implement it. The checkpoint must re-queue and re-notify rather than just resuming once, ensuring the agent addresses all comments before the lane proceeds to implementation.

## Acceptance Criteria

**Given** An implementation plan PR is open with unresolved comments,
**When** The checkpoint is resumed via POST /api/checkpoints/:stepId/resume,
**Then** The resume handler (resumeArtifactPRReviewLoop in checkpointGit.ts lines 785-820) detects unresolved comments and re-queues the checkpoint rather than proceeding.

---

**Given** The checkpoint is re-queued due to unresolved comments,
**When** The agent addresses the comments and revises the plan,
**Then** The checkpoint pauses again (loops) rather than auto-proceeding to implementation.

---

**Given** The implementation plan PR has zero unresolved comments,
**When** The checkpoint is resumed,
**Then** The lane proceeds to implementation without re-queuing.

---

**Given** The verify-and-harden work is complete,
**When** A regression test is added or existing tests are audited,
**Then** There is explicit test coverage proving the loop re-queues on unresolved comments and only proceeds when comments are resolved.

## In Scope

- Verify current behavior of resumeArtifactPRReviewLoop in checkpointGit.ts lines 785-820
- Verify current behavior of POST /api/checkpoints/:stepId/resume in routes/checkpoints.ts lines 179-398
- Verify current behavior of triggerPlanningArtifactPR in laneRunner.ts lines 441-532
- Fix the loop mechanism if it is currently one-shot (doesn't re-queue on unresolved comments)
- Add regression test proving the loop re-queues on unresolved comments
- Document the verified behavior in code comments if it's not already documented

## Out of Scope

- Changing the PR creation mechanism (triggerPlanningArtifactPR)
- Changing the basic checkpoint synchronous blocking mechanism (CheckpointQueueEntry)
- Changes to Checkpoint #2 (implementation review) — that's a separate issue
- Changes to multi-repo PR support — that's a separate issue
- Changes to the generate-lane-runbooks skill or runbook template — those are separate skill-maintainer tasks

## Constraints

- Existing plan-review checkpoint code is in triggerPlanningArtifactPR (laneRunner.ts:441-532), resumeArtifactPRReviewLoop (checkpointGit.ts:785-820), and POST /api/checkpoints/:stepId/resume (routes/checkpoints.ts:179-398)
- The checkpoint already blocks the lane synchronously via CheckpointQueueEntry
- The checkpoint already creates a PR of the implementation plan
- This is a verify-and-harden task, not a greenfield implementation — treat existing behavior as potentially correct and verify first before changing

## Dependencies

- None — this issue verifies existing behavior and can proceed in parallel with Issue C (ARC-1375)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1374-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1374-COMPLETION-SUMMARY.md` | TBD |