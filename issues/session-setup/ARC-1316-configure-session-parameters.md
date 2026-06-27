| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1289](https://proalpha.atlassian.net/browse/ARC-1289) ARC: Session Setup & Runbook Selection |
| Jira | [ARC-1316](https://proalpha.atlassian.net/browse/ARC-1316) |
| Created | 2026-06-27 |

# ARC-1316: Configure session parameters before start

## Goal
Before starting, supervisor configures: max concurrent lanes, retry count per step, agent escalation sequence, initial queue policy. Defaults provided for all parameters.

## Acceptance Criteria
1. Given concurrency set to N, when session runs, then at most N lanes run simultaneously.
2. Given defaults pre-populated, when supervisor starts without changes, then session runs with sensible defaults.
3. Given escalation sequence configured, when step fails and retries, then each retry uses next agent in sequence.

## In Scope
- Concurrency, retry count, escalation sequence, queue policy selection

## Out of Scope
- Per-runbook config, runtime concurrency changes
