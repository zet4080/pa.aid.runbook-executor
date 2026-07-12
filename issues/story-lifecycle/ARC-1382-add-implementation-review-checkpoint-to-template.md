| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-12 |
| Parent Epic | ARC-1373 — Story Lifecycle State Machine |
| Jira | ARC-1382 |

# ARC-1382: Add implementation-review checkpoint recognition to runbook template/docs/parser

## Goal

Extend HOW-THIS-WORKS.md, the generate_runbook tool's story-block template, and the keyword-based checkpoint-level parser (astWalker.ts) to recognize and correctly classify a new "IMPLEMENTATION REVIEW CHECKPOINT" gate type (alongside the existing "INDIVIDUAL PLAN CHECKPOINT" and "BATCH PLAN CHECKPOINT" conventions), so future-generated runbooks can express the new gate from ARC-1378 using the same keyword-sniffing convention already used for checkpointLevel derivation.

## Acceptance Criteria

Given a runbook story block contains a sub-step labeled with "IMPLEMENTATION REVIEW CHECKPOINT" (or similar established keyword), when astWalker.ts parses it, then it is correctly classified with the appropriate checkpointLevel.

---

Given HOW-THIS-WORKS.md is read by a new contributor, when they look for the 9-step story template, then it documents the new implementation-review checkpoint step in its correct position (after local-code-review, before completion-summary).

---

Given the generate_runbook tool is used to create a new runbook, when a HIGH story block is generated, then it includes the new implementation-review checkpoint sub-step by default in its template.

## In Scope

- HOW-THIS-WORKS.md docs
- astWalker.ts keyword parsing
- generate_runbook tool's story-block template

## Out of Scope

- actually regenerating/modifying any existing runbooks (that's separately handled per-lane as each lane adopts the new gate)

## Dependencies

- ARC-1378

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1382-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1382-COMPLETION-SUMMARY.md` | TBD |