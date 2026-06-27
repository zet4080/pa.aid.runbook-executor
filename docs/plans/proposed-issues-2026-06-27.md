# Proposed Issues — Runbook Executor
**Date:** 2026-06-27
**Project:** ARC
**Status:** Awaiting Gate 2 approval

---

## ARC: Core Infrastructure / story-1: Bootstrap web server and serve local UI

**Summary:** Bootstrap web server and serve local UI
**Description:** The tool runs as a local web server accessible at localhost. It serves the browser UI and provides the API backend for all frontend interactions. Runs on a fixed or configurable local port; no external network exposure.
**Acceptance Criteria:**
1. Given the tool is started from the command line, when the user opens `http://localhost:{port}` in a browser, then the UI loads without errors.
2. Given the server is running, when the user navigates to the root URL, then the runbook dashboard is rendered.
**In Scope:**
- Local web server startup, static UI serving, API backend skeleton
**Out of Scope:**
- Authentication, multi-user access, cloud/hosted deployment

---

## ARC: Core Infrastructure / story-2: Parse runbook markdown into typed step graph

**Summary:** Parse runbook markdown into typed step graph
**Description:** The tool reads `docs/plans/runbook-*.md` files and converts them into an in-memory step graph. Each step is typed as `agent`, `manual`, or `checkpoint` based on explicit markers (🔴🟡🟢) and structured fields in the markdown. Natural-language inference is not used.
**Acceptance Criteria:**
1. Given a runbook markdown file with 🔴🟡🟢 markers and checkbox steps, when the parser runs, then each step is represented with: type (agent/manual/checkpoint), checkbox state (complete/incomplete), and checkpoint level (red/yellow/green) where applicable.
2. Given a step with no explicit type field, when the parser encounters it, then it is classified as `manual` by default.
**In Scope:**
- Markdown parsing, step typing, checkpoint detection, checkbox state extraction
**Out of Scope:**
- Modifying runbook structure, authoring runbooks, YAML/JSON runbook formats

---

## ARC: Core Infrastructure / story-3: Implement sidecar state file

**Summary:** Implement sidecar state file
**Description:** The tool maintains a sidecar JSON/YAML state file alongside the planning repo to persist session state across restarts. The runbook markdown files are always the ground truth — sidecar is derived from and reconciled against markdown on startup.
**Acceptance Criteria:**
1. Given an active session, when the tool is restarted, then session state (selected runbooks, step progress, queue) is restored from the sidecar file.
2. Given sidecar and markdown disagree on checkbox state, when the tool starts, then the markdown state takes precedence.
**In Scope:**
- Sidecar file read/write, startup reconciliation with markdown ground truth
**Out of Scope:**
- Multi-session state, cloud sync, external state stores

---

## ARC: Core Infrastructure / story-4: Integrate opencode CLI subprocess

**Summary:** Integrate opencode CLI subprocess with JSON event stream
**Description:** The tool invokes `opencode run --format json` as a subprocess for each agent step. It reads the JSON event stream from stdout, detects completion and failure via process exit code and event fields, and surfaces the result to the execution engine. Each invocation starts a fresh session.
**Acceptance Criteria:**
1. Given an agent step, when the tool invokes `opencode run`, then stdout is consumed as a JSON event stream and rendered live in the UI.
2. Given the subprocess exits with a non-zero exit code or emits an error event, then the step is marked failed and the failure handling flow is triggered.
3. Given a completed subprocess, when the tool checks its exit code, then a zero exit code marks the step succeeded.
**In Scope:**
- Subprocess spawning, JSON event stream consumption, exit code detection, fresh-session-per-call
**Out of Scope:**
- opencode API/serve mode, persistent agent sessions, credential management (inherited from environment)

---

## ARC: Session Setup & Runbook Selection / story-1: Display runbook dashboard at startup

**Summary:** Display runbook dashboard at startup
**Description:** On startup, the tool scans the planning repo and presents all discovered runbooks in a collapsed summary view. Each runbook shows name, status (idle/running/blocked/complete), current wave, and overall progress. The dashboard loads within 3 seconds.
**Acceptance Criteria:**
1. Given a planning repo with runbook files, when the tool starts, then all runbooks are listed with name, status, current wave, and % progress within 3 seconds.
2. Given runbooks in various states, when displayed, then each runbook shows the correct status badge (idle/running/blocked/complete).
**In Scope:**
- Repo scanning, runbook status derivation, collapsed dashboard view, 3-second load target
**Out of Scope:**
- Editing runbooks, creating runbooks, detailed step view (that is expand-on-demand)

---

## ARC: Session Setup & Runbook Selection / story-2: Expand runbook detail on demand

**Summary:** Expand runbook detail on demand
**Description:** The supervisor can expand any runbook card to see the full progress tree: all waves, all stories, all steps with checkbox state, next pending step highlighted. Collapsing returns to the summary view.
**Acceptance Criteria:**
1. Given a collapsed runbook card, when the supervisor clicks to expand, then all waves and steps are shown with current checkbox state.
2. Given an expanded runbook, when the supervisor collapses it, then the card returns to summary view without losing state.
**In Scope:**
- Expand/collapse UI, full step tree rendering, next-step highlighting
**Out of Scope:**
- Editing step content, inline step execution from dashboard

---

## ARC: Session Setup & Runbook Selection / story-3: Select runbooks for session

**Summary:** Select runbooks to include in session
**Description:** The supervisor selects which runbooks to automate in the current session. Unselected runbooks remain visible but are not executed. Selection can be changed before the session starts.
**Acceptance Criteria:**
1. Given the dashboard, when the supervisor selects a subset of runbooks and starts the session, then only selected runbooks are executed.
2. Given a runbook is deselected before session start, when the session starts, then that runbook remains idle.
**In Scope:**
- Multi-select UI, session-scoped runbook list
**Out of Scope:**
- Adding/removing runbooks from the repo, mid-session runbook selection changes

---

## ARC: Session Setup & Runbook Selection / story-4: Configure session parameters before start

**Summary:** Configure session parameters before start
**Description:** Before starting execution, the supervisor configures: maximum concurrent lanes, retry count per failed step, agent escalation sequence, and initial queue policy. Defaults are provided for all parameters.
**Acceptance Criteria:**
1. Given the session config panel, when the supervisor sets concurrency to N, then at most N lanes run simultaneously.
2. Given default values pre-populated, when the supervisor starts without changing anything, then the session runs with sensible defaults.
3. Given an escalation sequence configured (e.g. build → senior-coder), when a step fails and retries, then each retry uses the next agent in the sequence.
**In Scope:**
- Concurrency, retry count, escalation sequence, queue policy selection
**Out of Scope:**
- Per-runbook configuration, runtime concurrency changes (that is queue policy change mid-session, covered in Epic 4)

---

## ARC: Parallel Lane Execution / story-1: Execute runbook steps sequentially within a lane

**Summary:** Execute runbook steps sequentially within a lane
**Description:** The tool executes steps within a single runbook lane one at a time, in order. Each step completes (or checkpoints/fails) before the next begins. Step type determines execution path: `agent` steps invoke opencode CLI, `manual` steps pause for supervisor, `checkpoint` steps trigger the checkpoint flow.
**Acceptance Criteria:**
1. Given a runbook lane with N steps, when execution starts, then steps execute in order and step N+1 does not start until step N completes.
2. Given a `manual` step, when the executor reaches it, then the lane pauses and the supervisor is notified.
3. Given an `agent` step, when the executor reaches it, then `opencode run` is invoked and output streams to the UI.
**In Scope:**
- Sequential step execution, step type routing (agent/manual/checkpoint), lane state transitions
**Out of Scope:**
- Parallel execution within a lane, skipping steps

---

## ARC: Parallel Lane Execution / story-2: Run multiple lanes concurrently up to concurrency limit

**Summary:** Run multiple lanes concurrently up to concurrency limit
**Description:** The tool runs selected runbook lanes in parallel, up to the configured maximum concurrency. When a lane finishes or blocks, a queued lane (if any) starts. Concurrency limit is set at session start.
**Acceptance Criteria:**
1. Given concurrency set to N and M>N selected runbooks, when the session starts, then exactly N lanes run simultaneously and remaining lanes queue.
2. Given a running lane completes, when capacity frees, then the next queued lane starts automatically.
**In Scope:**
- Concurrency management, lane queuing, capacity-triggered start
**Out of Scope:**
- Mid-session concurrency limit changes, priority-based lane ordering (that is queue policy, Epic 4)

---

## ARC: Parallel Lane Execution / story-3: Stream live agent output per lane in the UI

**Summary:** Stream live agent output per lane in the UI
**Description:** As an agent step executes, its stdout JSON event stream is rendered live in the UI in the lane's output panel. The supervisor can follow execution in real time without polling or refreshing.
**Acceptance Criteria:**
1. Given an executing agent step, when the opencode subprocess emits events, then each event appears in the lane output panel within 1 second.
2. Given multiple lanes running, when each emits output, then output is correctly isolated per lane panel.
**In Scope:**
- Live streaming UI, per-lane output isolation, JSON event rendering
**Out of Scope:**
- Log export, output search, output filtering

---

## ARC: Parallel Lane Execution / story-4: Update runbook checkbox state in real time

**Summary:** Update runbook checkbox state in real time
**Description:** As steps complete, the tool updates the corresponding checkboxes in the runbook markdown file immediately. The markdown file is always the ground truth — checkbox state reflects actual execution state at all times.
**Acceptance Criteria:**
1. Given a step completes successfully, when the tool updates state, then the corresponding `- [ ]` checkbox in the markdown file is updated to `- [x]` within 1 second.
2. Given a checkpoint is hit, when the lane pauses, then the markdown file reflects the paused state (checkbox not yet checked).
3. Given the tool crashes mid-execution, when restarted, then markdown checkbox state accurately reflects what completed before the crash.
**In Scope:**
- Real-time markdown checkbox updates, crash-safe state writing
**Out of Scope:**
- Modifying runbook structure beyond checkboxes, adding new steps

---

## ARC: Checkpoint Management / story-1: Detect checkpoint markers and pause lane execution

**Summary:** Detect checkpoint markers and pause lane execution
**Description:** When the executor reaches a step with a 🔴, 🟡, or 🟢 marker, it immediately pauses that lane and transitions it to `blocked` state. All three marker types trigger a pause regardless of content.
**Acceptance Criteria:**
1. Given a step with a 🔴, 🟡, or 🟢 marker, when the executor reaches it, then the lane pauses and transitions to `blocked` status.
2. Given a lane in `blocked` state, when the supervisor views the dashboard, then the lane shows `blocked` with the checkpoint type indicated.
**In Scope:**
- Marker detection (all three types), lane pause, blocked status transition
**Out of Scope:**
- Auto-resolving checkpoints, inferring checkpoint level from content

---

## ARC: Checkpoint Management / story-2: Deliver browser notification within 5 seconds of checkpoint hit

**Summary:** Deliver browser notification within 5 seconds of checkpoint hit
**Description:** When a lane hits a checkpoint, the tool sends a browser notification to alert the supervisor. The notification identifies the runbook, lane, and checkpoint type. Notification is delivered within 5 seconds of the checkpoint being reached.
**Acceptance Criteria:**
1. Given a lane hits a checkpoint, when 5 seconds elapse, then a browser notification has been delivered with runbook name, lane, and checkpoint type.
2. Given the browser tab is not in focus, when a checkpoint is hit, then the notification still appears via the browser notification API.
**In Scope:**
- Browser Notification API integration, checkpoint metadata in notification
**Out of Scope:**
- Email, Slack, or other notification channels

---

## ARC: Checkpoint Management / story-3: Display checkpoint queue with automatic priority scoring

**Summary:** Display checkpoint queue with automatic priority scoring
**Description:** All pending checkpoints are shown in an ordered queue panel. Each checkpoint is automatically scored by wait time and gate level (🟢 > 🟡 > 🔴). The queue reflects the current scheduling order.
**Acceptance Criteria:**
1. Given multiple pending checkpoints, when displayed, then they are ordered by automatic priority score (gate level + wait time).
2. Given a checkpoint has been waiting longer than another of equal gate level, then it ranks higher in the queue.
**In Scope:**
- Queue panel, automatic priority scoring, gate level weighting
**Out of Scope:**
- Manual reorder (Epic 4), policy-based scheduling (Epic 4)

---

## ARC: Checkpoint Management / story-4: Inline-edit checkpoint artifacts and persist handoff file to disk

**Summary:** Inline-edit checkpoint artifacts and persist handoff file to disk
**Description:** When a checkpoint involves an artifact (e.g. an implementation plan), the supervisor can read and edit it directly in the UI. Edits are persisted immediately to an ephemeral handoff file on disk so they survive a crash. The agent consumes the handoff file when execution resumes.
**Acceptance Criteria:**
1. Given a checkpoint with an associated artifact file, when the supervisor opens the checkpoint, then the artifact content is displayed in an editable panel.
2. Given the supervisor edits the artifact, when any character is typed, then the handoff file on disk is updated within 2 seconds (auto-save).
3. Given the tool restarts after a crash, when the checkpoint is re-opened, then the supervisor's edits are still present from the handoff file.
**In Scope:**
- Artifact display, inline editing, auto-save to handoff file, crash persistence
**Out of Scope:**
- Version history of edits, diff view, collaborative editing

---

## ARC: Checkpoint Management / story-5: Approve or reject checkpoint inline with feedback

**Summary:** Approve or reject checkpoint inline with feedback
**Description:** The supervisor approves or rejects a checkpoint directly in the UI. Approval resumes the lane. Rejection captures free-text feedback and halts the lane — the supervisor must explicitly restart or requeue before execution resumes.
**Acceptance Criteria:**
1. Given a pending checkpoint, when the supervisor clicks Approve, then the lane resumes from the next step.
2. Given a pending checkpoint, when the supervisor clicks Reject and enters feedback text, then the lane halts and the feedback is stored with the checkpoint record.
3. Given a rejected checkpoint, when the supervisor has not explicitly restarted the lane, then the lane remains halted.
**In Scope:**
- Approve/reject UI, free-text feedback capture, halt-on-rejection, explicit restart requirement
**Out of Scope:**
- Auto-requeue on rejection, partial approval

---

## ARC: Queue & Scheduling Policy / story-1: Display ordered checkpoint queue with priority scores

**Summary:** Display ordered checkpoint queue with priority scores
**Description:** The UI shows a persistent queue panel listing all pending checkpoints in priority order. Each entry shows: runbook name, checkpoint type (🔴🟡🟢), wait time, and priority score. The queue updates live as new checkpoints arrive or are resolved.
**Acceptance Criteria:**
1. Given pending checkpoints exist, when the supervisor views the queue panel, then all pending checkpoints are listed with type, runbook, wait time, and priority score.
2. Given a new checkpoint arrives, when added to the queue, then it appears in the correct priority position within 1 second.
**In Scope:**
- Queue panel UI, live updates, priority score display
**Out of Scope:**
- Manual reorder (story 2), policy selection (story 3)

---

## ARC: Queue & Scheduling Policy / story-2: Reorder checkpoint queue manually

**Summary:** Reorder checkpoint queue manually
**Description:** The supervisor can drag-and-drop or use controls to reorder the pending checkpoint queue. The new order takes effect before the next execution slot picks up work — no restart required.
**Acceptance Criteria:**
1. Given an ordered queue, when the supervisor moves a checkpoint to a different position, then the queue reflects the new order immediately.
2. Given the queue has been manually reordered, when the next execution slot frees up, then the tool picks the top item from the reordered queue.
**In Scope:**
- Manual reorder UI, immediate effect on scheduling
**Out of Scope:**
- Persisting manual order across sessions, bulk reorder

---

## ARC: Queue & Scheduling Policy / story-3: Select and switch queue policy mid-session

**Summary:** Select and switch queue policy mid-session
**Description:** The supervisor selects a queue policy at session start: `checkpoint-first`, `lane-first`, `balanced`, or `wave-gate-first`. The policy can be changed at any time during the session and takes effect on the next scheduling decision.
**Acceptance Criteria:**
1. Given policy set to `checkpoint-first`, when capacity frees, then the tool processes a queued checkpoint before advancing any lane.
2. Given policy set to `lane-first`, when capacity frees, then the tool advances an available lane step before processing checkpoints.
3. Given policy changed mid-session, when the next scheduling decision occurs, then the new policy is applied.
**In Scope:**
- Four policy modes, session-start selection, mid-session switching, next-decision effect
**Out of Scope:**
- Custom policy definitions, per-runbook policy overrides

---

## ARC: Failure Handling & Escalation / story-1: Retry failed agent steps up to configurable limit

**Summary:** Retry failed agent steps up to configurable limit
**Description:** When an agent step fails (non-zero exit code or error event), the tool automatically retries up to the configured retry limit. Each retry is logged in the lane output panel. After the limit is exhausted, the failure escalation flow is triggered.
**Acceptance Criteria:**
1. Given a step fails and retry count > 0 remaining, when the tool retries, then a new opencode subprocess is spawned for the same step.
2. Given retry count reaches the configured limit, when the last retry fails, then no further automatic retries occur and escalation is triggered.
**In Scope:**
- Automatic retry, retry counter, retry logging in UI
**Out of Scope:**
- Manual retry triggered by supervisor (that is post-checkpoint restart)

---

## ARC: Failure Handling & Escalation / story-2: Escalate agent level on each retry

**Summary:** Escalate agent level on each retry
**Description:** Each retry of a failed step uses the next agent in the configured escalation sequence (e.g. `build` → `senior-coder`). The escalation sequence is set at session start. If the sequence is exhausted before the retry limit, the last agent in the sequence is reused.
**Acceptance Criteria:**
1. Given escalation sequence `[build, senior-coder]` and a failing step, when the first retry runs, then `senior-coder` is used instead of `build`.
2. Given the escalation sequence is exhausted but retries remain, when subsequent retries run, then the last agent in the sequence is used.
**In Scope:**
- Per-retry agent escalation, sequence exhaustion fallback
**Out of Scope:**
- Dynamic escalation sequence changes mid-session

---

## ARC: Failure Handling & Escalation / story-3: Surface exhausted retries as supervisor checkpoint notification

**Summary:** Surface exhausted retries as supervisor checkpoint notification
**Description:** When all retries are exhausted and the step still fails, the lane pauses and the failure is added to the supervisor checkpoint queue — treated identically to a 🔴 checkpoint. The supervisor receives a browser notification and must explicitly decide how to proceed.
**Acceptance Criteria:**
1. Given all retries exhausted, when the final retry fails, then the lane pauses and a failure checkpoint appears in the queue.
2. Given the failure checkpoint, when the supervisor views it, then the full retry history and error output are visible.
3. Given the failure checkpoint, when a browser notification is sent, then it identifies the runbook, step, and number of retries attempted.
**In Scope:**
- Failure-as-checkpoint, retry history display, browser notification, lane halt
**Out of Scope:**
- Automatic failure recovery, rollback of partial changes

---

## ARC: Session History / story-1: Persist active session state across restarts

**Summary:** Persist active session state across restarts
**Description:** One active session exists per planning repo at a time. Session state (selected runbooks, step progress, queue, configuration) is persisted to the sidecar state file continuously so that restarting the tool restores the session without data loss.
**Acceptance Criteria:**
1. Given an active session, when the tool is restarted, then the session is restored: selected runbooks, step progress, pending checkpoints, and session config are all present.
2. Given two simultaneous starts pointing at the same repo, when the second starts, then it detects an active session and warns the supervisor rather than creating a second session.
**In Scope:**
- Session persistence, restart recovery, single-session-per-repo enforcement
**Out of Scope:**
- Multiple simultaneous sessions, cloud-based session sync

---

## ARC: Session History / story-2: Browse past sessions list

**Summary:** Browse past sessions list
**Description:** The supervisor can view a list of all past sessions for the current planning repo. Each entry shows: session date/time, runbooks included, final status, and duration. Sessions are read-only once closed.
**Acceptance Criteria:**
1. Given past sessions exist, when the supervisor opens the history view, then all past sessions are listed with date, runbooks, status, and duration.
2. Given a past session entry, when the supervisor selects it, then the session detail view opens.
**In Scope:**
- Session list UI, session metadata display, read-only past sessions
**Out of Scope:**
- Deleting sessions, exporting session data, comparing sessions

---

## ARC: Session History / story-3: View full step output history for any past session

**Summary:** View full step output history for any past session
**Description:** Within a past session, the supervisor can browse every step that was executed, its output (full agent event stream), status (completed/failed/checkpoint), and any supervisor feedback captured at checkpoints.
**Acceptance Criteria:**
1. Given a past session, when the supervisor opens it, then all executed steps are listed with status and timestamp.
2. Given a step in a past session, when the supervisor selects it, then the full agent output (event stream) is displayed.
3. Given a checkpoint in a past session, when viewed, then supervisor feedback (if any) is shown alongside the checkpoint artifact state at time of approval/rejection.
**In Scope:**
- Per-step output history, checkpoint feedback history, artifact state at checkpoint time
**Out of Scope:**
- Re-running steps from history, editing past session data
