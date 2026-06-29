# ARC-1312 — View full step output history for any past session

## Completed: 2026-06-29

### Acceptance Criteria and Evidence

**AC 1:** Given past session, when supervisor opens it, then all executed steps listed with status and timestamp.

Evidence:
- `packages/server/src/state/types.ts` — `StepRecord` interface has `stepId`, `status`, `startedAt`, `completedAt`
- `packages/server/src/runner/stepRunner.ts` — `executeStep` writes initial record (with `startedAt`) and completed record (with `status`, `completedAt`) to `state.stepLog`
- `packages/server/src/routes/sessions.ts` — `POST /api/sessions/close` snapshots `stepLog` into `ClosedSession.steps`; steps not in `stepLog` get `status: 'pending'`
- `packages/server/src/routes/sessions.ts` — `GET /api/sessions/history/:sessionId` returns full `ClosedSession` including `steps`
- `packages/ui/src/hooks/useSessionDetail.ts` — fetches detail endpoint; returns `{ session, loading, error }`
- `packages/ui/src/pages/SessionDetailPage.tsx` — renders `StepRow` for each record in `session.steps`, showing `stepId`, `StatusBadge`, and `startedAt` timestamp (or "Never run" for pending)
- Test: `packages/server/src/__tests__/sessions.test.ts` — "snapshots stepLog records into ClosedSession.steps" passes
- Test: `packages/server/src/__tests__/stepRunner.test.ts` — "ARC-1312: populates stepLog with correct step record after successful execution" passes
- Test: `packages/ui/src/pages/SessionDetailPage.test.tsx` — "renders step list with step ids and timestamps" passes

**AC 2:** Given step in past session, when selected, then full agent output displayed.

Evidence:
- `packages/server/src/runner/stepRunner.ts` — `executeStep` stores `result.events` (cast as `StoredEvent[]`) in the completed `StepRecord`
- `packages/ui/src/pages/SessionDetailPage.tsx` — `StepRow` component is expandable; when expanded shows event list via `extractEventText()` which renders `[text]`, `[tool_use]`, or `[type]` lines for each event
- Test: `packages/ui/src/pages/SessionDetailPage.test.tsx` — "expands step row to show events on click" passes

**AC 3:** Given checkpoint in past session, when viewed, then supervisor feedback shown alongside artifact state at approval/rejection time.

Evidence:
- `packages/server/src/state/types.ts` — `StepRecord.checkpointFeedback: string | null` field defined; populated by checkpoint-management stories (ARC-1303–ARC-1305); null until those stories land
- `packages/ui/src/pages/SessionDetailPage.tsx` — `StepRow` expanded section shows "Supervisor feedback:" label + text **only when** `checkpointFeedback!== null`
- Test: `packages/ui/src/pages/SessionDetailPage.test.tsx` — "shows checkpoint feedback when non-null" passes; "does not show checkpoint feedback section when null" passes

Note: AC 3 is structurally complete — the storage contract and UI rendering are fully implemented. Feedback will be non-null once ARC-1303–ARC-1305 (checkpoint-management) write feedback into step records.

### Files Changed

| File | Change |
|------|--------|
| `packages/server/src/state/types.ts` | Added `StepRecord` interface; `stepLog?` to `SidecarState`; `steps` to `ClosedSession` |
| `packages/server/src/runner/stepRunner.ts` | Updated `executeStep` to write `startedAt` and completed `StepRecord` to `stepLog` |
| `packages/server/src/routes/sessions.ts` | Updated `POST /api/sessions/close` to snapshot `stepLog` into `steps`, clear `stepLog`; added `GET /api/sessions/history/:sessionId` |
| `packages/ui/src/types/runbook.ts` | Added `StepRecord`, `StoredEvent` client-side mirror types; `steps` field to `ClosedSession` |
| `packages/ui/src/hooks/useSessionDetail.ts` | New file — hook for fetching single session detail |
| `packages/ui/src/pages/SessionDetailPage.tsx` | New file — session detail UI with expandable step rows |
| `packages/ui/src/App.tsx` | Wired `SessionDetailPage` replacing ARC-1311 placeholder |
| `packages/server/src/__tests__/sessions.test.ts` | Added stepLog snapshot tests, GET /:sessionId tests |
| `packages/server/src/__tests__/stepRunner.test.ts` | Added stepLog population tests |
| `packages/ui/src/hooks/useSessionDetail.test.ts` | New file — hook tests |
| `packages/ui/src/pages/SessionDetailPage.test.tsx` | New file — page render tests |

### Test Results

```
Server: 135 tests passed (13 test files)
UI: 89 tests passed (12 test files)
TypeScript: no errors (both packages)
```

### Deviations from plan

- **Import cycle mitigation (proactive):** The plan noted a risk that `state/types.ts` importing from `runner/types.ts` could cause a circular import. This was resolved proactively by defining `StoredEvent = Record<string, unknown>` inline in `state/types.ts` — no import from `runner/` needed. `StepRecord.events` uses `StoredEvent[]`; `stepRunner.ts` casts `result.events as StoredEvent[]` at the assignment boundary.
- **ARC-1311 and ARC-1312 types combined in one commit:** Because both stories land on the same branch, `StepRecord`/`StoredEvent` types and `stepLog` field were added to `state/types.ts` in the same commit as `ClosedSession`. The `ClosedSession` interface was authored with `steps: StepRecord[]` from the start to avoid a later schema migration on the same branch.

