# ARC-1303 Completion Summary

## Story
[ARC-1303](https://proalpha.atlassian.net/browse/ARC-1303): Display checkpoint queue with automatic priority scoring

## Status
✅ Complete

## Commit
- Branch: `ARC-1303` in `/repos/pa.aid.wsl-setup.sh`
- Commit: `d0eabd4`
- Message: `feat(checkpoint-management): ARC-1303 checkpoint queue panel with priority scoring`

## Acceptance Criteria Evidence

### AC1: Given multiple pending checkpoints, when displayed, then ordered by priority score (gate level + wait time)
- **File:** `packages/ui/src/lib/checkpointScore.ts` — `scoreCheckpointEntry()` and `sortCheckpointQueue()` implement formula: `levelWeight × 1_000_000 − waitSeconds` (level weight: high=1, medium=2, low=3)
- **Hook:** `packages/ui/src/hooks/useCheckpointQueue.ts` — calls `sortCheckpointQueue()` on every poll result before setting state
- **Test:** `src/lib/checkpointScore.test.ts` → `sortCheckpointQueue > low outranks medium outranks high — ordered ascending by priority`
- **Test:** `src/hooks/useCheckpointQueue.test.ts` → `ARC-1303 sorting > returns queue sorted by priority: high before medium before low`

### AC2: Given checkpoint waiting longer than equal-level peer, then it ranks higher
- **File:** `packages/ui/src/lib/checkpointScore.ts` — wait seconds subtracted from score so longer wait = lower score = higher rank
- **Test:** `src/lib/checkpointScore.test.ts` → `sortCheckpointQueue > longer wait wins within the same level`
- **Test:** `src/hooks/useCheckpointQueue.test.ts` → `ARC-1303 sorting > within same level, longer wait ranks first`

### AC3: Given pending checkpoint in queue, then queue item displays a [Resume] button that triggers the Resume flow
- **File:** `packages/ui/src/components/CheckpointQueuePanel.tsx` — each list item renders a `<button>Resume</button>` calling `onResume(entry.stepId)`
- **Hook:** `packages/ui/src/hooks/useCheckpointQueue.ts` — `resume(stepId)` optimistically removes entry and PATCHes `/api/state`; reverts on failure
- **Test:** `src/components/CheckpointQueuePanel.test.tsx` → `calls onResume with the correct stepId when [Resume] is clicked`
- **Test:** `src/hooks/useCheckpointQueue.test.ts` → `ARC-1303 resume > optimistically removes entry from queue and PATCHes /api/state`
- **Test:** `src/hooks/useCheckpointQueue.test.ts` → `ARC-1303 resume > reverts to original queue when PATCH fails`

### AC4: Given pending checkpoint in queue, then queue item displays a link to the associated Bitbucket PR
- **File:** `packages/ui/src/components/CheckpointQueuePanel.tsx` — renders `<a href={entry.prUrl}>View PR</a>` when `prUrl` present; omits otherwise
- **Test:** `src/components/CheckpointQueuePanel.test.tsx` → `renders PR link when prUrl is present`
- **Test:** `src/components/CheckpointQueuePanel.test.tsx` → `does not render PR link when prUrl is absent`

## Files Changed
| File | Change |
|------|--------|
| `packages/ui/src/lib/checkpointScore.ts` | New — pure scoring/sorting functions |
| `packages/ui/src/lib/checkpointScore.test.ts` | New — 8 tests |
| `packages/ui/src/lib/laneLabel.ts` | New — extracted from Dashboard.tsx |
| `packages/ui/src/hooks/useCheckpointQueue.ts` | Modified — sorting + optimistic resume() |
| `packages/ui/src/hooks/useCheckpointQueue.test.ts` | Modified — 6 new ARC-1303 tests |
| `packages/ui/src/components/CheckpointQueuePanel.tsx` | New — ordered queue panel component |
| `packages/ui/src/components/CheckpointQueuePanel.test.tsx` | New — 12 tests |
| `packages/ui/src/pages/Dashboard.tsx` | Modified — uses CheckpointQueuePanel, imports laneLabel |

## Test Results
- UI tests: **164 passed**, 0 failed (18 test files)
- Server tests: **261 passed**, 0 failed (19 test files in server package — no changes needed)
- TypeCheck (tsc --noEmit): clean
- Server build (tsc): clean
