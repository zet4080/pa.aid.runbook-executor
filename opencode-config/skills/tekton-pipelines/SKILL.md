---
name: tekton-pipelines
description: Use when creating or changing Tekton CI/CD pipelines, reusable tasks, build/deploy/release pipelines, or task templates
---
# Skill: tekton-pipelines

## Purpose

Quick reference for all Tekton pipelines and tasks (~110+ reusable tasks, ~80+ pipelines) in proALPHA CCOE platform. Covers authoring, debugging, and task reuse.

## When to Use

- Creating or changing Tekton CI/CD pipelines
- Creating or updating reusable Tekton tasks
- Building, deploying, or releasing pipeline templates
- Debugging pipeline failures
- Identifying which task to reuse

---

## Overview

The pipelines and tasks live in:
```
/workspace/ccoe/general.shared.iac/k8s/shared_apps/tekton-pipelines/
├── base/
│   ├── tasks/         (~110+ reusable Tekton tasks)
│   ├── pipelines/     (~80+ pipeline definitions)
│   ├── trigger_bindings/
│   ├── misc/
│   └── scripts/auto_approve/
└── overlays/
    ├── dev/   (env: dev,  domain: shared-dev.proalpha.dev)
    └── prod/  (env: prod, domain: shared.proalpha.dev)
```

Deployment is managed via **Kustomize overlays** — `base/` holds all definitions, overlays apply environment-specific patches.

---

## Environments

| Overlay | Environment | Domain |
|---------|-------------|--------|
| `overlays/dev` | `dev` | `shared-dev.proalpha.dev` |
| `overlays/prod` | `prod` | `shared.proalpha.dev` |

---

## Pipelines

### Build Pipelines

| Pipeline | Description | Key Steps |
|----------|-------------|-----------|
| **build-dotnet-application** | Builds .NET applications; runs unit tests, Sonar analysis, quality gates; publishes to CodeArtifact | fetch-repo → get-versioning → get-parameters → codeartifact-login → sonar-dotnet → [quality-gate-release] → publish → [quality-gate-pre] → cleanup |
| **build-dotnet-lib** | Builds .NET libraries | Same pattern as build-dotnet-application |
| **build-dotnet-netstandard** | Builds .NET NetStandard libraries | Same pattern as build-dotnet-application |
| **build-dotnet-backend** | Builds .NET backend services | Same pattern as build-dotnet-application |
| **build-npm-package** | Builds NPM packages, runs tests, publishes to CodeArtifact | fetch-repo → get-versioning → get-parameters → codeartifact-login → npm-install → build → tests → sonar-scan → [quality-gate-release] → package → publish → [quality-gate-pre] → cleanup |
| **build-npm-package-storybook** | Builds NPM packages including Storybook | Same pattern as build-npm-package |
| **build-npm-image** | Builds Docker images from NPM packages | Same pattern as build-npm-package |
| **build-npm-application** | Builds NPM applications | Same pattern as build-npm-package |
| **build-lambda-nodejs** | Builds AWS Lambda functions (Node.js) | fetch-repo → get-versioning → get-parameters → build → package |
| **build-lambda-python** | Builds AWS Lambda functions (Python) | fetch-repo → get-versioning → get-parameters → build → package |
| **build-generic-image** | Builds generic Docker images using buildah | fetch-repo → get-versioning → get-parameters → buildah-nextgen → [trigger-deployment] |
| **build-generic-mf** | Builds micro-frontend Docker images | Similar to build-generic-image |
| **build-generic-image-high-resource** | High-resource Docker builds (for memory/CPU-intensive images) | Similar to build-generic-image |
| **build-terraform-module** | Builds and tests Terraform modules | fetch-repo → terraform-fmt → terraform-docs → [tflint] |
| **build-gradle-min** | Builds Gradle-based projects (minimal) | fetch-repo → build steps |
| **build-aps-optsrv** | Builds APS Option Server (specialized internal build) | Specialized steps |
| **lint-npm** | Lints NPM packages only (no publish) | npm-install → npm run lint |

**Key params for build-generic-image:**
`git-commit-id`, `git-repo-name`, `dockerfile`, `docker-repository`, `tag`, `context`

---

### Deploy Pipelines

| Pipeline | Description | Key Steps |
|----------|-------------|-----------|
| **deploy-noprod** | Deploys to non-production environments (dev/staging) | fetch-repo → argocd-sync → get-deploy-parameter → deploy-s3-artifacts → run-api-tests → run-e2e-tests → generate-release-notes → get-parameters-pr → open-pull-request → auto-merge |
| **deploy-test** | Deploys to test environments | Similar to deploy-noprod |
| **deploy-prod** | Deploys to production | Similar to deploy-noprod, stricter gates |

---

### Provisioning Pipelines

| Pipeline | Description | Key Params | Key Steps |
|----------|-------------|------------|-----------|
| **provision-tenant-env** | Provisions new tenant infrastructure | `tenant`, `env` (production/innovation/trial/demo/dev), `create-routes`, `snow_task_number`, `snow_customer_email`, `uptime_period`, `tenant_subdomain` | acquire-mutex → get-parameters → fetch-repo → run-route-provisioning → run-tenant-manager → create-pr → wait-for-plan → approve → apply → wait-until-merged → release-mutex |
| **provision-tenant-routes** | Provisions tenant DNS/routes | `tenant`, `tenant-zone`, `stage` | Route provisioning steps |
| **deprovision-tenant-env** | Deprovisions a tenant environment | `tenant`, `env` | Reverse of provision-tenant-env |
| **deprovision-tenant-routes** | Deprovisions tenant routes | `tenant` | Route teardown steps |

> **Note:** Provisioning pipelines use mutex locks to prevent concurrent operations on the same tenant/environment.

---

### Release Pipelines

| Pipeline | Description | Key Params | Key Steps |
|----------|-------------|------------|-----------|
| **release-platform-components** | Automated release for monorepos. Detects changed components, bumps versions via Conventional Commits, creates SemVer tags. | `git-commit-id`, `git-repo-name`, `dry-run` | fetch-repo → detect-changes → compute-releases → create-tags → [update-config] → release-summary |

---

### Automation Pipelines

| Pipeline | Description | Key Params | Key Steps |
|----------|-------------|------------|-----------|
| **auto-approve-platform-pr** | Auto-approves PRs based on a policy file. | `pr-key`, `pr-base`, `pr-author` | fetch-repo → check-policy → approve-pr → comment-pr |

---

## Tasks

### Git Operations

| Task | Description | Key Params | Results |
|------|-------------|------------|---------|
| **git-clone** | Clone a git repository (Bitbucket Server) | `url`, `display-id`, `commit-id`, `depth`, `subdirectory`, `deleteExisting` | — |
| **git-clone-cloud** | Clone from Bitbucket Cloud | `url`, `display-id`, `commit-id`, `depth`, `sslVerify`, `subdirectory`, `deleteExisting` | — |
| **git-clone-cloud2** | Enhanced cloud clone with refspec support | Same as git-clone-cloud | — |
| **git-cli** | Run arbitrary git commands | `GIT_USER_NAME`, `GIT_USER_EMAIL`, `GIT_SCRIPT`, `SUBDIR`, `CHANGES_ONLY` | `commit hash` |
| **versioning** | Determine version from git tags/branch | `path`, `versionForImage`, `versionForPackage`, `trigger-source`, `display-id` | `image_version`, `package_versionprefix`, `package_versionsuffix`, `versiontype` |

---

### Package Management

| Task | Description | Key Params | Results |
|------|-------------|------------|---------|
| **npm-install** | Install NPM dependencies | `path`, `version`, `version-suffix`, `node_image` | — |
| **npm-run** | Run an npm script | `path`, `command`, `node_image` | `exit-code` |
| **yarn-install** | Install Yarn dependencies | `path`, `version` | — |
| **yarn-run** | Run a yarn script | `path`, `command` | `exit-code` |
| **codeartifact-login** | Authenticate to AWS CodeArtifact | `CODEARTIFACT_DOMAIN`, `DOMAIN_OWNER`, `path`, `PRIVATE_REPO`, `MIRROR_REPO`, `REPO_FORMAT` | — |

---

### Build Tasks

| Task | Description | Key Params |
|------|-------------|------------|
| **buildah-nextgen** | Build container images with buildah | `FORMAT`, `TLSVERIFY`, `DOCKERFILE`, `IMAGE`, `TAG`, `CONTEXT`, `PLATFORMS` |
| **buildah-high-resource** | High-resource container builds | Same as buildah-nextgen |
| **dotnet-publish** | Publish .NET packages | `CONTEXT`, `VERSIONPREFIX`, `VERSIONSUFFIX`, `SOLUTION`, `DOTNET_SDK_IMAGE`, `PACKAGE_NAME`, `REPOSITORY` |
| **push-nuget-to-codeartifact** | Push NuGet packages to CodeArtifact | `path`, `version`, `nuget-source` |
| **lambda-python-build-package** | Build Python Lambda package | `path`, `function_name` |
| **lambda-nodejs-build-package** | Build Node.js Lambda package | `path`, `function_name` |
| **lambda-python-tests** | Run Python Lambda tests | `path` |
| **lambda-nodejs-tests** | Run Node.js Lambda tests | `path` |

---

### Testing & Quality

| Task | Description | Key Params | Results |
|------|-------------|------------|---------|
| **sonar-dotnet** | SonarQube static analysis for .NET | `PROJECT_KEY`, `GIT_BRANCH`, `PR_KEY`, `PR_BASE`, `CONTEXT`, `VERSION`, `SOLUTION`, `SONAR_DOTNET_IMAGE` | `taskid`, `exit-code` |
| **sonar-scan** | SonarQube scan for JS/TS | `PROJECT_KEY`, `GIT_BRANCH`, `PR_KEY`, `PR_BASE`, `VERSION`, `path` | `taskid` |
| **check-quality-gate** | Verify SonarQube quality gate result | `taskid`, `versiontype`, `unit-tests-result` | — |
| **run-e2e-tests** | Run Java-based E2E tests | `env`, `subdirectory`, `reports_folder` | — |
| **run-node-e2e-tests** | Run Node.js E2E tests | `env`, `subdirectory`, `reports_folder`, `npm_cache_folder` | — |
| **run-gradle-e2e-tests** | Run Gradle E2E tests | `env`, `subdirectory`, `reports_folder`, `pipeline_tag` | — |
| **run-karate-apitests** | Run Karate API tests | `env`, `subdirectory`, `reports_folder` | — |
| **run-karate-apitests-gradle** | Run Karate API tests (Gradle variant) | `env`, `subdirectory`, `reports_folder` | — |

---

### Deployment Tasks

| Task | Description | Key Params |
|------|-------------|------------|
| **argocd-task-sync-and-wait** | Sync an ArgoCD application and wait for health | `s3-bucket`, `argocd-project` |
| **aws-s3-bucket-sync** | Sync files between S3 buckets | `source_buckets`, `target_buckets`, `delete-unknown-files`, `excludes` |
| **trigger-deployment** | Trigger a downstream deployment pipeline | `target-project`, `target-repository`, `apps`, `git-hash`, `artifact`, `artifact-type`, `deployment-type` |

---

### Bitbucket Operations

| Task | Description | Key Params | Results |
|------|-------------|------------|---------|
| **bitbucket-create-pr** | Create a Pull Request | `SOURCE_BRANCH`, `TARGET_BRANCH`, `REPOSITORY_SLUG`, `WORKSPACE` | `pullrequest-id` |
| **bitbucket-merge-pr** | Merge a Pull Request | `PULLREQUEST_ID`, `REPOSITORY_SLUG` | — |
| **bitbucket-approve-pr** | Approve a Pull Request | `PULLREQUEST_ID`, `REPOSITORY_SLUG`, `WORKSPACE`, `COMMIT_ID` | — |
| **bitbucket-comment-pr** | Post a comment on a PR | `PULLREQUEST_ID`, `REPOSITORY_SLUG`, `WORKSPACE`, `TEXT` | — |
| **bitbucket-wait-for-pr** | Poll until a PR is merged | `PULLREQUEST_ID`, `REPOSITORY_SLUG`, `WORKSPACE` | — |
| **bitbucket-wait-for-builds** | Wait for all builds on a PR to complete | `PULLREQUEST_ID`, `REPOSITORY_SLUG`, `WORKSPACE`, `COMMIT_ID`, `BUILD_NAME` | — |
| **bitbucket-check-changes** | Check if files matching a filter changed in a PR | `REPOSITORY_SLUG`, `FILTER`, `SOURCE_BRANCH`, `TARGET_BRANCH` | `filter-hit` |
| **bitbucket-tag-commit** | Tag a specific commit in Bitbucket | `TAG`, `REPOSITORY_SLUG`, `WORKSPACE` | — |

---

### Infrastructure & Utilities

| Task | Description | Key Params |
|------|-------------|------------|
| **get-parameters-*** | Extract `.pa-sdlc-config` YAML configuration from a repository | Varies by variant |
| **mutex-acquire** | Acquire a distributed lock (prevent concurrent runs) | `label`, `type` |
| **mutex-release** | Release a distributed lock | `label`, `type` |
| **snow-notification** | Send a notification to ServiceNow | `tenant`, `snow_task_number`, `is_failure`, `is_end`, `message` |
| **aws-cli** | Run arbitrary AWS CLI commands | `script` |
| **kubectl-cli** | Run arbitrary kubectl commands | `script` |
| **terraform-fmt** | Format Terraform files (check or fix) | — |
| **terraform-docs** | Generate Terraform module documentation | — |

---

## Common Patterns

### 1. Standard Build Flow

```
fetch-repo
  → get-versioning          # determines image_version / package_version from git tags
  → get-parameters          # reads .pa-sdlc-config from repo
  → [codeartifact-login]    # for packages that need npm/nuget registry
  → build                   # language-specific compile/package step
  → test                    # unit tests
  → [sonar-scan]            # static analysis
  → quality-gate            # blocks on Sonar gate result
  → publish                 # push artifact to CodeArtifact or registry
  → cleanup
```

### 2. Standard Deploy Flow

```
fetch-repo
  → argocd-sync             # sync the ArgoCD application
  → deploy-artifacts        # sync S3 assets if needed
  → run-tests               # API / E2E tests
  → create-pr               # open config PR for environment promotion
  → merge                   # auto-merge or wait for approval
```

### 3. Provisioning Flow (Tenant)

```
acquire-mutex               # prevent concurrent tenant operations
  → fetch-repo
  → provision               # run-route-provisioning / run-tenant-manager
  → create-pr               # open Terraform plan PR
  → wait-for-plan           # CI runs terraform plan
  → approve
  → apply
  → wait-until-merged
  → release-mutex
```

### 4. Key Tekton Conventions

| Convention | Detail |
|------------|--------|
| **Workspace** | All pipelines use a shared workspace named `ws` for source code |
| **Result passing** | `$(tasks.<taskname>.results.<resultname>)` |
| **Conditional execution** | `when` clauses with `input`, `operator`, `values` |
| **Retry logic** | Most tasks default to `retries: 3` |
| **Quality gate branching** | `RELEASE` build type triggers `quality-gate-release`; `PRE` build type triggers `quality-gate-pre` |
| **Mutex locks** | Used by provisioning pipelines to serialize tenant/env operations |
| **Versioning** | `versioning` task produces `image_version`, `package_versionprefix`, `package_versionsuffix`, `versiontype` results consumed by downstream tasks |

### 5. Quality Gate Logic

```
versiontype == "release"  →  quality-gate-release  (strict: blocks on gate failure)
versiontype == "pre"      →  quality-gate-pre       (lenient: warns but continues)
```

### 6. Release Versioning (release-platform-components)

- Reads **Conventional Commits** since last tag
- `feat:` → minor bump, `fix:` → patch bump
- Tags format: `<component>-v<MAJOR>.<MINOR>.<PATCH>`
- `dry-run` param prevents actual tag creation

---

## Quick Lookup

### "I need to…" Reference

| Goal | Use this pipeline/task |
|------|------------------------|
| Build and publish a .NET library | `build-dotnet-lib` |
| Build and publish a .NET service | `build-dotnet-application` |
| Build and publish an NPM package | `build-npm-package` |
| Build a Docker image | `build-generic-image` or `build-npm-image` |
| Build a Lambda function | `build-lambda-nodejs` / `build-lambda-python` |
| Deploy to dev/staging | `deploy-noprod` |
| Deploy to production | `deploy-prod` |
| Provision a new tenant | `provision-tenant-env` |
| Provision tenant routes | `provision-tenant-routes` |
| Deprovision a tenant | `deprovision-tenant-env` |
| Run a Sonar scan | `sonar-scan` (JS/TS) or `sonar-dotnet` (.NET) |
| Check a Sonar quality gate | `check-quality-gate` |
| Clone a repo | `git-clone` (Server) or `git-clone-cloud` (Cloud) |
| Login to CodeArtifact | `codeartifact-login` |
| Open / merge a Bitbucket PR | `bitbucket-create-pr` + `bitbucket-merge-pr` |
| Run E2E tests | `run-e2e-tests` / `run-node-e2e-tests` / `run-gradle-e2e-tests` |
| Run API tests | `run-karate-apitests` / `run-karate-apitests-gradle` |
| Sync ArgoCD | `argocd-task-sync-and-wait` |
| Tag a commit | `bitbucket-tag-commit` |
| Auto-approve a PR | `auto-approve-platform-pr` |
| Run Terraform formatting | `terraform-fmt` |
| Generate Terraform docs | `terraform-docs` |
| Notify ServiceNow | `snow-notification` |
| Prevent concurrent pipeline runs | `mutex-acquire` + `mutex-release` |
