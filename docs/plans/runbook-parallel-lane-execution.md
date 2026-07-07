# Runbook: Parallel Lane Execution — ARC-1290

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** executor run runbook lanes concurrently, streaming live agent output per lane while updating runbook checkbox state in real time
**Epic:** ARC-1290
**Lane:** parallel-lane-execution
**Wave(s):** 2
**Repos:** pa.aid.runbook-executor, pa.aid.conductor.ts
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1288 (core-infrastructure — opencode CLI subprocess)

**Out-of-checklist stories (in repo, not in active scope):** On-hold register tracks original Wave 1 stories pre-execution; all Wave 1 stories are now complete.

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

## HIGH story workflow
1. Read issue file
2. Write implementation plan → `implementation_plans/{lane}/{KEY}-implementation-plan.md`
3. 🔴 CHECKPOINT — human reviews plan before execution
4. Execute plan
5. Lint / tests pass
6. Write completion summary → `task-completions/{KEY}-COMPLETION-SUMMARY.md`
7. Commit + ☑

## MEDIUM/LOW batch workflow
1. Read all issue files in batch
2. Write all implementation plans
3. 🟡 CHECKPOINT — human reviews all plans
4. Execute all plans in sequence
5. Lint / tests pass for all
6. Write all completion summaries
7. Commit all + ☑ all

---

## Wave 1 — Foundation & Concurrency

> Gate: all 4 stories closed; unblock checkpoint-management lane (ARC-1291 → ARC-1301)

### 🔴 ARC-1291 — Execute runbook steps sequentially within a lane

- [x] **ARC-1291** — Execute runbook steps sequentially within a lane
  - [x] 🔒 Claimed: ✅
  - [x] Read `issues/parallel-lane-execution/ARC-1291-sequential-step-execution.md`
  - [x] Write `implementation_plans/parallel-lane-execution/ARC-1291-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1291-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(parallel-lane-execution): sequential step execution within lane`
  - [ ] Claimed: parallel-lane-execution / 2026-06-30 00:00

### 🔴 ARC-1292 — Run multiple lanes concurrently up to concurrency limit

- [x] **ARC-1292** — Run multiple lanes concurrently up to concurrency limit
  - [x] 🔒 Claimed: ✅
  - [x] Read `issues/parallel-lane-execution/ARC-1292-concurrent-lanes.md`
  - [x] Write `implementation_plans/parallel-lane-execution/ARC-1292-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1292-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(parallel-lane-execution): run multiple lanes concurrently up to concurrency limit`
  - [ ] Claimed: parallel-lane-execution / 2026-06-30 10:00

### 🔴 ARC-1293 — Stream live agent output per lane in the UI

- [x] **ARC-1293** — Stream live agent output per lane in the UI
  - [x] 🔒 Claimed: ✅
  - [x] Read `issues/parallel-lane-execution/ARC-1293-stream-agent-output.md`
  - [x] Write `implementation_plans/parallel-lane-execution/ARC-1293-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1293-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(parallel-lane-execution): stream live agent output per lane in UI`
  - [ ] Claimed: build-agent / 2026-06-30

### 🟡 Batch — Batch 1-A

- [x] 🔒 Claimed: ✅
- [x] Read all issues: ARC-1294
  - Files: `issues/parallel-lane-execution/ARC-1294-realtime-checkbox-updates.md`
- [x] Write all plans:
  - [x] `implementation_plans/parallel-lane-execution/ARC-1294-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1294 — Update runbook checkbox state in real time
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1294-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(parallel-lane-execution): real-time runbook checkbox state updates`
- [x] ☑ all complete

### Wave 1 gate
- [x] All active stories in this wave checked off
- [x] All PRs merged
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 2 — Conductor Automation — Reducing Agent Ceremony

> Gate: all 4 automation stories closed; conductor handles claim, worktree, artifact commit, and archive without agent ceremony

- [x] Confirm Confirm Wave 1 gate passed

### 🔴 ARC-1367 — Conductor: auto-claim story in runbook before dispatching agent

- [x] **ARC-1367** — Conductor: auto-claim story in runbook before dispatching agent Depends on ARC-1349 (runbook_claim_story) — Wave 1 must be done
  - [x] 🔒 Claimed: parallel-lane-execution / 2026-07-07 09:34
  - [x] Read `issues/parallel-lane-execution/ARC-1367-conductor-auto-claim-story.md`
  - [x] Write `implementation_plans/parallel-lane-execution/ARC-1367-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1367-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(parallel-lane-execution): conductor auto-claim story before agent dispatch`

### 🔴 ARC-1368 — Conductor: auto-provision feature git worktree for HIGH stories

- [ ] **ARC-1368** — Conductor: auto-provision feature git worktree for HIGH stories Depends on ARC-1367 (auto-claim must fire first)
  - [x] 🔒 Claimed: parallel-lane-execution / 2026-07-07 13:50
  - [x] Read `issues/parallel-lane-execution/ARC-1368-conductor-auto-provision-worktree.md`
  - [x] Write `implementation_plans/parallel-lane-execution/ARC-1368-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1368-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): conductor auto-provision feature git worktree`

### 🔴 Batch — Batch 2-A

- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1369, ARC-1370
  - Files: `issues/parallel-lane-execution/ARC-1369-conductor-auto-commit-planning-artifacts.md`, `issues/parallel-lane-execution/ARC-1370-conductor-auto-archive-on-completion.md`
- [ ] Write all plans:
  - [ ] `implementation_plans/parallel-lane-execution/ARC-1369-implementation-plan.md`
  - [ ] `implementation_plans/parallel-lane-execution/ARC-1370-implementation-plan.md`
- [ ] 🔴 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1369 — Conductor: auto-commit implementation plans and completion summaries to planning repo Depends on ARC-1359 (planning_commit) pattern
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1369-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): conductor auto-commit planning artifacts after step success`
- [ ] Execute ARC-1370 — Conductor: auto-update issue status and archive story artifacts on completion Depends on ARC-1358, ARC-1360, ARC-1369
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1370-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): conductor auto-archive story artifacts on completion`
- [ ] ☑ all complete

### On-Hold story — ARC-1291

If ARC-1288 (core-infrastructure) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/parallel-lane-execution/ARC-1291.md`
- [ ] Write `implementation_plans/parallel-lane-execution/ARC-1291-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1291-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(parallel-lane-execution): ARC-1291 implementation`

If not: log in On-Hold Register below and skip.

### On-Hold story — ARC-1292

If ARC-1291 (intra-lane) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/parallel-lane-execution/ARC-1292.md`
- [ ] Write `implementation_plans/parallel-lane-execution/ARC-1292-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1292-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(parallel-lane-execution): ARC-1292 implementation`

If not: log in On-Hold Register below and skip.

### On-Hold story — ARC-1293

If ARC-1291 (intra-lane) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/parallel-lane-execution/ARC-1293.md`
- [ ] Write `implementation_plans/parallel-lane-execution/ARC-1293-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1293-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(parallel-lane-execution): ARC-1293 implementation`

If not: log in On-Hold Register below and skip.

### On-Hold story — ARC-1294

If ARC-1291 (intra-lane) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/parallel-lane-execution/ARC-1294.md`
- [ ] Write `implementation_plans/parallel-lane-execution/ARC-1294-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1294-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(parallel-lane-execution): ARC-1294 implementation`

If not: log in On-Hold Register below and skip.

### Wave 2 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1291 | ARC-1288 (core-infrastructure) | — | Confirm ARC-1288 merged; start immediately |
| ARC-1292 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |
| ARC-1293 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |
| ARC-1294 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| 1 | 3 | 1 | 1 | 5 |
| 2 | 2 | 5 | 1 | 8 |
