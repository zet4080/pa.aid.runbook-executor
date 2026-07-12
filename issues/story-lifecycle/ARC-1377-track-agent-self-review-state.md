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
| Jira | ARC-1377 |

# ARC-1377: Track agent self-review (local-code-review) state explicitly in conductor state

## Goal

Surface the local-code-review skill's iteration count and pass/fail outcome into the conductor's persisted per-story state (currently this loop runs entirely inside the agent's own skill execution and is invisible to laneRunner.ts). This creates the necessary visibility for the new implementation-review gate (ARC-1378) to know when self-review has passed cleanly.

## Acceptance Criteria

Given an agent runs local-code-review during code execution, when the loop completes (pass or exhausted retries), then the outcome and iteration count are recorded in the per-story sidecar state.

---

Given self-review passes cleanly, when laneRunner.ts checks story state, then it can determine "self-review complete, clean" without re-invoking the skill.

---

Given self-review exhausts its retry limit still failing, when laneRunner.ts checks story state, then the story transitions to awaiting_escalation_checkpoint.

## In Scope

- sidecar state field for self-review outcome/iteration count
- laneRunner.ts read/write of this field around the local-code-review step

## Out of Scope

- changing local-code-review's own internal logic/skill behavior

## Dependencies

- ARC-1374

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/story-lifecycle/ARC-1377-implementation-plan.md` | Complete |
| Completion summary | `task-completions/ARC-1377-COMPLETION-SUMMARY.md` | Complete |