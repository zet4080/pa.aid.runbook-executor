# ARC-1313 Display Runbook Dashboard at Startup — Completion Summary

**Issue:** `issues/session-setup/ARC-1313-display-runbook-dashboard.md`
**Implementation Plan:** `implementation_plans/session-setup/ARC-1313-implementation-plan.md`
**Completed:** 2026-06-28
**Duration:** 1 session
**Branch:** `ARC-1313` (worktree `/repos/ARC-1313`)

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given planning repo with runbooks, when tool starts, then all runbooks listed with name, status, wave, % progress within 3 seconds | Passed | `GET /api/runbooks/summary` endpoint added; `useRunbookSummary` hook polls every 2 s (first response < 500 ms for 7 runbooks); `Dashboard.tsx` renders all summaries on mount. Integration test `src/__tests__/runbooks.summary.test.ts` (4 tests) validates 200 response with correct `RunbookSummary` shape. `npm run test --workspace=packages/server` → 92 tests passed. |
| 2 | Given runbooks in various states, when displayed, then each shows correct status badge | Passed | `StatusBadge.tsx` renders one of four variants (idle/running/blocked/complete) per status. `StatusBadge.test.tsx` asserts correct label text for all four statuses. `RunbookCard.test.tsx` asserts badge renders inside the card for a `running` fixture. `npm run test --workspace=packages/ui` → 21 tests passed. |

## Implementation Summary

### Server (8 files)
- **`packages/server/src/runbook/types.ts`** — `RunbookStatus` (`idle | running | blocked | complete`) and `RunbookSummary` types added (additive, no existing types changed).
- **`packages/server/src/runner/runningSteps.ts`** — New in-memory `Set<string>` tracker exporting `markRunning`, `markDone`, `getRunningStepIds`.
- **`packages/server/src/runner/stepRunner.ts`** — `markRunning` called before `runStep`; `markDone` called in `finally` block.
- **`packages/server/src/runbook/summarize.ts`** — Pure `deriveRunbookSummary()` function implementing priority-ordered status logic (running > blocked > complete > idle), progress calculation, and current-wave derivation.
- **`packages/server/src/routes/runbooks.ts`** — `GET /api/runbooks/summary` endpoint: globs `runbook-*.md`, parses each, calls `deriveRunbookSummary`, returns `RunbookSummary[]`.
- **`packages/server/src/__tests__/runningSteps.test.ts`** — 6 unit tests for the tracker.
- **`packages/server/src/__tests__/summarize.test.ts`** — 12 unit tests covering all status paths, priority ordering, progress math, current-wave logic.
- **`packages/server/src/__tests__/runbooks.summary.test.ts`** — 4 integration tests (supertest) for the new endpoint including 500 on missing `plansDir`.

### UI (15 files)
- **`packages/ui/src/types/runbook.ts`** — Client-side `RunbookStatus` and `RunbookSummary` types (manually mirrored, no shared package).
- **`packages/ui/src/hooks/useRunbookSummary.ts`** — Polling hook: fetches on mount, re-fetches every 2 000 ms, `loading` true only on first fetch, `error` cleared on success, `AbortController` cleanup on unmount.
- **`packages/ui/src/components/StatusBadge.tsx`** — shadcn `<Badge>` with status-to-variant mapping (secondary/default/destructive/outline).
- **`packages/ui/src/components/RunbookCard.tsx`** — shadcn `<Card>` with title, `<StatusBadge>`, wave label, shadcn `<Progress>` bar.
- **`packages/ui/src/pages/Dashboard.tsx`** — Full dashboard replacing stub: loading state, error state, empty state, responsive 2-column grid of `<RunbookCard>` components.
- shadcn primitives (`badge`, `card`, `progress`), `cn()` utility, Tailwind v4 config wired in.
- 4 UI test files (21 tests): `StatusBadge.test.tsx`, `RunbookCard.test.tsx`, `Dashboard.test.tsx`, `useRunbookSummary.test.ts`.

## Verification Steps

```bash
# Full test suite
cd /repos/ARC-1313/runbook-executor
npm test

# Server tests only
npm run test --workspace=packages/server

# UI tests only
npm run test --workspace=packages/ui

# Performance check (requires server running with REPO_ROOT set)
curl -w "\nTotal: %{time_total}s\n" http://localhost:4000/api/runbooks/summary
```

## Tests Added/Modified

| File | Tests | What it covers |
|------|-------|----------------|
| `packages/server/src/__tests__/runningSteps.test.ts` | 6 | `markRunning`, `markDone`, `getRunningStepIds` tracker |
| `packages/server/src/__tests__/summarize.test.ts` | 12 | Status derivation (all 4 statuses, priority order, progress %, current wave, zero-step runbook) |
| `packages/server/src/__tests__/runbooks.summary.test.ts` | 4 | `GET /api/runbooks/summary` — 200 with shape, 500 on missing dir |
| `packages/ui/src/components/StatusBadge.test.tsx` | 4 | Each status renders correct label text |
| `packages/ui/src/components/RunbookCard.test.tsx` | 6 | Card renders title, badge, wave, progress for fixture summary |
| `packages/ui/src/hooks/useRunbookSummary.test.ts` | 5 | Loading, success, error, abort, polling behaviour |
| `packages/ui/src/pages/Dashboard.test.tsx` | 6 | Loading state, error state, empty state, runbook list render |
| **Total** | **43 new tests** | (92 server + 21 UI = 113 total in suite) |

## Rollback Notes

None. All changes are additive: new route, new components, new tests. Removing the feature requires deleting the new files and removing the `router.get('/summary', ...)` line from `runbooks.ts`.

## Next Steps

- ARC-1314: Expand runbook detail on demand (in-page expand, depends on this dashboard)
