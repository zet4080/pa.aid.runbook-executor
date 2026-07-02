| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1355 |

# ARC-1355: Implement `run_lint` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/run_lint.ts` that auto-detects the linter for the current project and runs it — returning findings and pass/fail — so the agent does not need to manually detect and invoke the correct lint command during `local-code-review`.

## Problem

The `local-code-review` skill requires the agent to run the project linter, but must first detect which one is in use:
- Node/TypeScript → `npm run lint`
- Python → `ruff check .`
- Rust → `cargo clippy`
- Go → `golangci-lint run`

Agents frequently skip lint or use the wrong command when detection fails. This tool makes lint a single reliable call regardless of project language.

## Acceptance Criteria

**Given** a project with `package.json` containing a `lint` script,
**When** `run_lint({ directory })` is called,
**Then** the tool runs `npm run lint` and returns `{ passed: boolean, output: string, linter: "eslint" }`.

---

**Given** a project with `pyproject.toml` or `ruff.toml`,
**When** `run_lint` is called,
**Then** the tool runs `ruff check .` and returns `{ passed: boolean, output: string, linter: "ruff" }`.

---

**Given** a project with `Cargo.toml`,
**When** `run_lint` is called,
**Then** the tool runs `cargo clippy -- -D warnings` and returns `{ passed: boolean, output: string, linter: "clippy" }`.

---

**Given** a project with `go.mod`,
**When** `run_lint` is called,
**Then** the tool runs `golangci-lint run` and returns `{ passed: boolean, output: string, linter: "golangci-lint" }`.

---

**Given** a `package.json` exists but has no `lint` script,
**When** `run_lint` is called,
**Then** the tool returns `{ passed: null, linter: null, error: "No lint script found in package.json" }`.

---

**Given** no recognised linter is detected,
**When** `run_lint` is called,
**Then** the tool returns `{ passed: null, output: "", linter: null, error: "No linter detected" }`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/run_lint.ts`
- Framework/linter detection by file presence and `package.json` script inspection
- Runs the appropriate lint command in the given directory
- Returns: `{ passed: boolean | null, output: string, linter: string | null, error?: string }`
- Captures both stdout and stderr

## Out of Scope

- Running tests (that is `run_tests`, ARC-1354)
- Type checking (that is `run_typecheck`, ARC-1356)
- Auto-fixing lint errors

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$` for subprocess execution
- Input: `directory` (absolute path, defaults to `context.worktree` if omitted)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Bun shell (`Bun.$`) for subprocess execution
- `run_tests` (ARC-1354) — shares framework detection pattern; consider extracting shared detection helper

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1355-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1355-COMPLETION-SUMMARY.md` | TBD |
