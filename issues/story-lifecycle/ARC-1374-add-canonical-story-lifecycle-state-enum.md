| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1374 |

# ARC-1374: Add canonical StoryLifecycleState enum and per-story state field

## Goal

Define a single source-of-truth StoryLifecycleState enum (states: unclaimed, claimed, worktree_provisioning, agent_executing, retrying, awaiting_escalation_checkpoint, awaiting_plan_review, addressing_plan_comments, agent_self_review, awaiting_implementation_review, addressing_implementation_comments, awaiting_decision_review, addressing_decision_comments, completion_summary_drafting, archiving, archived) plus a StoryLifecyclePhase type ('plan'|'code'|'decision') in packages/server/src/state/types.ts, and persist a current-state field per story in the sidecar state. This is the foundation everything else in this epic builds on — no behavior change yet, types and persistence only.

## Acceptance Criteria

Given the new types are added to state/types.ts, when existing code compiles, then no existing behavior changes (pure additive type + field, no migration of call sites yet).

---

Given a story begins executing, when its lifecycle state changes, then the new per-story state field in the sidecar is updated and persisted across restarts.

---

Given the sidecar schema changes, when older sidecar.json files without the new field are loaded, then they load without error (backward compatible default state, e.g. 'unclaimed' or inferred from existing flags).

## In Scope

- state/types.ts additions
- sidecar.ts persistence of the new field
- migration/default-value handling for existing sidecar files

## Out of Scope

- migrating any actual call sites in laneRunner.ts/checkpointGit.ts/checkpoints.ts to use the new enum (that's later stories)
- deriving legacy enums as rollups (ARC-1381)

## Dependencies

- None.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1374-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1374-COMPLETION-SUMMARY.md` | TBD |