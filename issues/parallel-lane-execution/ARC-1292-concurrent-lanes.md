| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Completion | 100% |
| Last Updated | 2026-06-30 |
| Parent Epic | [ARC-1290](https://proalpha.atlassian.net/browse/ARC-1290) ARC: Parallel Lane Execution |
| Jira | [ARC-1292](https://proalpha.atlassian.net/browse/ARC-1292) |
| Created | 2026-06-27 |

# ARC-1292: Run multiple lanes concurrently up to concurrency limit

## Goal
Tool runs selected runbook lanes in parallel up to configured maximum. When a lane finishes or blocks, next queued lane starts if capacity available.

## Acceptance Criteria
1. Given concurrency N and M>N runbooks selected, when session starts, then exactly N lanes run and rest queue.
2. Given running lane completes, when capacity frees, then next queued lane starts automatically.

## In Scope
- Concurrency management, lane queuing, capacity-triggered start

## Out of Scope
- Mid-session concurrency limit changes, priority-based lane ordering
