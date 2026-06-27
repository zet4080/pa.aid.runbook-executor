---
name: platform-components
description: Use when editing Terraform/Atmos platform component stacks, discovery tags, shared/normal stacks, or Cloud Posse context provider settings
---
# Skill: platform-components
# Skill: @ccoe/platform-components

## Purpose

Quick reference for Terraform/Atmos platform component monorepo (53 components): discovery tags, shared vs normal stacks, Cloudposse Context Provider, component consumption in consumer IaC repos.

## When to Use

- Editing Terraform/Atmos platform component stacks
- Adding discovery tags or discovery templates
- Configuring shared vs normal stacks
- Consuming platform components in consumer IaC repo

---

## Overview

This repository is a **Terraform/Atmos component monorepo** containing **53 self-contained Terraform wrappers** for AWS infrastructure. Each component wraps an upstream module (typically from a private Bitbucket registry) and adds standardized naming, discovery tags, and discovery templates.

Consumer repos (like `ai.foundation.iac`) declare Atmos stacks, import component configs, and pull each component at a pinned version from this repo via `source.uri`. **This skill primarily documents how to consume components** — the developer-facing component authoring guide lives in `docs/component-development-guide.md` inside the components repo.

### Key Principles

- **Self-contained components**: Components never pass outputs to each other. When component B needs a resource from component A, it discovers it via standardized `atmos_discovery_*` tags using a discovery template.
- **Atmos orchestration**: Infrastructure is deployed via [Atmos](https://atmos.tools/), a Terraform orchestration tool that provides stack-based configuration management.
- **Discovery pattern**: Resources are discovered using AWS Resource Groups Tagging API — not through `terraform_remote_state` or module outputs.
- **Cloudposse Context Provider**: All components use the [Cloudposse Context Provider](https://github.com/cloudposse/terraform-provider-context) for standardized naming and tagging.
- **No manual tagging**: Releases are fully automated via Tekton pipeline on merge to `main`.

---

## CONSUMER GUIDE

This section is the primary reference for teams that **use** platform components in their own IaC repo. The authoritative consumer repo example is `ai.foundation.iac`.

### Consumer Model Overview

```
platform-components repo (git tags: vpc-v1.2.3, eks-v1.4.0, ...)
    ↑ source.uri + version
Consumer IaC repo (e.g. ai.foundation.iac)
  stacks/
    dev/core/components/vpc.yaml   ← declares source + vars
    dev/core/core_stack.yaml       ← imports component files
  atmos.yaml                       ← Atmos configuration
```

Each component in the consumer repo declares:
1. A `source.uri` pointing to the platform-components Git repo at a pinned version
2. The component's `vars` (networking CIDRs, flags, etc.)
3. A `provision.workdir.enabled: true` block

Context variables (`project`, `environment`, `account_id`, `region`) flow in from shared stack config files, not from per-component files.

### Consumer Repo Layout

```
stacks/
  backend.yaml              # S3 backend + AWS provider config (imported by every stack)
  project_settings.yaml     # settings.project + project_repository (imported by every stack)
  <env>/                    # e.g. dev, stg, prod
    env_config.yaml         # environment-wide settings + vars
    stack_ranges.yaml       # (optional) shared ALB priority ranges imported by non-core stacks
    <stack>/                # e.g. core, management, gateway, mcp-services, buddy
      <stack>_stack.yaml    # imports backend, project_settings, ../env_config, + one import per component
      components/
        <component>.yaml    # one file per component: source + vars
atmos.yaml                  # Atmos configuration (stacks base_path, components base_path, workflows, auth)
workflows/
  <stack>-<env>.yaml        # generated pull/plan/deploy/destroy workflow per stack+environment
platform.yaml               # pmt source-of-truth: environments, stacks, components, auth settings
```

### Key Config Files

**`stacks/backend.yaml`** — imported by every stack; defines S3 backend and AWS provider with assume-role:

```yaml
terraform:
  backend_type: s3
  backend:
    s3:
      bucket: "{{ .settings.backend_bucket }}"
      key: "atmos_terraform_state/{{ .atmos_stack }}/{{ .atmos_component }}.tfstate"
      region: "{{ .vars.region }}"
      encrypt: true
      kms_key_id: "arn:aws:kms:{{ .vars.region }}:{{ .vars.account_id }}:alias/aws/s3"
      use_lockfile: true
      assume_role:
        role_arn: "arn:aws:iam::{{ .vars.account_id }}:role/terraform-master-role"
  providers:
    aws:
      region: "{{ .vars.region }}"
      assume_role:
        role_arn: "arn:aws:iam::{{ .vars.account_id }}:role/terraform-master-role"
```

**`stacks/project_settings.yaml`** — imported by every stack; sets project identifier:

```yaml
settings:
  project: "aif"
  project_repository: "git@bitbucket.org:proalpha/ai.foundation.iac.git"
```

**`stacks/<env>/env_config.yaml`** — environment-wide settings, imported by every stack in that env:

```yaml
settings:
  backend_bucket: tf-state-ai-dev
  config_bucket_name: tf-state-ai-dev
  default_alb_target_group: traefik
  route53_zone_name: ai-dev.proalpha.dev
vars:
  account_id: "375353319064"
  environment: dev
  project: aif
  region: eu-central-1
```

**`stacks/<env>/<stack>/<stack>_stack.yaml`** — one import per component, plus `atmos_stack` and `stack_short_name`:

```yaml
import:
  - backend
  - project_settings
  - ../env_config
  - ./components/vpc
  - ./components/eks
  - ./components/alb
  # ...one import per component...

vars:
  atmos_stack: core-dev       # globally unique Atmos stack selector
  stack_short_name: core      # short label injected into non-shared resource names

terraform:
  auth:
    identities:
      dev-prj-engineer-admin:
        default: true

settings:
  atlantis:
    config_template_name: atlantis-config
    project_template:
      name: "st1-{component}-dev"
      workspace: default
      dir: ".workdir/terraform/core-dev-{component}"
      terraform_version: v1.5.0
      workflow: atmos-terraform
      apply_requirements:
        - mergeable
        - approved
      autoplan:
        enabled: true
        when_modified:
          - ../../../stacks/dev/core/components/{component}.yaml
```

**`stacks/<env>/<stack>/components/<component>.yaml`** — one per component; sets `source` + `vars`:

```yaml
components:
  terraform:
    vpc:
      source:
        uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/vpc"
        version: "main"      # pin to a tag like "vpc-v1.2.3" for production
      vars:
        vpc_cidr: "10.10.120.0/21"
        private_subnets:
          - "10.10.122.0/23"
          - "10.10.124.0/23"
          - "10.10.126.0/23"
        public_subnets:
          - "10.10.120.0/25"
          - "10.10.120.128/25"
          - "10.10.121.0/25"
        single_nat_gateway: false
      provision:
        workdir:
          enabled: true
```

> **Version pinning**: Use `version: "main"` for active development. For production stacks, pin to a released tag like `version: "vpc-v1.2.3"`. Tags are in the format `<component>-v<MAJOR>.<MINOR>.<PATCH>`.

### Real Consumer Examples

**`eks.yaml`** — EKS with Karpenter and a custom nodegroup:

```yaml
components:
  terraform:
    eks:
      source:
        uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/eks"
        version: "main"
      vars:
        cluster_version: "1.34"
        name: aif-dev
        enable_karpenter: true
        nodegroup_defaults_preset: default
        nodegroups_presets:
          ng_default: default
        nodegroups_map:
          ng_default:
            capacity_type: ON_DEMAND
            instance_types:
              - t3a.medium
              - t3.medium
              - t2.medium
            disk_size: 50
            min_capacity: 1
            max_capacity: 1
            bootstrap_extra_args: '"max-pods"=30'
            subnet_cidrs:
              - 10.10.122.0/23
              - 10.10.124.0/23
              - 10.10.126.0/23
            taints:
              - key: CriticalAddonsOnly
                value: "true"
                effect: NoSchedule
      provision:
        workdir:
          enabled: true
```

**`alb.yaml`** — ALB with named target groups, priority ranges, and EKS integration:

```yaml
components:
  terraform:
    alb:
      source:
        uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/alb"
        version: "main"
      vars:
        name: alb-ingress
        default_route_zone: "ai-dev.proalpha.dev"
        default_target_group: "yarp"
        enable_eks_integration: true
        hosts:
          - zone: ai-dev.proalpha.dev
            host: argocd.ai-dev.proalpha.dev
          - zone: ai-dev.proalpha.dev
            host: mcp.ai-dev.proalpha.dev
        rules:
          - host: argocd.ai-dev.proalpha.dev
            target_group: yarp
          - host: mcp.ai-dev.proalpha.dev
            target_group: yarp
        priority_ranges:
          core-dev: 1000-2999
          mcp-services-dev: 3000-3999
          management-dev: 4000-4999
          buddy-dev: 5000-5999
          gateway-dev: 6000-6999
        subnet_name_filters:
          - "*public*"
        target_groups:
          traefik:
            protocol: HTTP
            port: 32080
          yarp:
            protocol: HTTP
            port: 32082
          yarp-ssl:
            protocol: HTTPS
            port: 32443
      provision:
        workdir:
          enabled: true
```

**`dns.yaml`** — DNS zone with an ALB alias record:

```yaml
components:
  terraform:
    dns:
      source:
        uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/dns"
        version: "main"
      vars:
        zone_name: ai-dev.proalpha.dev
        records:
          - name: management
            type: A
            alias: true
            alb_name: alb-ingress
      provision:
        workdir:
          enabled: true
```

**`irsa-roles.yaml`** — IRSA roles with an inline policy document (gateway stack):

```yaml
components:
  terraform:
    irsa-roles:
      source:
        uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/irsa-roles"
        version: "main"
      vars:
        roles:
          - name: runtime-ingress-gateway
            role_name: runtime-ingress-gateway
            service_account: runtime-ingress-gateway
            namespace_patterns:
              - "dev-gateway*"
            policy_document: |
              {
                "Version": "2012-10-17",
                "Statement": [
                  {
                    "Effect": "Allow",
                    "Action": ["dynamodb:Query", "dynamodb:GetItem"],
                    "Resource": "arn:aws:dynamodb:eu-central-1:123456789012:table/my-table-*"
                  }
                ]
              }
      provision:
        workdir:
          enabled: true
```

### Shared vs Normal Stacks — The Consumer View

In the `ai.foundation.iac` repo, the **`core` stack** is the shared/foundation stack: it holds `vpc`, `eks`, `alb`, `karpenter`, `argocd`, `cert-manager`, `yarp-ingress-controller`, `waf`, `dns`, `dynamodb`, `masstransit`, `tokenvendor`, and cluster extensions. It sets `shared=true` (the default) on all those components, meaning their names have no stack label (`aif-dev`, not `aif-dev-core`).

Non-core stacks (`management`, `gateway`, `mcp-services`, `buddy`) are **normal stacks** — they contain project-scoped components and use `shared_vpc: true`, `shared_eks: true`, `shared_alb: true` to discover the shared infrastructure rather than creating new VPCs/clusters.

| Stack | `atmos_stack` | `stack_short_name` | Role |
|-------|--------------|-------------------|------|
| `core` | `core-dev` | `core` | Shared foundation: VPC, EKS, ALB, ArgoCD, Karpenter |
| `management` | `management-dev` | `management` | Project resources: buckets, SQS, IRSA, s3-secret |
| `gateway` | `gateway-dev` | `gateway` | API gateway layer: alb-rules, irsa-roles |
| `mcp-services` | `mcp-services-dev` | `mcp-services` | MCP-specific services |
| `buddy` | `buddy-dev` | `buddy` | Buddy-specific resources (e.g. `location`) |

The `stack_ranges.yaml` file in `stacks/dev/` provides a shared `priority_ranges` map imported by non-core stacks so each stack gets a non-overlapping ALB listener-rule priority band:

```yaml
# stacks/dev/stack_ranges.yaml
components:
  terraform:
    alb-rules:
      vars:
        priority_ranges:
          buddy-dev: 1000-1999
          core-dev: 2000-2999
          management-dev: 3000-3999
          mcp-services-dev: 4000-4999
          gateway-dev: 5000-5999
```

### Discovery / Shared-Flag Mechanics — Consumer View

When a normal-stack component (e.g. `irsa-roles` in `gateway`) needs to find the shared EKS cluster, it uses the `shared_eks` variable. Setting `shared_eks: true` tells the embedded `eks_data.tf` discovery template to look up the EKS cluster without a stack label — i.e., the shared cluster. Setting it `false` would look up an EKS cluster scoped to the current `stack_short_name`.

The same mechanism applies to `shared_vpc`, `shared_alb`, `shared_aurora`, `shared_waf`.

**Example:** the `irsa-roles` component in `gateway-dev` uses `shared_eks: true` (the default) so it discovers the EKS cluster created by `core-dev`. No explicit declaration needed — `shared_eks` defaults to `true` in most components.

**Example with explicit override:** a component that needs to scope its database to the current stack:

```yaml
vars:
  shared_aurora: false    # use the aurora cluster scoped to THIS stack, not the shared one
  stack_short_name: mystack
```

### Step-by-Step: Add a New Component to an Existing Stack

1. **Create the component file** at `stacks/<env>/<stack>/components/<component>.yaml`:

   ```yaml
   components:
     terraform:
       <component>:
         source:
           uri: "git::ssh://git@bitbucket.org/proalpha/platform-components.git//components/<component>"
           version: "main"   # pin to a tag for production
         vars:
           # component-specific variables — run `pmt man <component>` for reference
         provision:
           workdir:
             enabled: true
   ```

2. **Import it in the stack file** — add one line to the `import:` list in `<stack>_stack.yaml`:

   ```yaml
   import:
     - ...existing imports...
     - ./components/<component>
   ```

3. **Validate** that Atmos sees the new component:

   ```bash
   atmos describe stacks
   ```

4. **Plan** the component:

   ```bash
   atmos terraform plan <component> -s <stack>-<env>
   ```

5. **Apply** when ready:

   ```bash
   atmos terraform deploy <component> -s <stack>-<env>
   ```

6. **Regenerate workflows** if using `pmt`:

   ```bash
   pmt refresh
   ```

### Step-by-Step: Add a New Stack

#### Option A — Using `pmt` (recommended for new projects)

1. Edit `platform.yaml` to add the new stack under the target environment(s):

   ```yaml
   environments:
     - name: dev
       stacks:
         - short_name: my-new-stack
           stack_template: pm-base
           default_version: main
           atlantis_enabled: true
           components:
             - vpc
             - irsa-roles
   ```

2. Run `pmt refresh` to generate all stack/component/workflow files:

   ```bash
   pmt refresh
   ```

3. Fill in any `<PLEASE CUSTOMIZE>` placeholders in the generated component files.

4. Run `atmos describe stacks` to validate.

#### Option B — Manual file creation

1. Create the stack directory: `stacks/<env>/<stack>/components/`

2. Create `stacks/<env>/<stack>/<stack>_stack.yaml` following the anatomy above — import `backend`, `project_settings`, `../env_config`, and one entry per component file. Set `atmos_stack` and `stack_short_name` in `vars:`.

3. Create one `components/<component>.yaml` per component with `source.uri`, `vars`, and `provision.workdir.enabled: true`.

4. If your stack will add ALB listener rules, also import `../stack_ranges` to pick up the shared priority-range allocations, or define your own `priority_ranges` block.

5. Generate or manually create a workflow file at `workflows/<stack>-<env>.yaml` (see existing workflow files for the `pull/plan/deploy/destroy` pattern).

### Plan / Apply / Workflow Commands

```bash
# Plan a single component in a stack
atmos terraform plan <component> -s <stack>-<env>

# Deploy (apply) a single component
atmos terraform deploy <component> -s <stack>-<env>

# Destroy a single component
atmos terraform destroy <component> -s <stack>-<env>

# Run a full workflow (plan all components in a stack in dependency order)
atmos workflow plan -f <stack>-<env>

# Run a full deploy workflow
atmos workflow deploy -f <stack>-<env>

# Run a full destroy workflow (reverse order)
atmos workflow destroy -f <stack>-<env>

# Describe all stacks (validate configuration)
atmos describe stacks

# Describe all components
atmos describe components

# Authenticate via browser-based SSO
atmos auth login

# Show available identities/sessions
atmos auth list
```

> **Note on `deploy` vs `apply`**: The consumer repo uses `atmos terraform deploy` (not `apply`) as shown in the generated workflow files.

### `pmt` CLI Reference

`pmt` (pm_tool) is a Go CLI that generates Atmos stack files from `platform.yaml`. Key commands:

| Command | Purpose |
|---------|---------|
| `pmt init` | Create initial `platform.yaml` with `<PLEASE_CUSTOMIZE>` placeholders |
| `pmt --dry-run refresh` | Preview generated files without writing |
| `pmt refresh` | Generate/regenerate all stack, component, and workflow files |
| `pmt enable_deploy` | Generate deploy configuration for Atmos SSO |
| `pmt man <component>` | Read component documentation (e.g., `pmt man vpc`) |
| `pmt man <component> <topic>` | Read sub-topic (e.g., `pmt man alb priority_ranges`) |
| `pmt list component-templates` | List available component templates |
| `pmt list stack-templates` | List available stack templates |
| `pmt list environment-templates` | List available environment templates |
| `pmt environment add ...` | Add an environment via CLI |
| `pmt stack add ...` | Add a stack to environments via CLI |
| `pmt stack remove ...` | Remove a stack from environments |
| `pmt stack component add ...` | Add a component to a stack via CLI |

**Typical new-project workflow:**

```bash
# 1. Initialize platform.yaml
pmt init

# 2. Edit platform.yaml — set project, environments, stacks, components

# 3. Preview (optional)
pmt --dry-run refresh

# 4. Generate all files
pmt refresh

# 5. Fill in <PLEASE CUSTOMIZE> placeholders
grep -r 'PLEASE CUSTOMIZE' stacks/

# 6. Validate Atmos sees all stacks
atmos describe stacks

# 7. Authenticate and deploy
pmt enable_deploy
atmos auth login
atmos workflow deploy -f <stack>-<env>
```

---

## Architecture Concepts

### Stack Types

There are two types of stacks in this architecture:

| Type | `var.shared` | Stack label in name | Example Name | Usage |
|------|-------------|---------------------|--------------|-------|
| **Shared** | `true` | `""` (empty) | `aif-dev` | One per environment — VPC, EKS, shared ALB |
| **Normal** | `false` | Normalized `stack_short_name` | `aif-dev-gateway` | Project/stack-specific resources |

**Shared stacks** contain resources shared across multiple projects within an environment (e.g., the VPC, EKS cluster, shared ALB). There is exactly one shared stack per environment.

**Normal stacks** contain project-specific resources. Multiple normal stacks can exist per environment. When `shared=false`, `stack_short_name` **must** be provided — this is enforced by a `check` block.

### Discovery Pattern

Components discover each other's resources through:

1. **Discovery tags** — Standardized `atmos_discovery_*` tags applied to all resources during creation
2. **Discovery templates** — Terraform `.tmpl` files that look up resources by these tags via AWS Resource Groups Tagging API
3. **Template distribution** — Source `.tmpl` files are distributed to consumer components as `*_data.tf` files via the `scripts/template_distribution/` tooling

```
Component A → Creates resources with atmos_discovery_* tags
     ↓
Component B → Copies A's discovery template → Queries tags → Receives local.* values
```

**Key benefit**: Components are loosely coupled. There are no hard wiring between stacks; any component can discover any other by stack context.

> **CRITICAL**: `*_data.tf` files in consumer components are **generated** — they are copies of the source `.tmpl`. **Never edit `*_data.tf` in-place.** Always edit the source `.tmpl` and redistribute.

### Context Provider Naming

All components use the Cloudposse Context Provider with standardized settings:

```hcl
provider "context" {
  alias          = "<component_name>"
  enabled        = var.enabled
  delimiter      = "-"
  property_order = ["namespace", "project", "environment", "stack"]
  tags_key_case  = "lower"
  tags_value_case = "none"
  properties = {
    namespace   = { include_in_tags = true }
    project     = { include_in_tags = true }
    environment = { include_in_tags = true }
    stack       = { include_in_tags = true }
  }
  values = {
    namespace   = local.effective_namespace
    project     = var.project
    environment = var.environment
    stack       = local.<component>_stack
  }
}
```

Generated names follow the pattern: `namespace-project-environment[-stack]`

- AWS 63-character limit is enforced via a `postcondition` in `*_context.tf`
- `stack_short_name` max 12 chars, `[a-z0-9-]` only

### Discovery Tags

Every resource receives these tags for discovery by other components:

```hcl
atmos_discovery_namespace     = var.namespace
atmos_discovery_project       = var.project
atmos_discovery_environment   = var.environment
atmos_discovery_atmos_stack   = "<stack_short_name> or 'shared'"
atmos_discovery_resource_type = "<component_name>"
atmos_discovery_resource_key  = "default"
```

### module_tags vs tags

- **`module_tags`** — does **not** include a `Name` tag → passed to upstream modules
- **`tags`** — includes a `Name` tag → used on direct resource declarations

Do **not** swap these. Using `tags` (with `Name`) on upstream modules causes conflicts.

---

## Repository Layout

```
components/              # 53 Terraform component wrappers
tf-modules/
  platform-constants/    # Shared Terraform module (NOT a component, NOT released independently)
template-repos/
  core/                  # Atmos stack/environment skeletons for consumer repos
  wizards/               # Additional wizard templates
scripts/
  template_distribution/ # Copies *_discovery/*.tmpl → consumer *_data.tf files
  migration/             # Python tooling for Terragrunt → Atmos migration
  pre_migration/         # Pre-migration Python tooling
releases/
  release_<N>.yaml       # Coordinated major-release triggers
docs/
  component-development-guide.md  # Full anatomy + creation guide (829 lines)
  discovery-templates.md          # Discovery pattern deep-dive (1043 lines)
  release-process.md              # Release process details (508 lines)
  index.md
.release-config.yaml     # platform_version + non_releasable_patterns
renovate.json            # Renovate dependency update configuration
AGENTS.md                # Non-obvious conventions for AI agents
```

### Non-Releasable Patterns

The following paths do **not** trigger a component release when changed:

```yaml
non_releasable_patterns:
  - "*.md"
  - "docs/**"
  - ".gitignore"
  - ".github/**"
  - ".release-config.yaml"
  - "scripts/**"
  - "template-repos/**"
  - "tf-modules/**"
  - "releases/**"
  - "catalog-info.yaml"
  - "mkdocs.yml"
```

---

## Component Catalog

The repository contains **53 components**. Below is the complete catalog organized by category.

### Core Infrastructure (Foundation)

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **vpc** | Network foundation: VPC, subnets, routing, NAT, optional firewall. Discovery source for nearly all other components. | `vpc_cidr`, `private_subnets`, `public_subnets`, `single_nat_gateway`, `enable_nat_gateway`, `firewall_enable` |
| **eks** | EKS control plane and managed nodegroups. Depends on VPC discovery. | `cluster_version`, `nodegroups_presets`, `nodegroups_map`, `nodegroup_defaults_preset`, `enable_karpenter`, `enable_public_api_access`, `iam_roles` |
| **alb** | Application Load Balancer with routing structures, target groups, default routes. Depends on VPC; optionally EKS. | `target_groups`, `hosts`, `rules`, `default_route_zone`, `enable_eks_integration`, `alb_extension`, `priority_ranges` |
| **aurora** | Aurora PostgreSQL cluster. Depends on VPC discovery. | `engine_version`, `instance_type`, `autoscaling_enable`, `subnet_name_filters`, `storage_encrypted` |
| **kms** | KMS key management. Creates named KMS keys (encrypt/decrypt or sign/verify) with standardized aliases and discovery tags. | `keys` (map of key configs: `key_usage`, `key_spec`, `enable_key_rotation`, `deletion_window_in_days`, `policy`) |
| **rds** | RDS (non-Aurora) instance management. Supports postgres, mysql, mariadb, oracle, sqlserver. Has its own discovery template. | `engine`, `engine_version`, `instance_class`, `allocated_storage`, `multi_az`, `deletion_protection`, `parameters` |

### Kubernetes Add-ons

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **karpenter** | Karpenter-based node autoscaling for EKS. Depends on EKS discovery. | `replicas`, `node_classes_presets`, `node_pools_presets`, `custom_tags` |
| **argocd** | Argo CD control plane for GitOps. Depends on EKS discovery. | `url`, `config_bucket_name`, `git_repos`, `oidc` |
| **argocd-projects** | Argo CD project definitions. | `projects`, `cluster_config` |
| **argocd-stack-integration** | Argo CD stack integration. | `stack_name`, `app_path` |
| **cluster-autoscaler** | Kubernetes cluster autoscaler (legacy; prefer Karpenter). | `eks_cluster_name` |
| **metrics-server** | Kubernetes Metrics Server. | N/A |
| **kubernetes-dashboard** | Kubernetes Dashboard deployment. | N/A |
| **traefik-ingress** | Traefik ingress controller on EKS. Depends on EKS + ALB discovery. | N/A |
| **ingress-controller** | Generic ingress controller. | `ingress_class`, `replica_count` |
| **yarp-ingress-controller** | YARP ingress controller for .NET reverse proxy. | N/A |
| **sealed-secrets** | Bitnami Sealed Secrets controller. Uses tenant/stage context pattern. | `secret_name` |
| **sealed-secrets-k8s** | Sealed Secrets Kubernetes resources. | N/A |
| **kubecost** | Kubecost cost monitoring. | `cluster_name` |
| **cert-manager** | TLS certificate management (cert-manager). | `issuer`, `acme_server` |
| **eks-cluster-extensions** | EKS cluster extensions (e.g., EBS CSI driver). | `extensions` |
| **irsa-roles** | IAM Roles for Service Accounts (IRSA). Creates IAM roles bound to Kubernetes ServiceAccounts. Depends on EKS discovery. | `roles` (list of role defs: `name`, `service_account`, `namespace_patterns`, + `preset` OR `policy_document`) |
| **s3-secret** | Creates Kubernetes secrets from S3 files. Uses tenant/stage context + `shared_eks` for EKS discovery. | `secrets`, `shared_eks` |
| **tokenvendor** | Deploys tokenvendor service via ArgoCD Application on EKS. Creates IAM role with EKS Pod Identity + inline policy. Depends on EKS discovery. | `image_tag`, `cognito_pool_url`, `allowed_modules`, `chart_version`, `iam_role_name` |

### Networking & DNS

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **dns** | DNS zone management. Uses tenant/stage context pattern. Depends on VPC discovery. | `zone_name`, `records` |
| **route53** | Route53 record management. | `zone_name`, `records` |
| **privatelink-provider** | AWS PrivateLink endpoint service provider. Depends on VPC. | `vpc_id`, `subnet_ids` |
| **privatelink-consumer** | AWS PrivateLink consumer endpoint. Depends on VPC. | `service_name`, `vpc_id` |

### Database & Storage

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **aurora-db-users** | Manages database users for Aurora clusters. | `database_name`, `users` |
| **dynamodb** | DynamoDB table management. Uses tenant/stage context pattern. | `table_name`, `hash_key`, `range_key` |
| **buckets** | S3 bucket management. Uses tenant/stage context pattern. | `buckets` (map of bucket configs: `versioning`, `account_ids`, `role_ids`, `grant_write_permissions`) |
| **backup** | AWS Backup vaults and plans. | `vault_name`, `backup_plan` |

### ALB Extensions

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **alb-ec2-target** | EC2 target registration for ALB target groups. Has its own `tg_data.tf.tmpl` discovery template. | `target_group_arn`, `instance_id` |
| **alb-listener-rules** | Additional ALB listener rules. | `listener_arn`, `priority`, `rules` |
| **waf** | AWS WAF web ACL attached to ALB. Depends on ALB discovery. Has its own `waf_data.tf.tmpl` discovery template. | `rules`, `scope`, `alb_arn` |

### Monitoring & Logging

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **coralogix** | Coralogix logging integration. | `integration_type`, `application_name` |
| **monitoring-alerts** | CloudWatch alerting rules. | `alarm_name`, `metric_name`, `threshold` |
| **monitoring-coralogix** | Coralogix monitoring integration on EKS. | `application_name`, `subsystem_name` |
| **monitoring-dashboards** | CloudWatch dashboards. | `dashboard_name`, `widgets` |
| **monitoring-health-checks** | Route53 health checks. | `fqdn`, `type`, `port` |
| **monitoring-logs-pipeline** | Centralized logging pipeline. | `log_destination`, `filter_pattern` |
| **cost-dashboards** | Cost Explorer dashboards. | `time_period` |
| **cur** | Cost and Usage Report configuration. | `report_name`, `time_unit` |

### Application & Eventing

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **apigateway** | API Gateway REST API. | `rest_api_name`, `stage_name`, `route53_record` |
| **eventbus** | EventBridge event bus. | `event_bus_name`, `policy` |
| **sqs** | SQS queue management. | `queue_name`, `dead_letter_queue` |
| **masstransit** | MassTransit message broker integration. Uses tenant/stage context pattern. | `broker_type`, `connection_string` |
| **feature-flags** | Feature flag service. | `provider`, `instance_type` |

### Miscellaneous / Platform

| Component | Purpose | Key Variables |
|-----------|---------|---------------|
| **app-bootstrap** | Application bootstrap resources. | `bootstrap_script` |
| **cron-action** | Scheduled EventBridge rules. | `schedule`, `target` |
| **location** | AWS Location Service (maps, place index). | `map_name`, `place_index_name` |
| **noop-timer** | No-op timer (testing/placeholder). | N/A |
| **onlineseal** | Online Seal service for document signing. | `seal_url` |

---

## Standard Component Structure

Every component follows a consistent file layout:

```
components/<name>/
├── variables.tf                    # All variables: context + discovery + component-specific
├── versions.tf                     # Terraform and provider version constraints
├── <name>_context.tf               # Context provider, naming, discovery tags, validation
├── <name>_resources.tf  OR         # Main module call (both naming variants are in use)
│   <name>_module.tf    OR          # (some older components also use main.tf)
│   main.tf
├── outputs.tf                      # Outputs (typically disabled — discovery replaces them)
├── platform-constants.tf           # Optional: shared platform constants module
├── <name>_locals.tf                # Optional: complex computed locals
├── <dep>_data.tf                   # Consumed discovery templates (e.g., vpc_data.tf)
├── eks_discovery_control.tf        # Optional: EKS discovery control (conditional EKS usage)
├── <name>_discovery/               # Discovery template for OTHER components to consume
│   └── <name>_data.tf.tmpl         # Template file (source of truth)
├── presets/                        # Optional: YAML preset files
│   └── <category>/*.yaml
├── pm_tool/                        # PM tool integration
│   ├── pm_tool_resources.yaml
│   ├── component-templates/
│   │   ├── component.yaml          # Atmos component YAML template
│   │   └── metadata.yaml           # Component metadata and parameters
│   └── man-pages/
│       ├── main.man                # Primary help page
│       └── <topic>.man             # Additional topic pages
├── migration/                      # Migration support files
│   ├── state_mappings.yaml
│   └── variables_extraction_defs.yaml
└── examples/                       # Optional: example configurations
```

### File Naming Quick Reference

| File | Naming Pattern | Purpose |
|------|---------------|---------|
| Context | `<name>_context.tf` | Naming, tagging, validation for THIS component |
| Resources | `<name>_resources.tf` OR `<name>_module.tf` OR `main.tf` | Main module invocation |
| Locals | `<name>_locals.tf` | Complex local computations |
| Consumed template | `<dep>_data.tf` | Discovery template **copied from** another component — generated, do not edit |
| Source template | `<name>_discovery/<name>_data.tf.tmpl` | Discovery template for other components — edit this one |

---

## Context Variable Patterns

There are **two distinct context variable patterns** in use across the 53 components.

### Standard Pattern

Used by most components: `vpc`, `eks`, `alb`, `aurora`, `rds`, `kms`, `irsa-roles`, `karpenter`, `argocd`, `tokenvendor`, and most others.

```hcl
variable "namespace" {
  type        = string
  description = "Namespace (organization name). Can be empty string to minimize name length."
  default     = ""
}

variable "project" {
  type        = string
  description = "Project name"
}

variable "environment" {
  type        = string
  description = "Environment name (e.g., dev, staging, prod, sandbox)"
}

variable "atmos_stack" {
  type        = string
  description = "Atmos stack name. Normalized: underscores → hyphens"
  default     = null
}

variable "stack_short_name" {
  type        = string
  description = "Short stack name (max 12 chars, [a-z0-9-] only). Required when shared=false."
  default     = null
  validation {
    condition     = var.stack_short_name == null || (length(var.stack_short_name) <= 12 && length(var.stack_short_name) > 0)
    error_message = "stack_short_name must be between 1 and 12 characters."
  }
  validation {
    condition     = var.stack_short_name == null || can(regex("^[a-z0-9-]+$", var.stack_short_name))
    error_message = "stack_short_name must contain only lowercase letters (a-z), numbers (0-9), and hyphens (-)."
  }
}

variable "enabled" {
  type        = bool
  description = "Enable/disable this component"
  default     = true
}

variable "shared" {
  type        = bool
  description = "Whether this component is shared (no stack label) or stack-scoped"
  default     = true
}

variable "name" {
  type        = string
  description = "Optional explicit name override. When set, context-based name generation is skipped. Keep in *_context.tf for migration compatibility."
  default     = null
}
```

### Tenant/Stage Pattern

Used by a subset of components: `buckets`, `s3-secret`, `dynamodb`, `dns`, `masstransit`, `sealed-secrets` (legacy/different context model).

```hcl
variable "namespace" {
  type    = string
  default = ""
}

variable "tenant" {
  type    = string
  default = null
}

variable "environment" {
  type    = string
  default = null
}

variable "stage" {
  type    = string
  default = null
}

variable "project" {
  type    = string
  default = null
}

variable "region" {
  type    = string
  default = null
}

variable "account_id" {
  type    = string
  default = null
}

variable "atmos_stack" {
  type    = string
  default = null
}

variable "stack_short_name" {
  type    = string
  default = null
}

variable "enabled" {
  type    = bool
  default = true
}

variable "shared" {
  type    = bool
  default = true
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

---

## Key Component Variables

### vpc

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `vpc_cidr` | string | (required) | CIDR block for the VPC |
| `private_subnets` | list(string) | (required) | Private subnet CIDR blocks (one per AZ) |
| `public_subnets` | list(string) | (required) | Public subnet CIDR blocks (one per AZ) |
| `single_nat_gateway` | bool | `false` | Use a single NAT gateway (vs one per AZ) |
| `enable_nat_gateway` | bool | `true` | Create NAT gateway(s) |
| `firewall_enable` | bool | `false` | Enable AWS Network Firewall |

### eks

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `cluster_version` | string | `"1.34"` | Kubernetes version |
| `nodegroup_defaults_preset` | string | `"default"` | Preset name for nodegroup defaults |
| `nodegroups_presets` | map(string) | `{}` | Map of nodegroup name → preset name |
| `nodegroups_map` | map(any) | `{}` | Per-nodegroup overrides (merged with preset) |
| `nodegroup_defaults_override` | map(any) | `{}` | Global override applied to all nodegroups |
| `add_default_nodegroup` | bool | `true` | Create a default nodegroup automatically |
| `enable_public_api_access` | bool | `true` | Expose the EKS API server publicly |
| `enable_ssm_node_access` | bool | `true` | Enable SSM Session Manager on nodes |
| `enable_karpenter` | bool | `false` | Install Karpenter on the cluster |
| `filter_cluster_traffic` | bool | `false` | Enable cluster traffic filtering |
| `iam_roles` | list(any) | `[]` | Additional IAM roles to bind to the cluster |
| `eks_auto_discover_sso_roles` | bool | `true` | Auto-discover SSO roles for cluster access |
| `sso_role_patterns` | map(object) | (see below) | SSO role patterns to auto-discover — map keyed by role type (`admin`, `engineer`, `reader`), each with `name_regex`, `path_prefix`, `username`, `access_policy` |

### alb

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `target_groups` | map(object) | `{}` | Target group definitions (`protocol`, `port`) |
| `default_target_group` | string | `"traefik"` | Fallback target group name when `target_groups` is empty |
| `hosts` | list(object) | `[]` | List of hostnames for listener rules (`host`, `zone`, `balancer_record`, `is_private_zone`) |
| `rules` | list(object) | `[]` | Custom listener rules |
| `default_route_zone` | string | `""` | Route53 zone for default routes |
| `enable_default_routes` | bool | `false` | Enable all preset platform routes (argocd, pgadmin, etc.) |
| `default_route_services` | map(bool) | `{}` | Explicit enable/disable map per service key |
| `enable_eks_integration` | bool | `false` | Wire ALB to EKS |
| `alb_extension` | string | `""` | Suffix to allow multiple ALBs per environment |
| `create_lb` | bool | `true` | Whether to actually create the LB resource |
| `priority_ranges` | map(string) | `{}` | Priority range allocations per stack (format: `"start-end"`) |
| `lambda_targets` | list(object) | `null` | Lambda function targets |
| `sg_ingress_cidr` | list(string) | `null` | Security group ingress CIDR blocks (null = module-managed disabled) |
| `sg_egress_cidr` | list(string) | `null` | Security group egress CIDR blocks (null = module-managed disabled) |
| `enable_logging` | bool | `true` | Enable ALB access logging |
| `log_bucket` | string | `""` | S3 bucket for ALB access logs |
| `idle_timeout` | number | `60` | ALB idle timeout in seconds |
| `lb_tags` | map(string) | `{}` | Additional tags for the LB resource |
| `subnet_name_filters` | list(string) | `["*public*"]` | Subnet name filters — ALB defaults to public subnets |

### aurora

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `engine_version` | string | `"17.5"` | Aurora PostgreSQL engine version |
| `instance_type` | string | `null` | RDS instance class (e.g., `db.r6g.large`); null = serverless or module default |
| `autoscaling_enable` | bool | `false` | Enable read replica autoscaling |
| `subnet_name_filters` | list(string) | `["*private*"]` | Subnet name filters for placement |
| `storage_encrypted` | bool | `true` | Encrypt storage at rest |

### kms

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `keys` | map(object) | `{}` | Map of KMS keys to create. Key = logical name (used in alias/discovery). Each entry: `key_usage` (ENCRYPT_DECRYPT or SIGN_VERIFY), `key_spec` (optional), `enable_key_rotation` (optional), `deletion_window_in_days` (default 30), `policy` (optional JSON), `tags` (optional) |

### rds

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `engine` | string | (required) | RDS engine (`postgres`, `mysql`, `mariadb`, `oracle-ee`, `sqlserver-ex`) |
| `engine_version` | string | (required) | Engine version (e.g., `"16.3"` for postgres) |
| `instance_class` | string | `"db.t4g.medium"` | RDS instance class |
| `allocated_storage` | number | `20` | Initial storage in GiB |
| `max_allocated_storage` | number | `100` | Max storage for autoscaling (0 = disabled) |
| `multi_az` | bool | `false` | Enable Multi-AZ deployment |
| `deletion_protection` | bool | `true` | Protect from accidental deletion |
| `shared` | bool | `false` | RDS defaults to stack-scoped (not shared) |
| `shared_vpc` | bool | `true` | Discover shared VPC |
| `parameters` | list(object) | `[]` | DB parameter group parameters |

### irsa-roles

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `roles` | list(any) | `[]` | List of IRSA role definitions (see below) |

Each entry in `roles` must have:

```hcl
{
  name               = string                 # IAM role name suffix
  service_account    = string                 # Kubernetes ServiceAccount name
  namespace_patterns = list(string)           # K8s namespace patterns (supports wildcards)

  # Exactly ONE of:
  preset             = string                 # Name of a preset from irsa-roles/presets/
  policy_document    = string                 # Inline JSON IAM policy document

  # Optional:
  params             = map(string)            # Template parameters for preset
  role_name          = string                 # Verbatim IAM role name (bypasses auto-generated name)
  policy_name        = string                 # Verbatim IAM managed policy name (bypasses auto-generated name)
}
```

### buckets

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `buckets` | map(object) | `{}` | Map of S3 bucket configurations (see below) |

Each entry in `buckets` must have:

```hcl
{
  versioning            = optional(bool, true)
  account_ids           = optional(list(string), [])  # AWS account IDs allowed access
  role_ids              = optional(list(string), [])  # IAM role IDs allowed access
  grant_write_permissions = optional(bool, false)
  custom_tags           = optional(map(string), {})
  backup_tags           = optional(map(string), {})
}
```

### s3-secret

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `secrets` | map(object) | `{}` | Map of Kubernetes secret definitions (see below) |
| `shared_eks` | bool | `true` | Whether to use the shared EKS cluster for discovery |

Each entry in `secrets` must have:

```hcl
{
  bucket      = string                          # S3 bucket name
  secret_file = string                          # Object key/path in the bucket
  namespace   = string                          # Kubernetes namespace for the secret
  data_key    = optional(string, null)          # Key name inside the K8s secret's data map
  secret_type = optional(string, "Opaque")      # Kubernetes secret type
  annotations = optional(map(string), {})
}
```

### tokenvendor

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `image_tag` | string | (required) | Container image tag to deploy |
| `cognito_pool_url` | string | (required) | Cognito user pool URL |
| `allowed_modules` | list(string) | (see below) | K8s ServiceAccount subjects allowed to use tokenvendor |
| `chart_version` | string | `"0.1.0"` | Helm chart version from private S3 repo |
| `helm_repo_url` | string | `"s3://pa-private-helm-repo/tokenvendor"` | Helm chart repository URL |
| `iam_role_name` | string | `"tokenvendor"` | IAM role name (literal — not context-generated) |
| `destination_namespace` | string | `"apps"` | K8s namespace where tokenvendor is deployed |
| `service_account_name` | string | `"tokenvendor"` | K8s ServiceAccount name for tokenvendor pods |
| `shared_eks` | bool | `true` | Whether to use the shared EKS cluster |

---

## Discovery Templates

### How Discovery Works

1. Component A creates resources with `atmos_discovery_*` tags
2. Component B contains `A_data.tf` (distributed copy of `A/A_discovery/A_data.tf.tmpl`)
3. `A_data.tf` queries AWS Resource Groups Tagging API and exposes `local.*` values
4. Component B's module/resources consume `local.vpc_id`, `local.cluster_name`, etc.

### Important Rules for Discovery Templates

- **Variables consumed by a template** (e.g. `shared_vpc`, `subnet_name_filters`) **must be declared in the consumer's `variables.tf`** — NOT in the `.tmpl` file itself
- **Never edit `*_data.tf`** — it is a generated copy. Edit the source `.tmpl` in `<component>_discovery/` and redistribute via `scripts/template_distribution/`
- **If you rename or add a template**, update `scripts/template_distribution/discovery-templates.yaml`

### Available Discovery Templates

The following templates actually exist on disk (verified by glob of `components/*/*_discovery/*.tmpl`):

| Source Component | Template Path | Consumer File Name | Exposed Locals |
|-----------------|---------------|--------------------|----------------|
| vpc | `components/vpc/vpc_discovery/vpc_data.tf.tmpl` | `vpc_data.tf` | `vpc_id`, `vpc_name`, `vpc_cidr`, `vpc_arn`, `subnet_ids`, `subnet_cidrs`, `subnet_azs`, `vpc_data_discovery_private_subnet_ids`, `vpc_data_discovery_public_subnet_ids` |
| eks | `components/eks/eks_discovery/eks_data.tf.tmpl` | `eks_data.tf` | `cluster_name`, `cluster_id`, `cluster_arn`, `cluster_endpoint`, `cluster_version`, `cluster_ca_certificate`, `cluster_auth_token`, `cluster_oidc_issuer_url`, `cluster_oidc_issuer_arn`, `cluster_security_group_id`, `cluster_platform_version`, `cluster_asg_names`, `nodes_security_group_id`, `cluster_node_iam_role_name`, `cluster_node_iam_role_arn` |
| alb | `components/alb/alb_discovery/alb_data.tf.tmpl` | `alb_data.tf` | `alb_id`, `alb_name`, `alb_arn`, `alb_dns_name`, `alb_zone_id`, `https_listener_arn`, `http_listener_arn`, `alb_security_group_ids`, `alb_target_group_name_format`, `alb_priority_ranges`, `priority_offset` |
| aurora | `components/aurora/aurora_discovery/aurora_data.tf.tmpl` | `aurora_data.tf` | Aurora cluster looked up via `data.aws_rds_cluster.main` by context-generated name. Reference as `data.aws_rds_cluster.main.endpoint`, `.reader_endpoint`, `.arn`, `.port`, `.cluster_identifier` |
| rds | `components/rds/rds_discovery/rds_data.tf.tmpl` | `rds_data.tf` | `rds_instance_identifier`, `rds_instance_address`, `rds_instance_endpoint`, `rds_instance_port`, `rds_instance_arn` |
| buckets | `components/buckets/buckets_discovery/buckets_data.tf.tmpl` | `buckets_data.tf` | `bucket_id`, `bucket_arn`, `bucket_name`, `bucket_region` |
| dynamodb | `components/dynamodb/dynamodb_discovery/dynamodb_data.tf.tmpl` | `dynamodb_data.tf` | `table_name`, `table_arn`, `table_hash_key`, `table_range_key` |
| waf | `components/waf/waf_discovery/waf_data.tf.tmpl` | `waf_data.tf` | `waf_name` (WAF WebACL looked up via `data.aws_wafv2_web_acl.main`) |
| alb-ec2-target | `components/alb-ec2-target/tg_discovery/tg_data.tf.tmpl` | `tg_data.tf` | `target_group_arn`, `target_group_name`, `target_group_port`, `target_group_protocol`, `target_group_vpc_id`, `target_group_health_check`, `target_group_load_balancer_arns` |

> **Note**: `karpenter`, `argocd`, `irsa-roles`, `s3-secret`, `tokenvendor`, and `kms` do **not** have discovery templates — other components do not discover them by tag.

### Exposed Locals — VPC (`vpc_data.tf`)

| Local | Description |
|-------|-------------|
| `local.vpc_id` | VPC ID |
| `local.vpc_name` | VPC name |
| `local.vpc_cidr` | VPC CIDR block |
| `local.vpc_arn` | VPC ARN |
| `local.subnet_ids` | All filtered subnet IDs |
| `local.subnet_cidrs` | All filtered subnet CIDRs |
| `local.subnet_azs` | AZs for filtered subnets |
| `local.vpc_data_discovery_private_subnet_ids` | Private subnet IDs |
| `local.vpc_data_discovery_public_subnet_ids` | Public subnet IDs |

### Exposed Locals — EKS (`eks_data.tf`)

| Local | Description |
|-------|-------------|
| `local.cluster_name` | EKS cluster name |
| `local.cluster_id` | EKS cluster ID |
| `local.cluster_arn` | EKS cluster ARN |
| `local.cluster_endpoint` | EKS API server endpoint |
| `local.cluster_version` | Kubernetes version |
| `local.cluster_ca_certificate` | Base64-decoded CA certificate |
| `local.cluster_auth_token` | EKS authentication token |
| `local.cluster_oidc_issuer_url` | OIDC issuer URL without `https://` (for IRSA) |
| `local.cluster_oidc_issuer_arn` | OIDC issuer ARN (for IRSA) |
| `local.cluster_security_group_id` | Cluster security group ID |
| `local.cluster_platform_version` | EKS platform version |
| `local.cluster_asg_names` | Node group ASG names |
| `local.nodes_security_group_id` | Node security group ID |
| `local.cluster_node_iam_role_name` | Node IAM role name |
| `local.cluster_node_iam_role_arn` | Node IAM role ARN |

### Exposed Locals — ALB (`alb_data.tf`)

| Local | Description |
|-------|-------------|
| `local.alb_id` | ALB resource ID |
| `local.alb_name` | ALB name |
| `local.alb_arn` | ALB ARN |
| `local.alb_dns_name` | ALB DNS name |
| `local.alb_zone_id` | ALB hosted zone ID |
| `local.https_listener_arn` | HTTPS listener ARN |
| `local.http_listener_arn` | HTTP listener ARN |
| `local.alb_security_group_ids` | ALB security group IDs |
| `local.alb_target_group_name_format` | Target group name format string |
| `local.alb_priority_ranges` | Allocated priority ranges |
| `local.priority_offset` | Priority offset for this consumer |

### Exposed Locals — Aurora (`aurora_data.tf`)

Aurora discovery uses a context-generated name to look up `data.aws_rds_cluster.main` directly. Reference via:

| Attribute | Description |
|-----------|-------------|
| `data.aws_rds_cluster.main.cluster_identifier` | Aurora cluster identifier |
| `data.aws_rds_cluster.main.endpoint` | Writer endpoint |
| `data.aws_rds_cluster.main.reader_endpoint` | Reader endpoint |
| `data.aws_rds_cluster.main.arn` | Cluster ARN |
| `data.aws_rds_cluster.main.port` | Database port |

### Exposed Locals — RDS (`rds_data.tf`)

| Local | Description |
|-------|-------------|
| `local.rds_instance_identifier` | RDS instance identifier |
| `local.rds_instance_address` | Hostname of the RDS instance |
| `local.rds_instance_endpoint` | Full endpoint (host:port) |
| `local.rds_instance_port` | Database port |
| `local.rds_instance_arn` | Instance ARN |

RDS discovery requires `shared_rds` (bool, default `false`) declared in the consumer's `variables.tf`.

---

## Presets System

Some components support YAML preset files that define reusable configuration blocks.

### EKS Nodegroup Presets

Location: `components/eks/presets/nodegroups/`

| Preset File | Description |
|-------------|-------------|
| `default.yaml` | Standard on-demand nodes |
| `spot.yaml` | Spot instance nodes |
| `non-volatile-workload.yaml` | Nodes for stateful/non-volatile workloads |

**`default.yaml` values:**

```yaml
instance_types: [t3a.xlarge, t3.xlarge, m5.xlarge, m5a.xlarge, m6i.xlarge]
capacity_type: ON_DEMAND
min_capacity: 1
max_capacity: 1
disk_size: 50
platform: bottlerocket
ami_id: null
ami_type: null
block_device_name: /dev/xvdb
labels: {}
taints: []
tags: {}
ebs_tags: {}
bootstrap_extra_args: ""
post_bootstrap_user_data: ""
```

Usage in Atmos stack config:

```yaml
vars:
  nodegroup_defaults_preset: "default"
  nodegroups_presets:
    ng_default: "default"
    ng_spot: "spot"
  nodegroups_map:
    ng_default:
      min_capacity: 2
      max_capacity: 5
```

### ALB Default Routes Preset

Location: `components/alb/presets/default_routes.yaml`

Defines built-in platform routes created when `enable_default_routes = true` (or when services are explicitly enabled via `default_route_services`). Auth is **disabled by default** for all services; configure per-rule as needed.

| Route Key | Host Prefix | Auth Default |
|-----------|-------------|--------------|
| `argocd` | `argocd` | false |
| `pgadmin` | `pgadmin` | false |
| `kubecost` | `kubecost` | false |
| `k8s_dashboard` | `k8sdashboard` | false |
| `onlineseal` | `onlineseal` | false |

### IRSA Roles Presets

Location: `components/irsa-roles/presets/`

Contains JSON IAM policy templates. Currently includes:
- `s3-access.json` — S3 access policy template

Referenced by name via `preset` in `var.roles`. When using a preset, optional `params` can be passed to template placeholders.

---

## Key Dependencies

### Dependency Graph (Full)

```
vpc (priority 1000 — foundation)
    ├─► eks (priority 2000)
    │       ├─► karpenter (priority 3000+)
    │       ├─► argocd (priority 6000+)
    │       ├─► argocd-projects
    │       ├─► argocd-stack-integration
    │       ├─► cluster-autoscaler
    │       ├─► metrics-server
    │       ├─► kubernetes-dashboard
    │       ├─► traefik-ingress (depends on EKS + ALB)
    │       ├─► ingress-controller
    │       ├─► yarp-ingress-controller
    │       ├─► sealed-secrets
    │       ├─► sealed-secrets-k8s
    │       ├─► kubecost
    │       ├─► monitoring-coralogix
    │       ├─► cert-manager
    │       ├─► eks-cluster-extensions
    │       ├─► irsa-roles (depends on EKS discovery)
    │       ├─► s3-secret (depends on EKS discovery)
    │       └─► tokenvendor (depends on EKS discovery)
    ├─► alb (priority 5000+)
    │       ├─► alb-listener-rules (priority 7000+)
    │       ├─► alb-ec2-target
    │       └─► waf (priority 9000+)
    ├─► aurora (priority 8000+)
    │       └─► aurora-db-users
    ├─► rds
    ├─► kms
    ├─► privatelink-provider
    │       └─► privatelink-consumer
    ├─► dns
    │       └─► route53
    ├─► backup
    ├─► dynamodb
    ├─► sqs
    ├─► eventbus
    ├─► buckets
    ├─► feature-flags
    ├─► masstransit
    ├─► coralogix
    ├─► monitoring-alerts
    ├─► monitoring-dashboards
    ├─► monitoring-health-checks
    ├─► monitoring-logs-pipeline
    ├─► cost-dashboards
    ├─► cur
    ├─► location
    ├─► apigateway
    ├─► app-bootstrap
    ├─► cron-action
    ├─► onlineseal
    └─► noop-timer
```

### PM Tool Priority Values (from consumer repo README)

| Priority | Components |
|----------|------------|
| 1000 | `vpc` |
| 2000 | `eks` |
| 3000 | `karpenter` |
| 4000 | `traefik-ingress` |
| 5000 | `alb` |
| 6000 | `argocd` |
| 7000 | `alb-listener-rules` |
| 7500 | `argocd-projects` |
| 8000 | `aurora` |
| 9000 | `waf` |

---

## Non-Obvious Conventions

These are sourced from `AGENTS.md` — critical rules that are easy to miss:

### 1. No Outputs Between Components
Cross-component data sharing is done **exclusively** via `atmos_discovery_*` tags + discovery templates. Never use `terraform_remote_state`, output references, or any other cross-component data passing mechanism.

### 2. `*_data.tf` Files Are Generated — Never Edit In-Place
Files matching `*_data.tf` are distributed copies of source `.tmpl` files. If you need to change a discovery template:
- Edit the **source** `.tmpl` in `<component>_discovery/<component>_data.tf.tmpl`
- Redistribute using `scripts/template_distribution/`
- Never commit manual edits to `*_data.tf` files in consumer components

### 3. Discovery Template Variables Must Be Declared in Consumer's `variables.tf`
Variables used inside a discovery template (e.g., `shared_vpc`, `subnet_name_filters`) are resolved at consumption time. They **must** be declared in the consumer component's `variables.tf` — not inside the `.tmpl` file.

### 4. Naming Rules
- Format: `namespace-project-environment[-stack]`
- AWS 63-character limit enforced via `postcondition` in `*_context.tf`
- `stack_short_name`: max 12 chars, pattern `[a-z0-9-]` only
- `atmos_stack` is auto-normalized: underscores → hyphens

### 5. `var.name` Override
Every component supports `var.name` for explicit name override (preserving legacy names during Terragrunt → Atmos migration). Keep this logic in `*_context.tf`.

### 6. `module_tags` vs `tags`
- `module_tags` — no `Name` tag → pass to **upstream modules**
- `tags` — includes `Name` tag → use on **direct resource declarations**

Swapping these breaks naming in upstream modules or creates duplicate `Name` tags.

### 7. `shared` Flag Enforcement
- `shared=true` → stack label is empty, resource is environment-wide
- `shared=false` → stack label = `stack_short_name` (required when `shared=false`)
- Enforced by a Terraform `check` block in every component
- Note: `rds` defaults to `shared=false` (stack-scoped); most other components default to `shared=true`

### 8. PM Tool Asset Synchronization
If you add or change parameters in `pm_tool/component-templates/metadata.yaml`, you **must** mirror the change in **all three** locations:
- `pm_tool/component-templates/metadata.yaml`
- `pm_tool/component-templates/component.yaml`
- `pm_tool/man-pages/main.man`

### 9. Script Paths Are Hardcoded
`scripts/template_distribution/distribute_templates.py` expects absolute paths hardcoded in `discovery-templates.yaml` (currently pointing to `/mnt/home/tf-modules/platform-components/...`). If running in a different workspace, either symlink or edit the YAML paths before running.

---

## Editing Checklist

Before opening a PR for component changes, verify all of the following (from `AGENTS.md`):

1. **File layout**: Follow `docs/component-development-guide.md` structure exactly
2. **New cross-component dependency**: Edit source `.tmpl` — not distributed `*_data.tf`
3. **New/renamed discovery template**: Update `scripts/template_distribution/discovery-templates.yaml`
4. **Changed user-facing variables**: Update all three PM tool assets together:
   - `pm_tool/component-templates/metadata.yaml`
   - `pm_tool/component-templates/component.yaml`
   - `pm_tool/man-pages/main.man`
5. **Commit subject format**: Use `feat(<component>):` or `fix(<component>):` — scope is cosmetic but required by convention

---

## Release Process

### Versioning Model

```
<component>-v<MAJOR>.<MINOR>.<PATCH>
```

Each component is versioned independently. Versions are created automatically by the Tekton pipeline on merge to `main`.

Current `platform_version`: **1** (from `.release-config.yaml`)

### Conventional Commits → Version Bumps

| Commit Type | Version Bump | Example |
|-------------|-------------|---------|
| `feat(<component>): ...` | Minor | `feat(eks): add support for custom AMI` |
| `fix(<component>): ...` | Patch | `fix(vpc): correct NAT gateway policy` |
| `BREAKING CHANGE` / `!` | **Blocked** outside coordinated major releases | — |

**Scope is cosmetic** — the pipeline determines which component was affected from **changed file paths**, not from the commit scope. Using the correct scope is still required for convention.

### Release Modes

| Mode | Trigger | When to Use |
|------|---------|-------------|
| **Normal** | Any `feat`/`fix` commit merged to `main` | Day-to-day component changes |
| **Major** | Add `releases/release_<N>.yaml` with `platform_version: <current+1>` | Coordinated breaking changes across multiple components |

### Key Rules

- **Never tag manually** — the Tekton pipeline handles all tagging
- **Never use `BREAKING CHANGE` or `!`** outside of a coordinated major release
- Changes to files matching `non_releasable_patterns` (docs, scripts, etc.) do **not** trigger releases

---

## Quick Reference

### Universal Context Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `namespace` | string | `""` | Organization name |
| `project` | string | required | Project name |
| `environment` | string | required | Environment (dev/staging/prod/sandbox) |
| `atmos_stack` | string | `null` | Stack name (auto-normalized) |
| `stack_short_name` | string | `null` | Short stack name (max 12 chars, `[a-z0-9-]`) |
| `shared` | bool | `true` | Shared (env-wide) vs stack-scoped |
| `enabled` | bool | `true` | Enable/disable component |
| `name` | string | `null` | Explicit name override (migration support) |

### Discovery Control Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `shared_vpc` | bool | `true` | Use shared VPC |
| `shared_eks` | bool | `true` | Use shared EKS cluster |
| `shared_alb` | bool | `true` | Use shared ALB |
| `shared_aurora` | bool | `true` | Use shared Aurora cluster |
| `shared_waf` | bool | `true` | Use shared WAF WebACL |
| `shared_rds` | bool | `false` | Use shared RDS instance (defaults to stack-scoped) |
| `subnet_name_filters` | list(string) | `["*private*"]` | Subnet selection filters (ALB uses `["*public*"]`) |

### Key Files Per Component

| File | Purpose |
|------|---------|
| `variables.tf` | All input variables |
| `<component>_context.tf` | Naming, tagging, `var.name` override, validation |
| `<component>_resources.tf` / `<component>_module.tf` | Main module invocation |
| `<component>_discovery/<component>_data.tf.tmpl` | Discovery template source (edit this) |
| `<dep>_data.tf` | Consumed discovery template (generated, do not edit) |
| `presets/*.yaml` | YAML configuration presets |
| `pm_tool/man-pages/main.man` | Component user documentation |
| `pm_tool/component-templates/metadata.yaml` | PM tool metadata |

---

## Additional Resources

- [`docs/component-development-guide.md`](./docs/component-development-guide.md) — Full component anatomy and creation guide (829 lines)
- [`docs/discovery-templates.md`](./docs/discovery-templates.md) — Discovery pattern deep-dive (1043 lines)
- [`docs/release-process.md`](./docs/release-process.md) — Release process details (508 lines)
- [`AGENTS.md`](./AGENTS.md) — Non-obvious conventions for AI agents (authoritative source)
- [Atmos Documentation](https://atmos.tools/)
- [Cloudposse Context Provider](https://github.com/cloudposse/terraform-provider-context)

---
