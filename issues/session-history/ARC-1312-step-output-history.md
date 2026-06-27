| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Parent Epic | [ARC-1309](https://proalpha.atlassian.net/browse/ARC-1309) ARC: Session History |
| Jira | [ARC-1312](https://proalpha.atlassian.net/browse/ARC-1312) |
| Created | 2026-06-27 |

# ARC-1312: View full step output history for any past session

## Goal
Within a past session, supervisor browses every executed step with output (full agent event stream), status, and supervisor feedback captured at checkpoints.

## Acceptance Criteria
1. Given past session, when supervisor opens it, then all executed steps listed with status and timestamp.
2. Given step in past session, when selected, then full agent output displayed.
3. Given checkpoint in past session, when viewed, then supervisor feedback shown alongside artifact state at approval/rejection time.

## In Scope
- Per-step output history, checkpoint feedback history, artifact state at checkpoint time

## Out of Scope
- Re-running steps from history, editing past session data
