# ARC-1366 - Implement `get_checkpoint_context` and `send_checkpoint_notification` OpenCode tools

## Story
ARC-1366 — Implement `get_checkpoint_context` and `send_checkpoint_notification` OpenCode tools

## Lane
agent-tools

## Date
2026-07-07

## Commit
`7c0627a` in `pa.aid.config.md` repo, branch `ARC-1366`

## What was built:

1. `~/.config/opencode/tools/get_checkpoint_context.ts` — Loads live runbook + filesystem state for a paused checkpoint story. Accepts `runbook_path`, `story_key`, `instruction`. Exports `validateGetCheckpointArgs`, `findActiveStory`, `resolveImplementationPlanPath`, `buildContext` helpers. Returns `{ success: true, context: { story_key, title, lane, last_step_reached, implementation_plan_exists, implementation_plan_path, instruction } }`.

2. `~/.config/opencode/tools/send_checkpoint_notification.ts` — Emits a structured completion notification. Accepts `story_key`, `instruction`, `summary`. Exports `validateSendNotificationArgs`, `buildNotification`. Logs notification JSON to stdout and returns `{ success: true, notification: { story_key, instruction, summary, timestamp } }`.

3. `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts` — 7 test scenarios (success no plan, success plan exists, story not found, already checked off, missing instruction, missing runbook_path, last_step_reached logic). All pass.

4. `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts` — 4 test scenarios (success with ISO timestamp, missing story_key, missing instruction, missing summary). All pass.

5. Allowlists updated: `get_checkpoint_context` and `send_checkpoint_notification` added to build and senior-coder agent allowlists in both `opencode-sync-config.sh` (`pa.aid.wsl-setup.sh`) and `opencode.json` (`pa.aid.config.md`).

## Test results:
11 new tests pass; full suite 171/171 tests pass.

## Acceptance criteria coverage:
- AC1: active checkpoint → `{ success: true, context }` — covered by tests 1-2
- AC2: plan file exists → `implementation_plan_exists: true` — covered by test 2
- AC3: not found/already checked → error — covered by tests 3-4
- AC4: missing param → errors array — covered by tests 5-6
- AC5: send notification → `{ success: true, notification }` with stdout — covered by send test 1
- AC6: send missing param → errors array — covered by send tests 2-4