| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1361 |

# ARC-1361: Implement `preflight_check` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/preflight_check.ts` that verifies all required planning artifacts exist before the agent begins executing an implementation plan — returning a structured pass/fail result that the agent can act on immediately.

## Problem

The `execute-implementation-plan` skill starts with a pre-flight checklist:
- Does the issue file exist?
- Does the implementation plan exist and have no unresolved TBD/TODO placeholders?
- Is the runbook available?

Agents skip or partially perform this check, then encounter missing files mid-execution, wasting time and producing incomplete work.

## Acceptance Criteria

**Given** all required artifacts exist and the plan has no TBD placeholders,
**When** `preflight_check({ repo_path, issue_key, epic, lane })` is called,
**Then** the tool returns `{ passed: true, checks: [...all passed] }`.

---

**Given** the implementation plan file is missing,
**When** `preflight_check` is called,
**Then** `passed` is `false` and `checks` includes `{ name: "implementation_plan", passed: false, path: "..." }`.

---

**Given** the implementation plan contains lines with `TBD` or `TODO`,
**When** `preflight_check` is called,
**Then** `passed` is `false` and `checks` includes `{ name: "plan_no_placeholders", passed: false, details: ["line 42: TBD"] }`.

---

**Given** the runbook file for the lane is missing,
**When** `preflight_check` is called,
**Then** `passed` is `false` and `checks` includes `{ name: "runbook", passed: false, path: "..." }`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/preflight_check.ts`
- Checks: issue file exists, implementation plan exists, plan has no TBD/TODO, runbook exists
- Returns: `{ passed: boolean, checks: Array<{ name: string, passed: boolean, path?: string, details?: string[] }> }`
- Read-only — no file modifications

## Out of Scope

- Running tests or lint
- Resolving missing artifacts
- Jira status checks

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `repo_path` (absolute), `issue_key`, `epic`, `lane`
- Read-only — no side effects

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Planning repo artifact layout (issue, plan, runbook paths)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1361-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1361-COMPLETION-SUMMARY.md` | TBD |
