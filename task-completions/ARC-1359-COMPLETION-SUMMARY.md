# ARC-1359 Completion Summary — `planning_commit`

| Field | Value |
|-------|-------|
| Story | ARC-1359 |
| Lane | agent-tools |
| Date | 2026-07-05 |
| Status | Complete |
| Config repo commit | 6b2fc44 |
| Branch | ARC-1359 (pa.aid.config.md) |

## Summary

Implemented the `planning_commit` OpenCode tool at `~/.config/opencode/tools/planning_commit.ts`. The tool performs atomic `git add` + `git commit` + `git push` for planning repo artifacts, returning `{ hash, pushed, error? }`.

## Acceptance Criteria

| # | Criterion | Result |
|---|-----------|--------|
| 1 | Tool accepts `repo_path`, `files[]`, `message` | ✅ All three args defined with descriptions |
| 2 | `files: [.]` stages all modified tracked files | ✅ Passed through to `git add --.` |
| 3 | Commit hash extracted and returned | ✅ `extractCommitHash()` parses `[branch hash]` format |
| 4 | Push failure preserves commit — `hash` populated, `pushed: false` | ✅ Separate try/catch for push step |
| 5 | Empty `files` → error, no git ops | ✅ Validated before any git call |
| 6 | Empty `message` → error, no git ops | ✅ Validated before any git call |
| 7 | `planning_commit` added to `build` and `senior-coder` agent allowlists | ✅ Added to `~/.config/opencode/opencode.json` |

## Files Created

| File | Description |
|------|-------------|
| `~/.config/opencode/tools/planning_commit.ts` | Tool implementation |
| `~/.config/opencode/tools/tests/planning_commit.test.ts` | 7 unit tests |

## Test Results

```
✔ 1. extractCommitHash — parses hash from standard git commit output
✔ 2. extractCommitHash — returns empty string when no hash found
✔ 3. execute — empty files array → error result
✔ 4. execute — empty message → error result
✔ 5. execute — valid staged file + message → hash extracted, push fails gracefully
✔ 6. execute — files: [.] stages all modified files
✔ 7. execute — non-existent file in files list → git add error returned
tests 7 | pass 7 | fail 0
```

## Exported API

- `extractCommitHash(stdout: string): string` — parses commit hash from `git commit` stdout
- `execute(args, context): Promise<string>` — main tool logic, exported for testability
- `default` — registered `tool({...})` for OpenCode auto-discovery
