# ARC-1301 Implementation Plan — Detect checkpoint markers and pause lane execution

## Summary

This story makes the server correctly detect whether a runbook is blocked by a checkpoint queue entry and exposes the severity level of the blocking checkpoint so the dashboard UI can render a coloured indicator next to the status badge. Two bugs are fixed: (1) `isBlocked` in `summarize.ts` matches by `stepId` against the runbook's own step list, which silently fails for sub-steps or when queue entries carry a different ID than the parsed step; (2) `RunbookSummary` carries no checkpoint-level information, so the UI has no way to display severity. The fix replaces step-ID matching with `runbookPath`-based matching (queue entries already carry `runbookPath`) and adds `blockedCheckpointLevel` to both the server and UI summary types.

## Background & Analysis

### What already works

- `CheckpointQueueEntry` in `packages/server/src/state/types.ts` already has both `stepId` and `runbookPath` fields, and `checkpointLevel: 'high' | 'medium' | 'low'`.
- `deriveRunbookSummary` in `summarize.ts` correctly derives `running`, `complete`, and `idle` status. The `blocked` path compiles and is tested.
- The existing test suite (`summarize.test.ts`) passes with the current code, but the `makeQueue` fixture hardcodes `runbookPath: '/path/to/runbook.md'` while all test invocations pass `'/runbook.md'` as the first argument. This means the "blocked" tests pass today only because `isBlocked` uses step-ID matching — a mismatch in implementation intent.
- `RunbookCard.tsx` already renders `<StatusBadge status={status} />` and has a flex container around it that can accommodate an inline emoji.

### The `isBlocked` bug

Current code:
```ts
const queuedStepIds = new Set(checkpointQueue.map((e) => e.stepId));
const isBlocked = allSteps.some(
  (step) => queuedStepIds.has(step.id) && stepState[step.id]!== true,
);
```

Problem: A sub-step checkpoint (e.g. a checkpoint marker inside a sub-step list) would have a `stepId` that may not appear in `allSteps` (which only flattens top-level wave steps). Additionally, the queue is keyed by `runbookPath` for routing purposes — using path matching is simpler, more robust, and consistent with how the queue is intended to be consumed.

Fix: Replace with `checkpointQueue.some(e => e.runbookPath === runbookPath)`. This correctly blocks the runbook whenever any entry in the queue targets this runbook's path, regardless of step structure.

Acceptable edge case: If a queue entry exists for a step that has already been checked (stale state), the runbook will briefly show `blocked` until the executor clears the queue entry. This is acceptable — the queue should be cleared on resume anyway.

### `blockedCheckpointLevel` derivation

When blocked, compute the highest level among all matching queue entries: `high > medium > low`. If not blocked, the value is `null`.

### UI rendering decision

Render the checkpoint level emoji **inline next to** `StatusBadge` in `RunbookCard.tsx`, not inside the badge. The existing flex container at line 62–73 already has `gap-2` and `items-center`, making this a simple prepend before `<StatusBadge>`.

Level → emoji mapping:
- `'high'` → 🔴
- `'medium'` → 🟡
- `'low'` → 🟢
- `null` → (nothing rendered)

## Decisions

- **`isBlocked` fix:** use `runbookPath`-based matching for ALL queue entries (top-level and sub-step). Brief stale-state edge case is acceptable.
- **Checkpoint indicator placement:** render emoji (🔴/🟡/🟢) inline next to `StatusBadge` in `RunbookCard.tsx` (not inside the badge).

## Ordered Implementation Steps

### Step 1 — Add `blockedCheckpointLevel` to server `RunbookSummary`
**File:** `packages/server/src/runbook/types.ts`
**What:** Add `blockedCheckpointLevel: CheckpointLevel` field to `RunbookSummary`. `CheckpointLevel` is already defined in this file as `'high' | 'medium' | 'low' | null`.
**How:** Append the field after `checkedSteps`:
```ts
/** Highest checkpoint level among queue entries blocking this runbook; null if not blocked */
blockedCheckpointLevel: CheckpointLevel;
```

### Step 2 — Fix `isBlocked` and compute `blockedCheckpointLevel` in `summarize.ts`
**File:** `packages/server/src/runbook/summarize.ts`
**What:** Replace step-ID-based blocking detection with `runbookPath`-based detection, and compute `blockedCheckpointLevel`.
**How:**
1. Remove the `queuedStepIds` Set construction.
2. Replace `isBlocked` derivation:
   ```ts
   const matchingEntries = checkpointQueue.filter(e => e.runbookPath === runbookPath);
   const isBlocked = matchingEntries.length > 0;
   ```
3. Compute `blockedCheckpointLevel` using priority order `high > medium > low`:
   ```ts
   const levelOrder = ['high', 'medium', 'low'] as const;
   const blockedCheckpointLevel: CheckpointLevel = isBlocked
    ? levelOrder.find(lvl => matchingEntries.some(e => e.checkpointLevel === lvl))?? null
     : null;
   ```
4. Include `blockedCheckpointLevel` in the returned object.

### Step 3 — Fix test fixtures and add new test scenarios in `summarize.test.ts`
**File:** `packages/server/src/__tests__/summarize.test.ts`
**What:** Fix the path mismatch in `makeQueue` and add tests for `blockedCheckpointLevel` and sub-step checkpoint blocking.
**How:**
1. Change `makeQueue` to accept a `runbookPath` parameter (default `'/runbook.md'`) so fixtures match what tests pass as `runbookPath`. Update the existing `makeQueue` call signature to:
   ```ts
   function makeQueue(
     stepIds: string[],
     opts?: { runbookPath?: string; checkpointLevel?: CheckpointQueueEntry['checkpointLevel'] },
   ): CheckpointQueueEntry[]
   ```
   Default `runbookPath` to `'/runbook.md'` and `checkpointLevel` to `'high'`.
2. Update all existing `makeQueue(...)` callsites that pass `'/runbook.md'` as the test's first arg — no signature change needed for those since the default matches.
3. Add test: **sub-step checkpoint (queue entry whose stepId is not in allSteps) blocks the runbook** — pass a queue entry with `runbookPath: '/runbook.md'` and `stepId: 'SUBSTEP-X'`; assert `status === 'blocked'`.
4. Add test: **`blockedCheckpointLevel` is `'high'` when queue has a high entry**.
5. Add test: **`blockedCheckpointLevel` is `'medium'` when all entries are medium**.
6. Add test: **`blockedCheckpointLevel` is `'high'` when queue has both high and medium entries** (priority check).
7. Add test: **`blockedCheckpointLevel` is `null` when not blocked**.
8. Add test: **queue entry for a different runbookPath does not block this runbook** — ensures path-scoping is correct.

### Step 4 — Add `blockedCheckpointLevel` to UI `RunbookSummary`
**File:** `packages/ui/src/types/runbook.ts`
**What:** Mirror the server type change in the UI type.
**How:** Add field after `checkedSteps` in `RunbookSummary`:
```ts
/** Highest checkpoint level among queue entries blocking this runbook; null if not blocked */
blockedCheckpointLevel: 'high' | 'medium' | 'low' | null;
```

### Step 5 — Render checkpoint level indicator in `RunbookCard.tsx`
**File:** `packages/ui/src/components/RunbookCard.tsx`
**What:** When `status === 'blocked'`, render an emoji before `StatusBadge` indicating the checkpoint severity.
**How:**
1. Destructure `blockedCheckpointLevel` from `summary` at line 34.
2. Define a helper map or expression inside the component:
   ```ts
   const checkpointEmoji =
     status === 'blocked'
      ? { high: '🔴', medium: '🟡', low: '🟢' }[blockedCheckpointLevel?? '']?? null
       : null;
   ```
3. In the JSX, render before `<StatusBadge status={status} />`:
   ```tsx
   {checkpointEmoji && (
     <span aria-label={`Checkpoint level: ${blockedCheckpointLevel}`} role="img">
       {checkpointEmoji}
     </span>
   )}
   ```

## Test Strategy

### Unit tests (`summarize.test.ts`)
- All existing tests must continue to pass after fixture path fix.
- New tests cover: sub-step queue entries blocking by path, `blockedCheckpointLevel` values (`high`, `medium`, `low`, `null`), priority ordering (`high` beats `medium`), and cross-runbook isolation (different `runbookPath` does not block).

### UI component tests (if a test file exists for `RunbookCard`)
- If `RunbookCard.test.tsx` exists: add a snapshot or render test asserting the emoji appears when `status === 'blocked'` and `blockedCheckpointLevel` is non-null, and does not appear for other statuses.
- If no test file exists: the change is covered by the TypeScript type system and visual review.

### Type-check
- Run `tsc --noEmit` (or `pnpm typecheck`) in both `packages/server` and `packages/ui` to confirm no type errors.

### Integration
- Manual: start the dev server, inject a checkpoint queue entry via the API or state file, observe the dashboard card shows the correct emoji next to the blocked badge.

## Acceptance Criteria Mapping

| AC | How it is satisfied |
|----|-------------------|
| AC 1: Given step with 🔴/🟡/🟢 marker, when executor reaches it, then lane pauses and transitions to blocked status | Steps 2–3: `isBlocked` now matches by `runbookPath`; any queue entry (including sub-step checkpoints) causes `status: 'blocked'`. New unit tests assert this for all three marker types. |
| AC 2: Given blocked lane, when supervisor views dashboard, then lane shows blocked with checkpoint type indicated | Steps 4–5: `blockedCheckpointLevel` propagates to the UI via the summary type; `RunbookCard` renders the coloured emoji (🔴/🟡/🟢) inline next to the StatusBadge when blocked. |

## Risks & Notes

- **Stale queue entries:** After a checkpoint is resolved (supervisor presses Resume), the queue entry must be removed by the executor. If it lingers, the runbook will remain `blocked`. This is a concern for ARC-1302 (Resume flow), not this story.
- **`makeQueue` fixture migration:** The existing `stepId in checkpointQueue` tests that currently pass by accident (step-ID match coincidentally works because step IDs happen to be in `allSteps`) will still pass after the fix because the `runbookPath` will now match instead — provided the fixture path is corrected to `'/runbook.md'`. The fix must be applied before or alongside the logic change.
- **No breaking API change:** `RunbookSummary` is extended with a new required field. Any existing code that constructs a `RunbookSummary` literal (e.g. in other tests or mocks) will need to add `blockedCheckpointLevel: null`. Check for usages before committing.
