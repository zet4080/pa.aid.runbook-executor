| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1375 |

# ARC-1375: Extract generic PR-review-loop sub-machine parametrized by artifact kind

## Goal

Extract the existing ARC-1304 "open PR → check unresolved comments → address-loop-or-advance" logic into one shared, reusable function/module parametrized by artifactKind: 'plan' | 'implementation' | 'decision', with the resume/gate algorithm being: on resume, if the artifact's PR is merged, advance; if not merged, check unresolved comments — if any exist, the agent addresses them (revises the artifact file, commits, pushes), transitions through an "addressing_X_comments" state, and returns to the "awaiting_X_review" state (re-checkpoint, pause again); if not merged and zero unresolved comments, silently re-queue the checkpoint as "still awaiting merge" with no rejection and no agent action. No background auto-merge polling — this is driven only by explicit supervisor resume calls.

## Acceptance Criteria

Given an artifact PR is open and merged, when resume is triggered, then the shared function reports "advance" regardless of artifactKind.

---

Given an artifact PR is open, not merged, with unresolved comments, when resume is triggered, then the shared function triggers comment-addressing (revise file, commit, push) and re-queues the checkpoint.

---

Given an artifact PR is open, not merged, with zero unresolved comments, when resume is triggered, then the shared function re-queues the checkpoint silently with no agent action and no error/rejection.

---

Given three different artifactKind values are passed, when the function runs, then behavior is identical except for artifact-kind-specific messaging/labels (single implementation, not three near-duplicates).

## In Scope

- new shared module/function in checkpointGit.ts or a new file
- unit tests for all three resume branches across all three artifactKind values

## Out of Scope

- wiring this into the actual planning-artifact gate (ARC-1376) or the new implementation-review gate (ARC-1378) — this story only builds the reusable mechanism

## Dependencies

- ARC-1374

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1375-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1375-COMPLETION-SUMMARY.md` | TBD |