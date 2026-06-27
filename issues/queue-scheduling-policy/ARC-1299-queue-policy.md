| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1296](https://proalpha.atlassian.net/browse/ARC-1296) ARC: Queue & Scheduling Policy |
| Jira | [ARC-1299](https://proalpha.atlassian.net/browse/ARC-1299) |
| Created | 2026-06-27 |

# ARC-1299: Select and switch queue policy mid-session

## Goal
Supervisor selects queue policy at session start: checkpoint-first, lane-first, balanced, or wave-gate-first. Switchable mid-session; takes effect on next scheduling decision.

## Acceptance Criteria
1. Given policy checkpoint-first, when capacity frees, then tool processes queued checkpoint before advancing any lane.
2. Given policy lane-first, when capacity frees, then tool advances available lane step before processing checkpoints.
3. Given policy changed mid-session, when next scheduling decision occurs, then new policy applied.

## In Scope
- Four policy modes, session-start selection, mid-session switching

## Out of Scope
- Custom policy definitions, per-runbook policy overrides
