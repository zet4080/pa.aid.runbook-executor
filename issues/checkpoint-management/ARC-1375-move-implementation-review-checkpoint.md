| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-20 |
| Parent Epic | ARC-1295 — ARC: Checkpoint Management |
| Jira | ARC-1375 |

# ARC-1375: Move implementation review checkpoint (Checkpoint #2) to fire after Commit→Push→Open PR

## Goal

Reorder the implementation review checkpoint (Checkpoint #2, currently 'implementation' ArtifactKind in checkpointGit.ts:714) to fire unconditionally after Commit → Push → Open PR, rather than firing before commit and only when local-code-review returns 'clean'. The checkpoint must pause the lane, allow user review of the PR, and implement a review loop identical to Checkpoint #1: if resumed with unresolved PR comments, the agent addresses them and the checkpoint re-queues; only when comments are resolved does the lane proceed.

## Acceptance Criteria

**Given** Implementation work is complete and the story branch has been committed and pushed,
**When** The PR is opened,
**Then** The implementation review checkpoint fires unconditionally (not gated on local-code-review outcome).

---

**Given** The implementation review checkpoint fires,
**When** The checkpoint is created,
**Then** The checkpoint is created AFTER the commit and push, not before.

---

**Given** The implementation PR is open with unresolved comments,
**When** The checkpoint is resumed via POST /api/checkpoints/:stepId/resume,
**Then** The checkpoint detects unresolved comments and re-queues (loops) rather than proceeding.

---

**Given** The agent addresses PR comments and revises the implementation,
**When** The revision is committed and pushed,
**Then** The checkpoint pauses again (loops) rather than auto-proceeding.

---

**Given** The implementation PR has zero unresolved comments,
**When** The checkpoint is resumed,
**Then** The lane proceeds without re-queuing.

---

**Given** All changes are complete,
**When** Tests run,
**Then** There is explicit test coverage proving checkpoint ordering (commit happens BEFORE checkpoint is queued) and the review loop behavior.

## In Scope

- Decouple implementation review checkpoint from local-code-review skill outcome
- Move checkpoint creation to fire AFTER Commit → Push → Open PR in execution order
- Wire implementation review checkpoint to use the same review loop mechanism as Checkpoint #1 (comment-address + re-queue)
- Update laneRunner.ts lines 706-728 (current implementation checkpoint wiring)
- Update checkpointGit.ts:714 (current 'implementation' ArtifactKind reference)
- Add test coverage proving checkpoint ordering and review loop behavior
- Ensure local-code-review can still run as a pre-commit step if desired, but is not a precondition for the checkpoint firing

## Out of Scope

- Multi-repo PR support — that's Issue D (ARC-1376), depends on this issue being complete
- Changes to Checkpoint #1 (plan review) — that's Issue B (ARC-1374)
- Changes to the generate-lane-runbooks skill or runbook template — those are separate skill-maintainer tasks (will be noted as a dependency/related work)
- Changes to PR creation functions beyond ensuring they're called before the checkpoint fires

## Constraints

- Current implementation review checkpoint code is at checkpointGit.ts:714 ('implementation' ArtifactKind) and laneRunner.ts:706-728
- Current checkpoint only fires when local-code-review returns 'clean' — this gating must be removed
- Current checkpoint is positioned BEFORE the Commit step in execution order — this must be reversed
- PR creation functions are createPlanningArtifactPR (laneRunner.ts ~402-477) and createCheckpointBranchAndPR (checkpointGit.ts ~94-176) — these currently assume a single repo/remote
- The review loop mechanism to reuse is the same as Checkpoint #1 (resumeArtifactPRReviewLoop in checkpointGit.ts:785-820)

## Dependencies

- Issue B (ARC-1374) should verify the review loop mechanism works correctly for Checkpoint #1 before this issue reuses it for Checkpoint #2, but they can proceed in parallel if needed
- The generate-lane-runbooks skill and generate_runbook MCP tool currently emit the implementation review checkpoint marker BEFORE the Commit line in runbook templates (skill at /repos/pa.aid.runbook-executor/opencode-config/skills/generate-lane-runbooks/SKILL.md lines ~127-137, tool's buildHighStoryBlock function) — this ordering is wrong per new requirements. A separate skill-maintainer task will update the template; this issue's implementation should be coordinated with that change but does not own it.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1375-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1375-COMPLETION-SUMMARY.md` | TBD |