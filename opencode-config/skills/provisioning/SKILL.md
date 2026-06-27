---
name: provisioning
description: Use when working on proALPHA SaaS provisioning, tenant onboarding, product assignment, provisioning agents, message contracts, or IaC
---
# Skill: provisioning

## Purpose

Quick reference for proALPHA SaaS provisioning system: tenant onboarding, product assignment, provisioning agents, message contracts, IaC. Distributed .NET/C# message-driven architecture using MassTransit over AWS SQS.

## When to Use

- Adding new provisioning commands or parameter sets
- Tracing message flow through provisioning system
- Modifying IaC for provisioning infrastructure
- Debugging tenant onboarding or product-assignment flows
- Creating new provisioning agents

---

## Overview

The proALPHA SaaS provisioning system is a **distributed .NET/C# message-driven architecture** that orchestrates tenant onboarding, product assignments, and configuration changes across multiple AWS accounts (SaaS, Portal, AIF). It uses **MassTransit** for async, queue-based communication over AWS SQS.

### Key principles

- **Message contracts are the single source of truth.** All inter-service message types live in `pa.provisioning.messages.cs` and are distributed as a NuGet package.
- **The Provisioning Service is the orchestrator.** It holds command/parameter-set definitions as JSON files, creates `Provisioning` entities, and drives the state machine.
- **Agents are stateless workers.** Each agent consumes commands for its `Recipient` identifier, executes the work (Cognito, DB, App Catalog, …), then calls back the REST API with Success or Failure.
- **Commands chain via `ProvisioningSucceeded`.** A dependent command's `TriggerType` is set to `ProvisioningSucceeded`, so it fires automatically after its predecessor completes.

---

## Repository Map

| Repo | Workspace Path | Type | Purpose |
|------|---------------|------|---------|
| `pa.provisioning.messages.cs` | `/workspace/saas/pa.provisioning.messages.cs` | NuGet Library | Shared message contracts — the single source of truth for all MassTransit message types |
| `pa.provisioning.cs` | `/workspace/saas/pa.provisioning.cs` | Service | Central orchestrator: consumes `TenantProductAssigned`, resolves command/parameter-set definitions, publishes `ProvisioningCommandIssued` and `ParameterSetChanged` |
| `pa.administration.tenant.cs` | `/workspace/saas/pa.administration.tenant.cs` | Service | Tenant and product lifecycle management; publishes `TenantProductAssigned` |
| `pa.saas.provisioning.agent.cs` | `/workspace/saas/pa.saas.provisioning.agent.cs` | Agent | SaaS account provisioning (Cognito users, app clients, M2M users) |
| `ip.portal.provisioning.cs` | `/workspace/saas/ip.portal.provisioning.cs` | Agent | Portal account provisioning (app catalog, cartridges) |
| `aif.provisioning.cs` | `/workspace/aif/aif.provisioning.cs` | Agent (template) | AIF account provisioning; **use as the canonical template when creating a new agent** |
| `provisioning.module.iac` | `/workspace/ccoe_in/provisioning.module.iac` | IaC | Terraform/Atmos + Kubernetes manifests (EKS, Aurora PostgreSQL, Cognito, ArgoCD, SQS) |

---

## Architecture / Message Flow

```
┌─────────────────────────────────────────┐
│  pa.administration.tenant.cs            │
│  (Tenant Administration Service)        │
│                                         │
│  • Manages tenant / product lifecycle   │
│  • Validates TenantId (4–16 lowercase   │
│    alphanumeric chars)                  │
└──────────────┬──────────────────────────┘
               │ publishes
               ▼
      [ TenantProductAssigned ]
        { TenantId, ProductId }
               │
               ▼
┌─────────────────────────────────────────┐
│  pa.provisioning.cs                     │
│  (Provisioning Service)                 │
│                                         │
│  1. Consumes TenantProductAssigned      │
│  2. Looks up ProvisioningCommandDefs    │
│     matching TriggerType               │
│  3. Creates Provisioning entity (DB)   │
│  4. Resolves Parameters from tenant    │
│     data / TenantId / constants        │
│  5. Publishes ProvisioningCommandIssued │
│     to each Recipient                   │
└──────┬───────────────────────┬──────────┘
       │                       │
       │ publishes             │ publishes
       ▼                       ▼
[ ProvisioningCommandIssued ] [ ParameterSetChanged ]
  { CommandKey, Parameters,    { ParameterSetDefinitionId,
    ProvisioningId, Recipient }  ProvisioningId, Recipient,
                                 Parameters }
       │                       │
  ┌────▼────┐  ┌───────────────▼──────────┐
  │  SaaS   │  │  Portal Agent            │
  │  Agent  │  │  ip.portal.provisioning  │
  └────┬────┘  └───────────┬──────────────┘
       │                   │
       │  (also routes to) │
       │  ┌────────────────▼──┐
       │  │  AIF Agent        │
       │  │  aif.provisioning │
       │  └────────────────┬──┘
       │                   │
       └──────────┬────────┘
                  │ each agent:
                  │  1. Dispatches by CommandKey / ParameterSetDefinitionId
                  │  2. Executes job (Cognito, DB, App Catalog, …)
                  │  3. Calls REST API callback
                  ▼
  POST /api/provisionings/{id}/Success
  POST /api/provisionings/{id}/Failure
                  │
                  ▼
┌─────────────────────────────────────────┐
│  pa.provisioning.cs                     │
│  Updates Provisioning state             │
│  If dependent commands exist:           │
│    └─► publishes next                  │
│        ProvisioningCommandIssued        │
│        (TriggerType = ProvisioningSucceeded) │
└─────────────────────────────────────────┘

─── Parameter Set Flow ───────────────────
User / System ──► REST API (pa.provisioning.cs)
                   └─► ParameterSetChanged ──► each Recipient Agent
```

---

## Message Contracts

All contracts live in `/workspace/saas/pa.provisioning.messages.cs`. They are distributed as a NuGet package and referenced by every service and agent.

### `ProvisioningCommandIssued`

Sent by the **Provisioning Service → Agents** when a command must be executed.

```csharp
public class ProvisioningCommandIssued
{
    /// <summary>Matches the CommandKey in a ProvisioningCommandDefinition JSON file.</summary>
    public string CommandKey { get; set; }           // e.g. "CreateHumanAdministratorSaas"

    /// <summary>Resolved key-value parameters built from tenant data / constants.</summary>
    public CommandParameter[] Parameters { get; set; }

    /// <summary>UUID that identifies the Provisioning entity in the database.</summary>
    public string ProvisioningId { get; set; }

    /// <summary>Must match one entry in the Recipients array of the command definition.</summary>
    public string Recipient { get; set; }            // e.g. "SaaS Agent"
}
```

### `ParameterSetChanged`

Sent by the **Provisioning Service → Agents** when a parameter set is updated via the REST API.

```csharp
public class ParameterSetChanged
{
    public Guid ParameterSetDefinitionId { get; set; }
    public Guid ProvisioningId { get; set; }

    /// <summary>Must match one entry in the Recipients array of the parameter-set definition.</summary>
    public string Recipient { get; set; }

    /// <summary>Contains typed parameter buckets.</summary>
    public Parameters Parameters { get; set; }
    // Parameters.StringParameters       : Dictionary<string, string>
    // Parameters.StringListParameters   : Dictionary<string, string[]>
    // Parameters.BooleanParameters      : Dictionary<string, bool>
}
```

### `TenantProductAssigned`

Published by **Tenant Administration → Provisioning Service** when a product is assigned to a tenant.

```csharp
public class TenantProductAssigned
{
    public string TenantId { get; set; }   // 4–16 lowercase alphanumeric chars
    public string ProductId { get; set; }
}
```

---

## Provisioning Command Definitions

**Location:** `/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/ProvisioningCommands/`

One JSON file per command. The Provisioning Service loads all files at startup.

```json
{
  "CommandKey": "CreateHumanAdministratorSaas",
  "Recipients": ["SaaS Agent"],
  "TriggerType": "TenantProductAssigned",
  "Parameters": [
    { "Key": "Email",    "SourceType": "TenantParameter" },
    { "Key": "TenantId", "SourceType": "TenantId" }
  ]
}
```

### `TriggerType` values

| Value | When it fires |
|-------|--------------|
| `TenantProductAssigned` | A product is assigned to a tenant (entry point) |
| `ProvisioningSucceeded` | A preceding dependent provisioning command succeeded |

### `SourceType` values for Parameters

| Value | Resolves to |
|-------|------------|
| `TenantParameter` | A named parameter stored on the tenant entity |
| `TenantId` | The tenant's ID string |
| `Constant` | A hard-coded value defined in the definition |

---

## Parameter Set Definitions

**Location:** `/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/ParameterSets/`

One JSON file per parameter set. Agents receive a `ParameterSetChanged` message when a set is updated.

```json
{
  "Key": "Subdomain",
  "Recipients": ["SaaS Agent", "Portal Agent"],
  "Parameters": [
    { "Key": "Subdomain", "Type": "String" }
  ]
}
```

### Dependency configuration

`/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/DependencySettings/DependentParameterSetRecipients.json`

Defines which parameter sets must be propagated to dependent recipients when a change occurs.

---

## Agent Pattern

Every agent is structured identically. Use `aif.provisioning.cs` at `/workspace/aif/aif.provisioning.cs` as the canonical template for new agents.

### Key files in every agent

| File | Responsibility |
|------|---------------|
| `*Agent.Custom/ProvisioningCommandIssuedConsumer.cs` | MassTransit consumer for `ProvisioningCommandIssued`; filters by `Recipient`, dispatches to individual command handlers by `CommandKey` |
| `*Agent.Custom/ParameterSetChangedConsumer.cs` | MassTransit consumer for `ParameterSetChanged`; filters by `Recipient`, dispatches to parameter-set handlers |
| `*Agent.Core/Provisioning/ProvisioningClient.cs` | HTTP client; calls `POST /api/provisionings/{id}/Success` or `/api/provisionings/{id}/Failure` on the Provisioning Service REST API |
| `*Agent.Custom/Commands/` | One class per `CommandKey`; contains the actual business logic (Cognito API calls, DB writes, etc.) |
| `*Agent.Custom/ParameterSets/` | One class per parameter-set `Key`; applies the updated parameters |

### Agent `Recipient` identifiers

| Agent | Recipient string (must match JSON definitions) |
|-------|-----------------------------------------------|
| SaaS Agent | `"SaaS Agent"` |
| Portal Agent | `"Portal Agent"` |
| AIF Agent | configurable (set in `appsettings.json`) |

---

## Key Files Quick Reference

### `pa.provisioning.messages.cs` — `/workspace/saas/pa.provisioning.messages.cs`

| File | Purpose |
|------|---------|
| `src/Pa.Provisioning.Messages/ProvisioningCommandIssued.cs` | `ProvisioningCommandIssued` contract |
| `src/Pa.Provisioning.Messages/ParameterSetChanged.cs` | `ParameterSetChanged` contract |
| `src/Pa.Provisioning.Messages/TenantProductAssigned.cs` | `TenantProductAssigned` contract |
| `src/Pa.Provisioning.Messages/CommandParameter.cs` | `CommandParameter` value type |
| `src/Pa.Provisioning.Messages/Parameters.cs` | `Parameters` type (String / StringList / Boolean buckets) |

### `pa.provisioning.cs` — `/workspace/saas/pa.provisioning.cs`

| File | Purpose |
|------|---------|
| `src/Pa.Provisioning.Api/ProvisioningCommands/*.json` | Provisioning command definitions (one per command) |
| `src/Pa.Provisioning.Api/ParameterSets/*.json` | Parameter set definitions (one per set) |
| `src/Pa.Provisioning.Api/DependencySettings/DependentParameterSetRecipients.json` | Parameter set dependency config |
| `src/Pa.Provisioning.Api/Consumers/TenantProductAssignedConsumer.cs` | Entry-point consumer; triggers command resolution |
| `src/Pa.Provisioning.Api/Controllers/ProvisioningsController.cs` | REST API: `/api/provisionings/{id}/Success|Failure` |

### `pa.administration.tenant.cs` — `/workspace/saas/pa.administration.tenant.cs`

| File | Purpose |
|------|---------|
| `src/*/Publishers/TenantProductAssignedPublisher.cs` | Publishes `TenantProductAssigned` via MassTransit |
| `src/*/Validators/TenantIdValidator.cs` | Enforces 4–16 lowercase alphanumeric constraint |

### `pa.saas.provisioning.agent.cs` — `/workspace/saas/pa.saas.provisioning.agent.cs`

| File | Purpose |
|------|---------|
| `src/*Agent.Custom/ProvisioningCommandIssuedConsumer.cs` | Dispatches commands to handlers |
| `src/*Agent.Custom/ParameterSetChangedConsumer.cs` | Dispatches parameter-set updates to handlers |
| `src/*Agent.Core/Provisioning/ProvisioningClient.cs` | Calls Success/Failure REST endpoint |
| `src/*Agent.Custom/Commands/` | Per-command business logic classes |
| `src/*Agent.Custom/ParameterSets/` | Per-parameter-set handler classes |

### `ip.portal.provisioning.cs` — `/workspace/saas/ip.portal.provisioning.cs`

Same structure as `pa.saas.provisioning.agent.cs`; handles Portal-specific operations (app catalog, cartridges).

### `aif.provisioning.cs` — `/workspace/aif/aif.provisioning.cs`

Canonical agent template. Same structure as above; use when creating any new agent.

### `provisioning.module.iac` — `/workspace/ccoe_in/provisioning.module.iac`

| File / Directory | Purpose |
|-----------------|---------|
| `terraform/eks.tf` | EKS cluster |
| `terraform/aurora.tf` | Aurora PostgreSQL (provisioning service DB + tenant admin DB) |
| `terraform/cognito.tf` | AWS Cognito user pools |
| `terraform/masstransit.tf` | MassTransit / SQS message bus infrastructure |
| `terraform/argocd.tf` | ArgoCD GitOps deployment |
| `terraform/dev/` | Atmos config for dev environment |
| `terraform/staging/` | Atmos config for staging environment |
| `terraform/prod/` | Atmos config for production environment |
| `k8s/apps/base/apps/provisioning/service/provisioning-service.yaml` | ArgoCD Application CRD — Provisioning Service |
| `k8s/apps/base/apps/provisioning/agent/provisioning-agent.yaml` | ArgoCD Application CRD — SaaS Agent |
| `k8s/apps/base/apps/administration/tenant/administration-tenant.yaml` | ArgoCD Application CRD — Tenant Administration |

---

## How to Add a New Provisioning Command

A provisioning command is triggered automatically (by `TenantProductAssigned` or `ProvisioningSucceeded`) and dispatched to one or more agents.

**Step 1 — Add the command definition (Provisioning Service)**

Create a new JSON file in `/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/ProvisioningCommands/`:

```json
{
  "CommandKey": "MyNewCommand",
  "Recipients": ["SaaS Agent"],
  "TriggerType": "TenantProductAssigned",
  "Parameters": [
    { "Key": "TenantId", "SourceType": "TenantId" },
    { "Key": "SomeParam", "SourceType": "TenantParameter" }
  ]
}
```

- `CommandKey` must be unique across all definition files.
- `Recipients` must match the agent's configured `Recipient` identifier exactly.
- Set `TriggerType` to `ProvisioningSucceeded` if this command should only run after a preceding command completes; set it to `TenantProductAssigned` for an entry-point command.

**Step 2 — Implement the command handler in the target agent**

In the relevant agent repo (e.g., `/workspace/saas/pa.saas.provisioning.agent.cs`), create a new handler class in `src/*Agent.Custom/Commands/MyNewCommandHandler.cs`:

```csharp
public class MyNewCommandHandler
{
    private readonly ProvisioningClient _provisioningClient;
    // inject required services (Cognito, DB, etc.)

    public async Task HandleAsync(ProvisioningCommandIssued message)
    {
        var tenantId  = message.Parameters.Single(p => p.Key == "TenantId").Value;
        var someParam = message.Parameters.Single(p => p.Key == "SomeParam").Value;

        // --- execute business logic ---

        await _provisioningClient.SuccessAsync(message.ProvisioningId);
        // on error: await _provisioningClient.FailureAsync(message.ProvisioningId, reason);
    }
}
```

**Step 3 — Register the handler in the agent's consumer**

In `src/*Agent.Custom/ProvisioningCommandIssuedConsumer.cs`, add a dispatch case for the new `CommandKey`:

```csharp
case "MyNewCommand":
    await _myNewCommandHandler.HandleAsync(message);
    break;
```

**Step 4 — Update message contracts (if new parameter types are needed)**

If the command requires a new parameter type not yet in `CommandParameter`, update `/workspace/saas/pa.provisioning.messages.cs`, bump the NuGet package version, and update references in all consuming repos.

**Step 5 — Test end-to-end**

Publish a `TenantProductAssigned` event with a valid `TenantId`/`ProductId` pair and verify the Provisioning Service creates the entity, publishes `ProvisioningCommandIssued`, and the agent calls back with Success.

---

## How to Add a New Parameter Set

A parameter set is updated via the Provisioning Service REST API and pushed to agents as `ParameterSetChanged`.

**Step 1 — Add the parameter set definition (Provisioning Service)**

Create a new JSON file in `/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/ParameterSets/`:

```json
{
  "Key": "MyParameterSet",
  "Recipients": ["SaaS Agent", "Portal Agent"],
  "Parameters": [
    { "Key": "FeatureFlag", "Type": "Boolean" },
    { "Key": "ConfigValue",  "Type": "String" }
  ]
}
```

- `Key` must be unique across all parameter set definition files.
- `Recipients` must match agent `Recipient` identifiers exactly.
- Supported `Type` values: `String`, `StringList`, `Boolean`.

**Step 2 — Update dependency config if needed**

If this parameter set must be cascaded to additional recipients on change, update:
`/workspace/saas/pa.provisioning.cs/src/Pa.Provisioning.Api/DependencySettings/DependentParameterSetRecipients.json`

**Step 3 — Implement the parameter-set handler in each recipient agent**

In each recipient agent, create `src/*Agent.Custom/ParameterSets/MyParameterSetHandler.cs`:

```csharp
public class MyParameterSetHandler
{
    public async Task HandleAsync(ParameterSetChanged message)
    {
        var featureFlag = message.Parameters.BooleanParameters["FeatureFlag"];
        var configValue = message.Parameters.StringParameters["ConfigValue"];

        // apply the configuration update
    }
}
```

**Step 4 — Register the handler in the agent's consumer**

In `src/*Agent.Custom/ParameterSetChangedConsumer.cs`, add a dispatch case:

```csharp
case "MyParameterSet": // matched by ParameterSetDefinitionId resolved to Key
    await _myParameterSetHandler.HandleAsync(message);
    break;
```

**Step 5 — Trigger via REST API**

Call the Provisioning Service to push the update:
```
PUT /api/parameter-sets/MyParameterSet
{ "FeatureFlag": true, "ConfigValue": "hello" }
```

---

## How to Create a New Agent

Use `aif.provisioning.cs` at `/workspace/aif/aif.provisioning.cs` as the canonical template.

**Step 1 — Copy the template repo and rename**

Clone `aif.provisioning.cs` into a new repo (e.g., `myservice.provisioning.cs`) and rename all project/namespace references from `Aif` to your service name.

**Step 2 — Configure the Recipient identifier**

In `appsettings.json` (or the environment-specific override), set:

```json
{
  "Provisioning": {
    "Recipient": "MyService Agent"
  }
}
```

This value must match the `Recipients` array in every command and parameter-set definition that targets this agent.

**Step 3 — Register command and parameter-set handlers**

Implement handlers in `src/*Agent.Custom/Commands/` and `src/*Agent.Custom/ParameterSets/` following the pattern described in the sections above.

Wire them in `ProvisioningCommandIssuedConsumer.cs` and `ParameterSetChangedConsumer.cs`.

**Step 4 — Configure MassTransit / SQS**

The agent connects to SQS via MassTransit. Configure queue names in `appsettings.json`. The queue for `ProvisioningCommandIssued` and `ParameterSetChanged` is typically auto-created by the MassTransit conventions; confirm against the IaC in `provisioning.module.iac/terraform/masstransit.tf`.

**Step 5 — Update the Provisioning Service definitions**

Add the new `Recipient` string to the `Recipients` arrays in every command/parameter-set JSON file that the new agent should handle.

**Step 6 — Add Kubernetes / ArgoCD manifests**

In `/workspace/ccoe_in/provisioning.module.iac`, add:
- A new K8s manifest at `k8s/apps/base/apps/provisioning/agent/<agent-name>.yaml`
- Reference the new ECR image (`<account>.dkr.ecr.eu-central-1.amazonaws.com/<your-image-path>`)
- Register the ArgoCD Application CRD

**Step 7 — Configure auto-deployment**

Ensure the new agent's repo contains a `.pa-sdlc-config` pointing its auto-deployment target to `provisioning.module.iac`.

---

## Infrastructure

All infrastructure is declared in `/workspace/ccoe_in/provisioning.module.iac` using **Terraform/Atmos** for AWS resources and **Kubernetes manifests** for workloads deployed via **ArgoCD** (GitOps).

### Terraform resources

| File | Component | Description |
|------|-----------|-------------|
| `terraform/eks.tf` | EKS | Kubernetes cluster hosting all provisioning workloads |
| `terraform/aurora.tf` | Aurora PostgreSQL | Database for the Provisioning Service state and Tenant Administration |
| `terraform/cognito.tf` | Cognito | AWS Cognito user pools managed by the SaaS Agent |
| `terraform/masstransit.tf` | SQS / MassTransit | SQS queues and IAM permissions for the message bus |
| `terraform/argocd.tf` | ArgoCD | GitOps controller; watches K8s manifests and syncs deployments |

### Environment configs (Atmos)

| Directory | Environment |
|-----------|------------|
| `terraform/dev/` | Development |
| `terraform/staging/` | Staging |
| `terraform/prod/` | Production |

### Kubernetes manifests

| Manifest | Service |
|----------|---------|
| `k8s/apps/base/apps/provisioning/service/provisioning-service.yaml` | Provisioning Service |
| `k8s/apps/base/apps/provisioning/agent/provisioning-agent.yaml` | SaaS Agent |
| `k8s/apps/base/apps/administration/tenant/administration-tenant.yaml` | Tenant Administration |

### Deployment pipeline

Services auto-deploy via **Tekton CI → ArgoCD** (GitOps). Docker image repositories:

| Service | ECR Repository |
|---------|---------------|
| Provisioning Service | `740079610737.dkr.ecr.eu-central-1.amazonaws.com/pa/provisioning` |
| SaaS Agent | `pa/saas/provisioning/agent` |
| Portal Agent | `ip/portal/provisioning` |
| AIF Agent | `aif/provisioning` |
| Tenant Administration | `pa/administration/tenant` |

Auto-deployment target (`.pa-sdlc-config`): `provisioning.module.iac`

---

## Tenant Validation Rules

| Rule | Detail |
|------|--------|
| Tenant ID format | 4–16 lowercase alphanumeric characters only (`[a-z0-9]{4,16}`) |
| Validated in | `pa.administration.tenant.cs` — `TenantIdValidator.cs` |

---

## "I need to…" Quick Reference

| Goal | Repo | File(s) to change |
|------|------|-------------------|
| Add a new provisioning command | `pa.provisioning.cs` | Add JSON to `src/Pa.Provisioning.Api/ProvisioningCommands/` |
| Implement a command handler in the SaaS agent | `pa.saas.provisioning.agent.cs` | Add class in `src/*Agent.Custom/Commands/`; register in `ProvisioningCommandIssuedConsumer.cs` |
| Implement a command handler in the Portal agent | `ip.portal.provisioning.cs` | Add class in `src/*Agent.Custom/Commands/`; register in `ProvisioningCommandIssuedConsumer.cs` |
| Chain a command to run after another succeeds | `pa.provisioning.cs` | Set `"TriggerType": "ProvisioningSucceeded"` in the new command's JSON definition |
| Add a new parameter set | `pa.provisioning.cs` | Add JSON to `src/Pa.Provisioning.Api/ParameterSets/` |
| Handle a parameter set update in an agent | Target agent repo | Add class in `src/*Agent.Custom/ParameterSets/`; register in `ParameterSetChangedConsumer.cs` |
| Configure parameter set dependencies | `pa.provisioning.cs` | Edit `src/Pa.Provisioning.Api/DependencySettings/DependentParameterSetRecipients.json` |
| Add a new message contract type | `pa.provisioning.messages.cs` | Add class under `src/Pa.Provisioning.Messages/`; bump NuGet version; update all consumer repos |
| Create a new provisioning agent | `aif.provisioning.cs` (template) | Copy repo; rename namespaces; set `Recipient` in `appsettings.json`; implement handlers |
| Publish `TenantProductAssigned` from a new place | `pa.administration.tenant.cs` | Use / copy `TenantProductAssignedPublisher.cs` pattern |
| Change tenant ID validation rules | `pa.administration.tenant.cs` | `src/*/Validators/TenantIdValidator.cs` |
| Add a new K8s workload (e.g., new agent) | `provisioning.module.iac` | Add manifest under `k8s/apps/base/apps/provisioning/agent/`; register ArgoCD Application CRD |
| Change SQS queue configuration | `provisioning.module.iac` | `terraform/masstransit.tf` |
| Change Cognito user pool configuration | `provisioning.module.iac` | `terraform/cognito.tf` |
| Change Aurora DB configuration | `provisioning.module.iac` | `terraform/aurora.tf` |
| Deploy changes to dev / staging / prod | `provisioning.module.iac` | Update Atmos config in `terraform/dev/`, `terraform/staging/`, or `terraform/prod/` |
| Find where the Provisioning Service calls back | Any agent repo | `src/*Agent.Core/Provisioning/ProvisioningClient.cs` |
| Find the REST API endpoint for Success/Failure | `pa.provisioning.cs` | `src/Pa.Provisioning.Api/Controllers/ProvisioningsController.cs` |
| Trace how a tenant product assignment flows | — | Start at `pa.administration.tenant.cs` → `TenantProductAssigned` → `pa.provisioning.cs` `TenantProductAssignedConsumer.cs` → `ProvisioningCommandIssued` → agent `ProvisioningCommandIssuedConsumer.cs` |
