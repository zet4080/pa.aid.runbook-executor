| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Parent Epic | [ARC-1296](https://proalpha.atlassian.net/browse/ARC-1296) ARC: Queue & Scheduling Policy |
| Jira | [ARC-1298](https://proalpha.atlassian.net/browse/ARC-1298) |
| Created | 2026-06-27 |

# ARC-1298: Reorder checkpoint queue manually

## Goal
Supervisor reorders pending checkpoint queue via drag-and-drop or controls. New order takes effect before next execution slot — no restart required.

## Acceptance Criteria
1. Given ordered queue, when supervisor moves checkpoint to different position, then queue reflects new order immediately.
2. Given manually reordered queue, when next execution slot frees, then tool picks top item from reordered queue.

## In Scope
- Manual reorder UI, immediate scheduling effect

## Out of Scope
- Persisting manual order across sessions, bulk reorder
