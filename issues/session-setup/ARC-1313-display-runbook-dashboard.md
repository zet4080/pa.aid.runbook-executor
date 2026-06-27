| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1289](https://proalpha.atlassian.net/browse/ARC-1289) ARC: Session Setup & Runbook Selection |
| Jira | [ARC-1313](https://proalpha.atlassian.net/browse/ARC-1313) |
| Created | 2026-06-27 |

# ARC-1313: Display runbook dashboard at startup

## Goal
On startup, tool scans planning repo and presents all runbooks in collapsed summary view. Each shows name, status (idle/running/blocked/complete), current wave, and % progress. Loads within 3 seconds.

## Acceptance Criteria
1. Given planning repo with runbooks, when tool starts, then all runbooks listed with name, status, wave, % progress within 3 seconds.
2. Given runbooks in various states, when displayed, then each shows correct status badge.

## In Scope
- Repo scanning, status derivation, collapsed dashboard, 3-second load target

## Out of Scope
- Editing runbooks, detailed step view (expand-on-demand is ARC-1314)

## Dependencies
- ARC-1286 (runbook parser must exist)
