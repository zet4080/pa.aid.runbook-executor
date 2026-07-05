### Completion Summary for ARC-1356 - `run_typecheck` OpenCode Tool

#### What was implemented

**Tool file:** `~/.config/opencode/tools/run_typecheck.ts`
- `detectTypeChecker(directory)` — detects type checker via file priority: `tsconfig.json` → `mypy.ini` → `pyproject.toml` with `[tool.mypy]` → `Cargo.toml`
- `buildTypecheckResult(passed, checker, output, error?)` — pure result constructor
- Tool export with `execute()` — auto-detects and runs type checker, returns `{ passed, checker, output, error? }"
- Returns `null` checker result with error message when no type checker detected

**Test file:** `~/.config/opencode/tools/tests/run_typecheck.test.ts`
- 8 tests covering all detection scenarios — all pass
- Uses node:test + node:assert/strict with temp dir fixtures

**Allowlist updates:**
- `~/.config/opencode/opencode.json`: `run_typecheck: true` added to `debug`, `build`, `senior-coder` agent tool allowlists
- `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`: same 3 agent allowlists updated in the Python config generator

#### Test results

8/8 tests pass:
1. tsconfig.json → `{ checker: 'tsc', command: 'npx tsc --noEmit' }"
2. mypy.ini → `{ checker: 'mypy', command: 'mypy.' }"
3. pyproject.toml with [tool.mypy] → `{ checker: 'mypy', command: 'mypy.' }"
4. pyproject.toml without [tool.mypy] → `null`
5. Cargo.toml → `{ checker: 'cargo-check', command: 'cargo check' }"
6. Empty dir → `null`
7. tsconfig.json + Cargo.toml → tsc wins (priority order)
8. pyproject.toml (no mypy) + Cargo.toml → cargo-check wins

#### Commits

- Config repo (`pa.aid.config.md`), branch `ARC-1356`: `219d8b6` — `feat(agent-tools): implement run_typecheck OpenCode tool (ARC-1356)`
- wsl-setup repo (`pa.aid.wsl-setup.sh`), branch `main`: `d9cf285` — `feat(agent-tools): add run_typecheck to opencode.json allowlists (ARC-1356)`

#### Acceptance criteria

All criteria from the issue are met:
- Tool auto-detects type checker for TypeScript (tsc), Python (mypy via mypy.ini or pyproject.toml), and Rust (cargo check)
- Returns `{ passed, checker, output, error? }` result shape
- `pyproject.toml` without `[tool.mypy]` falls through to next detector (not an early exit)
- `run_typecheck` is enabled for `debug`, `build`, and `senior-coder` agents
- 8 unit tests verify all detection scenarios and edge cases