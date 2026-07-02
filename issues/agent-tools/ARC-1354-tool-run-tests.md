| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1354 |

# ARC-1354: Implement `run_tests` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/run_tests.ts` that auto-detects the test framework in the current project and runs the test suite — returning pass/fail status and output — so the agent does not need to manually detect and invoke the correct test command.

## Problem

The `local-code-review` skill requires the agent to run tests, but must first detect which framework is in use:
- Node/TypeScript → `npm test`
- Rust → `cargo test`
- Python → `pytest -v`
- Go → `go test ./...`

The agent frequently picks the wrong command (e.g. running `npm test` in a Python project), or skips detection and hard-codes a command that fails. A tool that handles detection and execution reliably reduces review failures and agent confusion.

## Acceptance Criteria

**Given** a project with `package.json` containing a `test` script,
**When** `run_tests({ directory })` is called,
**Then** the tool runs `npm test` in that directory and returns `{ passed: boolean, output: string, framework: "npm" }`.

---

**Given** a project with `Cargo.toml`,
**When** `run_tests` is called,
**Then** the tool runs `cargo test` and returns `{ passed: boolean, output: string, framework: "cargo" }`.

---

**Given** a project with `pyproject.toml` or `setup.py`,
**When** `run_tests` is called,
**Then** the tool runs `pytest -v` and returns `{ passed: boolean, output: string, framework: "pytest" }`.

---

**Given** a project with `go.mod`,
**When** `run_tests` is called,
**Then** the tool runs `go test ./...` and returns `{ passed: boolean, output: string, framework: "go" }`.

---

**Given** no recognised test framework is detected,
**When** `run_tests` is called,
**Then** the tool returns `{ passed: null, output: "", framework: null, error: "No test framework detected" }`.

---

**Given** the test command exits with a non-zero code,
**When** `run_tests` is called,
**Then** `passed` is `false` and the full output (stdout + stderr) is included.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/run_tests.ts`
- Framework detection by file presence: `package.json`, `Cargo.toml`, `pyproject.toml`, `setup.py`, `go.mod`
- Runs the appropriate test command in the given directory
- Returns: `{ passed: boolean | null, output: string, framework: string | null, error?: string }`
- Captures both stdout and stderr

## Out of Scope

- Lint checks (that is `run_lint`, ARC-1355)
- Type checking (that is `run_typecheck`, ARC-1356)
- Fixing test failures
- Parallel test execution across multiple frameworks

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$` for subprocess execution
- Input: `directory` (absolute path, defaults to `context.worktree` if omitted)
- Detection order: `package.json` → `Cargo.toml` → `pyproject.toml`/`setup.py` → `go.mod`

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Bun shell (`Bun.$`) for subprocess execution

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1354-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1354-COMPLETION-SUMMARY.md` | TBD |
