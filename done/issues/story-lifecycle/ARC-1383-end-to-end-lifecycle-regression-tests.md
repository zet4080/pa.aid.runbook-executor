| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Completion | 100% |
| Last Updated | 2026-07-15 |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1383 |

# ARC-1383: End-to-end lifecycle regression tests

## Goal

Write integration tests exercising the full claim-through-archive path across all new StoryLifecycleState states, including both retry loops (plan and code phases), all three review gates (plan/implementation/decision) with both their merged and comment-loop resume branches, and both new escalation-routing paths (worktree failure, archive-guard failure), to confirm the full migration behaves correctly end-to-end and no regressions were introduced relative to existing lane behavior.

## Acceptance Criteria

Given a full story lifecycle from claim to archive, when the integration test suite runs, then every state transition in the new StoryLifecycleState enum is exercised at least once.

---

Given the pre-migration test suite (existing laneRunner.test.ts, claimGit.test.ts, checkpointGit tests), when this story completes, then all existing tests still pass (no regressions).

---

Given the three review gates (plan, implementation, decision), when tests run their merge-gated resume algorithm, then both the "merged → advance" and "not merged + comments → address-loop" branches are covered for all three artifactKinds.

## In Scope

- new integration test files
- updates to existing test files as needed for the migrated behavior

## Out of Scope

- any new production code (this is a test-only story, aside from minor testability refactors if strictly required)

## Dependencies

- ARC-1374
- ARC-1375
- ARC-1376
- ARC-1377
- ARC-1378
- ARC-1379
- ARC-1380
- ARC-1381
- ARC-1382

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1383-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1383-COMPLETION-SUMMARY.md` | TBD |