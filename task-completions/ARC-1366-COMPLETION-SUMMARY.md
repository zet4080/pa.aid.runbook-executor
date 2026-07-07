# ARC-1366 Implement `get_checkpoint_context` and `send_checkpoint_notification` OpenCode tools — Completion Summary

**Issue:** `issues/agent-tools/ARC-1366-tool-resume-checkpoint.md`
**Implementation Plan:** `implementation_plans/agent-tools/ARC-1366-implementation-plan.md`
**Completed:** 2026-07-07
**Lane:** agent-tools
**Commit:** `7c0627a` in `pa.aid.config.md` repo, branch `ARC-1366`

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given active checkpoint w/ runbook_path+story_key+instruction → returns `{ success: true, context: {...} }` populated from live state | Passed | Tests 1–2 in `get_checkpoint_context.test.ts`: `1. Success — active checkpoint, no plan file` and `2. Success — active checkpoint, plan file exists` both pass |
| 2 | Given plan file exists → `implementation_plan_exists: true` with resolved absolute path | Passed | Test 2: `2. Success — active checkpoint, plan file exists → implementation_plan_exists: true` — writes temp plan file, confirms exists=true and path resolves correctly |
| 3 | Story not found or already checked off → `{ success: false, error: "No active checkpoint found for <key>" }` | Passed | Tests 3–4: `3. Story not found in runbook → findActiveStory returns null` and `4. Story already checked off → findActiveStory returns null` |
| 4 | Missing required param → `{ success: false, errors: ["<field> is required"] }` without file reads | Passed | Tests 5–6: `5. Missing instruction param` and `6. Missing runbook_path param` — validation short-circuits before any I/O |
| 5 | send_checkpoint_notification with story_key+instruction+summary → `{ success: true, notification: {..., timestamp} }` and logs to stdout | Passed | Test 1 in `send_checkpoint_notification.test.ts`: `1. Success — buildNotification returns correct shape with ISO timestamp` |
| 6 | send_checkpoint_notification missing param → `{ success: false, errors: [...] }` without emitting notification | Passed | Tests 2–4: `2. Missing story_key`, `3. Missing instruction`, `4. Missing summary` — no stdout emission on validation failure |

## Implementation Summary

Two focused OpenCode tools were created to enable checkpoint resume with supervisor instruction:

- **`get_checkpoint_context.ts`**: Uses `parseRunbook()` from `./lib/parser.ts` to load live runbook state, locates the story by key (stripping namespace prefix via `step.id.split('__').pop()`), determines `last_step_reached` from the last checked sub-step, resolves the implementation plan path using `dirname(dirname(runbook_path))/implementation_plans/{lane}/{story_key}-implementation-plan.md`, and returns the full context package including the supervisor's instruction.

- **`send_checkpoint_notification.ts`**: Validates required fields, builds a notification object with ISO timestamp via `buildNotification()`, emits it via `console.log(JSON.stringify(notification))` to stdout so OpenCode session UI captures it, and returns `{ success: true, notification }`.

Both tools follow the one-tool-per-file pattern with `export default tool({...})` from `@opencode-ai/plugin`. Both tool names were added to the `build` and `senior-coder` allowlists in `opencode-sync-config.sh` and `opencode.json`.

## Verification Steps

```bash
# Run ARC-1366 tests specifically
node --experimental-strip-types --test ~/.config/opencode/tools/tests/get_checkpoint_context.test.ts
node --experimental-strip-types --test ~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts

# Run full tools test suite
node --experimental-strip-types --test ~/.config/opencode/tools/tests/*.test.ts

# Verify allowlist entries
grep "get_checkpoint_context\|send_checkpoint_notification" ~/.config/opencode/opencode.json
```

## Tests Added

- `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts` — 7 tests covering: success (no plan), success (plan exists), story not found, already checked off, missing instruction, missing runbook_path, last_step_reached logic
- `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts` — 4 tests covering: success with ISO timestamp, missing story_key, missing instruction, missing summary

**Full suite result:** 171 tests pass, 0 fail (includes all 11 new ARC-1366 tests)

## Rollback Notes

None. These are additive new tool files. Removal is as simple as deleting the two tool files and two test files. No migrations.

## Next Steps

None. ARC-1366 is the final story in the Wave 2 agent-tools batch. With this story complete, the Wave 2 gate can be closed.
