| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1380 |

# ARC-1380: Fix archive-guard silent failure — route to escalation checkpoint

## Goal

Currently, when archiveStoryArtifacts's guards fail (e.g. missing completion-summary file or missing issue file), the function just logs a warning and returns with no state change, leaving the story stuck with no visibility. Route this failure explicitly into the awaiting_escalation_checkpoint state instead.

## Acceptance Criteria

Given archiveStoryArtifacts's guard check fails (missing required file), when the check fails, then the story transitions to awaiting_escalation_checkpoint instead of silently returning.

---

Given the story is in awaiting_escalation_checkpoint due to an archive-guard failure, when the supervisor resumes after fixing the issue (e.g. writing the missing file), then the archiving operation is re-attempted.

## In Scope

- planningArchiver.ts guard-failure paths
- routing into the escalation-checkpoint queue

## Out of Scope

- changing what the guards actually check for

## Dependencies

- ARC-1374

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1380-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1380-COMPLETION-SUMMARY.md` | TBD |