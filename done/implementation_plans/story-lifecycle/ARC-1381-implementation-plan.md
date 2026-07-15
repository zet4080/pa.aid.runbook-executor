# ARC-1381: Derive legacy status enums as computed rollups from canonical state - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1381-derive-legacy-status-enums-as-rollups.md`  
**Completion Summary:** `task-completions/ARC-1381-COMPLETION-SUMMARY.md` (TBD)  
**Approach:** Single approach — rollup computation pattern established  
**Owner:** build agent  
**Date:** 2026-07-15

## Scope & Alignment

Convert LaneStatus, RunbookStatus, and ClosedSession.status from independently-set state to pure computed rollups derived from canonical StoryLifecycleState. This implements the final step of the state machine migration, eliminating the three legacy status enums as independent sources of truth.

**Acceptance criteria mapping:**
- AC1 (lane 'done' when all stories archived) → Step 2: aggregate lane state computation
- AC2 (lane 'paused' when any story in gate) → Step 2: aggregate lane state computation
- AC3 (no independent status-setting code remains) → Step 3: remove all manual status-setting sites

## Assumptions & Dependencies

- ARC-1374: StoryLifecycleState enum exists with all states defined
- ARC-1376: Planning-artifact gate migrated to shared sub-machine
- ARC-1378: Implementation-review gate implemented
- All story state transitions now push StoryLifecycleState via sidecar
- Sidecar state provides per-story lifecycle records accessible to laneRunner and route handlers

## Implementation Steps

### Step 1: Add rollup computation functions to state/types.ts

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/state/types.ts`

**Action:** Add three pure functions that compute legacy enum values from StoryLifecycleState arrays:

```typescript
export function computeLaneStatus(stories: StoryLifecycleRecord[]): LaneStatus
export function computeRunbookStatus(stories: StoryLifecycleRecord[]): RunbookStatus
export function computeClosedSessionStatus(stories: StoryLifecycleRecord[]): 'complete' | 'partial'
```

**Logic:**
- `LaneStatus`: 'done' if all archived; 'paused' if any in gate state (awaiting_escalation_checkpoint, awaiting_plan_review, awaiting_implementation_review, awaiting_decision_review); 'running' if any non-gate active state; 'idle' if all unclaimed; 'queued' otherwise
- `RunbookStatus`: 'complete' if all archived; 'blocked' if any in gate; 'running' if any active; 'idle' otherwise
- `ClosedSessionStatus`: 'complete' if all stories in session reached archived; 'partial' otherwise

**Gate states set:** `['awaiting_escalation_checkpoint', 'awaiting_plan_review', 'awaiting_implementation_review', 'awaiting_decision_review']`

**Verification:** Unit tests in new file `state/types.test.ts` cover all four branches for each rollup (empty array, all archived, mixed with gate, mixed without gate).

---

### Step 2: Replace laneRunner.ts volatile map with computed getter

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/runner/laneRunner.ts`

**Action:**
- Remove module-level `laneStatusMap: Map<string, LaneStatus>` declaration (line 31)
- Replace `getLaneState()` (lines 62–64) with new implementation:
  ```typescript
  function getLaneState(laneId: string): LaneStatus {
    const stories = sidecar.getStoriesForLane(laneId);
    return computeLaneStatus(stories);
  }
  ```
- Remove `restoreLaneStatus()` function entirely (lines 80–82) — no longer needed
- Remove all calls to `laneStatusMap.set()` throughout laneRunner.ts (lines 550, 647, 683, 739, 772, 820)

**New sidecar method required:**
- Add `getStoriesForLane(laneId: string): StoryLifecycleRecord[]` to sidecar interface in `state/sidecar.ts`
- Implementation filters sidecar state by lane identifier derived from story key prefix or runbook metadata

**Verification:** Existing laneRunner.test.ts tests should pass unchanged; modify assertions that expect specific `laneStatusMap` side effects to instead assert on sidecar state.

---

### Step 3: Migrate RunbookStatus to computed getter in runbook/summarize.ts

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/runbook/summarize.ts`

**Action:**
- Locate function that computes `RunbookStatus` (line 45 per exploration)
- Replace step-completion-based logic with call to `computeRunbookStatus(stories)` from Step 1
- Pass `ParsedRunbook` and sidecar state to function; extract stories via `sidecar.getStoriesForRunbook(runbookPath)`

**New sidecar method required:**
- Add `getStoriesForRunbook(runbookPath: string): StoryLifecycleRecord[]` to sidecar interface
- Implementation filters sidecar state by stories belonging to the runbook identified by path

**Verification:** Existing runbook tests should pass; check that UI state mirrors match new computed values.

---

### Step 4: Migrate ClosedSession.status to computed value at close time

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/routes/sessions.ts`

**Action:**
- Locate session-close handler (lines 109–121 per exploration)
- Replace manual `status` assignment with:
  ```typescript
  status: computeClosedSessionStatus(session.stories)
  ```
- Ensure `session.stories` contains full `StoryLifecycleRecord[]` snapshot at close time

**Verification:** Session-close test in `__tests__/sessions.test.ts` or equivalent verifies both 'complete' and 'partial' branches.

---

### Step 5: Remove legacy status setters and validate elimination

**Files:** All files across codebase

**Action:**
- Search for remaining references to `laneStatusMap.set()`, `RunbookStatus =`, `ClosedSession.status =` assignment patterns
- Remove any dead code paths revealed by Steps 2–4
- Validate no route handlers or background jobs independently set these three enums

**Verification:**
```bash
rg "laneStatusMap\.set" --type ts
rg "RunbookStatus\s*=" --type ts
rg "status\s*=\s*['\"]complete|partial" --type ts
```
All searches return zero results (except test mocks and type definitions).

---

## Testing & Validation

1. **Unit tests:** `state/types.test.ts` covers all rollup branches
2. **Integration tests:** Existing `laneRunner.test.ts` updated to assert sidecar state instead of volatile map state
3. **Manual verification:** Run conductor against existing runbook; verify lane pauses correctly at gates, resumes correctly after merge, transitions to 'done' when all stories archived
4. **Regression suite:** Run full test suite: `npm test` in `/repos/pa.aid.conductor.ts/packages/server/`

## Risks & Open Questions

**Risk:** Sidecar `getStoriesForLane` / `getStoriesForRunbook` methods may not exist yet — if missing, add them as part of Step 2/3 implementation.

**Risk:** Lane identifier derivation from story key may be ambiguous if multiple lanes exist in same session — verify sidecar state includes explicit `laneId` field on `StoryLifecycleRecord`, or derive from runbook metadata.

**Open question:** Should `computeLaneStatus` handle edge case where lane has zero stories? Current assumption: return 'idle'. Verify expected behavior with supervisor if ambiguous.
