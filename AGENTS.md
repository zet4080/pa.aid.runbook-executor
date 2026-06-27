# AGENTS.md — pa.aid.runbook-executor

## What This Repo Contains

| Folder | Contents |
|--------|----------|
| `docs/requirements/` | Approved requirements doc |
| `docs/plans/` | Proposed issues, create-epics log, workorder (when generated) |
| `issues/` | One folder per epic; one markdown file per story |
| `implementation_plans/` | Per-story implementation plans (created during execution) |
| `task-completions/` | Completion summaries per story |
| `done/` | Archived completed items |
| `opencode-config/skills/` | Agent skills available in this repo |

## Workflow

| Step | Action | Artifact |
|------|--------|----------|
| 1. Issue | Define what to build | `issues/{epic}/{KEY}-{title}.md` |
| 2. Plan | Write implementation plan | `implementation_plans/{KEY}-plan.md` |
| 3. Execute | Implement, test, commit | feature branch named `{KEY}` |
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
{KEY}: {short summary}

{detailed description of what changed and why}
```

## Issue Authoring

Issue files follow the template in `opencode-config/skills/create-issue/`. Each file includes: frontmatter table, goal, acceptance criteria (Given/When/Then), in scope, out of scope, constraints, dependencies.

## Execution Planning

Once workorder is generated, see `docs/plans/workorder.md` and `docs/plans/runbook-*.md` for lane-based execution instructions.

## MCP Endpoints

| Service | Purpose |
|---------|---------|
| atlassian | Jira + Confluence (JIRA_API_TOKEN, CONFLUENCE_API_TOKEN) |
| bitbucket | PR management (BITBUCKET_PASSWORD) |
| sonarqube | Code quality gates (SONARQUBE_TOKEN) |
| aws-mcp / eks-mcp | AWS infrastructure (AWS credentials) |
