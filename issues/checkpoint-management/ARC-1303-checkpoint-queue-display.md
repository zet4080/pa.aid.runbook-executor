| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1303](https://proalpha.atlassian.net/browse/ARC-1303) |
| Created | 2026-06-27 |

# ARC-1303: Display checkpoint queue with automatic priority scoring

## Goal
Pending checkpoints shown in ordered queue panel. Each scored by wait time and gate level (🟢 > 🟡 > 🔴). Queue reflects current scheduling order. Each queue item shows a [Resume] button and a link to the associated Bitbucket PR.

## Acceptance Criteria
1. Given multiple pending checkpoints, when displayed, then ordered by priority score (gate level + wait time).
2. Given checkpoint waiting longer than equal-level peer, then it ranks higher.
3. Given pending checkpoint in queue, then queue item displays a [Resume] button that triggers the Resume flow for that checkpoint.
4. Given pending checkpoint in queue, then queue item displays a link to the associated Bitbucket PR.

## In Scope
- Queue panel, automatic priority scoring, gate level weighting, [Resume] button per queue item, PR link per queue item

## Out of Scope
- Manual reorder (ARC-1298), policy scheduling (ARC-1299)

## Dependencies
- ARC-1302 (PR creation and [Resume] card)