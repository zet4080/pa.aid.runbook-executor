| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Parent Epic | [ARC-1309](https://proalpha.atlassian.net/browse/ARC-1309) ARC: Session History |
| Jira | [ARC-1311](https://proalpha.atlassian.net/browse/ARC-1311) |
| Created | 2026-06-27 |

# ARC-1311: Browse past sessions list

## Goal
Supervisor views list of all past sessions for current planning repo. Each entry: date/time, runbooks included, final status, duration. Read-only once closed.

## Acceptance Criteria
1. Given past sessions exist, when supervisor opens history view, then all listed with date, runbooks, status, duration.
2. Given past session entry, when supervisor selects it, then session detail view opens.

## In Scope
- Session list UI, session metadata display, read-only past sessions

## Out of Scope
- Deleting sessions, exporting, comparing sessions
