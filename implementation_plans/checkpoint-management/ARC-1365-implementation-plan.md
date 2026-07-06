# ARC-1365 Display agent last message on checkpoint Resume card - Implementation Plan

**Issue:** `issues/checkpoint-management/ARC-1365-agent-last-message-on-checkpoint.md`
**Completion Summary:** `task-completions/ARC-1365-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established
**Owner:** build-agent
**Date:** 2026-07-06

---

## Scope & Alignment

This plan covers adding `lastAgentMessage` capture at checkpoint-hit time on the server and rendering it on the Resume card in the UI.

| AC | Step |
|----|------|
| AC1 — short message shown inline | Step 4 (`ResumeCard` extension) |
| AC2 — long message truncated with "Show more…" | Step 4 (`ResumeCard` extension) |
| AC3 — modal shows full message | Step 3 (`AgentMessageModal`) + Step 4 |
| AC4 — closing modal restores truncated view, no card state lost | Step 4 (local modal state) |
| AC5 — no message section when absent | Step 4 (graceful empty state) |

---

## Assumptions & Dependencies

- ARC-1301 merged: `runLane()` in `laneRunner.ts` is the authoritative checkpoint creation point.
- ARC-1302 merged: `ResumeCard` component exists and receives a full `CheckpointQueueEntry` prop.
- ARC-1305 merged: Resume card show/hide lifecycle is in place.
- The 200-character truncation threshold is a compile-time UI constant (`MESSAGE_TRUNCATE_LENGTH = 200`), not configurable at runtime.
- Modal must be keyboard-dismissible (Escape key) for basic accessibility.
- `lastAgentMessage` is populated at checkpoint-hit time from the last `TextEvent` seen by the lane runner — not read lazily from subsequent output.
- No existing modal/dialog component exists in the UI package; a new `AgentMessageModal` component must be created.

---

## Implementation Steps

### Step 1: Extend `CheckpointQueueEntry` with `lastAgentMessage`

**Files:**
- `packages/server/src/state/types.ts`
- `packages/ui/src/types/runbook.ts`

**Action:** Add `lastAgentMessage?: string` to the `CheckpointQueueEntry` interface in both locations. The field is optional so existing checkpoint entries without it remain valid.

**Verification:** `tsc --noEmit` completes with no errors in both packages.

---

### Step 2: Capture last `TextEvent` message in `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**Action:** Inside `runLane()`, declare a `let lastAgentMessage: string | undefined` variable before the subprocess event loop. On each `TextEvent` received from the subprocess event stream, update `lastAgentMessage = event.part.text`. When constructing the `CheckpointQueueEntry` object (both for sub-step gate checkpoints and top-level checkpoint steps), populate `lastAgentMessage` from this variable. Leave it `undefined` if no `TextEvent` has been received.

**Verification:** After running a lane that hits a checkpoint, the written sidecar file contains a `lastAgentMessage` field on the relevant `CheckpointQueueEntry` entry.

---

### Step 3: Create `AgentMessageModal` component

**File:** `packages/ui/src/components/AgentMessageModal.tsx`

**Action:** Create a new React component `AgentMessageModal` with props `{ message: string; onClose: () => void }`. It renders a full-screen overlay backdrop and a centered dialog box containing the full `message` text (plain text, no markdown rendering). A close button calls `onClose`. A `useEffect` attaches a `keydown` listener that calls `onClose` on Escape; the listener is cleaned up on unmount. No external modal library is used.

**Verification:** Component renders with message text, Escape key fires `onClose`, close button fires `onClose`, overlay is visible over page content.

---

### Step 4: Extend `ResumeCard` to display `lastAgentMessage`

**File:** `packages/ui/src/components/ResumeCard.tsx`

**Action:**
- Add `MESSAGE_TRUNCATE_LENGTH = 200` as a module-level constant.
- Import and conditionally render `AgentMessageModal`.
- Add `showModal` local state (`useState<boolean>(false)`).
- In the render output, below existing card content, add a message section only when `entry.lastAgentMessage` is non-empty:
  - If `entry.lastAgentMessage.length <= MESSAGE_TRUNCATE_LENGTH`: render the full text inline.
  - If `entry.lastAgentMessage.length > MESSAGE_TRUNCATE_LENGTH`: render the first 200 characters followed by a "Show more…" button. Clicking the button sets `showModal(true)`.
- When `showModal` is true, render `AgentMessageModal` with the full message text and `onClose={() => setShowModal(false)}`.
- When `entry.lastAgentMessage` is absent or empty: render nothing (no empty box, no placeholder).

**Verification:** All five AC scenarios are exercisable in the browser against a running dev server. Card state (PR link, step ID, level emoji) is unaffected when modal opens and closes.

---

### Step 5: Write and extend tests

**Files:**
- `packages/ui/src/components/ResumeCard.test.tsx` (extend existing)
- `packages/ui/src/components/AgentMessageModal.test.tsx` (new file)

**Action:** Follow the vitest + `@testing-library/react` + `userEvent` pattern already established in `ResumeCard.test.tsx`.

New `ResumeCard` test scenarios:
- Short message (≤ 200 chars): message text appears inline, no "Show more…" button present.
- Long message (> 200 chars): truncated text and "Show more…" button present, full text not visible initially.
- Clicking "Show more…": `AgentMessageModal` appears with full message text.
- Absent `lastAgentMessage`: no message section in the rendered output.

New `AgentMessageModal` test scenarios:
- Component renders full message text.
- Clicking close button fires `onClose`.
- Pressing Escape fires `onClose`.

**Verification:** `npm test --workspace=packages/ui` passes with all new cases green; no existing tests regress.

---

## Testing & Validation

Run the following to verify end-to-end correctness:

1. `tsc --noEmit` in both `packages/server` and `packages/ui` — no type errors.
2. `npm test --workspace=packages/ui` — all ResumeCard and AgentMessageModal tests green.
3. Manual smoke test against a running dev server (`npm run dev`): trigger a checkpoint, observe Resume card showing the last agent message, verify truncation and modal behavior.

---

## Risks & Open Questions

- **No existing modal precedent:** The UI has no reusable Modal/Dialog component. The inline `AgentMessageModal` is intentionally minimal — if a shared dialog is needed later, it can be extracted. No scope concern for this story.
- **TextEvent timing:** If the agent emits no `TextEvent` before hitting a checkpoint (e.g., checkpoint is the very first step), `lastAgentMessage` will be `undefined`, which is the correct graceful-empty behavior per AC5.
- **Concurrent lane writes:** `writeSidecar()` is already serialized (established in ARC-1294); adding one more string field to `CheckpointQueueEntry` does not introduce new concurrency concerns.
