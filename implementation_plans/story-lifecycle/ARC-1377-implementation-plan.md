# ARC-1377 Track agent self-review (local-code-review) state explicitly in conductor state - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1377-track-agent-self-review-state.md`
**Completion Summary:** `task-completions/ARC-1377-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. This story follows two precedents already set in this epic: (1) the additive-optional-sidecar-field + pure-helper-function pattern used by ARC-1374 for `storyLifecycle` (`getStoryLifecycleState`/`setStoryLifecycleState`), applied here to a new `selfReview` field; and (2) the text-scanning-of-agent-events pattern already used by `laneRunner.ts`'s existing `extractLastAgentMessage` helper, applied here to detect the local-code-review skill's own structured output markers (`✅ Local review clean (iteration N/3).` / `🔴 ESCALATE — local review iteration N/3 still has unresolved findings.`, both verbatim strings already mandated by `~/.config/opencode/skills/local-code-review/SKILL.md` Steps 4–5). No competing design exists to choose between — the issue's Out of Scope explicitly forbids changing the skill's own logic, which makes "read the skill's existing mandated output text" the only viable signal source. Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that it may be skipped when exploration reveals a clear existing pattern to follow.
**Owner:** build agent
**Date:** 2026-07-12

## Scope & Alignment

This plan covers exactly the two In-Scope items from the issue:

1. A new sidecar state field recording the local-code-review outcome (`'clean' | 'escalated'`) and iteration count per story, plus pure read/write helpers for it (mirroring `getStoryLifecycleState`/`setStoryLifecycleState` from ARC-1374).
2. `laneRunner.ts` reading the agent's emitted events after an agent step's `executeStep` call resolves, detecting whether the local-code-review skill's own structured pass/escalate marker text is present anywhere in those events, and — when detected — writing the new self-review record and, on the escalated branch only, transitioning the story's canonical `StoryLifecycleState` (from ARC-1374, already merged to `main`) to `awaiting_escalation_checkpoint` and pausing the lane with a new checkpoint queue entry.

Explicitly NOT covered (per issue's Out of Scope): no change to `~/.config/opencode/skills/local-code-review/SKILL.md` or its internal loop/iteration-cap logic. This plan only reads the skill's already-mandated output text; it does not alter what the skill does or how many iterations it runs.

Acceptance criteria mapping:
- **AC1** (outcome + iteration count recorded when the loop completes, pass or exhausted) → Step 1 (types) + Step 2 (`setSelfReviewRecord`) + Step 4 (wiring both branches in `runLane`) + Steps 3 & 5 (tests proving persistence for both the clean and escalated cases).
- **AC2** (clean → `laneRunner.ts` can determine "self-review complete, clean" without re-invoking the skill) → Step 2 (`getSelfReviewRecord` — pure read, no skill invocation) + Step 5 (test asserting the persisted record is queryable after the fact via `getSelfReviewRecord`).
- **AC3** (exhausted → story transitions to `awaiting_escalation_checkpoint`) → Step 4 (escalation branch calls `setStoryLifecycleState(..., 'awaiting_escalation_checkpoint')` and pushes a high-level `CheckpointQueueEntry`, pausing the lane) + Step 5 (test asserting `storyLifecycle[storyKey].state === 'awaiting_escalation_checkpoint'`, one checkpoint entry, and lane status `'paused'`).

## Assumptions & Dependencies

- **Dependency ARC-1374 (satisfied, merged to `main`):** `StoryLifecycleState` (including the `awaiting_escalation_checkpoint` value), `StoryLifecycleRecord`, `SidecarState.storyLifecycle`, and the pure helpers `getStoryLifecycleState`/`setStoryLifecycleState` are already present in `packages/server/src/state/types.ts` and `packages/server/src/state/sidecar.ts` on `main` (commit `28637bb`, "Merged in ARC-1374"). This plan imports and calls `setStoryLifecycleState` — it does not redefine or duplicate it.
- **The local-code-review skill's own iteration counter is what AC3 means by "exhausts its retry limit."** The issue's Goal sentence says "Surface the local-code-review skill's iteration count" — this is the skill's own internal loop counter (hardcoded max 3, per `local-code-review/SKILL.md`), which is a distinct concept from `SessionConfig.retryLimit` (the unrelated ARC-1306 subprocess-level retry mechanism already wired into `runLane`'s `while (retriesUsed < retryLimit ...)` loop). This plan does not touch `retryLimit`, `retriesUsed`, or the ARC-1306 retry loop's own checkpoint-push branch.
- **Detection happens after the subprocess-level retry loop, from the final `result`.** A single agent step's sub-step segment (e.g. "Execute plan", "Run `local-code-review`...", "Lint / tests pass", "Write completion summary", "Commit") is dispatched as one `executeStep` call per attempt; the local-code-review skill's own output (including intermediate "Iteration N/3" progress lines and the final clean/escalate verdict line) all land in that one call's `RunResult.events` array, regardless of how many ARC-1306 subprocess-level retries occurred first. This plan reads only the final, post-retry-loop `result.events`.
- **The agent process typically exits successfully (`result.outcome === 'success'`) even when local-code-review escalates internally** — escalating is the skill instructing the agent to report to the human and stop, not a crash or nonzero exit. This is why AC3's transition cannot be inferred from `result.outcome` alone and must come from scanning the events for the skill's own escalate marker text.
- **`stripNamespace(step.id)` is the correct storyKey convention** — already used for `claimKey` at the auto-claim call site in `runLane`, matching the convention documented in the ARC-1374 plan.
- **No route change needed.** `GET /api/state` already spreads the full `SidecarState` object (`res.json({ ...state, laneState })`), so the new `selfReview` field is automatically visible once populated — no change to `routes/state.ts`. `PATCH /api/state`'s allowlist is untouched — `selfReview` is internally managed only, like `stepLog` and (per ARC-1374) `storyLifecycle`.
- **No wiring into ARC-1378 in this story.** ARC-1378 (the new human implementation-review gate) depends on ARC-1377, not the reverse — it does not exist yet. `getSelfReviewRecord` is built and unit-tested here with no real caller yet, mirroring the precedent already set by ARC-1374 (`getStoryLifecycleState`/`setStoryLifecycleState` built with no real caller) and ARC-1375 (`resumeArtifactPRReviewLoop` built with no real caller).
- Execution happens in the standard git worktree (`/repos/ARC-1377`, branched from `main`), auto-provisioned by the existing ARC-1368 mechanism in `runLane` — no manual worktree setup required by this plan.

## Implementation Steps

### Step 1 — Add self-review types and the sidecar field

**Files:** `packages/server/src/state/types.ts`

**Action:**
Add, grouped near the existing `StoryLifecycleState`/`StoryLifecycleRecord` block (same section, following the same additive-union-type style):

- `export type SelfReviewOutcome = 'clean' | 'escalated';` — the two terminal outcomes of the local-code-review loop (pass vs iteration cap exhausted), matching the skill's own two exit conditions (Step 5 "If CLEAN" / "If NEEDS_FIXES and iteration = 3").
- `export interface SelfReviewRecord { outcome: SelfReviewOutcome; iterations: number; updatedAt: string; }` — `iterations` is the iteration number reported in the skill's own final verdict line (1–3); `updatedAt` is an ISO timestamp, following the same convention as `StoryLifecycleRecord.updatedAt`.

Add a new optional field to the `SidecarState` interface, placed after `storyLifecycle` (from ARC-1374) with a doc comment matching the style of that field's comment:

- `selfReview?: Record<string, SelfReviewRecord>;` — doc comment: "Per-story local-code-review outcome and iteration count (ARC-1377), keyed by bare story key (e.g. 'ARC-1377'). Absent when the story has not yet completed a self-review pass, or on sidecars persisted before this story. Undefined is the honest 'not yet reviewed' state — callers use `getSelfReviewRecord` and treat `undefined` as such, with no forced default (unlike `getStoryLifecycleState`, there is no sensible default outcome to invent)."

**Verification:** `run_typecheck` on `packages/server` — no type errors introduced. Confirm the new field is optional (`selfReview?:`) so no existing `SidecarState` literal in the codebase requires updating.

### Step 2 — Add persistence helper functions to sidecar.ts

**Files:** `packages/server/src/state/sidecar.ts`

**Action:**
Extend the existing `import { ... } from './types.js'` statement to also import `SelfReviewOutcome` and `SelfReviewRecord` as type-only imports.

Add two new exported pure functions (no file I/O, immutable — matching the exact style of `getStoryLifecycleState`/`setStoryLifecycleState` added by ARC-1374 in the same file), placed directly after those two functions under a new section comment (`// Self-review tracking helpers (ARC-1377) — pure functions, no file I/O.`):

- `getSelfReviewRecord(state: SidecarState, storyKey: string): SelfReviewRecord | undefined` — returns `state.selfReview?.[storyKey]`. Unlike `getStoryLifecycleState`, this returns `undefined` rather than a forced default when absent, since "has not yet self-reviewed" has no sensible default outcome value. This is the single read choke point satisfying AC2 — any future caller (e.g. ARC-1378) checks `getSelfReviewRecord(state, storyKey)?.outcome === 'clean'` without re-invoking the skill.
- `setSelfReviewRecord(state: SidecarState, storyKey: string, outcome: SelfReviewOutcome, iterations: number): SidecarState` — returns a new `SidecarState` with `selfReview` shallow-copied and the entry for `storyKey` replaced by `{ outcome, iterations, updatedAt: new Date().toISOString() }`. Does not mutate the input `state` object.

**Verification:** `run_typecheck` passes. Unit tests in Step 3 exercise both functions directly.

### Step 3 — Unit tests for the new sidecar helpers

**Files:** `packages/server/src/__tests__/sidecar.test.ts`

**Action:** Add a new `describe('getSelfReviewRecord')` / `describe('setSelfReviewRecord')` block (placed directly after the existing `describe('setStoryLifecycleState', ...)` block from ARC-1374) covering:

- `getSelfReviewRecord` returns `undefined` when `selfReview` is entirely absent from the state object.
- `getSelfReviewRecord` returns `undefined` when `selfReview` is present but does not contain the queried `storyKey`.
- `getSelfReviewRecord` returns the stored record for a `storyKey` present in `selfReview`.
- `setSelfReviewRecord` returns a new object (input `state` reference unchanged / not mutated) with the new entry present under `selfReview[storyKey]`, `outcome` and `iterations` matching the arguments, and an ISO `updatedAt`.
- `setSelfReviewRecord` called twice for two different `storyKey`s preserves both entries (no clobbering).
- `setSelfReviewRecord` called twice for the same `storyKey` overwrites the prior entry for that key only (e.g. `'clean'`/1 then `'escalated'`/3 — final record is the second call's values).
- Round-trip: apply `setSelfReviewRecord`, then `writeSidecar` + fresh `readSidecar` (simulating a restart) — assert the read-back `selfReview[storyKey]` deep-equals what was set. Direct evidence for AC1's "recorded in the per-story sidecar state" persisting across restarts.
- Backward-compat load: write a raw JSON sidecar fixture to disk with `version: 1` and all currently-required fields but no `selfReview` key (simulating a pre-ARC-1377 sidecar), call `readSidecar`, assert it returns non-null without throwing, then call `getSelfReviewRecord` on the result and assert it returns `undefined`.

**Verification:** `run_tests` — all new and existing tests in `sidecar.test.ts` pass.

### Step 4 — Detect self-review outcome and wire it into runLane

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**

Extend the existing `import { writeSidecar } from '../state/sidecar.js';` statement to also import `getSelfReviewRecord`, `setSelfReviewRecord`, and `setStoryLifecycleState`. Add a type-only import of `SelfReviewOutcome` from `../state/types.js` alongside the existing `CheckpointQueueEntry` import.

Add a new helper function, placed directly after the existing `extractLastAgentMessage` function (same section, same style — both scan an `OpenCodeEvent[]` for text content):

- `parseLocalCodeReviewOutcome(events: OpenCodeEvent[]): { outcome: SelfReviewOutcome; iterations: number } | undefined` — iterates `events` in order; for each `TextEvent`, tests its `part.text` against two patterns: the clean marker `` /✅ Local review clean \(iteration (\d+)\/3\)/ `` and the escalate marker `` /🔴 ESCALATE — local review iteration (\d+)\/3 still has unresolved findings/ ``. When a `TextEvent`'s text contains one or more matches of either pattern, the function keeps only the *last* match found within that text (using a global-flag scan, since a single streamed text chunk can legitimately contain multiple iteration-progress markers followed by the final verdict). It then compares across events strictly in array order, always overwriting its running "last known outcome" whenever the current event yields any match — so the function's return value is the last chronological occurrence of either marker across the whole `events` array (correctly ignoring earlier "Iteration N/3" progress lines that are not the final verdict). Returns `undefined` if neither pattern is found anywhere in `events` (self-review sub-step was not part of this dispatch, or the agent never reached a verdict).

In `runLane`, insert a new block immediately after the existing `if (result.outcome === 'failed') { ... }` retry-handling block closes (i.e. after the ARC-1306 retry loop has either succeeded or already `return`ed on exhaustion) and before the existing "check for an unchecked CHECKPOINT/GATE sub-step" block:

1. Call `parseLocalCodeReviewOutcome(result.events)`.
2. If it returns `undefined`, do nothing — fall through unchanged to the existing pending-gate logic (no self-review marker was present in this dispatch).
3. If it returns a value:
   a. Compute `storyKey = stripNamespace(step.id)`, matching the existing `claimKey` convention.
   b. Read `current = getState()`; compute `withRecord = setSelfReviewRecord(current, storyKey, parsedOutcome.outcome, parsedOutcome.iterations)`.
   c. If `parsedOutcome.outcome === 'clean'`: `setState(withRecord)`, `writeSidecar(withRecord.repoRoot, withRecord)`, and fall through to the existing pending-gate logic unchanged (AC2 — no lane pause on clean; the story simply now has a queryable clean record).
   d. If `parsedOutcome.outcome === 'escalated'`: compute `withLifecycle = setStoryLifecycleState(withRecord, storyKey, 'awaiting_escalation_checkpoint')`; build a `CheckpointQueueEntry` with `stepId: step.id`, `runbookPath`, `checkpointLevel: 'high'`, `hitAt: new Date().toISOString()`, `lastAgentMessage: extractLastAgentMessage(result.events)` — mirroring the exact shape of the existing ARC-1306 retry-exhausted checkpoint entry; append it to `withLifecycle.checkpointQueue`; call `setState`/`writeSidecar` with the resulting state; set `laneStatusMap.set(runbookPath, 'paused')`; `return` from `runLane` immediately (do not fall through to the pending-gate/mark-step-checked logic — AC3's "does not proceed" requirement).

**Verification:** `run_typecheck` passes. `run_lint` reports no new violations. Unit tests in Step 5 exercise both branches end-to-end through `runLane`.

### Step 5 — Unit tests for runLane self-review wiring

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:** Add a new `describe('runLane — self-review outcome tracking (ARC-1377)', ...)` block (placed directly after the existing `describe('runLane — retry behaviour (ARC-1306)', ...)` block), using the file's existing `makeAgentStep`/`makeSidecarState`/`executeStepMock` helpers and a new small local helper that builds a `TextEvent`-shaped object for use in `makeRunResult`-style fixtures. Cover:

- **Clean outcome:** `executeStepMock` resolves with `events` containing one `TextEvent` whose text is `"✅ Local review clean (iteration 1/3).\nNo blockers..."`. After `runLane` completes: `getState().selfReview['<storyKey>']` equals `{ outcome: 'clean', iterations: 1, updatedAt: <ISO string> }`; `getLaneState(RUNBOOK_PATH)` is `'done'` (lane does NOT pause); `getState().checkpointQueue` is empty.
- **Escalated outcome:** `executeStepMock` resolves with `outcome: 'success'` (subprocess exit 0) but `events` containing one `TextEvent` whose text is `"🔴 ESCALATE — local review iteration 3/3 still has unresolved findings.\n..."`. After `runLane` completes: `getState().selfReview['<storyKey>']` equals `{ outcome: 'escalated', iterations: 3, updatedAt: <ISO string> }`; `getState().storyLifecycle['<storyKey>'].state` equals `'awaiting_escalation_checkpoint'`; `getState().checkpointQueue` has exactly one entry with `stepId` equal to the step's id and `checkpointLevel: 'high'`; `getLaneState(RUNBOOK_PATH)` is `'paused'`.
- **No marker present:** ordinary successful step with plain-text events containing neither marker — `getState().selfReview` remains `undefined` after `runLane` completes (regression guard: detection does not misfire on ordinary steps).
- **Last-occurrence correctness:** `events` contains three `TextEvent`s in order — an intermediate `"=== Local Code Review — Iteration 1/3 ==="` progress line, a second intermediate `"=== Local Code Review — Iteration 2/3 ==="` line, then a final `"✅ Local review clean (iteration 2/3)."` verdict — asserts the recorded `selfReview` entry has `iterations: 2` (the final verdict), not a false match on the intermediate progress lines (which do not match either terminal-verdict regex).
- **Detection runs on the post-retry final result:** combine with the existing ARC-1306 retry pattern — `executeStepMock` fails twice then succeeds on the third call, with the third call's resolved `events` containing the clean marker. Asserts `selfReview` is recorded from the third (final, successful) call's events, not the first two failed attempts.
- **No double-push regression:** `executeStepMock` resolves with `outcome: 'failed'` on every call with `retryLimit: 0` (so the existing ARC-1306 branch pushes a high checkpoint and returns before reaching the new Step 4 code at all) even though those failed attempts' `events` happen to also contain an escalate-marker-shaped string — asserts `getState().checkpointQueue` still has exactly one entry (proving the two mechanisms cannot double-push, by construction of the early `return` in the existing retry-exhausted branch).

**Verification:** `run_tests` — all new tests pass; no existing `laneRunner.test.ts` test changes behavior (the new block is purely additive and the new code path is only reached when a self-review marker is present, so pre-existing tests using plain-text-free `events: []` fixtures are unaffected).

### Step 6 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite, typecheck, and lint to confirm the additive change introduces zero regressions elsewhere (in particular the wider `laneRunner.test.ts` suite — sub-step checkpoint detection, auto-claim, auto-commit, auto-archive, worktree provisioning, and planning-artifact PR detection describe blocks — none of which reference `selfReview`/`parseLocalCodeReviewOutcome` and must continue to pass unmodified).

**Verification:** `run_typecheck` returns clean; `run_tests` returns all green with only `types.ts`, `sidecar.ts`, `sidecar.test.ts`, `laneRunner.ts`, and `laneRunner.test.ts` modified; `run_lint` reports no new violations.

## Testing & Validation

- `run_typecheck` — confirms the new `SelfReviewOutcome`/`SelfReviewRecord` types, the new `SidecarState.selfReview` optional field, the two new `sidecar.ts` helper signatures, and the new `parseLocalCodeReviewOutcome` helper in `laneRunner.ts` all compile cleanly against every existing call site.
- `run_tests` — confirms:
  - New `sidecar.test.ts` scenarios (Step 3) prove default-`undefined` behavior, correct read/write, immutability, no-clobber, round-trip persistence, and backward-compat loading — direct evidence for AC1 and AC2.
  - New `laneRunner.test.ts` scenarios (Step 5) prove both the clean path (record written, no pause) and the escalated path (record written, canonical lifecycle state transitioned, checkpoint pushed, lane paused) end-to-end through `runLane` — direct evidence for AC1, AC2, and AC3.
  - The full existing suite (`sidecar.test.ts`'s pre-existing tests, `reconcile.test.ts`, `checkpoints.test.ts`, `state.test.ts`, `stepRunner.test.ts`, `sessions.test.ts`, `runbooks.summary.test.ts`, `runbooks.detail.test.ts`, and every pre-existing `laneRunner.test.ts` describe block) passes unmodified.
- `run_lint` — confirms no style/lint violations in the five edited files.

## Risks & Open Questions

- **Fragile coupling to the local-code-review skill's exact output wording.** `parseLocalCodeReviewOutcome` depends on the literal strings `"✅ Local review clean (iteration N/3)."` and `"🔴 ESCALATE — local review iteration N/3 still has unresolved findings."`, both currently mandated by `~/.config/opencode/skills/local-code-review/SKILL.md` Steps 4–5. If that skill's output wording changes in the future, detection silently stops matching (the function returns `undefined`, and no `selfReview` record is written — a graceful no-detection fallback, not a crash or false positive). This coupling is an accepted consequence of the issue's Out of Scope explicitly forbidding changes to the skill's own logic in this story; a future change to the skill's wording should be paired with a corresponding update to this regex, but that is not this story's concern.
- **AC3's "exhausts its retry limit" refers to the skill's own internal iteration cap (max 3), not `SessionConfig.retryLimit`.** These are two independent concepts — this plan explicitly does not touch the ARC-1306 subprocess-retry mechanism. If this interpretation is wrong, the fix is confined to Step 4's control flow and does not affect the new types/helpers in Steps 1–2.
- **No wiring into ARC-1378 in this story, by design.** `getSelfReviewRecord` is built and tested but has no real caller yet (ARC-1378, which depends on this story, will be the first consumer). This mirrors the precedent already set by ARC-1374 and ARC-1375 in this same epic.
- **Low-probability false-positive risk:** if an unrelated sub-step within the same dispatched segment happens to emit text that coincidentally matches one of the two marker regexes verbatim (including the specific emoji and "iteration N/3" phrasing), `parseLocalCodeReviewOutcome` would misattribute that text as a genuine self-review verdict. Given the specificity of the marker strings (emoji + exact phrase + captured iteration number), this is accepted as a negligible risk rather than something this story guards against structurally.
