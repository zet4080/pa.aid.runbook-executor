# Runbook: Agent Workflow Improvements — ARC-1344

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Eliminate PR requirement for runbook artifacts, introduce a decision document pattern for async human decisions, and update all planning skills to use the non-interactive checkpoint model.
**Epic:** ARC-1344
**Lane:** agent-workflow
**Repos:** pa.aid.runbook-executor (planning), `~/.config/opencode/skills/` (skill files modified during execution)
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** none — begin immediately

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |


## Wave 1 — Agent Workflow Improvements

**Start condition:**
- [ ] No cross-lane prerequisites — begin immediately

### ARC-1345 — Fix runbook-commit workflow — no PR for runbook artifacts 🟢 LOW

- [x] **ARC-1345** — Fix runbook-commit workflow — no PR for runbook artifacts
  - [x] 🔒 Claimed: agent-workflow / 2026-07-01 20:03
  - [x] Read `issues/agent-workflow/ARC-1345-fix-runbook-commit-workflow.md`
  - [x] Write `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1345-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-workflow): fix runbook-commit workflow no PR for runbook artifacts`

### ARC-1346 — Introduce decision document pattern for async human decisions 🔴 HIGH

- [x] **ARC-1346** — Introduce decision document pattern for async human decisions
  - [x] 🔒 Claimed: agent-workflow / 2026-07-01 20:20
  - [x] Read `issues/agent-workflow/ARC-1346-decision-document-pattern.md`
  - [x] Write `implementation_plans/agent-workflow/ARC-1346-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1346-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-workflow): introduce decision document pattern for async human decisions`

### ARC-1347 — Update planning skills for non-interactive checkpoint model 🔴 HIGH

- [ ] Confirm ARC-1346 complete
- [ ] **ARC-1347** — Update planning skills for non-interactive checkpoint model
  - [x] 🔒 Claimed: agent-workflow / 2026-07-01 21:00
  - [ ] Read `issues/agent-workflow/ARC-1347-update-skills-noninteractive.md`
  - [ ] Write `implementation_plans/agent-workflow/ARC-1347-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1347-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-workflow): update planning skills for non-interactive checkpoint model`

🟢 **WAVE 1 GATE** — ARC-1345, ARC-1346, and ARC-1347 closed; agent-workflow lane complete


## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1347 | ARC-1346 (decision document pattern) | — | Begin ARC-1347 only after ARC-1346 is complete |


## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 3 (ARC-1345, ARC-1346, ARC-1347) | 0 | 1 | 4 |
| **Total** | **3** | **0** | **1** | **4** |
