# ARC-1314 Expand Runbook Detail on Demand — Implementation Plan

**Issue:** `issues/session-setup/ARC-1314-expand-runbook-detail.md`
**Completion Summary:** `task-completions/ARC-1314-COMPLETION-SUMMARY.md` (TBD)
**Approach:** In-place expand/collapse on RunbookCard using local React state + new detail API endpoint
**Owner:** Build agent
**Date:** 2026-06-29

---

## Scope & Alignment

This plan adds expand/collapse behaviour to RunbookCard so the supervisor can reveal the full step tree (all waves → steps → sub-steps with checkbox state and next-step highlight) and collapse back to the summary view without losing in-memory state.

| AC | Step(s) that satisfy it |
|----|------------------------|
| AC-1: Click expand → all waves and steps shown with checkbox state | Steps 1, 2, 3 |
| AC-2: Click collapse → returns to summary view without losing state | Step 3 (local state preserved in component; not reset on toggle) |

**Out of scope:** editing step content, inline step execution from the detail view.

---

## Assumptions & Dependencies

- ARC-1313 is complete. `RunbookCard`, `useRunbookSummary`, `RunbookSummary` type, and `GET /api/runbooks/summary` all exist at their established paths.
- The server's `parseRunbook()` already returns a full `ParsedRunbook` with `waves → steps → subSteps` (see `packages/server/src/runbook/types.ts`). No server parser changes are needed.
- A new `GET /api/runbooks/detail` endpoint will be added that accepts a `path` query parameter and returns the `ParsedRunbook` for that file plus the current `stepState` slice — this is the server-side requirement for AC-1 (step checkbox state).
- The `stepState` is already stored in `SidecarState` (keyed by step id). The detail endpoint reads it from `getState()`.
- No client-side routing is needed — expand/collapse is in-page. The expanded detail view replaces the card body content while the card header/title remain visible.
- The worktree for this batch is at `/repos/ARC-1314` (to be created from `main` branch).
- All file paths below are relative to the repo root of `pa.aid.conductor.ts` unless otherwise noted.

---

## Implementation Steps

### Step 1: Add `GET /api/runbooks/detail` server endpoint

**Files:**
- `packages/server/src/routes/runbooks.ts` (modify — add new route handler)
- `packages/server/src/__tests__/runbooks.detail.test.ts` (new)

**Action:**

Add a route handler for `GET /api/runbooks/detail?path=<absolute-path>` in `runbooks.ts`.

Handler logic:
1. Read the `path` query parameter. If absent or empty, return `400 { error: "path query parameter is required" }`.
2. Resolve the path — confirm it is a readable file under the plans directory (`resolvePlansDir()`). If the file is outside `plansDir` or does not exist, return `400 { error: "invalid path" }`.
3. Call `parseRunbook(resolvedPath)` to get the `ParsedRunbook`.
4. Read current `stepState` from `getState()`.
5. Return `{ runbook: ParsedRunbook, stepState: Record<string, boolean> }` as JSON.

The route must be registered before the summary route (Express matches in order).

**Response shape:**
```
{ runbook: ParsedRunbook, stepState: Record<string, boolean> }
```

**Verification:** Unit test covering: valid path returns 200 with runbook + stepState; missing `path` param returns 400; path outside plans dir returns 400.

---

### Step 2: Add client-side types and `useRunbookDetail` hook

**Files:**
- `packages/ui/src/types/runbook.ts` (modify — add `RunbookStep`, `RunbookSubStep`, `RunbookWave`, `ParsedRunbook` mirror types and `RunbookDetailResponse`)
- `packages/ui/src/hooks/useRunbookDetail.ts` (new)
- `packages/ui/src/hooks/useRunbookDetail.test.ts` (new)

**Action:**

**`src/types/runbook.ts`** — append client-side mirrors of the server's step-tree types (additive, no changes to existing `RunbookSummary` / `RunbookStatus` types):
- `RunbookSubStep: { label: string; checked: boolean }$
- `RunbookStep: { id: string; label: string; type: string; checked: boolean; checkpointLevel: string | null; subSteps: RunbookSubStep[] }$
- `RunbookWave: { title: string; steps: RunbookStep[] }$
- `ParsedRunbookDetail: { metadata: RunbookMetadata; waves: RunbookWave[] }$
- `RunbookDetailResponse: { runbook: ParsedRunbookDetail; stepState: Record<string, boolean> }`

**`src/hooks/useRunbookDetail.ts`** — custom hook signature:
```
useRunbookDetail(path: string | null): { detail: RunbookDetailResponse | null; loading: boolean; error: string | null }
```

Behaviour:
- When `path` is `null`, return `{ detail: null, loading: false, error: null }` immediately (no fetch).
- When `path` is a non-null string, fetch `GET /api/runbooks/detail?path=<encoded-path>`.
- Set `loading: true` during fetch; set `loading: false` on completion (success or error).
- Set `detail` on success; set `error` on non-2xx or network failure.
- When `path` changes from a previous non-null value to a new one, re-fetch.
- Clean up with `AbortController` on unmount or `path` change.

**Verification:** Hook test scenarios: null path → no fetch, returns null detail; non-null path → fetch called with correct URL, returns parsed response; error path → error set on non-2xx.

---

### Step 3: Extend `RunbookCard` with expand/collapse and step tree rendering

**Files:**
- `packages/ui/src/components/RunbookCard.tsx` (modify)
- `packages/ui/src/components/StepTree.tsx` (new)
- `packages/ui/src/components/RunbookCard.test.tsx` (modify — add expand/collapse tests)
- `packages/ui/src/components/StepTree.test.tsx` (new)

**Action:**

**`StepTree.tsx`** — new component rendering the full parsed runbook step tree.

Props: `{ waves: RunbookWave[]; stepState: Record<string, boolean>; nextStepId: string | null }`

Rendering contract:
- For each wave: render a wave title header.
- For each step in the wave: render a checkbox (disabled, read-only — reflecting `stepState[step.id]`), the step label, and a visual "next step" highlight (e.g. `aria-current="step"` and a ring/border) if `step.id === nextStepId`.
- For each sub-step: render indented, with its own checked indicator.
- The component is purely presentational — no click handlers on checkboxes (editing is out of scope).

**`nextStepId` derivation** (done inside `RunbookCard` before passing to `StepTree`):
- Iterate waves and steps in order; the first step where `stepState[step.id]!== true` is the next step.
- Pass its `id` as `nextStepId`; pass `null` if all steps are checked.

**`RunbookCard.tsx`** modifications:
- Add `useState<boolean>(false)` for `isExpanded`.
- Add an expand/collapse toggle button in `CardHeader` (a chevron icon or text label "Expand" / "Collapse"), with `aria-expanded` set to the current state.
- When `isExpanded === false`: render existing summary body (wave name, progress bar) — unchanged from ARC-1313.
- When `isExpanded === true`: call `useRunbookDetail(summary.path)`. Render:
  - Loading state: "Loading detail…" text.
  - Error state: error message.
  - Loaded: `<StepTree>` with waves, stepState, and derived nextStepId.
- State (`isExpanded`) is held in the component; collapsing does not reset the fetched detail (the hook caches the last fetched result as `detail`).

**Verification:**
- `RunbookCard.test.tsx`: toggle expand button renders; after expand, `aria-expanded="true"` is present; collapse button is present in expanded state.
- `StepTree.test.tsx`: given a two-wave fixture, all wave titles render; checked steps show checked indicator; first unchecked step has `aria-current="step"`.

---

## Testing & Validation

| Layer | Command | What it covers |
|-------|---------|----------------|
| Server unit | `npm run test --workspace=packages/server` | New detail endpoint (400 validation, 200 response shape) |
| UI unit | `npm run test --workspace=packages/ui` | `useRunbookDetail` hook, `StepTree` rendering, `RunbookCard` expand/collapse toggle |
| Full suite | `npm test` (root) | All of the above |

Manual verification:
1. Start server: `REPO_ROOT=/repos/pa.aid.runbook-executor npm run dev`
2. Navigate to `http://localhost:5173`; confirm dashboard loads showing summary cards.
3. Click expand on one card; confirm wave/step tree appears with checkbox states.
4. Confirm the next unchecked step is visually highlighted.
5. Click collapse; confirm returns to summary (progress bar, wave name visible).
6. Click expand again; confirm previously loaded tree re-renders (no extra flash — detail hook retains last value).

---

## Risks & Open Questions

| # | Risk | Likelihood | Mitigation |
|---|------|-----------|-----------|
| 1 | Path validation allows path traversal if not restricted to plansDir | Low | Confirm resolved path starts with `plansDir` using `path.resolve()` comparison; return 400 if not |
| 2 | Step tree can be long (50+ items) — layout may overflow card | Low | Cap max-height on expanded area with overflow-y: auto; fine-tune in styling pass |
| 3 | `stepState` in detail response may lag markdown for freshly-checked boxes | Low | Same behaviour as summary endpoint — reconcile is done at `GET /api/runbooks/summary` call; detail reads from the same reconciled store |
