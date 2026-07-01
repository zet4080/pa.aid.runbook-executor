# ARC-1291 Execute Runbook Steps Sequentially Within a Lane — Implementation Plan
**Issue:** `issues/parallel-lane-execution/ARC-1291-sequential-step-execution.md`
**Completion Summary:** `task-completions/ARC-1291-COMPLETION-SUMMARY.md` (TBD)
**Approach:** In-memory parsed runbook cache + `laneRunner` module
**Owner:** agent
**Date:** 2026-06-30

## Scope & Alignment

This plan delivers the sequential step execution engine — the foundation that all subsequent parallel-lane-execution stories (ARC-1292, ARC-1293, ARC-1294) and the entire checkpoint-management lane (ARC-1301+) depend on.

Acceptance criteria mapping:

- **AC 1** (steps execute in order; N+1 does not start until N completes): covered by Steps 3 and 5 — `laneRunner` iterates steps sequentially using `await executeStep()`, and wiring to session start fires that runner.
- **AC 2** (manual step → lane pauses, supervisor notified): covered by Steps 1 and 3 — `manual` is removed from the type system; what was `manual` is now correctly `checkpoint`. `laneRunner` pauses on any `checkpoint` step and pushes to `checkpointQueue`.
- **AC 3** (agent step → opencode run invoked, output streams to UI): covered by Steps 2, 3, 3a, 5 — buildPromptFromStep constructs the prompt from step metadata; executeStep handles subprocess invocation and event-bus streaming; laneRunner calls it; sessions route wires lane start.

## Assumptions & Dependencies

- ARC-1288 is merged: `executeStep(stepId, prompt)` in `packages/server/src/runner/stepRunner.ts` is the single-step execution entry point and is production-ready.
- ARC-1286 is merged: `parseRunbook(filePath)` produces a `ParsedRunbook` with typed `waves[]` → `steps[]`. Step IDs are already namespaced by lane.
- The requirements document (`docs/requirements/2026-06-27-runbook-executor.md`, Section 5 Execution) states: *"Runbook steps must be explicitly typed (`agent` / `manual`) — tool does not infer step type from natural language."* Open Question 3 resolution in the same document confirms steps are typed via structured markers (e.g. `agent:`, `prompt:`, `command:`), not natural-language inference. In the runbook format used by this project, the 🔴/🟡/🟢 checkpoint markers are the explicit type signal: HIGH/MEDIUM/LOW map to checkpoint-or-agent. Steps with no keyword are agent steps (they run the opencode CLI without a supervisor gate). This makes `none → agent` the correct fallback, not `none → manual` as currently coded.
- `POST /api/sessions/start` in `packages/server/src/routes/sessions.ts` already has a comment: *"ARC-1291 will extend it to spawn actual lane executor processes."* This is the correct hook point.
- `SidecarState.selectedRunbooks` holds the absolute paths of runbooks chosen for the session. The lane runner resolves parsed runbooks by path from the cache.
- `SidecarState.checkpointQueue: CheckpointQueueEntry[]` is the existing supervisor notification channel. Checkpoint steps push to this queue and halt the lane.
- The `label` and `subSteps` fields on `RunbookStep` are the available metadata for constructing an agent prompt. `buildPromptFromStep` (Step 3a) derives the prompt from these fields. A dedicated `prompt:` field in the runbook format is anticipated by the requirements but not yet present; when added, `buildPromptFromStep` should prefer it over the derived form.
- `writeSidecar` is not yet serialized across concurrent lane writes — the existing `NOTE(ARC-1294)` comment in `sidecar.ts` documents this known limitation. Since ARC-1291 only runs one lane (no concurrency yet), this is acceptable. ARC-1294 will address concurrent writes.
- Lane execution is fire-and-forget relative to the HTTP response. `POST /api/sessions/start` returns immediately after setting `sessionStartedAt`; lane execution runs asynchronously in the background.

## Implementation Steps

### Step 1: Fix the step type system

**Files:**
- `packages/server/src/runbook/types.ts`
- `packages/server/src/runbook/astWalker.ts`

**Action:**

In `types.ts`, remove `'manual'` from the `StepType` union. The corrected type is `'agent' | 'checkpoint'` only.

In `astWalker.ts`, update `deriveStepTypeAndLevel` with the corrected mapping:
- `HIGH` → `{ type: 'checkpoint', checkpointLevel: 'high' }` (unchanged)
- `MEDIUM` → `{ type: 'agent', checkpointLevel: 'medium' }` (unchanged)
- `LOW` → `{ type: 'checkpoint', checkpointLevel: 'low' }` (was `manual` — fix to `checkpoint`)
- no keyword → `{ type: 'agent', checkpointLevel: null }` (was `manual` — fix to `agent`)

No other logic in `astWalker.ts` references `StepType` by value — the type is passed through from `deriveStepTypeAndLevel` into `parseStepsFromList` and then into each `RunbookStep`. There are no switch statements or type guards on `'manual'` elsewhere in this file.

Scan for any other files that reference the string literal `'manual'` as a `StepType` value and update them to use `'agent'` or `'checkpoint'` as appropriate. Expected locations: none beyond `astWalker.ts` based on Phase 0 exploration, but the scan must confirm.

**Verification:** TypeScript compiles without error (`npm run build` in `packages/server`). Existing parser tests in `packages/server/src/runbook/parser.test.ts` and `__tests__/` pass. Any test fixture that asserted `type: 'manual'` must be updated to `'checkpoint'` (LOW steps) or `'agent'` (no-keyword steps).

---

### Step 2: Create the parsed runbook cache module

**Files:** `packages/server/src/runner/runbookCache.ts` (new)

**Action:** Create a module that holds the in-memory cache of parsed runbooks. The cache is a `Map<string, ParsedRunbook>` keyed by absolute runbook file path.

Exports:
- `setCache(runbooks: ParsedRunbook[]): void` — replaces the entire cache with the provided runbooks, keyed by `runbook.metadata.lane` or the file path. Since the cache must be queryable by the file path used in `selectedRunbooks`, the key must be the file path. The `ParsedRunbook` type does not carry its source path; `setCache` must accept pairs `Array<{ path: string; runbook: ParsedRunbook }>` or the caller must zip paths with runbooks. Use `Array<{ path: string; runbook: ParsedRunbook }>` as the parameter type to be unambiguous.
- `getRunbook(path: string): ParsedRunbook | undefined` — returns the cached runbook for a given absolute path, or `undefined` if not found.
- `_resetCacheForTesting(): void` — clears the cache; exposed for test isolation only, not part of the public contract.

In `packages/server/src/index.ts`, after `const parsedRunbooks = runbookFiles.map(parseRunbook)`, call `setCache` with the zipped path+runbook pairs before `setState`.

**Verification:** Unit test: call `setCache` with two entries, verify `getRunbook` returns the correct `ParsedRunbook` for each path and `undefined` for an unknown path. Verify `_resetCacheForTesting` clears the cache. No changes to existing server startup tests.

---

### Step 3: Create the `laneRunner` module

**Files:** `packages/server/src/runner/laneRunner.ts` (new)

**Action:** Create the sequential lane execution engine.

**Lane state tracking:** a module-level `Map<string, LaneStatus>` where `LaneStatus = 'idle' | 'running' | 'paused' | 'done'` and the key is the runbook file path. Expose `getLaneState(runbookPath: string): LaneStatus` for consumers (ARC-1293 streaming UI, ARC-1292 concurrency manager).

**Primary function signature:**

```ts
export async function runLane(
  runbookPath: string,
  steps: RunbookStep[],
): Promise<void>
```

**Execution logic:**

1. Set lane status to `'running'`.
2. Iterate `steps` in array order. For each step:
   - If `step.checked` is `true`: skip (already completed — respects markdown ground truth on resume).
   - If `step.type === 'agent'`: call `await executeStep(step.id, step.label)`. On `RunResult.outcome === 'failed'`, stop iteration and set lane status to `'paused'` (ARC-1306 will add retry logic here). On `'success'`, continue to the next step.
   - If `step.type === 'checkpoint'`: read current state via `getState()`, push a `CheckpointQueueEntry` for this step to `state.checkpointQueue`, call `setState` and `writeSidecar` to persist the updated queue, set lane status to `'paused'`, and stop iteration. Do not call `executeStep` for checkpoint steps.
3. If the loop completes without hitting a checkpoint or failure, set lane status to `'done'`.

The `CheckpointQueueEntry` pushed for a checkpoint step must include: `stepId: step.id`, `runbookPath`, `checkpointLevel: step.checkpointLevel ?? 'low'`, `hitAt: new Date().toISOString()`.

**Exports:** `runLane`, `getLaneState`, `LaneStatus` type, `_resetLaneStateForTesting` (test isolation).

**Verification:** Unit tests in `__tests__/laneRunner.test.ts` (Step 6). TypeScript compiles without error.

---

### Step 3a: Add `buildPromptFromStep` helper to `laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Add an internal (non-exported) helper function with the signature:

```ts
function buildPromptFromStep(step: RunbookStep): string
```

The function constructs a prompt string from the step's existing fields:
- Opens with the step identity line: `"You are executing step {step.id}: {step.label}."`
- If `step.subSteps` is non-empty and at least one sub-step has `checked: false`, appends `"Complete the following tasks in order:"` followed by each unchecked sub-step label as a numbered list item (checked sub-steps are omitted from the list).
- If `step.subSteps` is empty, or all sub-steps are already `checked: true`, appends `"The sub-steps are already completed — verify and confirm."` instead of the numbered list.

In the agent step execution path of `runLane`, replace the direct use of `step.label` as the `executeStep` prompt with a call to `buildPromptFromStep(step)`.

**Verification:** Unit test scenarios in `__tests__/laneRunner.test.ts` (added in Step 6): given a step with label and two unchecked sub-steps, assert the returned string contains the step id, the label, and both sub-step labels in numbered form; given a step with no sub-steps, assert the returned string contains the identity line and the "already completed" clause with no numbered list.

---

### Step 5: Wire `laneRunner` to `POST /api/sessions/start`

**Files:** `packages/server/src/routes/sessions.ts`

**Action:** After the existing `writeSidecar` / `setState` block that records `sessionStartedAt`, add the lane-launch logic:

1. Read `state.selectedRunbooks` (already available as `current.selectedRunbooks`).
2. For each runbook path in `selectedRunbooks`:
   a. Call `getRunbook(path)` from `runbookCache`. If the cache returns `undefined` for a path, log a warning and skip that runbook.
   b. Collect all unchecked steps from all waves in the parsed runbook, flattening waves into a single ordered step array. Steps with `checked: true` are included in the array (the lane runner skips them individually) to preserve ordering.
   c. Fire `runLane(path, steps)` as a detached async task: `void runLane(path, steps).catch(...)`. Do not await. Log errors from the catch handler.
3. The HTTP response (`res.json({ ok: true, sessionStartedAt })`) is returned before the lane tasks begin executing — this is the intended fire-and-forget behavior.

Import `getRunbook` from `../runner/runbookCache.js` and `runLane` from `../runner/laneRunner.js`.

**Verification:** Integration test: mock `runLane` via `vi.mock`, call `POST /api/sessions/start` with a selected runbook in state, assert `runLane` was called with the correct path and step array. Assert the HTTP response is `200` with `sessionStartedAt` set. Assert the response does not wait for `runLane` to resolve (use a mock that never resolves and confirm the response arrives immediately).

---

### Step 6: Add tests for `laneRunner`

**Files:** `packages/server/src/__tests__/laneRunner.test.ts` (new)

**Action:** Write unit tests covering the following scenarios. Mock `executeStep` via `vi.mock('../runner/stepRunner.js')` and `writeSidecar` / `setState` via their respective module mocks.

Scenarios:
- **Sequential ordering:** given three agent steps, assert `executeStep` is called in step-array order and each call awaits the previous (use ordered mock responses to confirm).
- **Skip checked steps:** given a step with `checked: true` followed by an unchecked agent step, assert `executeStep` is called only once (for the unchecked step).
- **Checkpoint step pauses lane:** given an agent step followed by a checkpoint step, assert `executeStep` is called for the agent step, then the checkpoint pushes to `checkpointQueue`, lane status transitions to `'paused'`, and the subsequent steps are not executed.
- **Failed agent step pauses lane:** given an agent step whose `executeStep` resolves with `outcome: 'failed'`, assert the lane status transitions to `'paused'` and the next step is not executed.
- **All steps complete → done:** given two agent steps that both succeed, assert lane status transitions to `'done'` after the second step.
- **Lane status tracking:** call `getLaneState` before, during (with a never-resolving mock), and after execution to verify `'idle'` → `'running'` → `'done'` transitions.
- **`buildPromptFromStep` with sub-steps:** given a step with a label and two unchecked sub-steps, assert the returned string contains the step id, the label, and both sub-step labels in numbered form.
- **`buildPromptFromStep` without sub-steps:** given a step with a label and an empty `subSteps` array, assert the returned string contains the identity line and no numbered list.

**Verification:** `npm test` in `packages/server` passes all tests including the new suite.

---

### Step 7: Tick the cross-lane dependency gate

**Files:** `docs/plans/runbook-parallel-lane-execution.md` (planning repo)

**Action:** Check the gate checkbox `- [ ] Confirm core-infrastructure ARC-1288 merged` at the top of Wave 1 in the runbook, since ARC-1288 is confirmed merged (it is the basis for `executeStep` already in the codebase).

**Verification:** The checkbox reads `- [x] Confirm core-infrastructure ARC-1288 merged` after the edit. Commit this change to the planning repo on `main` as part of the completion summary commit.

---

## Testing & Validation

**Type system correctness:**
- `StepType` compiles as `'agent' | 'checkpoint'` with no `'manual'` variant.
- Re-run the full parser test suite: all existing runbook fixtures parse without TypeScript errors; steps previously typed as `'manual'` now appear as `'checkpoint'` (LOW-level) or `'agent'` (no keyword).

**Unit test suite (`npm test --workspace=packages/server`):**
- All pre-existing tests pass without modification except fixture updates for `type: 'manual'` → `'checkpoint'` or `'agent'`.
- New `runbookCache.test.ts`-level scenarios pass (may be co-located in an existing test file).
- New `laneRunner.test.ts` suite passes all scenarios described in Step 6.
- Sessions route test: `POST /api/sessions/start` fires `runLane` per selected runbook and returns immediately.

**End-to-end smoke test (manual):**
1. Start the server (`npm run dev` in `/repos/pa.aid.conductor.ts`).
2. Confirm a runbook is selected via `PATCH /api/state` (set `selectedRunbooks`).
3. Call `POST /api/sessions/start`.
4. Verify the response returns `{ ok: true, sessionStartedAt: <ISO> }` immediately.
5. Poll `GET /api/state` and observe `stepState` updating as agent steps complete.
6. When a checkpoint step is reached, verify `checkpointQueue` contains a new entry and `getLaneState` reports `'paused'` (via a debug log or temporary diagnostic endpoint).

## Risks & Open Questions

- **Prompt fidelity:** `buildPromptFromStep` uses `step.label` and `step.subSteps` to construct the agent instruction. If a runbook step's label is terse or its sub-steps are purely administrative (e.g. "🔒 Claimed:" entries), the generated prompt may be sparse. The requirements anticipate an explicit `prompt:` field in the runbook format; when that is added, `buildPromptFromStep` should check `step.prompt ?? buildFromLabel(step)` as a fallback chain. This is out of scope for ARC-1291.
- **Lane resume after restart:** When the server restarts mid-session, `sessionStartedAt` is preserved in the sidecar but no lane is running. The supervisor must re-call `POST /api/sessions/start` to resume. This is acceptable for ARC-1291 scope. Full resume-from-checkpoint is a future concern (ARC-1301+).
- **Concurrent writes:** `writeSidecar` in `laneRunner` (for checkpoint queue updates) is not yet serialized against concurrent lane writes. This is safe for ARC-1291 since only one lane runs at a time. ARC-1294 will introduce a write queue; `laneRunner` must be updated to use it at that point.
- **`manual` in existing test fixtures:** Any test that asserts `type: 'manual'` will need updating. A grep for `'manual'` across `__tests__/` and `__fixtures__/` before starting implementation will scope the cleanup work.
