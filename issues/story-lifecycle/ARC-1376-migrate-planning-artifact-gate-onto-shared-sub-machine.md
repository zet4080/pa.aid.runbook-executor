| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1376 |

# ARC-1376: Migrate planning-artifact gate onto shared PR-review sub-machine (supersedes ARC-1372)

## Goal

Rewire ARC-1371's existing planning-artifact PR gate (which currently gates resume on PR-merge status via a hard poller, with manual resume rejected via 409 while the PR is open) to use the new shared sub-machine from ARC-1375 instead, for both plan and decision artifact kinds. This is the direct technical successor to and supersedes the now-closed issue ARC-1372. Remove the old poller-based merge-detection and 409-rejection logic entirely in favor of the explicit-resume-driven algorithm.

## Acceptance Criteria

Given a planning-artifact PR is open with unresolved comments, when resume is triggered, then the system does NOT reject with 409 and instead routes through the shared comment-addressing flow.

---

Given a planning-artifact PR is merged, when resume is triggered, then the system advances to the next phase (AGENT_EXECUTING(code) for plan artifacts).

---

Given the old planningPRPollers background polling mechanism, when this migration lands, then it is removed (no longer needed — merge detection happens only on explicit resume).

---

Given decision documents previously used a status:pending/answered markdown field mechanism, when this migration lands, then decision documents also use the new PR-merge-gated sub-machine instead (via artifactKind: 'decision'), and the old markdown-field mechanism is removed.

## In Scope

- checkpointGit.ts, checkpoints.ts, laneRunner.ts changes to route plan/decision artifact checkpoints through ARC-1375's shared function
- removal of planningPRPollers
- removal of decision status:pending/answered field parsing

## Out of Scope

- the new implementation-review gate (ARC-1378) — that's a separate artifactKind wired in a later story

## Dependencies

- ARC-1375

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1376-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1376-COMPLETION-SUMMARY.md` | TBD |