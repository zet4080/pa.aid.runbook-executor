| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1290](https://proalpha.atlassian.net/browse/ARC-1290) ARC: Parallel Lane Execution |
| Jira | [ARC-1293](https://proalpha.atlassian.net/browse/ARC-1293) |
| Created | 2026-06-27 |

# ARC-1293: Stream live agent output per lane in the UI

## Goal
As agent step executes, stdout JSON event stream rendered live in lane output panel. Supervisor follows execution in real time without polling.

## Acceptance Criteria
1. Given executing agent step, when subprocess emits events, then each appears in lane panel within 1 second.
2. Given multiple lanes running, when each emits output, then output correctly isolated per lane panel.

## In Scope
- Live streaming UI, per-lane output isolation, JSON event rendering

## Out of Scope
- Log export, output search/filtering
