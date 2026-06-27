| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1296](https://proalpha.atlassian.net/browse/ARC-1296) ARC: Queue & Scheduling Policy |
| Jira | [ARC-1297](https://proalpha.atlassian.net/browse/ARC-1297) |
| Created | 2026-06-27 |

# ARC-1297: Display ordered checkpoint queue with priority scores

## Goal
Persistent queue panel lists all pending checkpoints in priority order. Each entry: runbook name, checkpoint type, wait time, priority score. Updates live.

## Acceptance Criteria
1. Given pending checkpoints, when supervisor views queue panel, then all listed with type, runbook, wait time, priority score.
2. Given new checkpoint arrives, when added, then appears in correct priority position within 1 second.

## In Scope
- Queue panel UI, live updates, priority score display

## Out of Scope
- Manual reorder (ARC-1298), policy selection (ARC-1299)

## Dependencies
- ARC-1303 (checkpoint queue with priority scoring)
