| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1356 |

# ARC-1356: Implement `run_typecheck` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/run_typecheck.ts` that auto-detects and runs the type checker for the current project — returning errors and pass/fail — used in `local-code-review` after lint passes.

## Problem

The `local-code-review` skill requires the agent to run type checking after lint: `npx tsc --noEmit` for TypeScript, `mypy .` for Python. Agents skip this step when they cannot determine the correct command or when the project lacks an obvious config file.

## Acceptance Criteria

**Given** a TypeScript project with `tsconfig.json`,
**When** `run_typecheck({ directory })` is called,
**Then** the tool runs `npx tsc --noEmit` and returns `{ passed: boolean, output: string, checker: "tsc" }`.

---

**Given** a Python project with `mypy.ini` or `pyproject.toml` containing `[tool.mypy]`,
**When** `run_typecheck` is called,
**Then** the tool runs `mypy .` and returns `{ passed: boolean, output: string, checker: "mypy" }`.

---

**Given** a Rust project (Cargo.toml) — type checking is covered by `cargo build` or `cargo check`,
**When** `run_typecheck` is called,
**Then** the tool runs `cargo check` and returns `{ passed: boolean, output: string, checker: "cargo-check" }`.

---

**Given** no type checker is detected,
**When** `run_typecheck` is called,
**Then** returns `{ passed: null, checker: null, error: "No type checker detected — skipping" }`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/run_typecheck.ts`
- Detection by file presence: `tsconfig.json`, `mypy.ini`, `pyproject.toml` with mypy section, `Cargo.toml`
- Returns: `{ passed: boolean | null, output: string, checker: string | null, error?: string }`

## Out of Scope

- Running tests or lint
- Auto-fixing type errors

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$`
- Input: `directory` (absolute path, defaults to `context.worktree`)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `run_lint` (ARC-1355) — typically run before type-check in review flow

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1356-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1356-COMPLETION-SUMMARY.md` | TBD |
