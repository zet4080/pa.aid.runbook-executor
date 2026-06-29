| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1305](https://proalpha.atlassian.net/browse/ARC-1305) |
| Created | 2026-06-27 |

# ARC-1305: Display persistent [Resume] card and handle Resume click per lane

## Goal
Session UI displays a persistent [Resume] card for each suspended checkpoint lane. Supervisor clicks [Resume] to signal "I have finished reviewing." This is not an approval — it triggers the executor to check for unresolved PR comment threads. Each lane's [Resume] card is independent; clicking one does not affect others.

## Acceptance Criteria
1. Given checkpoint lane suspended, then [Resume] card visible in session UI showing PR link and checkpoint context.
2. Given supervisor clicks [Resume], then executor wakes and reads all unresolved PR comment threads via Bitbucket MCP (triggering ARC-1304 flow).
3. Given [Resume] card active, then card remains visible until either (a) supervisor clicks [Resume], or (b) lane is manually cancelled.
4. Given multiple lanes suspended simultaneously, then each has its own independent [Resume] card; clicking one does not affect the others.

## In Scope
- [Resume] card component, click handler, card persistence, per-lane isolation

## Out of Scope
- Approve/reject buttons, free-text rejection feedback (removed from model), auto-proceed without Re-notify

## Dependencies
- ARC-1302 (PR creation and initial [Resume] card display)
- ARC-1304 (comment thread reading and re-notify loop)