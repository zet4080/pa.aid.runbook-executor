```markdown
# ARC-1315 Select Runbooks to Include in Session — Completion Summary

**Issue:** `issues/session-setup/ARC-1315-select-runbooks-for-session.md`
**Implementation Plan:** `implementation_plans/session-setup/ARC-1315-implementation-plan.md`
**Completed:** 2026-06-29
**Duration:** ~1h
**Cost:** tracked via session

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given dashboard, when supervisor selects subset and starts session, then only selected runbooks execute | Passed | `useSessionState.test.ts` — "toggleRunbook calls PATCH /api/state with updated selectedRunbooks"; `Dashboard.test.tsx` — "passes selected=true to the card whose path is in selectedRunbooks". `selectedRunbooks` array is persisted to sidecar and will be used by ARC-1291 executor as the session runbook list. |
| 2 | Given deselected runbook, when session starts, then that runbook remains idle | Passed | `Dashboard.test.tsx` — "passes selected=true to card whose path is in selectedRunbooks"; deselected card receives `selected=false`; `selectedRunbooks` in state excludes the deselected path |

## Implementation Summary

### `useSessionState` hook (Step 1)
New hook at `packages/ui/src/hooks/useSessionState.ts`. Fetches `GET /api/state` on mount to load `selectedRunbooks` and `config`. Exposes `toggleRunbook(path)` which optimistically updates local state and fires `PATCH /api/state { selectedRunbooks }`. Reverts on PATCH error. Also exposes `updateConfig(partial)` (used by ARC-1316) and `startSession()`.

### `RunbookCard` selection checkbox (Step 2)
Extended `RunbookCardProps` with `selected: boolean` and `onToggleSelect: (path: string) => void`. Added a checkbox in `CardHeader` with `aria-label="Select {title}"`, `checked={selected}`, and `onChange` calling `onToggleSelect(summary.path)`. Selected cards gain a blue ring via Tailwind `ring-2 ring-blue-500`.

### Dashboard wiring (Step 3)
`Dashboard.tsx` now calls `useSessionState()` to get `selectedRunbooks` and `toggleRunbook`. Passes `selected` and `onToggleSelect` to each `RunbookCard`. Renders "N of M selected" summary row above the grid. Shows "Loading selection…" while selection is loading.

### Client-side types (shared with ARC-1316)
Added `QueuePolicy` and `SessionConfig` types to `packages/ui/src/types/runbook.ts` for use by `useSessionState` and the upcoming `SessionConfigPanel`.

## Verification Steps

```bash
cd /repos/ARC-1314/runbook-executor && npm test
cd packages/ui && npx tsc --noEmit
```

## Tests Added/Modified

- `packages/ui/src/hooks/useSessionState.test.ts` — 6 tests: loads state from GET; error on non-2xx; toggleRunbook adds/removes; toggleRunbook calls PATCH; updateConfig calls PATCH with merged config
- `packages/ui/src/components/RunbookCard.test.tsx` — 3 new tests: checkbox renders; checked when selected=true; calls onToggleSelect on click
- `packages/ui/src/pages/Dashboard.test.tsx` — 3 new tests: "N of M selected" row; loading selection text; selected card has checked checkbox

## Rollback Notes

Additive. `RunbookCard` now requires `selected` and `onToggleSelect` props — all existing callers (only Dashboard) have been updated.

## Next Steps

- ARC-1316: configure session parameters before start (SessionConfigPanel + Start Session button)
```
