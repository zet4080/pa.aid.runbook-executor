# ARC-1294 Update Runbook Checkbox State in Real Time — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1294-update-checkbox-state.md`
**Completion Summary:** `task-completions/ARC-1294-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Synchronous write-back already in place — verify concurrent safety, close NOTE debt, add crash-recovery tests
**Owner:** build-agent
**Date:** 2026-06-30

---

## Scope & Alignment

The core checkbox write-back feature was implemented as part of the ARC-1293 fix commit (`d7ff6e2`). The following is already in production on the `runbook-executor` integration branch:

- `packages/server/src/runbook/markdownWriter.ts` — `markStepCheckedInMarkdown` and `markSubStepsCheckedInMarkdown`, both using atomic write (`.tmp` → rename).
- `packages/server/src/runner/laneRunner.ts` calls both write-back functions at the correct points: step success with no pending gate → `markStepCheckedInMarkdown`; step success with pending checkpoint gate → `markSubStepsCheckedInMarkdown` for pre-gate sub-steps.
- 19 tests in `markdownWriter.test.ts` and 7 write-back tests in `laneRunner.test.ts` already pass.
- Crash recovery via `step.checked === true` skip-guard in `runLane` is already implemented.

**What this plan addresses:**

| AC | Status | Remaining work |
|----|--------|----------------|
| AC 1 — checkbox flips within 1 second of step completion | ✅ Implemented | Confirm no async gap; add inline comment; close `NOTE(ARC-1294)` debt in `sidecar.ts` and `types.ts` |
| AC 2 — paused/checkpoint steps NOT checked, only completed steps | ✅ Implemented | Add explicit concurrent-lane test confirming checkpoint paths skip `markStepCheckedInMarkdown` |
| AC 3 — crash/restart: markdown accurately reflects completed steps | ✅ Implemented (skip-guard) | Add explicit crash-recovery integration test in `laneRunner.test.ts` |

**AC mapping:**

| AC | Satisfied by |
|----|-------------|
| AC 1 — checkbox flips within 1 second | `markStepCheckedInMarkdown` called synchronously after `executeStep` resolves; no async gap between step completion and file write. Steps 1 and 2 confirm this. |
| AC 2 — paused/checkpoint steps NOT checked | `markStepCheckedInMarkdown` is only called in the no-pending-gate branch; checkpoint paths return before reaching the call. Step 3 adds explicit concurrent-write test coverage. |
| AC 3 — crash recovery reads markdown accurately | `runLane` skips `step.checked === true` steps — these values come from the parsed runbook, which is re-parsed from markdown at server startup. Step 4 adds an integration test exercising crash restart with pre-checked markdown. |

---

## Assumptions & Dependencies

- ARC-1291, ARC-1292, and ARC-1293 are merged to the `runbook-executor` integration branch. All existing `markdownWriter.ts` and `laneRunner.ts` write-back code is in production.
- Node.js single-threaded event loop: synchronous calls between `await` boundaries cannot interleave. The `getState()` → `setState()` → `writeSidecar()` block in `laneRunner.ts` is synchronous throughout — no `await` between these calls — so concurrent lanes cannot observe a torn state write.
- Similarly, `markStepCheckedInMarkdown` and `markSubStepsCheckedInMarkdown` use synchronous `readFileSync`/`writeFileSync`/`renameSync`. They cannot interleave with each other within a single Node.js process.
- The markdown file is the crash-recovery ground truth as stated in the issue. `SidecarState.stepState` is a secondary index; the authoritative skip-guard in `runLane` is `step.checked === true`, which comes from the parsed runbook (i.e., from the markdown file). The runbook cache is re-populated from disk at server startup, so a fresh parse reflects all checkboxes written before the crash.
- The worktree for ARC-1294 is branched off `runbook-executor`: `git -C /repos/pa.aid.wsl-setup.sh worktree add /repos/ARC-1294 -b ARC-1294 runbook-executor`.

---

## Implementation Steps

### Step 1: Add inline comment confirming synchronous write path

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Add a one-line comment immediately before the `markStepCheckedInMarkdown` call site:
`// Synchronous write-back — no await between executeStep resolution and this call; cannot interleave with other lanes (Node.js single thread). AC 1.`

Add a matching comment before the `markSubStepsCheckedInMarkdown` call in the sub-step checkpoint path.

**Verification:** `tsc --noEmit` in `packages/server` compiles cleanly. The comments are visible in the diff.

---

### Step 2: Remove NOTE(ARC-1294) debt markers

**Files:**
- `packages/server/src/state/sidecar.ts`
- `packages/server/src/state/types.ts`

**Action:** Remove the two `NOTE(ARC-1294)` comment blocks:

- In `sidecar.ts`: replace the `NOTE(ARC-1294)` lines inside the `writeSidecar` JSDoc with: `// Concurrent safety: the getState/setState/writeSidecar sequence in laneRunner is synchronous; no await between reads and writes, so concurrent lanes cannot interleave (Node.js single thread).`
- In `types.ts`: remove the two `NOTE(ARC-1294)` lines from the `stepState` field comment. The remaining comment `// Runbook markdown is always ground truth — markdown wins on any conflict.` is sufficient.

**Verification:** `grep -r "NOTE(ARC-1294)"` in `packages/server/src` finds no matches. `tsc --noEmit` passes.

---

### Step 3: Add concurrent-write safety test

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:** Add a test scenario in the `runLane — markdown write-back on step completion` describe block:

- **Concurrent checkpoint push — no state loss:** Fire two `runLane` calls concurrently (both with a single agent step that succeeds but has an unchecked `INDIVIDUAL PLAN CHECKPOINT` sub-step) using `Promise.all`. After both settle, assert that `getState().checkpointQueue` has length 2 — both checkpoint entries are present, not one overwriting the other. Assert `markStepCheckedInMarkdown` was NOT called (both lanes hit checkpoint gates). This confirms the synchronous `getState()` → `setState()` pattern accumulates correctly when lanes interleave across `await` boundaries.

Note: because `pushedSubStepCheckpoints` is a module-level Set, each step id must be unique to avoid the idempotency guard suppressing the second push. Use distinct step ids for each lane.

**Verification:** `vitest run` in `packages/server` passes all tests including the new scenario.

---

### Step 4: Add crash-recovery integration tests

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:** Add a new describe block `runLane — crash recovery from markdown state` with the following scenarios:

- **Partial restart — first step pre-checked, second runs:** `steps` array where the first step has `checked: true` and the second has `checked: false`. Run `runLane`. Assert `executeStep` is called exactly once (for the second step). Assert `markStepCheckedInMarkdown` is called with the second step's id. This confirms the skip-guard correctly resumes at the first uncompleted step.

- **Full restart — all steps pre-checked:** `steps` array where all steps have `checked: true`. Run `runLane`. Assert `executeStep` is not called. Assert lane transitions to `'done'`. This confirms that a fully-completed lane on restart exits cleanly without re-executing any work.

**Verification:** `vitest run` in `packages/server` passes all new tests.

---

### Step 5: Verify full test suite

**Files:** all test files under `packages/server/src/__tests__/`

**Action:** Run `vitest run` from `packages/server/`. Run `tsc --noEmit` from `packages/server/`. Both must pass with zero errors and no regressions.

**Verification:** Clean output from both commands. Zero regression against the baseline 230-test suite. New tests for Steps 3 and 4 pass.

---

## Testing & Validation

All automated verification is via `vitest run` and `tsc --noEmit` in `packages/server`.

**Scenario coverage:**

- **AC 1 — within-1-second checkbox flip:** The write path is synchronous after `await executeStep(...)` resolves. No polling, no deferred queue. The checkbox is updated in the same Node.js microtask continuation as step completion. Step 1 documents this by comment; existing `laneRunner.test.ts` write-back tests verify the call is made correctly.
- **AC 2 — paused/checkpoint steps NOT checked:** Existing test `does NOT call markStepCheckedInMarkdown when step succeeds but checkpoint gate is pending` in `laneRunner.test.ts` covers the single-lane case. Step 3 adds a concurrent two-lane variant.
- **AC 3 — crash recovery:** Step 4 adds two explicit scenarios: partial restart (first step pre-checked, second runs) and full restart (all pre-checked, lane exits done without executing).
- **Concurrent write safety:** Step 3 confirms both concurrent checkpoint queue entries are preserved — no state loss under concurrent lane execution.
- **Atomic markdown write:** Existing tests in `markdownWriter.test.ts` confirm `.tmp` file is absent after write and content is correct.

**End-to-end smoke test (manual):**
1. Start the server (`npm run dev` in `runbook-executor/`) with `REPO_ROOT` pointing to a planning repo.
2. Select two runbooks; click "Start Session".
3. As each lane's agent steps complete, confirm the corresponding top-level checkboxes flip from `[ ]` to `[x]` in the markdown file within ~1 second.
4. When a checkpoint is hit, confirm the lane's in-progress step checkbox is NOT checked (lane paused).
5. Kill the server. Restart. Confirm the runbook file still reflects the correct checkbox states from before the crash. Re-start the session and confirm already-checked steps are skipped.

---

## Risks & Open Questions

- **`step.checked` comes from parser at session-start time, not live markdown:** The skip-guard in `runLane` uses `step.checked` as passed at `runLane` invocation time. The runbook cache is re-populated at server startup from disk, so crash recovery correctly picks up all checkboxes written before the crash. The risk only materializes if a live session's runbook markdown is externally edited mid-session — out of scope for this story.
- **Two lanes checking the same step ID:** The runbook parser namespaces step IDs by lane, so two lanes can never share a step ID. `markStepCheckedInMarkdown` matches on the de-namespaced raw ID (e.g. `ARC-1294`) — if two runbooks contain the same step key, the first matching line would be updated. This is not expected in practice; runbooks have distinct steps. No mitigation needed.
- **NOTE debt markers:** Two comment blocks referencing `NOTE(ARC-1294)` remain in `sidecar.ts` and `types.ts`. Step 2 removes them. If Step 2 is skipped, these comments will remain as misleading TODOs after ARC-1294 closes.
