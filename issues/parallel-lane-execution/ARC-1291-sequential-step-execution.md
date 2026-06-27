| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1290](https://proalpha.atlassian.net/browse/ARC-1290) ARC: Parallel Lane Execution |
| Jira | [ARC-1291](https://proalpha.atlassian.net/browse/ARC-1291) |
| Created | 2026-06-27 |

# ARC-1291: Execute runbook steps sequentially within a lane

## Goal
Tool executes steps within a single lane one at a time, in order. Each step completes (or checkpoints/fails) before next begins. Step type routes execution: agent → opencode CLI, manual → supervisor pause, checkpoint → checkpoint flow.

## Acceptance Criteria
1. Given lane with N steps, when execution starts, then steps execute in order and step N+1 does not start until step N completes.
2. Given manual step, when reached, then lane pauses and supervisor notified.
3. Given agent step, when reached, then opencode run invoked and output streams to UI.

## In Scope
- Sequential execution, step type routing, lane state transitions

## Out of Scope
- Parallel execution within a lane, skipping steps

## Dependencies
- ARC-1286 (parser), ARC-1288 (opencode CLI integration)
