| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-11 |
| Parent Epic | ARC-1295 — ARC: Checkpoint Management |
| Jira | ARC-1372 |

# ARC-1372: Unify planning-artifact checkpoint gate with PR comment-resolution loop

## Goal

Unify the planning-artifact checkpoint gate (ARC-1371, which currently gates resume on PR-merge status) with the standard checkpoint comment-resolution loop (ARC-1304, which gates resume on zero unresolved PR comments), so that when an agent writes an implementation plan, commits/pushes it, and opens a PR, the checkpoint resume logic checks for unresolved PR comments rather than requiring a merge. If unresolved comments exist, route to comment-addressing (agent reads/answers comments, revises the plan file, tool commits+pushes the revision, and a new checkpoint/pause is inserted) in a loop. Once zero unresolved comments remain, hand off to the agent to begin implementation.

## Problem

ARC-1371 gates resume of a planning-artifact checkpoint (implementation plan or decision document PR) on the PR being merged. ARC-1304 gates resume of a standard execution checkpoint on there being zero unresolved PR comments, looping through comment-addressing until the thread is clear. These two gates are inconsistent: a planning-artifact PR with open, unresolved review comments can still be sitting unmerged while an agent has no defined way to iterate on the comments before merge, and there is no comment-addressing loop wired into the planning-artifact path at all. This forces supervisors to either leave comments unresolved and merge anyway, or manually ping the agent out-of-band to revise the plan. The two gate mechanisms need to be unified so planning-artifact PRs use the same comment-resolution loop as standard checkpoints, and resume is driven by "zero unresolved comments" rather than "PR merged".

## Acceptance Criteria

**Given** A planning-artifact PR is open with unresolved comments,
**When** Resume is triggered,
**Then** The system does NOT reject with 409 and instead routes to the comment-addressing flow (like ARC-1304's addressPRComments).

---

**Given** A planning-artifact PR is open with zero unresolved comments,
**When** Resume is triggered,
**Then** The system hands off to the agent for implementation (does not require the PR to be merged).

---

**Given** Comments are addressed and the implementation plan file is revised,
**When** The revision is committed,
**Then** It is also pushed to the PR branch and a new checkpoint entry is queued (pause again) rather than auto-proceeding.

---

**Given** The unification lands,
**When** Existing ARC-1371 merge-based behavior is exercised in tests,
**Then** Equivalent coverage is preserved or explicitly superseded by new comment-based tests (no silent regression).

## In Scope

- Modify gate logic in checkpointGit.ts for the planning-artifact resume path
- Modify gate logic in checkpoints.ts for the planning-artifact resume path
- Modify gate logic in laneRunner.ts for the planning-artifact resume path
- Add unit/integration tests for the new comment-based resume behavior on planning-artifact checkpoints

## Out of Scope

- Changing ARC-1304's existing standard-checkpoint behavior
- Changing ARC-1367 (claim) mechanism except where directly integrated as a dependency
- Changing ARC-1369 (auto-commit) mechanism except where directly integrated as a dependency
- Changing ARC-1370 (auto-archive) mechanism except where directly integrated as a dependency

## Constraints

- Design Considerations: This unification affects checkpoint lifecycle logic currently encoded as scattered conditional flags across checkpointGit.ts/checkpoints.ts/laneRunner.ts (lane-paused?, has-PR?, PR-merged?, decision-answered?, and now comment-thread-state). Before writing the implementation plan for this issue, use the write-design-spec skill to evaluate whether to model the planning-artifact checkpoint lifecycle as an explicit finite state machine (proposed states: DRAFTED -> PR_OPEN -> ADDRESSING_COMMENTS (loop back to PR_OPEN) -> HANDED_OFF -> IMPLEMENTING, with AWAITING_DECISION_ANSWER as a parallel state for decision-type artifacts) rather than continuing with ad-hoc conditional logic. This is a design decision to be resolved via write-design-spec, not assumed.

## Dependencies

- ARC-1371 (Conductor: open PR for implementation plans and decision documents as approval gate — provides the planning-artifact PR creation and pause mechanism this issue unifies)
- ARC-1304 (standard checkpoint comment-resolution loop — provides the addressPRComments flow this issue reuses for planning artifacts)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1372-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1372-COMPLETION-SUMMARY.md` | TBD |