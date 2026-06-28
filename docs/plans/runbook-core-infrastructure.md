# Runbook: Core Infrastructure — ARC-1284

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Users and agents can start the runbook executor tool locally, have its runbooks parsed into a structured step graph, persist session state across restarts, and drive agent steps through the opencode CLI — providing the complete runtime foundation all other lanes build on.
**Epic:** ARC-1284
**Lane:** core-infrastructure
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** none

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

## Wave 1 — Core Runtime Foundation

> Gate: none — start immediately

### 🔴 HIGH — ARC-1285: Bootstrap web server

- [x] **ARC-1285** — Bootstrap web server and serve local UI
  - [x] 🔒 Claimed: build / 2026-06-27 (session start)
  - [x] Read `issues/core-infrastructure/ARC-1285-bootstrap-web-server.md`
  - [x] Write `implementation_plans/core-infrastructure/ARC-1285-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1285-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(core-infrastructure): bootstrap web server and local UI serving`

### 🔴 HIGH — ARC-1286: Parse runbook markdown

- [x] **ARC-1286** — Parse runbook markdown into typed step graph
  - [x] 🔒 Claimed: build / 2026-06-28 (session start)
  - [x] Read `issues/core-infrastructure/ARC-1286-parse-runbook-markdown.md`
  - [x] Write `implementation_plans/core-infrastructure/ARC-1286-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1286-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(core-infrastructure): parse runbook markdown into typed step graph`

### 🟡 Batch — State & subprocess integration (ARC-1287, ARC-1288)

> Note: ARC-1288 is 🔴 HIGH — individual plan checkpoint required before execution.

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1287, ARC-1288
- [ ] Write all implementation plans:
  - [ ] `implementation_plans/core-infrastructure/ARC-1287-implementation-plan.md`
  - [ ] `implementation_plans/core-infrastructure/ARC-1288-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT (ARC-1287)
- [ ] Execute ARC-1287
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1287-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(core-infrastructure): implement sidecar state file`
- [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT (ARC-1288)
- [ ] Execute ARC-1288
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1288-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(core-infrastructure): integrate opencode CLI subprocess with JSON event stream`
- [ ] ☑ batch complete

### Wave 1 gate
- [ ] All Wave 1 stories checked off (ARC-1285, ARC-1286, ARC-1287, ARC-1288)
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify dependent lanes:
  - session-setup (unblocked by ARC-1286)
  - session-history (unblocked by ARC-1287)
  - parallel-lane-execution (unblocked by ARC-1288)
  - failure-handling (unblocked by ARC-1288)

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 3 (ARC-1285, ARC-1286, ARC-1288) | 1 (ARC-1287) | 1 | 5 |
| **Total** | **3** | **1** | **1** | **5** |
