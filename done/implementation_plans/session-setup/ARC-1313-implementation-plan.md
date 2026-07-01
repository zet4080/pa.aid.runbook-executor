# ARC-1313 Display Runbook Dashboard at Startup — Implementation Plan

**Issue:** `issues/session-setup/ARC-1313-display-runbook-dashboard.md`
**Completion Summary:** `task-completions/ARC-1313-display-runbook-dashboard-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Server-side enrichment + React hooks + Tailwind CSS + shadcn/ui
**Owner:** Build agent
**Date:** 2026-06-28

---

## Scope & Alignment

This plan implements the collapsed runbook dashboard shown on tool startup. It covers:

1. A new `GET /api/runbooks/summary` endpoint that returns each runbook enriched with computed `status` (idle / running / blocked / complete), the current `wave`, and `progressPct`.
2. An in-memory `runningSteps` tracker in the server so the summary endpoint can derive "running" status without touching the ephemeral event bus.
3. Tailwind CSS + shadcn/ui wiring in `packages/ui` (install + config).
4. A `useRunbookSummary` hook that fetches the summary endpoint and auto-refreshes every 2 seconds (to meet the ≤ 3 s load target and keep the dashboard live).
5. UI components: `RunbookCard`, `StatusBadge`, `ProgressBar`, and the filled-in `Dashboard` page.
6. Vitest + React Testing Library setup for the UI package and unit/integration tests for all new code.

**Out of scope (ARC-1314):** expand-on-demand step detail view; editing runbooks.

### Acceptance Criteria → Steps

| AC | Step(s) that satisfy it |
|----|------------------------|
| AC-1: All runbooks listed with name, status, wave, % progress within 3 seconds | Steps 2, 5, 6, 7 |
| AC-2: Each runbook shows correct status badge | Steps 3, 4, 5, 6 |

---

## Assumptions & Dependencies

- **ARC-1286 (runbook parser)** is complete. `parseRunbook()` exists at `packages/server/src/runbook/parser.ts` and returns `ParsedRunbook`. Confirmed present.
- `GET /api/runbooks` already exists and returns `ParsedRunbook[]`. The new `/api/runbooks/summary` endpoint is additive — it does not replace or alter the existing route.
- `SidecarState.stepState` is `Record<string, boolean>` (stepId → markdown-checked). It is NOT a source of "running" status — running state is ephemeral and must be tracked separately (see Step 3).
- `SidecarState.checkpointQueue` holds pending checkpoint entries. A runbook is "blocked" if it has a non-empty intersection between its step IDs and the queue.
- The `packages/ui` Vite dev server already proxies `/api` to `localhost:4000` via `vite.config.ts`. No proxy changes needed.
- shadcn/ui requires a `paths` alias (`@/*`) in tsconfig and vite config. Both will be added in Step 1.
- No client-side routing library is needed — the dashboard is the only view (ARC-1314 expand-on-demand will be in-page).
- The worktree is at `/repos/ARC-1313`. All file paths below are relative to `/repos/ARC-1313/`.

---

## Implementation Steps

---

### Step 1: Configure Tailwind CSS + shadcn/ui in `packages/ui`

**Files:**
- `packages/ui/package.json`
- `packages/ui/vite.config.ts`
- `packages/ui/tsconfig.json`
- `packages/ui/tailwind.config.js` *(new)*
- `packages/ui/postcss.config.js` *(new)*
- `packages/ui/src/index.css` *(new or modify)*
- `packages/ui/components.json` *(new — shadcn config)*

**Action:**

Install deps:
```
tailwindcss postcss autoprefixer
@radix-ui/react-slot class-variance-authority clsx tailwind-merge
```

1. Add `tailwind.config.js` with `content: ["./src/**/*.{ts,tsx}"]`.
2. Add `postcss.config.js` referencing tailwindcss and autoprefixer plugins.
3. Create `src/index.css` with the three Tailwind directives (`@tailwind base/components/utilities`). Import it in `src/main.tsx`.
4. Add path alias `@` → `./src` in `vite.config.ts` (`resolve.alias`) and in `tsconfig.json` (`paths: { "@/*": ["./src/*"] }`). Set `baseUrl: "."` in tsconfig.
5. Add `components.json` (shadcn CLI config) pointing to `src/components/ui/` as the component output directory, using the `default` style and `zinc` base color.
6. Scaffold shadcn `badge`, `card`, and `progress` primitives into `src/components/ui/` (run `npx shadcn@latest add badge card progress` or copy the component source manually — no registry network access needed).

**Verification:** `npm run lint --workspace=packages/ui` passes (TypeScript compiles). `npm run build --workspace=packages/ui` produces a `dist/` without errors. Tailwind classes present in the built CSS.

---

### Step 2: Add `RunbookSummary` type to shared types

**Files:**
- `packages/server/src/runbook/types.ts` *(modify — add new exported interfaces)*

**Action:**

Add three new exported types after the existing `ParsedRunbook` interface:

- `RunbookStatus`: `'idle' | 'running' | 'blocked' | 'complete'`
- `RunbookSummary`: `{ path: string; metadata: RunbookMetadata; status: RunbookStatus; currentWave: string | null; progressPct: number; totalSteps: number; checkedSteps: number; }`

No changes to existing types — additive only.

**Verification:** `npm run test --workspace=packages/server` still passes (existing tests unchanged).

---

### Step 3: Add `runningSteps` in-memory tracker

**Files:**
- `packages/server/src/runner/runningSteps.ts` *(new)*
- `packages/server/src/runner/stepRunner.ts` *(modify)*

**Action:**

Create `runningSteps.ts` as a module exporting:
- `markRunning(stepId: string): void` — adds stepId to an internal `Set<string>`
- `markDone(stepId: string): void` — removes stepId from the set
- `getRunningStepIds(): ReadonlySet<string>` — returns the current set

Modify `executeStep` in `stepRunner.ts` to call `markRunning(stepId)` before `runStep(...)` and `markDone(stepId)` in a `finally` block after `runStep(...)` resolves or rejects.

**Verification:**
- Unit test in `src/__tests__/runningSteps.test.ts`: after `markRunning('x')`, `getRunningStepIds()` contains `'x'`; after `markDone('x')`, it does not.
- `npm run test --workspace=packages/server` passes.

---

### Step 4: Implement status derivation function

**Files:**
- `packages/server/src/runbook/summarize.ts` *(new)*
- `packages/server/src/__tests__/summarize.test.ts` *(new)*

**Action:**

Create `summarize.ts` exporting a pure function:

```ts
deriveRunbookSummary(
  runbookPath: string,
  runbook: ParsedRunbook,
  stepState: Record<string, boolean>,
  runningStepIds: ReadonlySet<string>,
  checkpointQueue: CheckpointQueueEntry[],
): RunbookSummary
```

**Status derivation rules (in priority order):**

1. **running** — any step in this runbook has its `id` in `runningStepIds`.
2. **blocked** — any step in this runbook has its `id` in `checkpointQueue` (matched by `entry.stepId`) and that step is not checked.
3. **complete** — all steps across all waves have `stepState[step.id] === true`. If there are no steps, status is `complete`.
4. **idle** — none of the above.

**Progress calculation:**
- `totalSteps` = count of all `RunbookStep` records across all waves.
- `checkedSteps` = count where `stepState[step.id] === true`.
- `progressPct` = `totalSteps === 0 ? 100 : Math.round((checkedSteps / totalSteps) * 100)`.

**Current wave:**
- `currentWave` = title of the first wave that contains at least one unchecked step; `null` if all steps are checked (i.e., complete).

**Verification:** Unit tests in `summarize.test.ts` covering:
- All-unchecked runbook → status `idle`, progressPct `0`
- All-checked runbook → status `complete`, progressPct `100`
- Runbook with a stepId in `runningStepIds` → status `running`
- Runbook with a stepId in `checkpointQueue` → status `blocked`
- Running takes priority over blocked
- `currentWave` is the first wave with unchecked steps
- Zero-step runbook → status `complete`, progressPct `100`

---

### Step 5: Add `GET /api/runbooks/summary` endpoint

**Files:**
- `packages/server/src/routes/runbooks.ts` *(modify — add new route handler)*
- `packages/server/src/__tests__/runbooks.summary.test.ts` *(new)*

**Action:**

Add a second route handler in `runbooks.ts` before `export default router`:

```
GET /api/runbooks/summary
```

Handler logic:
1. Read `plansDir` (same as existing route: `path.resolve(process.cwd(), 'docs', 'plans')`).
2. Glob `runbook-*.md` files from that directory.
3. Parse each with `parseRunbook()`.
4. Retrieve current state via `getState()` (gives `stepState` and `checkpointQueue`).
5. Retrieve `getRunningStepIds()` from `runningSteps.ts`.
6. Call `deriveRunbookSummary()` for each runbook, passing the relative `path` as the `runbookPath`.
7. Return `RunbookSummary[]` as JSON.

Error handling: if `plansDir` is not readable, return `500` with an error message consistent with the existing `/runbooks` route pattern.

**Performance:** all parsing is synchronous (`readdirSync` + `parseRunbook`). For the 3-second target: with ≤ 7 runbook files (current count in planning repo), this completes in well under 100 ms. No caching needed at this stage.

**Verification:** Integration test in `runbooks.summary.test.ts` using `supertest` against the Express `app`:
- Mock `process.cwd()` to point at a fixture directory with a stub `docs/plans/runbook-test.md`.
- Assert `GET /api/runbooks/summary` returns 200 with an array containing a `RunbookSummary` with correct `status`, `progressPct`, `currentWave`, and `metadata`.
- Assert missing `plansDir` returns 500.

---

### Step 6: Build UI components

**Files:**
- `packages/ui/src/types/runbook.ts` *(new)*
- `packages/ui/src/hooks/useRunbookSummary.ts` *(new)*
- `packages/ui/src/components/StatusBadge.tsx` *(new)*
- `packages/ui/src/components/RunbookCard.tsx` *(new)*
- `packages/ui/src/pages/Dashboard.tsx` *(replace stub)*

**Action:**

**`src/types/runbook.ts`**
Mirror the server-side `RunbookSummary` and `RunbookStatus` types. These are typed manually (no shared package) to keep the UI package self-contained. Include: `RunbookStatus`, `RunbookSummary`.

**`src/hooks/useRunbookSummary.ts`**
Custom hook signature: `useRunbookSummary(): { summaries: RunbookSummary[]; loading: boolean; error: string | null }`.

Behaviour:
- On mount, fetch `GET /api/runbooks/summary` and set `summaries`.
- Set `loading: true` during the first fetch only; subsequent refreshes do not flip loading back to true (prevents flash).
- Set `error` if fetch throws or response is non-2xx; clear error on next successful fetch.
- Poll every 2000 ms via `setInterval`. Clear interval on unmount.
- Abort in-flight fetch on unmount using `AbortController`.

**`src/components/StatusBadge.tsx`**
Props: `{ status: RunbookStatus }`. Renders a shadcn `<Badge>` with a variant and label per status:

| status | variant | label |
|--------|---------|-------|
| idle | secondary | Idle |
| running | default (blue) | Running |
| blocked | destructive | Blocked |
| complete | outline (green) | Complete |

**`src/components/RunbookCard.tsx`**
Props: `{ summary: RunbookSummary }`. Renders a shadcn `<Card>` with:
- `CardHeader`: runbook title (`summary.metadata.title`) + `<StatusBadge>` aligned right.
- `CardContent`: one row with "Wave: {currentWave ?? '—'}" and "Progress: {progressPct}%".
- shadcn `<Progress>` bar showing `progressPct`.

The card is non-interactive (no click handler) — expand-on-demand is ARC-1314.

**`src/pages/Dashboard.tsx`** (replace stub)
Renders the full dashboard:
- Header: `<h1>Runbook Dashboard</h1>` with a subtitle showing runbook count.
- Loading state: a simple "Loading…" text (shown only on first load, `loading === true && summaries.length === 0`).
- Error state: a red alert div showing the error message.
- Runbook list: `summaries.map(s => <RunbookCard key={s.path} summary={s} />)` in a responsive CSS grid (2 columns on ≥ 768 px, 1 column on mobile).
- Empty state: "No runbooks found." when `summaries.length === 0` and not loading.

**Verification:**
- `npm run lint --workspace=packages/ui` (TypeScript) passes.
- `npm run build --workspace=packages/ui` produces clean `dist/`.
- Open `http://localhost:5173` in dev mode and confirm all 7 runbooks are listed with name, status badge, wave, and progress bar.

---

### Step 7: Configure Vitest + React Testing Library for `packages/ui`

**Files:**
- `packages/ui/package.json` *(modify — add test deps and test script)*
- `packages/ui/vite.config.ts` *(modify — add `test` config block)*
- `packages/ui/src/hooks/useRunbookSummary.test.ts` *(new)*
- `packages/ui/src/components/StatusBadge.test.tsx` *(new)*
- `packages/ui/src/components/RunbookCard.test.tsx` *(new)*
- `packages/ui/src/pages/Dashboard.test.tsx` *(new)*

**Action:**

Install test deps into `packages/ui`:
```
vitest @vitest/ui jsdom
@testing-library/react @testing-library/user-event @testing-library/jest-dom
```

Add `"test": "vitest run"` to `packages/ui/package.json` scripts.

Add Vitest config to `vite.config.ts`:
```ts
test: { environment: 'jsdom', globals: true, setupFiles: ['./src/test-setup.ts'] }
```

Create `src/test-setup.ts` importing `@testing-library/jest-dom`.

Update root `package.json` `"test"` script to run tests in both workspaces:
```
npm run test --workspace=packages/server && npm run test --workspace=packages/ui
```

**Test scenarios for UI components (describe intent, not implementation):**

- `StatusBadge`: each of the four status values renders the correct label text.
- `RunbookCard`: given a `RunbookSummary` fixture with `status: 'running'`, renders the title, "Running" badge, wave name, and progress value.
- `useRunbookSummary`: mock `fetch` to return a summary array; assert hook returns correct `summaries` after fetch resolves; assert `loading` is `true` before fetch resolves; assert `error` is set on non-2xx response.
- `Dashboard`: mock `useRunbookSummary` to return a fixture with 2 summaries; assert 2 `RunbookCard` elements render; mock loading state and assert "Loading…" text is shown; mock error state and assert error message is shown.

**Verification:** `npm run test --workspace=packages/ui` passes with all scenarios green.

---

## Testing & Validation

### Automated tests

| Layer | Command | What it covers |
|-------|---------|----------------|
| Server unit | `npm run test --workspace=packages/server` | `runningSteps.ts` tracker, `summarize.ts` derivation logic, `/api/runbooks/summary` integration |
| UI unit | `npm run test --workspace=packages/ui` | `StatusBadge`, `RunbookCard`, `Dashboard`, `useRunbookSummary` hook |
| Full suite | `npm test` (root) | All of the above |

### Manual end-to-end verification

1. Start server from the `pa.aid.runbook-executor` planning repo root: `REPO_ROOT=/repos/pa.aid.runbook-executor npm run dev`.
2. Navigate to `http://localhost:5173`.
3. Confirm dashboard loads within 3 seconds showing all 7 runbooks (core-infrastructure, session-setup, parallel-lane-execution, checkpoint-management, queue-scheduling-policy, failure-handling, session-history).
4. Confirm each card shows: title from runbook `metadata.title`, a status badge, current wave (or `—`), and a progress bar.
5. For a runbook with no steps checked, confirm `status: idle` and `progressPct: 0`.
6. Manually mark a step as checked in a runbook markdown file, restart server, confirm `progressPct` increases and `status` moves to `complete` if all steps are checked.
7. Confirm the 2-second polling works: modify a runbook markdown on disk, trigger a server restart, and within 2-4 seconds the UI updates without a page reload.

### Performance target

`GET /api/runbooks/summary` must respond in < 500 ms with all 7 runbooks. Verify with `curl -w "%{time_total}" http://localhost:4000/api/runbooks/summary`.

---

## Risks & Open Questions

| # | Risk | Likelihood | Mitigation |
|---|------|-----------|-----------|
| 1 | shadcn/ui registry requires network access at install time | Medium | The `add` command fetches from `registry.shadcn.com`. If the dev environment has no outbound internet, copy the three component sources (`badge.tsx`, `card.tsx`, `progress.tsx`) from the shadcn GitHub repo directly into `src/components/ui/`. |
| 2 | `path` field in `RunbookSummary` may differ between OS (absolute vs relative) | Low | The server always passes the resolved absolute path from `plansDir`. The client uses `path` only as a React `key`, not for display. Consistent. |
| 3 | No step IDs in the sidecar on fresh start (empty `stepState`) | Low | `loadOrCreateState` seeds `stepState` from markdown at startup. Status derivation handles `stepState[id] === undefined` as falsy (unchecked) — same behaviour as `false`. |
| 4 | Running status is lost on server restart | Low | The `runningSteps` set is in-memory only. A restart clears it. This is acceptable: if the server restarts, the step is no longer running. The `stepRunner` `finally` block ensures `markDone` is always called on normal exit. An abrupt kill will reset everything — acceptable given ARC-1291's serialisation model. |
| 5 | 3-second load target with slow markdown parsing | Low | Parsing 7 runbook files synchronously is benchmarked at < 100 ms. If runbook count grows significantly, introduce a lazy parse cache keyed by `mtime`. Not needed now. |
| 6 | Tailwind purge may strip shadcn dynamic classes | Medium | Set `safelist` in `tailwind.config.js` for any programmatically constructed class names used in `StatusBadge` (status-to-variant mapping). Alternatively, use `clsx`/`tailwind-merge` with static strings only. |
