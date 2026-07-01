# AGENTS.md — pa.aid.runbook-executor

## What This Repo Contains

| Folder | Contents |
|--------|----------|
| `docs/requirements/` | Approved requirements doc |
| `docs/plans/` | Workorder, runbooks, proposed issues, create-epics log, session logs |
| `issues/` | One folder per epic; one markdown file per story |
| `implementation_plans/` | Per-story implementation plans (created during execution) |
| `task-completions/` | Completion summaries per story |
| `done/` | Archived completed items |
| `opencode-config/skills/` | Agent skills available in this repo |

## Sibling Repositories

| Repo | Path | Purpose |
|------|------|---------|
| `pa.aid.conductor.ts` | `/repos/pa.aid.conductor.ts` | Standalone runbook executor application — server, UI, and all application code live here |

**Application code target:** All runbook executor application source code (server, UI, etc.) belongs in `pa.aid.conductor.ts`, not in this planning repo.

**Application code location:** Root of `pa.aid.conductor.ts` (code is not in a subdirectory).

## Branching Strategy (pa.aid.conductor.ts)

All runbook executor feature work targets `pa.aid.conductor.ts`. The `main` branch stays clean until the full feature set is complete.

| Rule | Detail |
|------|--------|
| Long-lived integration branch | `runbook-executor` — all feature work merges here |
| Feature branches | Branch off `runbook-executor`; named exactly `<KEY>` (e.g. `ARC-1285`) |
| Merge target | Feature branches merge back into `runbook-executor`, never directly to `main` |
| Final merge | `runbook-executor` → `main` only when all epics are complete |

**Branch creation command:**
```bash
git -C /repos/pa.aid.conductor.ts worktree add /repos/<KEY> -b <KEY> main
```

**Runbook artifact commits** (runbook checkoffs, implementation plans, completion summaries) always go to `main` in `pa.aid.runbook-executor` — never into feature branches or `pa.aid.conductor.ts`.

## Workflow

| Step | Action | Artifact |
|------|--------|----------|
| 1. Issue | Define what to build | `issues/{epic}/{KEY}-{title}.md` |
| 2. Plan | Write implementation plan | `implementation_plans/{lane}/{KEY}-implementation-plan.md` |
| 3. Execute | Implement, test, commit in `pa.aid.conductor.ts` | feature branch named exactly `{KEY}`, branched off `main` |
| 4. Complete | Write completion summary | `task-completions/{KEY}-COMPLETION-SUMMARY.md` |

## Epics

| Epic Key | Summary | Issues Folder |
|----------|---------|---------------|
| ARC-1284 | ARC: Core Infrastructure | `issues/core-infrastructure/` |
| ARC-1289 | ARC: Session Setup & Runbook Selection | `issues/session-setup/` |
| ARC-1290 | ARC: Parallel Lane Execution | `issues/parallel-lane-execution/` |
| ARC-1295 | ARC: Checkpoint Management | `issues/checkpoint-management/` |
| ARC-1296 | ARC: Queue & Scheduling Policy | `issues/queue-scheduling-policy/` |
| ARC-1300 | ARC: Failure Handling & Escalation | `issues/failure-handling/` |
| ARC-1309 | ARC: Session History | `issues/session-history/` |

## Commit Message Format

```
{type}({lane}): {short summary}

{detailed description of what changed and why}
```

For planning artifacts: `docs({lane}): {short summary}`

## Issue Authoring

Issue files follow the template in `opencode-config/skills/create-issue/`. Each file includes: frontmatter table, goal, acceptance criteria (Given/When/Then), in scope, out of scope, constraints, dependencies.

## Execution Planning

Full operational reference: see `docs/plans/HOW-THIS-WORKS.md`.

Lane runbooks (one per epic):
- `docs/plans/runbook-core-infrastructure.md`
- `docs/plans/runbook-session-setup.md`
- `docs/plans/runbook-parallel-lane-execution.md`
- `docs/plans/runbook-checkpoint-management.md`
- `docs/plans/runbook-queue-scheduling-policy.md`
- `docs/plans/runbook-failure-handling.md`
- `docs/plans/runbook-session-history.md`

## MCP Endpoints

| Service | Purpose |
|---------|---------|
| atlassian | Jira + Confluence (JIRA_API_TOKEN, CONFLUENCE_API_TOKEN) |
| bitbucket | PR management (BITBUCKET_PASSWORD) |
| sonarqube | Code quality gates (SONARQUBE_TOKEN) |
| aws-mcp / eks-mcp | AWS infrastructure (AWS credentials) |
