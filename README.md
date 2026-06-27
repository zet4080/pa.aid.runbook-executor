# pa.aid.runbook-executor

Planning repo for the **Runbook Executor** — a local web-based tool to automate multi-lane runbook execution from pa.aid planning repos.

## What Is This?

The Runbook Executor eliminates manual orchestration of agent-driven development runbooks. A supervisor points the tool at a planning repo, selects which runbooks to automate, and the tool executes steps lane by lane — pausing at 🔴🟡🟢 checkpoints for supervisor review, streaming live agent output, managing retries with escalation, and maintaining a prioritized checkpoint queue.

The supervisor remains in control at every checkpoint: viewing and editing artifacts inline, approving or rejecting with feedback, and choosing queue scheduling policy.

## Quick Navigation

| Section | Path | Contents |
|---------|------|----------|
| Requirements | `docs/requirements/2026-06-27-runbook-executor.md` | Full requirements, constraints, success criteria |
| Proposed Issues | `docs/plans/proposed-issues-2026-06-27.md` | All 26 stories with AC and scope |
| Epics Log | `docs/plans/create-epics-log-2026-06-27.md` | Jira epic and story keys |
| Issues | `issues/` | One folder per epic, one file per story |
| Implementation Plans | `implementation_plans/` | Per-story plans (created during execution) |
| Completion Summaries | `task-completions/` | Evidence of completed stories |

## Epics & Stories

7 epics, 26 stories in Jira project ARC:

| Epic | Key | Stories |
|------|-----|---------|
| Core Infrastructure | ARC-1284 | ARC-1285 – ARC-1288 |
| Session Setup & Runbook Selection | ARC-1289 | ARC-1313 – ARC-1316 |
| Parallel Lane Execution | ARC-1290 | ARC-1291 – ARC-1294 |
| Checkpoint Management | ARC-1295 | ARC-1301 – ARC-1305 |
| Queue & Scheduling Policy | ARC-1296 | ARC-1297 – ARC-1299 |
| Failure Handling & Escalation | ARC-1300 | ARC-1306 – ARC-1308 |
| Session History | ARC-1309 | ARC-1310 – ARC-1312 |

## Next Steps

Invoke `write-workorder` to produce the execution plan with parallel lanes and wave-based milestones.
