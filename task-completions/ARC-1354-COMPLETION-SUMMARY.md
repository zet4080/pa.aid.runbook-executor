---
# ARC-1354 Completion Summary — `run_tests` OpenCode Tool

| Field | Value |
|-------|-------|
| Story | ARC-1354 |
| Title | Implement `run_tests` OpenCode tool |
| Date | 2026-07-05 |
| Lane | agent-tools |
| Config repo commit | `a9a0521` (branch `ARC-1354`)

## What Was Built

Two files created in `pa.aid.config.md` repo:

| File | Path |
|------|------|
| Tool implementation | `tools/run_tests.ts` |
| Unit tests | `tools/tests/run_tests.test.ts` |

### `run_tests.ts`

Self-contained OpenCode tool that:
- Exports `detectFramework(directory)` — checks for `package.json`, `Cargo.toml`, `pyproject.toml`, `setup.py`, `go.mod` in priority order; returns `{ framework, command }` or `null`
- Exports `buildResult(passed, framework, output, error?)` — pure constructor; omits `error` key when not provided
- Exports `RunTestsResult` interface: `{ passed: boolean | null, framework: string | null, output: string, error?: string }`
- `execute(args, context)` resolves directory from `args.directory?? context.worktree`, detects framework, runs command with `Bun.$` `.cwd().quiet().nothrow()`, returns combined stdout+stderr with exit code mapped to `passed`

## Test Results

```diff
▶ detectFramework
  ✔ 1: detects npm from package.json (1.801199ms)
  ✔ 2: detects cargo from Cargo.toml (0.17008ms)
  ✔ 3: detects pytest from pyproject.toml (0.116345ms)
  ✔ 4: detects pytest from setup.py (0.148948ms)
  ✔ 5: detects go from go.mod (0.114888ms)
  ✔ 6: returns null for empty directory (0.108768ms)
  ✔ 7: npm wins over cargo when both present (priority order) (0.117768ms)
✔ detectFramework (3.338084ms)
▶ buildResult
  ✔ 8: passed=true, no error key present (0.172091ms)
  ✔ 9: passed=false, no error key present (0.088186ms)
  ✔ 10: passed=null with error field present (0.133755ms)
✔ buildResult (0.54056ms)
ℹ tests 10
ℹ pass 10
ℹ fail 0
```

**10/10 tests pass.**

## Allowlist Verification

```
grep run_tests ~/.config/opencode/opencode.json
        