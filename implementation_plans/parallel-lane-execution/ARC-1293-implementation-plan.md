---
# ARC-1293 Stream Live Agent Output Per Lane in the UI — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1293-stream-agent-output.md`
**Completion Summary:** `task-completions/ARC-1293-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Per-lane SSE endpoint over existing eventBus (pattern established by `routes/steps.ts`)
**Owner:** build-agent
**Date:** 2026-06-30

---

## Scope & Alignment

This plan delivers live per-lane output streaming for the active session UI. It covers:

- A new server-side SSE endpoint `GET /api/lanes/:runbookPath/stream` that fans out `eventBus` events for all steps currently executing within a lane.
- A new UI hook `useLaneStream` that opens and manages the SSE connection for a given runbook path.
- A new UI component `LaneOutputPanel` that subscribes via `useLaneStream` and renders incoming events in real time, one panel per lane.
- Mounting `LaneOutputPanel` instances in `Dashboard.tsx` for each selected runbook when a session is active.

**AC mapping:**

| AC | Satisfied by |
|---|---|
| AC 1 — each subprocess event appears in the lane panel within 1 second | Step 1 (server SSE) + Step 2 (eventBus fan-out) + Step 3 (`useLaneStream`) + Step 4 (`LaneOutputPanel`) |
| AC 2 — output correctly isolated per lane panel | Step 1 (per-`runbookPath` endpoint) + Step 2 (fan-out subscribes per step in the active lane) + Step 4 (one panel instance per runbook path) |

---

## Assumptions & Dependencies

- ARC-1291 is merged: `runLane` and `executeStep` are in production. `eventBus.publish(stepId, event)` is called for every subprocess event during step execution.
- ARC-1292 is merged: `laneScheduler` and `getLaneState` are available; `laneStatusMap` owns the canonical lane status.
- The `eventBus` module exports `subscribe(stepId, listener)` and `publish(stepId, event)`. No changes to `eventBus` are required for this story.
- `laneRunner.ts` does not expose which `stepId` is currently executing for a given lane. The fan-out bridge (Step 2) must track this itself by observing the step-execution lifecycle.
- `GET /api/state` already returns `laneState: Record<string, LaneStatus>` (from `routes/state.ts`). The UI can read this to know which lanes are active and for which runbook paths to render output panels.
- `SidecarState.selectedRunbooks` is the authoritative list of lane paths for a session — the same array already drives `laneState` in `GET /api/state`.
- The UI `Dashboard.tsx` controls session lifecycle state (`hasActiveSession`). Lane output panels are mounted when a session is active.
- All SSE streams share the existing Express server — no new server process or WebSocket upgrade is required.
- Node.js single-threaded execution guarantees: registering and unregistering step subscriptions between `await` boundaries in `runLane` is safe. Fan-out bridge subscribes when a step starts and unsubscribes when it ends.

---

## Implementation Steps

### Step 1: Add a per-lane active-step registry in `runner/laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Add a module-level `Map<string, string>` named `laneActiveStep` that records the stepId currently executing within each lane, keyed by `runbookPath`.

Export a read function `getActiveLaneStep(runbookPath: string): string | undefined` — returns the stepId of the step currently executing in that lane, or `undefined` if the lane is idle/paused/done.

Within `runLane`, update `laneActiveStep` around each agent step's `executeStep` call:
- Set `laneActiveStep.set(runbookPath, step.id)` immediately before calling `executeStep`.
- Delete `laneActiveStep.delete(runbookPath)` immediately after `executeStep` resolves (whether success or failure), before the lane status is changed.

Extend `_resetLaneStateForTesting` to also clear `laneActiveStep`.

**Verification:** `tsc --noEmit` compiles cleanly. After calling `_resetLaneStateForTesting()`, `getActiveLaneStep` returns `undefined` for any path.

---

### Step 2: Create `runner/laneEventFanout.ts` — SSE bridge from eventBus to per-lane stream

**Files:** `packages/server/src/runner/laneEventFanout.ts` (new file)

**Action:** Create a module that bridges the per-step `eventBus` to per-lane listeners. It maintains:

- `laneListeners: Map<string, Set<LaneEventListener>>` — per-`runbookPath` listener sets.
- `stepUnsubscribes: Map<string, () => void>` — cleanup callbacks returned by `eventBus.subscribe`, keyed by `stepId`.

Define and export:

```ts
export type LaneEventListener = (stepId: string, event: OpenCodeEvent) => void;

export function subscribeLane(runbookPath: string, listener: LaneEventListener): () => void
export function notifyLaneStep(runbookPath: string, stepId: string): void
export function endLaneStep(runbookPath: string, stepId: string): void
export function _resetFanoutForTesting(): void
```

`subscribeLane(runbookPath, listener)` adds the listener to `laneListeners[runbookPath]` and returns an unsubscribe function that removes it (cleaning up the Set entry when empty).

`notifyLaneStep(runbookPath, stepId)` is called from `laneRunner.ts` when a step begins. It calls `eventBus.subscribe(stepId, event => deliverToLane(runbookPath, stepId, event))`, stores the returned unsubscribe in `stepUnsubscribes[stepId]`, and fans out future events for that step to all current `laneListeners[runbookPath]` entries.

`endLaneStep(runbookPath, stepId)` is called from `laneRunner.ts` when a step ends. It calls `stepUnsubscribes.get(stepId)?.()` to unsubscribe from the eventBus and removes the entry from `stepUnsubscribes`.

`deliverToLane(runbookPath, stepId, event)` (internal) — iterates `laneListeners[runbookPath]` and calls each listener with `(stepId, event)`.

**Verification:** `tsc --noEmit` compiles cleanly. Unit test: register a lane listener, call `notifyLaneStep`, publish an event to the eventBus for that stepId, and assert the lane listener was called with `(stepId, event)`. Call `endLaneStep` and publish again; assert the listener is not called.

---

### Step 3: Integrate `laneEventFanout` into `laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Import `notifyLaneStep` and `endLaneStep` from `./laneEventFanout.js`.

In `runLane`, within the agent step execution block:
- Call `notifyLaneStep(runbookPath, step.id)` immediately before calling `executeStep`.
- Call `endLaneStep(runbookPath, step.id)` in a `finally` block wrapping the `executeStep` call, so it fires regardless of success or failure.

This must be done consistently with the `laneActiveStep` updates from Step 1: the order is `laneActiveStep.set` → `notifyLaneStep` → `await executeStep` → `endLaneStep` (finally) → `laneActiveStep.delete`.

**Verification:** `tsc --noEmit` compiles cleanly. Existing `laneRunner` unit tests still pass.

---

### Step 4: Add `GET /api/lanes/:encodedPath/stream` SSE endpoint to `routes/steps.ts`

**Files:** `packages/server/src/routes/steps.ts`

**Action:** Add a new route to the existing `steps` router:

``
GET /api/lanes/:encodedPath/stream
``

The `encodedPath` parameter is the URL-encoded absolute runbook file path (same encoding as `SidecarState.selectedRunbooks` entries — the client encodes with `encodeURIComponent`).

On request:
1. Decode `encodedPath` via `decodeURIComponent(req.params.encodedPath)` to get `runbookPath`.
2. Set SSE headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`, then call `res.flushHeaders()`.
3. Initialize a `seq` counter at 0.
4. Call `subscribeLane(runbookPath, (stepId, event) => { res.write(`data: ${JSON.stringify({ runbookPath, stepId, event, seq: seq++ })}\n\n`); })`, storing the returned unsubscribe function.
5. On `req.on('close',...)`, call the unsubscribe function and end the response.

The SSE wire format per event:

``
data: {"runbookPath":"<path>","stepId":"<stepId>","event":<OpenCodeEvent>,"seq":<int>}\n\n
``

No terminal "done" event is needed — the client should tolerate the connection closing when the lane finishes (the lane status in `GET /api/state` is the source of truth for completion).

Import `subscribeLane` and `LaneEventListener` from `../runner/laneEventFanout.js`.

**Verification:** Manual: start a session, open the SSE stream for a selected runbook path, confirm events arrive within 1 second of the subprocess emitting them. Unit test: mock `subscribeLane`, mount the route, send a GET request, confirm SSE headers are set and the subscriber is registered.

---

### Step 5: Create `hooks/useLaneStream.ts` in the UI

**Files:** `packages/ui/src/hooks/useLaneStream.ts` (new file)

**Action:** Implement a React hook:

```ts
interface LaneStreamEvent {
  runbookPath: string;
  stepId: string;
  event: Record<string, unknown>;
  seq: number;
}

export function useLaneStream(runbookPath: string | null): LaneStreamEvent[]
```

Behaviour:
- Returns an ordered array of all `LaneStreamEvent` objects received since mount (or since `runbookPath` last changed).
- When `runbookPath` is `null`, the hook is idle and returns `[]`.
- When `runbookPath` is non-null, opens an `EventSource` to `/api/lanes/${encodeURIComponent(runbookPath)}/stream`.
- Each SSE `message` event: parses the JSON payload from `event.data`, appends it to the events array via `setState(prev => [...prev, parsed])`.
- On `EventSource.onerror`: does not throw — leaves the accumulated events in place. The panel renders what it has received.
- On cleanup (unmount or `runbookPath` change): calls `eventSource.close()` and resets the events array to `[]`.

**Verification:** Unit test using a mock `EventSource` (or `vi.fn()` factory): verify the hook starts with `[]`, appends parsed events as messages arrive, and clears on cleanup.

---

### Step 6: Create `components/LaneOutputPanel.tsx`

**Files:** `packages/ui/src/components/LaneOutputPanel.tsx` (new file)

**Action:** Implement a presentational component:

```ts
interface LaneOutputPanelProps {
  runbookPath: string;
  laneLabel: string;
}

export function LaneOutputPanel({ runbookPath, laneLabel }: LaneOutputPanelProps)
```

Behaviour:
- Calls `useLaneStream(runbookPath)` to obtain the event stream.
- Renders a labelled panel (using `laneLabel` as the heading — derived from the runbook filename, formatted by the caller).
- When no events have arrived: renders a placeholder "Waiting for agent output…" message.
- For each event in the stream: renders one line using the same `extractEventText` logic already present in `SessionDetailPage.tsx` (move or duplicate this helper — see note below). Displays the `stepId` as a prefix on each line so the supervisor can distinguish output across step boundaries within the same lane.
- Scrolls to the bottom automatically as new events arrive (use a `useEffect` with a `ref` on the scroll container).

**Note on `extractEventText`:** The helper is currently private to `SessionDetailPage.tsx`. It should be extracted to `lib/formatEvent.ts` and imported by both `SessionDetailPage.tsx` and `LaneOutputPanel.tsx`. This extraction is part of this step.

**Verification:** Unit test with a mocked `useLaneStream` returning a fixed event array: assert each event is rendered with its `stepId` prefix and the formatted text. Assert "Waiting for agent output…" renders when the array is empty.

---

### Step 7: Mount `LaneOutputPanel` instances in `Dashboard.tsx`

**Files:** `packages/ui/src/pages/Dashboard.tsx`

**Action:** When `hasActiveSession` is true and `selectedRunbooks` is non-empty, render one `LaneOutputPanel` per selected runbook below the session action buttons.

The `laneLabel` prop is derived from the runbook path filename: strip the `runbook-` prefix and `.md` suffix (mirrors the `laneFromRunbookPath` logic on the server), then capitalise the first letter. Pass the raw `runbookPath` as the `runbookPath` prop.

The panels are rendered as a vertically stacked list, one per lane. Each panel is independent — `LaneOutputPanel` manages its own `useLaneStream` subscription.

**Verification:** Manual: start a session with two runbooks selected; confirm two output panels appear, each labelled with their lane name, and each populates independently as agents execute. Unit test: render `Dashboard` with `hasActiveSession = true` (via local state manipulation in a test wrapper) and `selectedRunbooks = ['path/runbook-foo.md', 'path/runbook-bar.md']`; assert two `LaneOutputPanel` elements are rendered.

---

### Step 8: Extract `extractEventText` to `lib/formatEvent.ts`

**Files:**
- `packages/ui/src/lib/formatEvent.ts` (new file)
- `packages/ui/src/pages/SessionDetailPage.tsx`

**Action:** Move the `extractEventText` function from `SessionDetailPage.tsx` into `packages/ui/src/lib/formatEvent.ts`. Export it. Update `SessionDetailPage.tsx` to import it from `@/lib/formatEvent`. `LaneOutputPanel.tsx` (Step 6) also imports from `@/lib/formatEvent`.

**Verification:** `tsc --noEmit` compiles cleanly. Existing `SessionDetailPage` tests pass without modification.

---

### Step 9: Add server-side unit tests for `laneEventFanout`

**Files:** `packages/server/src/__tests__/laneEventFanout.test.ts` (new file)

**Action:** Write unit tests covering the following scenarios. Use `vi.mock('../runner/eventBus.js')` to control event delivery. Call `_resetFanoutForTesting()` in `beforeEach`.

Scenarios:
- **Fan-out to single lane listener:** register one lane listener via `subscribeLane`, call `notifyLaneStep`, publish an event to the eventBus for that stepId, assert the lane listener received `(stepId, event)`.
- **Fan-out to multiple lane listeners for same lane:** register two listeners for the same `runbookPath`, assert both receive the same event.
- **Isolation between lanes:** register a listener for lane A and another for lane B, publish an event for a step in lane A, assert only lane A's listener is called.
- **Unsubscribe stops delivery:** call the unsubscribe returned by `subscribeLane`, publish a new event, assert the listener is not called.
- **`endLaneStep` stops eventBus subscription:** call `notifyLaneStep`, then `endLaneStep`, then publish an event for that stepId to the eventBus, assert no lane listener is called.
- **Multiple sequential steps in one lane:** call `notifyLaneStep` for step1, publish events, call `endLaneStep` for step1; then call `notifyLaneStep` for step2, publish events; assert the lane listener received events for both steps with their respective stepIds, and that step1 events after `endLaneStep` are not delivered.

**Verification:** `vitest run` in `packages/server` passes all new tests.

---

### Step 10: Verify full test suite and type-check

**Files:** all test files under `packages/server/src/__tests__/` and `packages/ui/src/`

**Action:** Run `vitest run` from both `packages/server/` and `packages/ui/`. Run `tsc --noEmit` from both packages. All must pass with zero errors.

**Verification:** Clean output from all four commands.

---

## Testing & Validation

All automated verification is via `vitest run` and `tsc --noEmit` in both packages.

**Scenario coverage:**

- **AC 1 — within-1-second delivery:** events published to the eventBus by `executeStep` → `publish(stepId, event)` are forwarded via `notifyLaneStep` → `laneEventFanout` → SSE write. No polling, no buffering delay. The event path is synchronous from `publish` call to `res.write` call; network round-trip is the only latency.
- **AC 2 — lane isolation:** each `LaneOutputPanel` opens its own `EventSource` to its own `runbookPath` endpoint. The server's `laneListeners` map is keyed by `runbookPath` — events for lane A are never delivered to lane B's listener set.
- **No-event state:** `LaneOutputPanel` renders "Waiting for agent output…" when no events have arrived.
- **Step-boundary labelling:** each event line is prefixed with `stepId` so the supervisor can identify which step produced each line.
- **EventSource error resilience:** `useLaneStream` does not throw on `EventSource.onerror`; accumulated events remain visible.
- **Panel cleanup on session close:** when `hasActiveSession` becomes false, `LaneOutputPanel` instances unmount and `useLaneStream` closes the `EventSource` connection.
- **Server-side fan-out tests:** all scenarios in Step 9 pass.
- **UI hook and component tests:** `useLaneStream` and `LaneOutputPanel` unit tests pass with mocked event sources.

**End-to-end smoke test (manual):**
1. Start the server (`npm run dev` in `/repos/pa.aid.conductor.ts`).
2. Select two runbooks; click "Start Session".
3. Confirm two lane output panels appear below the session controls.
4. As each lane executes agent steps, confirm events appear in the correct panel within ~1 second.
5. Confirm no cross-lane event bleed (lane A's events appear only in lane A's panel).

---

## Risks & Open Questions

- **`EventSource` reconnect on stream close:** When `runLane` completes (lane reaches `done`), the SSE connection is closed by the server when the `req` closes. The browser's `EventSource` will attempt to reconnect after a timeout. `useLaneStream` must not clear accumulated events on reconnect — only on unmount or `runbookPath` prop change. The `onerror` handler should be a no-op (leave events in place); any reconnect simply appends more events.
- **URL encoding of runbook paths:** The `encodedPath` route parameter uses `encodeURIComponent` on the client. Express does not double-decode URL parameters by default (`app.set('router caseSensitive', true)` is not the concern here — the issue is that `/` characters in the absolute path, after encoding, become `%2F` which some reverse proxies decode before Express sees them). If the dev server proxy (Vite) normalises `%2F`, a query-parameter encoding (`GET /api/lanes/stream?path=<encodedPath>`) may be more robust. This is flagged as a risk; the plan uses the path-parameter approach and notes the proxy consideration. If integration tests reveal proxy decode issues, the query-parameter variant is a drop-in replacement with no semantic changes.
- **Historical events on late connect:** The SSE stream delivers only events emitted after the client connects. If the UI renders the panel after a step has already produced output (e.g. slow initial page load), earlier events are not replayed. For this story, the in-session `stepLog` (already stored in `SidecarState.stepLog` by `executeStep`) is the source of truth for completed-step output; the stream delivers live events only. A future story may render `stepLog` events on panel mount as a historical seed — out of scope here.
- **Concurrent same-path subscriptions:** If two browser tabs open the same lane stream simultaneously, both receive all events (the `laneListeners` Set holds both listeners). This is benign for a single-supervisor tool.

---
