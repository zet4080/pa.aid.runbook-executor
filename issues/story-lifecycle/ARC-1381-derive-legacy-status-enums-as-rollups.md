| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1381 |

# ARC-1381: Derive legacy status enums as computed rollups from canonical state

## Goal

Stop independently setting LaneStatus, RunbookStatus, and ClosedSession.status by hand throughout laneRunner.ts and related route/state code. Instead, compute all three of them on read as rollups derived from the new canonical StoryLifecycleState (e.g. a lane is 'paused' iff any story within it is in a gate state; 'done' iff all stories are archived; 'running' iff any story is in an automatic/non-gate state), so there is exactly one source of truth instead of four values that can drift out of sync.

## Acceptance Criteria

Given all stories in a lane are archived, when LaneStatus is read, then it computes to 'done' without being independently set anywhere.

---

Given at least one story in a lane is sitting in any gate state (awaiting_escalation_checkpoint, awaiting_plan_review, awaiting_implementation_review, awaiting_decision_review), when LaneStatus is read, then it computes to 'paused'.

---

Given the migration lands, when existing code that independently sets LaneStatus/RunbookStatus/ClosedSession.status is searched for, then no such independent-setting code remains (all three are pure derived/computed getters).

## In Scope

- laneRunner.ts, state/types.ts, any route code that reads/sets these three enums

## Out of Scope

- WaveLifecycleState (that's a new concept, not a migration of an existing enum — not covered by this story; may warrant its own future story if not already covered elsewhere in this epic)

## Dependencies

- ARC-1374
- ARC-1376
- ARC-1378

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1381-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1381-COMPLETION-SUMMARY.md` | TBD |