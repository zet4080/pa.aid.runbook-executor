```markdown
---
| Field | Value |
|-------|-------|
| Key | ARC-1365 |
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Epic | ARC-1295 — ARC: Checkpoint Management |
| Created | 2026-07-06 |
---

# ARC-1365: Display agent last message on checkpoint Resume card

## Goal

When a lane pauses at a checkpoint, the supervisor needs immediate context about what the agent was doing. This story captures the agent's last message at checkpoint-hit time and surfaces it on the Resume card — inline for short messages, and via a modal dialog for longer ones.

## Acceptance Criteria

**1.**
- **Given** a lane hits a checkpoint and the Resume card is displayed
- **When** the agent's last message is ≤ 200 characters
- **Then** the full message text is shown inline on the Resume card

**2.**
- **Given** a lane hits a checkpoint and the Resume card is displayed
- **When** the agent's last message exceeds 200 characters
- **Then** the first 200 characters are displayed with a "Show more…" affordance

**3.**
- **Given** the supervisor clicks "Show more…" on the Resume card
- **When** the modal opens
- **Then** a modal dialog displays the complete agent message text

**4.**
- **Given** the supervisor closes the modal dialog
- **When** the modal is dismissed
- **Then** the Resume card returns to its truncated inline view with no loss of card state

**5.**
- **Given** no last message is available in the checkpoint data at checkpoint-hit time
- **When** the Resume card is displayed
- **Then** no message section is rendered (graceful empty state — no placeholder text, no empty box)

## In Scope

- Extend the checkpoint data structure (sidecar state / checkpoint event) to include a `lastAgentMessage` field, populated at checkpoint-hit time
- Display `lastAgentMessage` inline on the Resume card when the message is ≤ 200 characters
- Truncate messages longer than 200 characters on the Resume card and show a "Show more…" affordance
- Modal dialog that opens on "Show more…" click and displays the full message text
- Graceful empty state: no message section rendered when `lastAgentMessage` is absent or empty

## Out of Scope

- Storing full agent conversation history beyond the single last message
- Displaying message history or a message timeline
- Message search, filtering, or copy-to-clipboard functionality
- Email, Slack, or push delivery of the agent message text
- Modifying how the agent produces or formats its output messages
- Markdown rendering of the message (plain text only)

## Constraints

- The 200-character truncation threshold is a UI constant; it must not be user-configurable in this story
- Modal must be keyboard-dismissible (Escape key) to meet basic accessibility requirements
- `lastAgentMessage` must be captured at checkpoint-hit time — not read lazily after the fact

## Dependencies

- ARC-1301 — checkpoint detection and lane pause (provides checkpoint event data structure)
- ARC-1302 — Resume card component (the UI component being extended)
- ARC-1305 — Resume card show/hide lifecycle

## Completion Tracking

| Field | Value |
|-------|-------|
| Implementation Plan | `implementation_plans/checkpoint-management/ARC-1365-implementation-plan.md` (TBD) |
| Completion Summary | `task-completions/ARC-1365-COMPLETION-SUMMARY.md` (TBD) |
```