# ARC-1366 `get_checkpoint_context` and `send_checkpoint_notification` — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1366-tool-resume-checkpoint.md`
**Completion Summary:** `task-completions/ARC-1366-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Two focused tool files — pattern established by existing agent-tools lane tools
**Owner:** agent-tools
**Date:** 2026-07-07

---

## Scope & Alignment

Create two new OpenCode tools:

| Tool file | Purpose | AC covered |
|---|---|---|
| `~/.config/opencode/tools/get_checkpoint_context.ts` | Load live runbook + filesystem state for a paused checkpoint; embed supervisor instruction | AC 1–4 |
| `~/.config/opencode/tools/send_checkpoint_notification.ts` | Emit structured completion notification once instruction executed | AC 5–6 |
| `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts` | Unit tests: success, plan-exists, not-found, already-checked, missing-param | AC 1–4 |
| `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts` | Unit tests: success, missing-param | AC 5–6 |
| `opencode-sync-config.sh` + `opencode.json` | Add both tools to `build` and `senior-coder` allowlists | In Scope |

---

## Assumptions & Dependencies

- `parseRunbook()` available at `./lib/parser.ts`; `ParsedRunbook` type at `./lib/types.ts`
- `@opencode-ai/plugin` available at `~/.config/opencode/node_modules/`
- `existsSync` from `node:fs`; `join`, `dirname` from `node:path`
- Planning repo root is derivable from `runbook_path` by going up two levels: `dirname(dirname(runbook_path))` (runbook lives at `{root}/docs/plans/runbook-*.md`)
- Implementation plan path pattern: `{planningRepoRoot}/implementation_plans/{lane}/{story_key}-implementation-plan.md`
- `lane` is `parsed.metadata.lane` (e.g. `"agent-tools"`)
- Story namespace prefix must be stripped: `step.id.split('__').pop()` to match `story_key`
- "Active checkpoint" = story exists in runbook AND `step.checked === false`
- `last_step_reached` = label of last sub-step where `checked === true`; `null` if no sub-steps are checked
- `console.log(JSON.stringify(notification))` emits to stdout captured by OpenCode session UI
- Bun test runner: `bun test` invoked from the `tests/` directory

---

## Implementation Steps

### Step 1: Create `~/.config/opencode/tools/get_checkpoint_context.ts`

**Files:** `~/.config/opencode/tools/get_checkpoint_context.ts`

**Exported helpers (for testability):**

- `validateGetCheckpointArgs(args): string[]` — returns array of error strings for any empty/missing field among `runbook_path`, `story_key`, `instruction`
- `findActiveStory(parsed: ParsedRunbook, story_key: string): RunbookStep | null` — walks `parsed.waves[].steps[]`, strips namespace prefix, returns step if `checked === false`; returns `null` if not found or already checked
- `resolveImplementationPlanPath(runbook_path: string, lane: string, story_key: string): string` — computes `join(dirname(dirname(runbook_path)), 'implementation_plans', lane, `${story_key}-implementation-plan.md`)`
- `buildContext(step: RunbookStep, parsed: ParsedRunbook, runbook_path: string, instruction: string): object` — assembles and returns the context object

**`execute()` logic:**
1. `validateGetCheckpointArgs` — if errors, return `{ success: false, errors }`
2. `parseRunbook(runbook_path)` — catch parse errors; return structured error if file unreadable
3. `findActiveStory(parsed, story_key)` — if null, return `{ success: false, error: "No active checkpoint found for <story_key>" }`
4. `resolveImplementationPlanPath(runbook_path, parsed.metadata.lane, story_key)` — check `existsSync`
5. `last_step_reached` — scan `step.subSteps` for last entry where `checked === true`; take `label`; null if none
6. Return `{ success: true, context: { story_key, title, lane, last_step_reached, implementation_plan_exists, implementation_plan_path, instruction } }`

**Verification:** Tests in Step 4 pass; direct invocation in OpenCode returns expected JSON shape.

---

### Step 2: Create `~/.config/opencode/tools/send_checkpoint_notification.ts`

**Files:** `~/.config/opencode/tools/send_checkpoint_notification.ts`

**Exported helpers:**

- `validateSendNotificationArgs(args): string[]` — returns error strings for empty/missing `story_key`, `instruction`, `summary`
- `buildNotification(story_key, instruction, summary): object` — returns `{ story_key, instruction, summary, timestamp: new Date().toISOString() }`

**`execute()` logic:**
1. `validateSendNotificationArgs` — if errors, return `{ success: false, errors }`
2. Build notification via `buildNotification(story_key, instruction, summary)`
3. `console.log(JSON.stringify(notification))` — emit to stdout
4. Return `{ success: true, notification }`

**Verification:** Tests in Step 5 pass; invocation in OpenCode logs notification to stdout.

---

### Step 3: Update allowlists in `opencode-sync-config.sh` and `opencode.json`

**Files:**
- `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
- `~/.config/opencode/opencode.json`

**Action:** Add `"get_checkpoint_context": True` and `"send_checkpoint_notification": True` to the `tools` dict for both `build` and `senior-coder` agents in `opencode-sync-config.sh`. Apply the same additions directly to `~/.config/opencode/opencode.json` for immediate use without waiting for a full wsl-setup run.

**Verification:** `grep -c get_checkpoint_context opencode-sync-config.sh` returns 2 (build + senior-coder); same in `opencode.json`.

---

### Step 4: Create `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts`

**Files:** `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts`

**Test scenarios (using Bun test runner):**

| Scenario | What to verify |
|---|---|
| Success — active checkpoint, no plan | Returns `{ success: true, context }` with `implementation_plan_exists: false` |
| Success — active checkpoint, plan exists | Returns `{ success: true, context }` with `implementation_plan_exists: true` and correct path |
| Story not found in runbook | Returns `{ success: false, error: "No active checkpoint found for <key>" }` |
| Story already checked off | Returns `{ success: false, error: "No active checkpoint found for <key>" }` |
| Missing `instruction` param | Returns `{ success: false, errors: ["instruction is required"] }` |
| Missing `runbook_path` param | Returns `{ success: false, errors: ["runbook_path is required"] }` |
| `last_step_reached` logic | Returns label of last checked sub-step; `null` when none checked |

Tests import `validateGetCheckpointArgs`, `findActiveStory`, `resolveImplementationPlanPath`, `buildContext` directly (not via the tool entry point) to stay pure/synchronous where possible.

**Verification:** `bun test tests/get_checkpoint_context.test.ts` exits 0, all cases pass.

---

### Step 5: Create `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts`

**Files:** `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts`

**Test scenarios:**

| Scenario | What to verify |
|---|---|
| Success | Returns `{ success: true, notification: { story_key, instruction, summary, timestamp } }` with ISO timestamp |
| Missing `story_key` | Returns `{ success: false, errors: ["story_key is required"] }` |
| Missing `instruction` | Returns `{ success: false, errors: ["instruction is required"] }` |
| Missing `summary` | Returns `{ success: false, errors: ["summary is required"] }` |

Tests import `validateSendNotificationArgs` and `buildNotification` directly.

**Verification:** `bun test tests/send_checkpoint_notification.test.ts` exits 0, all cases pass.

---

### Step 6: Run full test suite

**Files:** `~/.config/opencode/tools/tests/`

**Action:** Run `bun test tests/get_checkpoint_context.test.ts tests/send_checkpoint_notification.test.ts` from `~/.config/opencode/tools/`. Verify all tests pass with exit code 0.

**Verification:** Zero test failures; no `console.error` output from Bun runtime.

---

## Testing & Validation

- All unit tests run via `bun test tests/get_checkpoint_context.test.ts tests/send_checkpoint_notification.test.ts`
- Each AC item has a corresponding test scenario (see Steps 4–5)
- Allowlist verification: `grep get_checkpoint_context opencode-sync-config.sh | wc -l` = 2

---

## Risks & Open Questions

- `console.log` in `send_checkpoint_notification` emits to Bun stdout which OpenCode captures in the session stream — verify it doesn't interfere with the JSON return value (they are separate channels)
- `parsed.metadata.lane` may be `null` for runbooks without a `**Lane:**` metadata line — add null guard in `resolveImplementationPlanPath()` and return `{ success: false, error: "Lane metadata not found in runbook" }` if null
