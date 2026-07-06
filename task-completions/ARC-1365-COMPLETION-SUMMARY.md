# ARC-1365 - Display agent last message on checkpoint Resume card

## Traceability
- **Key:** ARC-1365
- **Title:** Display agent last message on checkpoint Resume card
- **Branch:** ARC-1365
- **Commit:** 5af49a3
- **Lane:** checkpoint-management

## Acceptance Criteria

**AC1.** Given lane hits checkpoint + Resume card displayed, When last message ≤ 200 chars → full message shown inline on Resume card.

**AC2.** Given lane hits checkpoint + Resume card displayed, When last message > 200 chars → first 200 chars shown with 'Show more…' affordance.

**AC3.** Given supervisor clicks 'Show more…', When modal opens → modal dialog displays complete agent message text.

**AC4.** Given supervisor closes modal dialog, When modal dismissed → Resume card returns to truncated view, no card state lost.

**AC5.** Given no last message in checkpoint data, When Resume card displayed → no message section rendered (graceful empty state — no placeholder, no empty box).

## Evidence

### AC1 — Short message inline
- `ResumeCard.tsx` renders `entry.lastAgentMessage` inline when `length <= MESSAGE_TRUNCATE_LENGTH` (200)
- `data-testid='agent-message-inline'` renders the full text
- Test: `ResumeCard.test.tsx` — 'renders short message (≤ 200 chars) inline with no Show more… button' — PASSED
- Test: 'renders exactly 200 chars inline without Show more… when message is exactly 200 chars' — PASSED

### AC2 — Long message truncated + Show more…
- `ResumeCard.tsx` renders `entry.lastAgentMessage.slice(0, 200)` + 'Show more…' button when `length > 200`
- Test: 'renders truncated message with Show more… button when message exceeds 200 chars' — PASSED

### AC3 — Modal shows full message on click
- `AgentMessageModal.tsx` created: renders full message in `data-testid='agent-message-modal-text'`
- `ResumeCard.tsx` opens modal via local `showModal` state when 'Show more…' button clicked
- Test: 'opens AgentMessageModal with full message when Show more… is clicked' — PASSED
- `AgentMessageModal.test.tsx` — 'renders the full message text' — PASSED

### AC4 — Closing modal restores truncated view
- Modal state is `useState<boolean>(false)` in `ResumeCard` — closing sets it back to false
- Card state (PR link, stepId, level emoji, Resume button) is unaffected
- Test: 'closes AgentMessageModal and returns to truncated view when modal is dismissed' — PASSED (Escape key dismissal; card still rendered with Resume button and Show more… intact)

### AC5 — Graceful empty state
- `ResumeCard.tsx` only renders message section when `entry.lastAgentMessage` is truthy
- Test: 'does NOT render message section when lastAgentMessage is absent' — PASSED
- Test: 'does NOT render message section when lastAgentMessage is empty string' — PASSED

## Test results
- **Server:** 293 tests, 19 test files — all PASSED
- **UI:** 192 tests, 19 test files — all PASSED
- **New tests:** 7 in `AgentMessageModal.test.tsx` + 8 new cases in `ResumeCard.test.tsx`
- **tsc --noEmit:** clean in both packages

## Files modified
- `packages/server/src/state/types.ts` — added `lastAgentMessage?: string` to `CheckpointQueueEntry`
- `packages/server/src/runner/laneRunner.ts` — imported `OpenCodeEvent`, `TextEvent`; added `extractLastAgentMessage()`; populated field at both checkpoint creation sites
- `packages/ui/src/types/runbook.ts` — added `lastAgentMessage?: string` to `CheckpointQueueEntry`
- `packages/ui/src/components/AgentMessageModal.tsx` — new file (modal component)
- `packages/ui/src/components/AgentMessageModal.test.tsx` — new file (7 tests)
- `packages/ui/src/components/ResumeCard.tsx` — added modal state, message section, graceful empty state
- `packages/ui/src/components/ResumeCard.test.tsx` — extended with 8 new test cases
