---
name: _index
description: Use when no specific skill is obvious and a complete skill registry is needed to choose the correct pa.aid-owned skill
---
# Skill: _index
## Skills Registry & Auto-Loader

## Purpose

Meta-skill: discover all available skills when current task domain is unclear. Provides registry and decision rules to select correct domain skill.

## When to Use

- No specific skill obvious at task start
- Need complete pa.aid skill registry
- Need help selecting between multiple applicable skills
- Not for executing task itself (load domain skill after _index)

---



**Step 1 — Load the index** *(you are here)*
Call `skill("_index")` to discover all available skills and their trigger conditions.

**Step 2 — Load the matching skill**
Identify the skill whose domain and trigger keywords match the current task, then call `skill("<skill-name>")` to inject its full instructions and resources into the conversation.

If two skills seem relevant, load the one whose trigger keywords are the closest match. Load only one skill per task.

---

## Skills Registry

### Domain Skills

| Skill Name | Domain | Trigger Keywords |
|---|---|---|
| `provisioning` | proALPHA SaaS tenant provisioning and product assignment | provisioning, tenant, product assignment, SaaS agent, portal agent, AIF agent, ProvisioningCommandIssued, ParameterSetChanged, TenantProductAssigned |
| `ccoe_mod` | CCoE Terraform reusable modules (`ccoe_mod/` git sources) | ccoe_mod, terraform module, vpc, eks, aurora, cognito, argocd, monitoring, cert-manager, s3-secret, elasticache, pgadmin |
| `platform-components` | Terraform/Atmos platform component monorepo (49 components) | atmos, platform-components, component, stack, discovery, cloudposse, atmos_discovery, shared stack, normal stack |
| `tekton-pipelines` | Tekton CI/CD pipelines and reusable tasks | tekton, pipeline, task, CI/CD, build pipeline, deploy pipeline, release pipeline, kustomize, git-clone, sonar, buildah, argocd-task-sync |
| `tekton-status` | Query live Tekton PipelineRun status via CloudWatch | tekton pipelinerun status, cloudwatch logs, eks audit logs, pipeline stuck, pipeline not visible |
| `apigateway` | API Gateway configuration | pa.routing.apigateway, appsettings.json, YARP, routing, apigateway routes, swagger aggregation, jwt authentication gateway |
| `pa-core` | proALPHA core .NET libraries (`Pa.Core.*`) and baseapp Helm chart | Pa.Core, pa.core, NuGet, Program.cs, AccessAndEntitlement, ApiVersioning, Boot, EFCore, ErrorHandling, HealthCheck, Identity, Logging, MessageBus, Redis, Telemetry, baseapp, values.yaml, DatabaseConfiguration, MessageBusConfiguration, WebServerConfiguration |

### Process / Workflow Skills

| Skill Name | Domain | Trigger Keywords |
|---|---|---|
| `add-story-to-epic` | Adding one story to existing epic and syncing planning artifacts | add story, epic story, planning artifacts, sync story |
| `ansible-local-test` | Test Ansible role changes locally | ansible, playbook, role, local test, validate playbook, windows ansible |
| `ansible-role-testing` | Test proALPHA Ansible roles/playbooks through RTU under `/repos` | ansible role testing, RTU, RoleTestingUtility, deploy.proalpha.ansiblegalaxy, deploy.proalpha.playbooks |
| `ansible-role-writing` | Write proALPHA Ansible roles, playbooks, custom modules | ansible role writing, argument_specs, tests/test.yml, FQCN, idempotency, win_shell, win_command |
| `capture-requirements` | Capturing requirements from idea/doc/page | requirements, capture, idea, specification, doc to requirements |
| `caveman` | Ultra-compressed communication mode | caveman mode, compress communication, terse mode |
| `caveman-commit` | Write commit messages in caveman style | commit message, /commit, generate commit |
| `caveman-help` | Quick-reference card for caveman modes | caveman help, caveman reference, caveman commands |
| `caveman-review` | Ultra-compressed PR/code review comments | caveman review, compressed review, terse code review |
| `close-issue` | Archiving completed issue artifacts | close issue, archive issue, done/, completed issue |
| `compress` | Compress memory files into caveman format | compress memory, compress CLAUDE.md, compress todos |
| `create-epics` | Turn planning-repo requirements into Jira epics and issues | create epics, planning repo, jira epics, requirements to epics |
| `create-issue` | Write a new engineering issue | create issue, new issue, engineering issue, write issue |
| `execute-implementation-plan` | Implementing an approved plan | execute plan, implement plan, approved plan, implementation |
| `finishing-a-development-branch` | Work complete, need to merge/PR/cleanup | finish branch, merge branch, PR cleanup, development done |
| `generate-lane-runbooks` | Generate per-lane agent runbooks from a workorder | lane runbooks, agent runbooks, workorder runbooks |
| `local-code-review` | Pre-PR local review/fix loop | local review, pre-PR review, code review loop, lint check |
| `openedge-guid` | Generate OpenEdge GUID / UUID hex identifiers | openedge guid, uuid, guid, hex identifier |
| `pr-review` | Full Bitbucket PR review with Jira context and SonarQube | PR review, bitbucket review, pull request review, sonarqube |
| `pr-triage` | Triage today's PRs to find ones needing deeper review | PR triage, triage PRs, daily PRs, review queue |
| `restructure-epics` | Restructure Jira epics into feature layout | restructure epics, epic layout, feature epics, reorganize epics |
| `start-execution-session` | Starting or resuming work on a planning repo runbook | start session, execution session, resume runbook, planning repo session |
| `sync-runbook` | Reconcile runbook/workorder to actual work done | sync runbook, update runbook, runbook out of date, agent forgot to check off, I did work manually, reconcile runbook, sync workorder |
| `systematic-debugging` | Any bug, test failure, unexpected behavior | bug, debug, test failure, unexpected behavior, broken, error |
| `test-driven-development` | Any feature or bugfix implementation | TDD, test-driven, feature implementation, bugfix, write tests first |
| `using-git-worktrees` | Starting feature work needing isolation | worktree, git worktree, feature isolation, parallel branches |
| `using-skills` | How to discover and invoke skills | how to use skills, find skill, skill discovery, invoke skill |
| `write-completion-summary` | Documenting completed work | completion summary, done summary, work summary, document completed |
| `write-design-spec` | Technical design spec before implementation planning | design spec, technical spec, architecture doc, design document |
| `write-implementation-plan` | Translating issue into execution plan | implementation plan, execution plan, translate issue, plan from issue |
| `write-workorder` | Generate wave-based workorder doc | workorder, work order, wave-based, workorder doc |
| `writing-skills` | Creating or editing skills | write skill, create skill, edit skill, new skill, skill authoring |

---

## Decision Rules

### Domain Skills

| I need to… | Load skill |
|---|---|
| Add or modify a provisioning command or parameter set | `provisioning` |
| Create or change a provisioning agent (SaaS, Portal, AIF) | `provisioning` |
| Modify or debug tenant onboarding / product-assignment flows | `provisioning` |
| Change a provisioning message contract | `provisioning` |
| Modify IaC in `provisioning.module.iac` | `provisioning` |
| Write or modify Terraform referencing a `git::ssh://git.proalpha.com/cce_mod/` source | `ccoe_mod` |
| Add infrastructure: VPC, EKS, Aurora, Cognito, ALB, SQS, DynamoDB, ECR, WAF | `ccoe_mod` |
| Ask "which CCoE module do I use for X?" | `ccoe_mod` |
| Write or modify an Atmos stack configuration | `platform-components` |
| Add or modify a platform component in the Terraform monorepo | `platform-components` |
| Understand `atmos_discovery_*` tags, shared vs. normal stacks, or the Cloudposse Context Provider | `platform-components` |
| Create or modify a Tekton pipeline or task | `tekton-pipelines` |
| Debug a CI/CD pipeline failure | `tekton-pipelines` |
| Add a build, deploy, release, provisioning, or deprovisioning pipeline | `tekton-pipelines` |
| Understand available reusable Tekton tasks (git-clone, sonar-dotnet, buildah, argocd-sync, etc.) | `tekton-pipelines` |
| Check status of a running or stuck Tekton PipelineRun via CloudWatch | `tekton-status` |
| Configuring API Gateway routes, clusters, auth, CORS, rate limits, Swagger aggregation, appsettings.json for the API Gateway | `apigateway` |
| Add or configure any `Pa.Core.*` NuGet package in a .NET service | `pa-core` |
| Set up `Program.cs` / `Startup.cs` for a service using pa.core libraries | `pa-core` |
| Write or edit a `values.yaml` for a service using the `baseapp` Helm chart | `pa-core` |
| Map Helm deployment parameters (`databaseconfiguration`, `busconfiguration`, `redis`, `authentication`) to .NET config env vars | `pa-core` |

### Process / Workflow Skills

| I need to… | Load skill |
|---|---|
| Test ansible role changes, run ansible locally, validate playbook against Windows | `ansible-local-test` |
| Test proALPHA Ansible roles/playbooks via RTU under `/repos` | `ansible-role-testing` |
| Write or review required structure for proALPHA Ansible roles/playbooks/modules | `ansible-role-writing` |
| Write a technical design spec before implementation planning | `write-design-spec` |
| Capture requirements from an idea, doc, or page | `capture-requirements` |
| Switch to ultra-compressed communication mode | `caveman` |
| Write a commit message (/commit, "generate commit") | `caveman-commit` |
| Get a quick-reference card for all caveman modes | `caveman-help` |
| Write ultra-compressed PR/code review comments | `caveman-review` |
| Archive a completed issue to done/ | `close-issue` |
| Add one story to an existing epic and sync planning artifacts | `add-story-to-epic` |
| Turn planning-repo requirements into Jira epics and issues | `create-epics` |
| Compress memory files (CLAUDE.md, todos) into caveman format | `compress` |
| Write a new engineering issue | `create-issue` |
| Implement an approved plan | `execute-implementation-plan` |
| Finish a development branch (merge/PR/cleanup) | `finishing-a-development-branch` |
| Generate per-lane agent runbooks from a workorder | `generate-lane-runbooks` |
| Run pre-PR local review/fix loop | `local-code-review` |
| Generate OpenEdge GUID / hyphenless UUID hex identifiers | `openedge-guid` |
| Do a full Bitbucket PR review with Jira context and SonarQube | `pr-review` |
| Triage today's PRs to find ones needing deeper review | `pr-triage` |
| Restructure Jira epics into feature layout | `restructure-epics` |
| Start or resume work on a planning repo runbook | `start-execution-session` |
| Reconcile runbook/workorder to actual work done after manual or forgotten steps | `sync-runbook` |
| Debug any bug, test failure, or unexpected behavior | `systematic-debugging` |
| Implement any feature or bugfix (test-driven) | `test-driven-development` |
| Start feature work needing isolation via git worktrees | `using-git-worktrees` |
| Discover and invoke skills | `using-skills` |
| Document completed work | `write-completion-summary` |
| Translate an issue into an execution plan | `write-implementation-plan` |
| Generate a wave-based workorder doc | `write-workorder` |
| Create or edit a skill | `writing-skills` |

---

## Loading a Skill

Once you have identified the correct skill name from the registry above, make this tool call:

```
skill(name="<skill-name>")
```

**Examples:**
```
skill(name="provisioning")
skill(name="ccoe_mod")
skill(name="platform-components")
skill(name="tekton-pipelines")
skill(name="apigateway")
skill(name="pa-core")
skill(name="systematic-debugging")
skill(name="write-design-spec")
skill(name="test-driven-development")
```

Do not proceed with implementation until the matching skill has been loaded and its instructions have been injected into the conversation.

---

> **Note:** If no skill matches the current task, proceed without loading one — not every task requires a skill.
