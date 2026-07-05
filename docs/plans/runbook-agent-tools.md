# Runbook: OpenCode Agent Tools — ARC-1348-epic

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Agents can invoke purpose-built tools for all mechanical planning-repo and code-quality operations, eliminating manual markdown parsing, fragile git command chains, and framework-detection guesswork from the skill pipeline.
**Epic:** ARC-1348-epic
**Lane:** agent-tools
**Repos:** pa.aid.runbook-executor (planning), `~/.config/opencode/tools/` (tool files created during execution), `pa.aid.conductor.ts` (parser reference)
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** none — begin immediately

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

## Wave 1 — Runbook Navigation Tools

> Gate: none — begin immediately

### 🔴 HIGH — ARC-1348: `runbook_find_next_story`

- [x] **ARC-1348** — Implement `runbook_find_next_story` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-02 14:00
  - [x] Read `issues/agent-tools/ARC-1348-tool-runbook-find-next-story.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1348-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1348-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement runbook_find_next_story tool`

### 🔴 HIGH — ARC-1349: `runbook_claim_story`

- [x] Confirm `runbook_find_next_story` complete
- [x] **ARC-1349** — Implement `runbook_claim_story` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-02 14:52
  - [x] Read `issues/agent-tools/ARC-1349-tool-runbook-claim-story.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1349-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1349-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement runbook_claim_story tool`

### 🔴 HIGH — ARC-1362: `validate_runbook`

- [x] **ARC-1362** — Implement `validate_runbook` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-03 09:40
  - [x] Read `issues/agent-tools/ARC-1362-tool-validate-runbook.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1362-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1362-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement validate_runbook tool`

### 🔴 HIGH — ARC-1363: `generate_runbook`

- [x] **ARC-1363** — Implement `generate_runbook` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-03 11:33
  - [x] Read `issues/agent-tools/ARC-1363-tool-generate-runbook.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1363-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1363-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement generate_runbook tool`

### 🔴 HIGH — ARC-1350: `runbook_check_step`

- [x] Confirm `runbook_claim_story` complete
- [x] **ARC-1350** — Implement `runbook_check_step` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-04 14:38
  - [x] Read `issues/agent-tools/ARC-1350-tool-runbook-check-step.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1350-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1350-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement runbook_check_step tool`

### 🔴 HIGH — ARC-1351: `runbook_check_story`

- [x] Confirm `runbook_claim_story` complete
- [x] **ARC-1351** — Implement `runbook_check_story` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-04 20:37
  - [x] Read `issues/agent-tools/ARC-1351-tool-runbook-check-story.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1351-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1351-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement runbook_check_story tool`

### Wave 1 gate

- [x] All Wave 1 stories checked off
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — Wave 1 closed; Wave 2 may begin

## Wave 2 — Code Quality & Issue Tools

> Gate: Wave 1 closed (ARC-1349 complete — required by ARC-1360)

### 🔴 HIGH — ARC-1353: `get_current_issue`

- [x] **ARC-1353** — Implement `get_current_issue` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-05 08:15
  - [x] Read `issues/agent-tools/ARC-1353-tool-get-current-issue.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1353-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1353-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement get_current_issue tool`

### 🔴 HIGH — ARC-1354: `run_tests`

- [x] **ARC-1354** — Implement `run_tests` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-05 09:36
  - [x] Read `issues/agent-tools/ARC-1354-tool-run-tests.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1354-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1354-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(agent-tools): implement run_tests tool`

### 🔴 HIGH — ARC-1355: `run_lint`

- [x] **ARC-1355** — Implement `run_lint` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-05 09:57
  - [x] Read `issues/agent-tools/ARC-1355-tool-run-lint.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1355-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1355-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement run_lint tool`

### 🔴 HIGH — ARC-1358: `update_issue_status`

- [ ] **ARC-1358** — Implement `update_issue_status` OpenCode tool
  - [x] 🔒 Claimed: agent-tools / 2026-07-05 10:32
  - [x] Read `issues/agent-tools/ARC-1358-tool-update-issue-status.md`
  - [x] Write `implementation_plans/agent-tools/ARC-1358-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1358-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement update_issue_status tool`

### 🔴 HIGH — ARC-1360: `archive_issue`

- [ ] Confirm `runbook_check_story` tool complete
- [ ] Confirm `update_issue_status` tool complete
- [ ] **ARC-1360** — Implement `archive_issue` OpenCode tool
  - [ ] 🔒 Claimed: _(fill in: agent-tools / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/agent-tools/ARC-1360-tool-archive-issue.md`
  - [ ] Write `implementation_plans/agent-tools/ARC-1360-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1360-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement archive_issue tool`

### 🟡 Batch — Supporting tools (ARC-1352, ARC-1356, ARC-1357, ARC-1359, ARC-1361, ARC-1364)

- [ ] 🔒 Claimed: _(fill in: agent-tools / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1352, ARC-1356, ARC-1357, ARC-1359, ARC-1361, ARC-1364
- [ ] Write all implementation plans:
  - [ ] `implementation_plans/agent-tools/ARC-1352-implementation-plan.md`
  - [ ] `implementation_plans/agent-tools/ARC-1356-implementation-plan.md`
  - [ ] `implementation_plans/agent-tools/ARC-1357-implementation-plan.md`
  - [ ] `implementation_plans/agent-tools/ARC-1359-implementation-plan.md`
  - [ ] `implementation_plans/agent-tools/ARC-1361-implementation-plan.md`
  - [ ] `implementation_plans/agent-tools/ARC-1364-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1352 (`runbook_check_dependencies`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1352-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement runbook_check_dependencies tool`
- [ ] Execute ARC-1356 (`run_typecheck`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1356-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement run_typecheck tool`
- [ ] Execute ARC-1357 (`get_branch_diff`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1357-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement get_branch_diff tool`
- [ ] Execute ARC-1359 (`planning_commit`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1359-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement planning_commit tool`
- [ ] Execute ARC-1361 (`preflight_check`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1361-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement preflight_check tool`
- [ ] Execute ARC-1364 (`generate_issue_file`)
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1364-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(agent-tools): implement generate_issue_file tool`
- [ ] ☑ batch complete

### Wave 2 gate

- [ ] All Wave 2 stories checked off (ARC-1352, ARC-1353, ARC-1354, ARC-1355, ARC-1356, ARC-1357, ARC-1358, ARC-1359, ARC-1360, ARC-1361, ARC-1364)
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — agent-tools lane complete; all 16 tools deployed to `~/.config/opencode/tools/`

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 6 (ARC-1348, ARC-1349, ARC-1362, ARC-1363, ARC-1350, ARC-1351) | 0 | 1 | 7 |
| Wave 2 | 5 (ARC-1353, ARC-1354, ARC-1355, ARC-1358, ARC-1360) | 1 (ARC-1352, ARC-1356, ARC-1357, ARC-1359, ARC-1361, ARC-1364) | 1 | 7 |
| **Total** | **11** | **1** | **2** | **14** |
