# HOW-THIS-WORKS.md — Operational Reference for Runbook Executor Planning Repo

This document is the single authoritative operational reference for working in the pa.aid.runbook-executor planning repository.

---

## 1. What this repo is

pa.aid.runbook-executor is a planning repo for a local web-based tool (the Runbook Executor) that automates multi-lane runbook execution from pa.aid planning repos. The tool eliminates manual orchestration: a supervisor points it at a planning repo, selects runbooks, and it executes steps lane by lane — pausing at checkpoints for review, streaming live agent output, managing retries with escalation, and maintaining a prioritized checkpoint queue.

This repo contains requirements, issues, implementation plans, completion summaries, and execution runbooks. Development is driven by agents executing stories against issues in the `issues/` folder. Each story is a discrete unit of work with acceptance criteria, scope boundaries, and a defined artifact chain. Agents claim stories, write implementation plans, execute code changes, and produce completion summaries — all tracked through runbook checklists.

---

## 2. The three planning documents

| Document | Path | Purpose |
|----------|------|---------|
| Requirements | `docs/requirements/2026-06-27-runbook-executor.md` | Full requirements, constraints, success criteria |
| Workorder | `docs/plans/workorder.md` | Lane overview, story list, effort estimates, cross-lane dependencies |
| Runbooks | `docs/plans/runbook-{lane}.md` (7 files) | Executable per-lane checklists with checkpoint model |

---

## 3. Roles

| Role | Responsibility |
|------|----------------|
| Coding agent | Executes stories: reads issues, writes plans, implements code, runs tests, writes completion summaries |
| Review agent | Validates implementation against acceptance criteria; runs lint/tests; classifies findings as BLOCKER/ISSUE/ADVISORY |
| Human supervisor | Approves plans at checkpoints; reviews wave gate checklists; handles escalations and cross-lane coordination |

### What the review agent checks

1. Diff matches acceptance criteria in issue file
2. No regressions in existing tests
3. Lint and type-check pass
4. Implementation plan steps all executed
5. Completion summary maps evidence to each AC item

---

## 4. Checkpoint model

| Symbol | Tier | Trigger | Who acts |
|--------|------|---------|----------|
| 🔴 | HIGH individual | Before executing any HIGH-risk story | Coding agent presents plan; human supervisor approves before code is written |
| 🟡 | MEDIUM/LOW batch | Before executing a batch of MEDIUM/LOW stories | Coding agent presents all plans in batch; human supervisor approves before any story in batch executes |
| 🟢 | Wave gate | After all stories in a wave are checked off | Human supervisor verifies wave gate checklist; signals dependent lanes |

### Risk classification

| Risk | Definition | Stories in this repo |
|------|------------|---------------------|
| 🔴 HIGH | New architectural component, cross-system integration, or blocks 2+ other lanes | ARC-1285, ARC-1286, ARC-1288, ARC-1291, ARC-1292, ARC-1293, ARC-1301, ARC-1304, ARC-1306, ARC-1308, ARC-1310, ARC-1313 |
| 🟡 MEDIUM | Incremental feature within established pattern | ARC-1287, ARC-1294, ARC-1297–1299, ARC-1302, ARC-1303, ARC-1305, ARC-1307, ARC-1311, ARC-1312, ARC-1314–1316 |
| 🟢 LOW | Cosmetic, config, or docs change | None in this repo |

---

## 5. Agent workflows

### 9-step MEDIUM/LOW batch workflow

1. Load `start-execution-session` skill — identify lane, wave, next batch
2. Confirm cross-lane pre-checks are satisfied
3. Claim batch in runbook (fill 🔒 Claimed row)
4. Read all issue files in batch
5. Write all implementation plans for batch
6. 🟡 BATCH PLAN CHECKPOINT — present all plans to human supervisor; wait for approval
7. Execute stories in order: implement → local-code-review → lint/tests → completion summary → commit
8. Check off each story checkbox in runbook
9. Check wave gate; if all stories in wave complete, run wave gate checklist and notify supervisor

### 9-step HIGH individual workflow

1. Load `start-execution-session` skill — identify lane, wave, next HIGH story
2. Confirm cross-lane pre-checks are satisfied
3. Claim story in runbook (fill 🔒 Claimed row)
4. Read issue file
5. Write implementation plan
6. 🔴 INDIVIDUAL PLAN CHECKPOINT — present plan to human supervisor; wait for approval; do not write any code before approval
7. Execute plan: implement → local-code-review (iterate max 3 times for BLOCKER/ISSUE) → lint/tests pass
8. Write completion summary; commit
9. Check off story checkbox in runbook; check wave gate

---

## 6. Per-story artifact chain

1. **Issue file:** `issues/{epic}/{KEY}-{title}.md` — defines what to build, AC, scope
2. **Implementation plan:** `implementation_plans/{lane}/{KEY}-implementation-plan.md` — HOW to build, ordered steps
3. **Code:** feature branch named exactly `{KEY}` — implementation + tests
4. **Completion summary:** `task-completions/{KEY}-COMPLETION-SUMMARY.md` — evidence mapped to each AC item

**Branch naming rule:** exactly `{KEY}`, no prefix or suffix (e.g., `ARC-1285`, not `feature/ARC-1285`).

---

## 7. How to start a session

### First time

1. Read this file
2. Read `docs/plans/workorder.md`
3. Load `start-execution-session` skill
4. Identify which lane to work — check start conditions
5. Open the lane's runbook
6. Find first unchecked, unclaimed story
7. Follow agent workflow for its risk level

### Resuming

1. Load `start-execution-session` skill
2. Open the lane's runbook
3. Find first unchecked story
4. Check its state (no plan / plan exists / plan approved / code done / summary missing)
5. Continue from where work stopped

---

## 8. How to review an implementation plan

1. Plan steps map 1-to-1 to acceptance criteria in the issue file
2. No step adds scope not in the issue
3. Test strategy covers each AC item
4. Cross-lane dependencies noted if plan touches shared interfaces

---

## 9. How to review a wave close

1. Every story checkbox in the wave is checked off in the runbook
2. Every story has a completion summary in `task-completions/`
3. All PRs for stories in this wave are merged to main
4. No open BLOCKER or ISSUE findings from local-code-review
5. Dependent lanes have been notified of the gate passing

---

## 10. Failure and recovery

### Agent fails mid-story

The story is partially done. Resume: open runbook → find story → check what sub-steps are checked → continue from last checked sub-step. Do not re-execute already-committed work.

### Story blocked

Another story this one depends on is not yet merged. Add to On-Hold Register in runbook. Set a reminder to re-check. Do not start the story. The lane can continue with unblocked stories if any remain.

### Wave fails gate

One or more gate checklist items cannot be checked. Stop. Do not signal dependent lanes. Fix the failing item (re-run review, write missing summary, merge open PR). Then re-check gate.

### Emergency stop

Comment `STOP` in the relevant runbook with reason and timestamp. All agents working that lane must halt. Do not merge. Human supervisor decides next action.

---

## 11. Cross-supervisor coordination

| Event | Who signals | Who receives | What to do |
|-------|-------------|--------------|------------|
| Wave gate passes in lane X | Agent completing last story | Human supervisor | Check gate checklist; signal any dependent lane agents |
| Dependent lane unblocked | Human supervisor | Coding agent for that lane | Start lane — confirm pre-check and proceed |
| Story blocked (cross-lane dep missing) | Coding agent | Human supervisor | Add to On-Hold Register; supervisor tracks unblock |
| Scope change discovered | Coding agent | Human supervisor | Stop; open new issue or defer; do not implement out-of-scope |
| Security-sensitive change found | Coding agent | Human supervisor | Stop immediately; human sign-off required before proceeding |

---

## 12. On-Hold items protocol

### How to add

Write the story key, what it is blocked by, the date checked, and what action to take when unblocked into the On-Hold Register table in the relevant runbook.

### How to check

When a blocking story is merged, scan all On-Hold Registers in all runbooks for entries blocked by that story. Move those stories back to active.

### How to unblock

Remove from On-Hold Register; confirm pre-check at top of wave; claim story; proceed with normal workflow.

---

## 13. Commit message format

Format:
```
{type}({lane}): {short summary}

{detailed description of what changed and why}
```

For planning artifacts (runbook checkoffs, implementation plans, completion summaries), use: `docs({lane}): {short summary}`

Example:
```
feat(core-infrastructure): bootstrap web server and local UI serving

Adds Express server with static file serving for the local React UI.
Serves on configurable port (default 4000). Adds health-check endpoint.
Relates to ARC-1285.
```

---

## 14. Authoring skills reference

Skills are located in `opencode-config/skills/`:

- **create-issue:** template for new story files
- **write-implementation-plan:** translates issue into ordered technical steps
- **execute-implementation-plan:** executes an approved plan
- **local-code-review:** runs lint, tests, reviews diff; returns BLOCKER/ISSUE/ADVISORY findings
- **write-completion-summary:** documents evidence mapped to AC items
- **close-issue:** archives all three artifacts to done/
- **start-execution-session:** bootstrap skill — loads context, finds next story, checks wave gates

---

## 15. Lane overview

| Lane | Epic Key | Stories | Start Condition | Runbook |
|------|----------|---------|-----------------|---------|
| core-infrastructure | ARC-1284 | 4 | immediately | [docs/plans/runbook-core-infrastructure.md](runbook-core-infrastructure.md) |
| session-setup | ARC-1289 | 4 | after ARC-1286 merged | [docs/plans/runbook-session-setup.md](runbook-session-setup.md) |
| parallel-lane-execution | ARC-1290 | 4 | after ARC-1288 merged | [docs/plans/runbook-parallel-lane-execution.md](runbook-parallel-lane-execution.md) |
| checkpoint-management | ARC-1295 | 5 | after ARC-1291 merged | [docs/plans/runbook-checkpoint-management.md](runbook-checkpoint-management.md) |
| queue-scheduling-policy | ARC-1296 | 3 | after ARC-1303 merged | [docs/plans/runbook-queue-scheduling-policy.md](runbook-queue-scheduling-policy.md) |
| failure-handling | ARC-1300 | 3 | after ARC-1288 merged | [docs/plans/runbook-failure-handling.md](runbook-failure-handling.md) |
| session-history | ARC-1309 | 3 | after ARC-1287 merged | [docs/plans/runbook-session-history.md](runbook-session-history.md) |

---

*Last updated: 2026-06-27*
