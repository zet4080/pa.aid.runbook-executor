| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Completion | 100% |
| Last Updated | 2026-06-29 |
| Parent Epic | [ARC-1284](https://proalpha.atlassian.net/browse/ARC-1284) ARC: Core Infrastructure |
| Jira | [ARC-1286](https://proalpha.atlassian.net/browse/ARC-1286) |
| Created | 2026-06-27 |

# ARC-1286: Parse runbook markdown into typed step graph

## Goal
Reads `docs/plans/runbook-*.md` files and converts them into an in-memory step graph. Each step typed as agent/manual/checkpoint from explicit markers (🔴🟡🟢) and structured fields. Natural-language inference not used.

## Acceptance Criteria
1. Given a runbook with 🔴🟡🟢 markers and checkboxes, when parser runs, then each step has: type (agent/manual/checkpoint), checkbox state, and checkpoint level.
2. Given a step with no explicit type field, when parsed, then classified as manual by default.

## In Scope
- Markdown parsing, step typing, checkpoint detection, checkbox state extraction

## Out of Scope
- Modifying runbook structure, authoring runbooks, YAML/JSON formats

## Constraints
- Runbook format: docs/plans/runbook-*.md with 🔴🟡🟢 markers and hierarchical checkboxes
- Must not modify runbook structure — only update checkbox state
