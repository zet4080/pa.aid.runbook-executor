---
# Requirements: Runbook Executor

**Date:** 2026-06-27
**Status:** Draft
**Author:** agent

---

## 1. Context & Motivation

The pa.aid agent-driven development process uses planning repos with markdown runbooks to coordinate multi-lane feature delivery. Runbooks contain structured steps executed by AI agents, gated by supervisor checkpoints. Today execution is manual — a human orchestrates each step, invokes agents one at a time, and tracks state by editing markdown files. A web-based automation tool would eliminate this manual orchestration, letting one supervisor oversee multiple parallel lanes while retaining full control at checkpoints.

---

## 2. Problem Statement

Executing runbooks today requires the supervisor to manually invoke agents step by step, switch between lanes, track checkpoint state, and maintain runbook markdown by hand. This is slow, error-prone, and does not scale to multiple parallel lanes. There is no mechanism to queue supervisor attention or prioritize review work.

---

## 3. Target Users & Personas

| Persona | Role | Context |
|---------|------|---------|
| Supervisor | Developer / tech lead | Running automation sessions locally, reviewing checkpoint PRs in Bitbucket, and resuming lanes via [Resume] cards in the session UI |

---

## 4. Goals & Non-Goals

**Goals:**
- Automate multi-lane runbook execution with minimal supervisor intervention
- Surface checkpoints promptly and let supervisor handle them inline
- Give supervisor full control over execution order, queue priority, and scheduling policy
- Keep runbook markdown files as the source of truth at all times

**Non-Goals:**
- Multi-user / team collaboration features
- Hosted / cloud deployment
- Managing planning repo structure or authoring runbooks
- Replacing opencode agents — tool orchestrates them, does not reimplement them

---

## 5. Requirements

### Session Setup
- Tool accepts a path to a planning repo at startup
- Tool scans `docs/plans/runbook-*.md` and presents all runbooks with status, current wave, and progress
- Supervisor selects which runbooks to include in the session
- Supervisor configures: max concurrent lanes, retry count per step, escalation agent sequence, queue policy
- Tool maintains a sidecar state file for session continuity; runbook markdown is always ground truth

### Execution
- Tool executes runbook steps sequentially within each lane
- Multiple lanes run concurrently up to the configured concurrency limit
  - Agent steps invoke `opencode` CLI subprocesses using global agent/skill config (`~/.config/opencode/`)
  - Runbook steps must be explicitly typed (`agent` / `manual`) — tool does not infer step type from natural language
  - Agent output streams live in the UI
  - Execution auto-continues after non-checkpoint steps complete successfully

### Checkpoints
- 🔴, 🟡, and 🟢 markers in runbook markdown trigger a checkpoint pause in that lane
- On checkpoint hit, executor creates a git branch, pushes relevant artifacts (planning repo artifacts or code artifacts depending on checkpoint type), and opens a Bitbucket Pull Request via the Bitbucket MCP endpoint
- A persistent **[Resume]** card is displayed in the session UI within 5 seconds of the checkpoint hit, showing: checkpoint title, PR link, checkpoint type, and runbook/lane context
- The suspended lane does not block other lanes — other lanes continue executing (non-blocking)
- Each lane has its own checkpoints and PRs; per-lane isolation is maintained throughout
- Checkpoint added to supervisor queue with automatic priority score (wait time + gate level)
- Supervisor reviews the PR in Bitbucket using inline comments to indicate required changes; PR approval/rejection state is decorative — the executor does not act on it
- Supervisor clicks **[Resume]** in the session UI to signal "I have finished reviewing"
- On [Resume] click, executor reads all unresolved comment threads on the PR via Bitbucket MCP
  - If unresolved comments exist: executor addresses each comment as a required action item, commits the result, replies on the PR thread ("Addressed in commit {sha}"), resolves the thread, then re-sends the [Resume] card with note "Agent addressed N comments — [view changes]" and waits for [Resume] again
  - If no unresolved comments exist: lane proceeds to next step
- This loop repeats until supervisor clicks [Resume] with zero unresolved threads
- All unresolved inline comments on the PR — whether on planning artifacts or code — are treated as required action items

### Queue Management
- Pending checkpoints visible as an ordered queue
- Supervisor can reorder queue manually at any time; takes effect before next execution slot
- Queue policy governs scheduling when capacity frees: `checkpoint-first`, `lane-first`, `balanced`, `wave-gate-first`
- Policy selectable at session start, switchable mid-session

### Failure Handling
- Failed agent steps retry up to a configurable limit
- Each retry escalates agent level (e.g. `build` → `senior-coder`)
- After max retries exhausted, lane pauses and supervisor is notified like a checkpoint

### History & State
- One active session per planning repo at a time
- Past sessions browsable with full step output history
- Runbook checkbox state in markdown updated in real time to reflect execution state

---

## 6. Constraints

- Runs locally only (`localhost`) — no hosted deployment
- Uses global opencode agent/skill config (`~/.config/opencode/`) — no per-repo agent config override
- Credentials inherited from `setup.env` environment (Jira, Confluence, Bitbucket, AWS, LiteLLM)
- Agent invocation via `opencode` CLI subprocess — fresh session per agent call
- Runbook markdown format: `docs/plans/runbook-*.md` with 🔴🟡🟢 checkpoint markers and hierarchical checkboxes
- Must not modify runbook structure — only update checkbox state

---

## 7. Success Criteria

1. Supervisor can point tool at planning repo and see all runbooks with status within 3 seconds of startup
2. Supervisor can select runbooks and start parallel execution in one action
3. Every checkpoint pauses the lane, creates a Bitbucket PR, and delivers a persistent [Resume] card in the session UI within 5 seconds
4. Supervisor can review checkpoint artifacts as PR inline comments in Bitbucket and resume the lane by clicking [Resume] in the session UI — no separate tool required
5. Lane proceeds only when supervisor clicks [Resume] with zero unresolved PR comment threads — executor re-notifies after addressing comments rather than auto-proceeding
6. When all lanes are blocked, tool stops consuming resources and waits
7. Runbook checkbox state in markdown accurately reflects execution state at all times
8. Failed steps retry up to configurable limit with escalating agent level before pausing for supervisor
9. Supervisor can reorder pending checkpoint queue at any time — takes effect before next execution slot
10. Past sessions browsable with full step output history
11. When capacity frees, tool auto-picks highest-priority queued checkpoint; manual reorder immediately reflected
12. Queue policy selectable at session start and switchable mid-session; change takes effect on next scheduling decision

---

## 8. Open Questions

| # | Question | Owner | Urgency | Resolution |
|---|----------|-------|---------|------------|
| 1 | Does `opencode` CLI support headless/non-interactive mode? | Architecture | 🔴 Blocker | **Resolved.** `opencode run [message]` is documented for scripting. `opencode serve` provides headless API. Known GitHub issues about permission prompts in CI — treat as requiring integration testing before relying on unattended execution. |
| 2 | What is the output format of `opencode` CLI — how to detect completion vs failure? | Architecture | 🔴 Blocker | **Resolved.** Use `opencode run --format json` — outputs a JSON event stream. Detect completion/failure via process exit code + JSON event/error fields. Do not rely on raw text parsing. |
| 3 | How does the tool detect that a runbook step requires agent invocation vs is a manual/human step? | Architecture | 🟡 Important | **Resolved.** Steps must be explicitly typed in runbook markdown (e.g. `type: agent` vs `type: manual`) or inferred from structured fields (`agent:`, `prompt:`, `command:`). Natural-language inference alone is insufficient. Runbook format must be extended to support this. |
| 4 | How does the supervisor signal approval at a checkpoint? | Product | 🟡 Important | **Resolved.** PR approve/reject state is decorative. The executor reads unresolved comment threads on [Resume] click. If unresolved comments exist, executor addresses all of them and re-notifies. Lane proceeds only when [Resume] is clicked with zero unresolved threads. |

---

## 9. Out of Scope

- Multi-user collaboration or shared sessions
- Cloud/hosted deployment
- Runbook authoring or planning repo scaffolding
- Replacing or reimplementing opencode agents
- Notification channels beyond browser notifications (email, Slack, etc.)
- Multiple planning repos in a single session
---
