| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1304](https://proalpha.atlassian.net/browse/ARC-1304) |
| Created | 2026-06-27 |

# ARC-1304: Inline-edit checkpoint artifacts and persist handoff file to disk

## Goal
When checkpoint involves artifact (e.g. implementation plan), supervisor reads and edits it in UI. Edits persisted immediately to ephemeral handoff file — survives crash. Agent consumes handoff file when execution resumes.

## Acceptance Criteria
1. Given checkpoint with artifact file, when supervisor opens checkpoint, then artifact displayed in editable panel.
2. Given supervisor edits artifact, when any character typed, then handoff file updated on disk within 2 seconds.
3. Given tool restarts after crash, when checkpoint re-opened, then supervisor edits still present from handoff file.

## In Scope
- Artifact display, inline editing, auto-save to handoff file, crash persistence

## Out of Scope
- Edit version history, diff view, collaborative editing
