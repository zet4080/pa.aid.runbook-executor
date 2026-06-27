# pa.aid.runbook-executor

Planning repository for the Runbook Executor — a local web-based tool that automates multi-lane runbook execution from pa.aid planning repos.

---

## What Is This?

pa.aid.runbook-executor is a planning repo for the Runbook Executor — a local web-based tool that automates multi-lane runbook execution from pa.aid planning repos. The tool eliminates manual orchestration: a supervisor points it at a planning repo, selects runbooks, and it executes steps lane by lane — pausing at checkpoints for review, streaming live agent output, managing retries with escalation, and maintaining a prioritized checkpoint queue.

This repo contains requirements, issues, implementation plans, runbooks, completion summaries, and execution runbooks. Development is driven by agents executing stories against issues in the `issues/` folder. For the complete operational reference, see `docs/plans/HOW-THIS-WORKS.md`.

---

## Quick Navigation

| Section | Path | Contents |
|---------|------|----------|
| Requirements | `docs/requirements/2026-06-27-runbook-executor.md` | Full requirements, constraints, success criteria |
| Workorder | `docs/plans/workorder.md` | Lane overview, story list, effort estimates, dependencies |
| Runbooks | `docs/plans/runbook-{lane}.md` | Executable per-lane checklists (7 files) |
| Operational reference | `docs/plans/HOW-THIS-WORKS.md` | Checkpoint model, agent workflows, wave gates, failure recovery |
| Issues | `issues/` | One folder per epic, one file per story |
| Implementation plans | `implementation_plans/` | Per-story plans (created during execution) |
| Completion summaries | `task-completions/` | Evidence of completed stories |

---

## Project Planning Documents

Full operational reference: see `docs/plans/HOW-THIS-WORKS.md`.

| Document | Path | Purpose |
|----------|------|---------|
| Requirements | `docs/requirements/2026-06-27-runbook-executor.md` | Full requirements, constraints, success criteria |
| Workorder | `docs/plans/workorder.md` | Lane overview, story list, effort estimates, cross-lane dependencies |
| Runbooks | `docs/plans/runbook-{lane}.md` (7 files) | Executable per-lane checklists with checkpoint model |

---

## Execution Model Summary

- 7 parallel feature lanes, each with its own runbook
- 26 stories total; 37h wall-clock on critical path
- 3-tier checkpoint model: 🔴 individual plan, 🟡 batch plan, 🟢 wave gate
- ~6.5h supervisor time across 37h wall-clock (~18% supervision ratio)
- No global sync gates — per-story supervisor reviews only

### Supervision Budget

| Lane | Stories | Wall-clock | Agent-hours total | Supervision |
|------|---------|------------|------------------|-------------|
| core-infrastructure | 4 | 18h | 18h | 4 × 15 min = 1h |
| session-setup | 4 | 12h | 12h | 4 × 15 min = 1h |
| parallel-lane-execution | 4 | 16h | 16h | 4 × 15 min = 1h |
| checkpoint-management | 5 | 17h | 17h | 5 × 15 min = 1.25h |
| queue-scheduling-policy | 3 | 10h | 10h | 3 × 15 min = 0.75h |
| failure-handling | 3 | 10h | 10h | 3 × 15 min = 0.75h |
| session-history | 3 | 9h | 9h | 3 × 15 min = 0.75h |
| **Total** | **26** | **37h** (parallel) | **92h** | **6.5h** |

---

## Parallel Lane Diagram

```
T=0       T=8       T=15      T=19      T=25      T=31      T=37
│         │         │         │         │         │         │
├─── core-infrastructure ─────────────────────────────────────┤  18h
          ├─── session-setup ──────────┤                          12h (done T=20)
                    ├─── parallel-lane-execution ────────────┤    16h (done T=31)
                              ├─── checkpoint-management ──────────────┤  17h (done T=36)
                                        ├─── queue-scheduling-policy ──┤  10h (done T=37)
                    ├─── failure-handling ──────────┤               10h (done T=25)
          ├─── session-history ──────────┤                           9h (done T=20)
```

---

## Issue Scope and Status

| Epic | Key | Stories | Status |
|------|-----|---------|--------|
| ARC: Core Infrastructure | ARC-1284 | 4 | Not started |
| ARC: Session Setup & Runbook Selection | ARC-1289 | 4 | Not started |
| ARC: Parallel Lane Execution | ARC-1290 | 4 | Not started |
| ARC: Checkpoint Management | ARC-1295 | 5 | Not started |
| ARC: Queue & Scheduling Policy | ARC-1296 | 3 | Not started |
| ARC: Failure Handling & Escalation | ARC-1300 | 3 | Not started |
| ARC: Session History | ARC-1309 | 3 | Not started |

---

## Developer Setup

```bash
git clone <repo-url>
cd pa.aid.runbook-executor
```

Configure MCP endpoints before starting agent sessions:

| Service | Purpose |
|---------|---------|
| atlassian | Jira + Confluence (JIRA_API_TOKEN, CONFLUENCE_API_TOKEN) |
| bitbucket | PR management (BITBUCKET_PASSWORD) |
| sonarqube | Code quality gates (SONARQUBE_TOKEN) |
| aws-mcp / eks-mcp | AWS infrastructure (AWS credentials) |

---

## Authoring Standards

Issue and skill authoring standards live in `opencode-config/skills/`:

- **create-issue:** template for new story files
- **write-implementation-plan:** translates issue into ordered technical steps
- **execute-implementation-plan:** executes an approved plan
- **local-code-review:** runs lint, tests, reviews diff; returns BLOCKER/ISSUE/ADVISORY findings
- **write-completion-summary:** documents evidence mapped to AC items
- **close-issue:** archives all three artifacts to done/
- **start-execution-session:** bootstrap skill — loads context, finds next story, checks wave gates
