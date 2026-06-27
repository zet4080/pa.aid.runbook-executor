| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1300](https://proalpha.atlassian.net/browse/ARC-1300) ARC: Failure Handling & Escalation |
| Jira | [ARC-1308](https://proalpha.atlassian.net/browse/ARC-1308) |
| Created | 2026-06-27 |

# ARC-1308: Surface exhausted retries as supervisor checkpoint notification

## Goal
When all retries exhausted, lane pauses and failure added to supervisor queue as 🔴 checkpoint. Browser notification sent; supervisor must decide how to proceed.

## Acceptance Criteria
1. Given all retries exhausted, when final retry fails, then lane pauses and failure checkpoint appears in queue.
2. Given failure checkpoint, when supervisor views it, then full retry history and error output visible.
3. Given failure checkpoint, when browser notification sent, then it identifies runbook, step, retries attempted.

## In Scope
- Failure-as-checkpoint, retry history display, browser notification, lane halt

## Out of Scope
- Automatic recovery, rollback of partial changes

## Dependencies
- ARC-1306 (retry mechanism), ARC-1302 (browser notification)
