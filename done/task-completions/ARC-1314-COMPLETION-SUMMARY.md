```markdown
# ARC-1314 Expand Runbook Detail on Demand — Completion Summary

**Issue:** `issues/session-setup/ARC-1314-expand-runbook-detail.md`
**Implementation Plan:** `implementation_plans/session-setup/ARC-1314-implementation-plan.md`
**Completed:** 2026-06-29
**Duration:** ~1.5h
**Cost:** tracked via session

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given collapsed runbook card, when supervisor clicks expand, then all waves and steps shown with checkbox state | Passed | `RunbookCard.test.tsx` — "renders StepTree when detail is loaded" and "clicking expand sets aria-expanded='true'"; `StepTree.test.tsx` — "renders all step labels", "renders checked step with a checked checkbox" |
| 2 | Given expanded runbook, when supervisor collapses, then returns to summary view without losing state | Passed | `RunbookCard.test.tsx` — "clicking expand then collapse sets aria-expanded back to false"; `isExpanded` state held in component; `useRunbookDetail` hook retains last fetched detail value on collapse |

## Implementation Summary

### Server (Step 1)
Added `GET /api/runbooks/detail?path=<absolute-path>` endpoint to `packages/server/src/routes/runbooks.ts`. The endpoint validates the path is non-empty and within the plans directory (path traversal guard using `path.resolve()` comparison), parses the runbook via `parseRunbook()`, and returns `{ runbook: ParsedRunbook, stepState: Record<string, boolean> }`. Registered before the `/runbooks/summary` route.

### Client Types (Step 2)
Extended `packages/ui/src/types/runbook.ts` with client-side mirrors: `RunbookSubStep`, `RunbookStep`, `RunbookWave`, `ParsedRunbookDetail`, `RunbookDetailResponse`.

### `useRunbookDetail` hook (Step 2)
New hook at `packages/ui/src/hooks/useRunbookDetail.ts`. Returns `{ detail, loading, error }`. When `path` is null, no fetch is made. Fetches once per unique non-null path; caches last result. Uses `AbortController` for cleanup on unmount or path change.

### `StepTree` component (Step 3)
New presentational component at `packages/ui/src/components/StepTree.tsx`. Renders all waves → steps → sub-steps. Read-only checkboxes (disabled). First unchecked step receives `aria-current="step"` and a blue highlight ring.

### `RunbookCard` expand/collapse (Step 3)
Modified `packages/ui/src/components/RunbookCard.tsx` to add `isExpanded` state, expand/collapse toggle button with `aria-expanded`, and conditional rendering: collapsed = summary view (unchanged), expanded = StepTree with loading/error states. `deriveNextStepId` utility computes the first unchecked step across waves.

### Fix: React 19 dual-instance (pre-existing)
Added `"overrides": { "react": "^19.0.0", "react-dom": "^19.0.0" }` to root `package.json` to prevent npm workspaces from hoisting React 18 alongside React 19, which caused `useState null ref` errors in all UI tests.

## Verification Steps

```bash
# Run full test suite
cd /repos/ARC-1314/runbook-executor && npm test

# Type-check both packages
cd packages/server && npx tsc --noEmit
cd packages/ui && npx tsc --noEmit
```

## Tests Added/Modified

- `packages/server/src/__tests__/runbooks.detail.test.ts` — 5 tests: valid path returns 200 with runbook + stepState; missing path returns 400; empty path returns 400; path outside plansDir returns 400; non-existent file returns 400
- `packages/ui/src/hooks/useRunbookDetail.test.ts` — 6 tests: null path skips fetch; non-null path fetches correct URL; error on non-2xx; error on network failure; re-fetches on path change
- `packages/ui/src/components/StepTree.test.tsx` — 8 tests: wave titles render; step labels render; checked/unchecked checkbox state; aria-current on nextStepId; no aria-current when null; sub-steps render; no sub-step list when empty
- `packages/ui/src/components/RunbookCard.test.tsx` — 13 tests (6 original + 7 new): expand button present with aria-expanded="false"; click expand sets aria-expanded="true"; click collapse reverts; loading state shown; error state shown; StepTree renders when detail loaded; path passed to hook on expand

## Rollback Notes

Additive changes only. The detail endpoint is a new route; no existing routes modified. `RunbookCard` is backward-compatible — summary view unchanged.

## Next Steps

- ARC-1315: select runbooks to include in session (checkbox multi-select on RunbookCard)
- ARC-1316: configure session parameters before start
```