# Runbook: Parallel Lane Execution — ARC-1290

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** The executor can run multiple runbook lanes concurrently, streaming live agent output per lane while updating runbook checkbox state in real time.
**Epic:** ARC-1290
**Lane:** parallel-lane-execution
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1288 (core-infrastructure — opencode CLI subprocess)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Wave 1 — Foundation & Concurrency (ARC-1291, ARC-1292, ARC-1293, ARC-1294)

- [ ] Confirm core-infrastructure ARC-1288 merged

### ARC-1291 — Execute runbook steps sequentially within a lane 🔴 HIGH

- [ ] **ARC-1291** — Execute runbook steps sequentially within a lane
  - [x] 🔒 Claimed: parallel-lane-execution / 2026-06-30 00:00
  - [ ] Read `issues/parallel-lane-execution/ARC-1291-sequential-step-execution.md`
  - [ ] Write `implementation_plans/parallel-lane-execution/ARC-1291-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1291-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): sequential step execution within a lane`

### ARC-1292 — Run multiple lanes concurrently up to concurrency limit 🔴 HIGH

- [ ] **ARC-1292** — Run multiple lanes concurrently up to concurrency limit
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/parallel-lane-execution/ARC-1292-concurrent-lanes.md`
  - [ ] Write `implementation_plans/parallel-lane-execution/ARC-1292-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1292-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): concurrent lane execution up to concurrency limit`

### ARC-1293 — Stream live agent output per lane in the UI 🔴 HIGH

- [ ] **ARC-1293** — Stream live agent output per lane in the UI
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/parallel-lane-execution/ARC-1293-stream-agent-output.md`
  - [ ] Write `implementation_plans/parallel-lane-execution/ARC-1293-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1293-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): stream live agent output per lane in UI`

### Batch: ARC-1294 🟡 MEDIUM

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1294
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1294
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1294-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(parallel-lane-execution): real-time runbook checkbox state updates`
- [ ] ☑ batch complete

### 🟢 Wave 1 Gate

- [ ] ARC-1291 completion summary written and committed
- [ ] ARC-1292 completion summary written and committed
- [ ] ARC-1293 completion summary written and committed
- [ ] ARC-1294 completion summary written and committed
- [ ] 🟢 WAVE 1 GATE — all 4 stories closed; unblock checkpoint-management lane (ARC-1291 → ARC-1301)

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1291 | ARC-1288 (core-infrastructure) | — | Confirm ARC-1288 merged; start immediately |
| ARC-1292 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |
| ARC-1293 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |
| ARC-1294 | ARC-1291 (intra-lane) | — | Start after ARC-1291 committed |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 3 | 1 | 1 | 5 |
| **Total** | **3** | **1** | **1** | **5** |
