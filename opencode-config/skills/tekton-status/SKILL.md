---
name: tekton-status
description: Use when checking live Tekton PipelineRun status, CloudWatch audit logs, EKS audit logs, stuck pipelines, or missing PipelineRun visibility
---
# Skill: tekton-status

## Purpose

Retrieve live Tekton PipelineRun status via EKS control-plane audit logs in CloudWatch Logs Insights. Reliable read-only fallback when EKS MCP Kubernetes client fails for SSO profiles.

## When to Use

- Checking live Tekton PipelineRun status
- Retrieving stuck or missing PipelineRun visibility
- Querying EKS or CloudWatch audit logs for pipeline data
- Debugging PipelineRun failures with timing/task progress

---

## Overview

This skill documents the **proven, read-only method** to retrieve the live status of a Tekton PipelineRun in the `shared` EKS cluster by querying the EKS control-plane audit logs stored in **CloudWatch Logs Insights**.

**When to use it:** You have a PipelineRun name (e.g., `pa.dms.adminservice.cs-build-dotnet-application-r6mkk`) and need its status, start/completion times, and task progress. This method is the **reliable fallback** because the EKS MCP Kubernetes client fails to authenticate for the `shared` SSO profile (see [Known Pitfalls](#known-pitfalls)).

| Target | Value |
|--------|-------|
| AWS Account | `740079610737` |
| Region | `eu-central-1` |
| Cluster | `shared` |
| Tekton namespace | `ci` |
| Tekton API group | `tekton.dev/v1` |

---

## Environment / Prerequisites

| Requirement | Details |
|-------------|---------|
| **AWS profile** | `shared` |
| **Region** | `eu-central-1` |
| **Account** | `740079610737` |
| **Audit logging enabled** | `cluster.logging.clusterLogging[type=audit].enabled: true` — if disabled the log group will be empty |
| **CloudWatch log group** | `/aws/eks/<cluster-name>/cluster`; for `shared`: `/aws/eks/shared/cluster` |
| **Log retention** | `shared` cluster = 90 days; `shared-ci` = 72 hours — verify with `describe-log-groups` |
| **IAM permissions (read-only)** | `logs:DescribeLogGroups`, `logs:StartQuery`, `logs:GetQueryResults`, `logs:DescribeLogStreams`, `logs:GetLogEvents` |
| **Tekton namespace** | All PipelineRuns run in the single `ci` namespace on `shared` |

---

## Step 1 — Cluster Discovery

Confirm the active identity, available clusters, and that audit logging and the log group are in place before running queries.

```bash
# Confirm you are operating as the expected identity in account 740079610737
aws sts get-caller-identity
```

```bash
# List all EKS clusters in the region — should return: shared, shared-ci
aws eks list-clusters --region eu-central-1
```

```bash
# Inspect the shared cluster; verify logging.clusterLogging has audit: enabled: true
aws eks describe-cluster --name shared --region eu-central-1
```

Check the output for:
```json
"logging": {
  "clusterLogging": [
    { "types": ["audit"], "enabled": true }
  ]
}
```

```bash
# Confirm the log group exists and check its retention period (should be 90 days)
aws logs describe-log-groups \
  --log-group-name-prefix /aws/eks/shared/cluster \
  --region eu-central-1
```

---

## Step 2 — Key Insight: Why Event Objects, Not PipelineRun Objects

The audit level configured per resource type is what makes this method work. Understanding the distinction is essential:

| Resource | Audit Level | What is logged |
|----------|-------------|----------------|
| `PipelineRun` / `TaskRun` (`tekton.dev/v1`) | **Metadata** | verb, name, namespace, timestamps — **no status body** |
| Kubernetes `Event` objects (emitted by the Tekton controller) | **RequestResponse** | **Full body**, including the human-readable progress message |

The Tekton controller emits Kubernetes `Event` objects whose `message` field contains strings like:

```
Tasks Completed: 8 (Failed: 0, Cancelled 0), Skipped: 1
```

**Therefore: always query `Event` objects, not `PipelineRun` objects.** PipelineRun entries in the audit log lack `.status.conditions` and contain no useful state information.

---

## Step 3 — The Query Pattern (Start → Get)

CloudWatch Logs Insights queries are **asynchronous**: you start a query and then poll for results.

```bash
# Step 1 — Submit the query; returns a queryId
aws logs start-query \
  --log-group-name /aws/eks/shared/cluster \
  --start-time <EPOCH_SECONDS_START> \
  --end-time <EPOCH_SECONDS_END> \
  --query-string "<QUERY_STRING>" \
  --region eu-central-1

# Step 2 — Retrieve results using the queryId from Step 1
aws logs get-query-results --query-id <queryId> --region eu-central-1
```

> **Warning:** `start-query` and `get-query-results` must be called back-to-back **within the same session token cycle**. A session token rotation between the two calls will cause `get-query-results` to fail. If this happens, re-run `start-query` and immediately fetch results.

### Computing the Time Window

Use a Python snippet to convert a relative time window into epoch seconds:

```python
import time
now = int(time.time())
start = now - (7 * 24 * 3600)    # 7-day window — sufficient for most runs
# For the full retention window on shared (90 days):
# start = now - (90 * 24 * 3600)
```

---

## Step 4 — The Queries

### Query A — Find a PipelineRun by Suffix / Confirm Namespace

Use the unique random suffix (last 5 characters of the PipelineRun name) to locate all audit events for that run. This reveals the namespace (`ci`), the full run name, and its UID.

```
fields @timestamp, @message
| filter @message like /r6mkk/
| sort @timestamp desc
| limit 50
```

Replace `r6mkk` with the actual run suffix from your PipelineRun name.

---

### Query B — Get Final Status

Filter to only the Event objects that carry a `Tasks Completed` progress message, then take the most recent row — that is the final state.

```
fields @timestamp, @message
| filter @message like /r6mkk/ AND @message like /Tasks Completed/
| sort @timestamp desc
| limit 10
```

**Interpreting the `reason` field:**

| `reason` value | Meaning |
|----------------|---------|
| `Succeeded` | PipelineRun completed successfully |
| `Running` | PipelineRun is still in progress |
| `Failed` | PipelineRun failed |

The `message` field gives the full task count breakdown, e.g.: `Tasks Completed: 8 (Failed: 0, Cancelled 0), Skipped: 1`.

---

### Query C — Full Progress Timeline (Start Time + Per-Task Checkpoints)

Same filter as Query B, but sorted ascending to reconstruct the timeline from start to finish. The **earliest `Running` row** with `Tasks Completed: 0` is the PipelineRun start time.

```
fields @timestamp, @message
| filter @message like /r6mkk/ AND @message like /Tasks Completed/
| sort @timestamp asc
| limit 20
```

---

### Query D — Search When the Suffix Is Unknown

If you only know the repository name or pipeline name, search by those terms instead.

**By repository and pipeline name:**
```
fields @timestamp, @message
| filter @message like /adminservice/ AND @message like /build-dotnet-application/
| sort @timestamp desc
| limit 50
```

**All recent PipelineRun `update` activity in `ci`:**
```
fields @timestamp, @message
| filter @message like /pipelineruns/ AND @message like /"verb":"update"/
| sort @timestamp desc
| limit 10
```

---

## Step 5 — Extracting Structured Data

Each result row's `@message` field is a **raw JSON audit event**. The relevant paths are:

| JSON Path | Contains |
|-----------|----------|
| `requestObject.reason` / `responseObject.reason` | `Running`, `Succeeded`, or `Failed` |
| `requestObject.message` / `responseObject.message` | `"Tasks Completed: N..."` progress string |
| `responseObject.involvedObject.name` | The PipelineRun name |
| `responseObject.involvedObject.namespace` | The namespace (`ci`) |
| `responseObject.firstTimestamp` / `.lastTimestamp` | Event timestamps |
| `objectRef.namespace` / `objectRef.name` | Namespace / name (Metadata-level events) |
| `stageTimestamp` | When the API server completed processing this call |
| `verb` | The Kubernetes API verb: `create`, `update`, `patch`, etc. |

### Python Extraction Skeleton

```python
import json

def extract_tekton_event(row):
    for field in row:
        if field.get('field') == '@message':
            try:
                audit = json.loads(field['value'])
                req = audit.get('requestObject', {})
                resp = audit.get('responseObject', {})
                return {
                    'timestamp': audit.get('stageTimestamp'),
                    'reason':    req.get('reason') or resp.get('reason'),
                    'message':   req.get('message') or resp.get('message'),
                    'namespace': audit.get('objectRef', {}).get('namespace'),
                    'name':      audit.get('objectRef', {}).get('name'),
                    'verb':      audit.get('verb'),
                }
            except Exception:
                pass
    return {}
```

Call this function on each row returned by `get-query-results` → `results[*]`.

---

## Worked Example

**PipelineRun:** `pa.dms.adminservice.cs-build-dotnet-application-r6mkk`
**Cluster:** `shared` | **Namespace:** `ci` | **Date:** 2026-06-01

Running Query C (ascending timeline) against the 7-day window produces:

| `@timestamp` | `reason` | `message` |
|---|---|---|
| `2026-06-01T11:42:03Z` | `Running` | Tasks Completed: 0 (Failed: 0, Cancelled 0), Incomplete: 9, Skipped: 0 |
| `2026-06-01T11:42:28Z` | `Running` | Tasks Completed: 1 (Failed: 0, Cancelled 0), Incomplete: 8, Skipped: 0 |
| `2026-06-01T11:42:35Z` | `Running` | Tasks Completed: 2 (Failed: 0, Cancelled 0), Incomplete: 7, Skipped: 0 |
| `2026-06-01T11:43:17Z` | `Running` | Tasks Completed: 4 (Failed: 0, Cancelled 0), Incomplete: 5, Skipped: 0 |
| `2026-06-01T11:48:47Z` | `Running` | Tasks Completed: 5 (Failed: 0, Cancelled 0), Incomplete: 3, Skipped: 1 |
| `2026-06-01T11:51:37Z` | `Running` | Tasks Completed: 6 (Failed: 0, Cancelled 0), Incomplete: 2, Skipped: 1 |
| `2026-06-01T11:52:16Z` | `Running` | Tasks Completed: 7 (Failed: 0, Cancelled 0), Incomplete: 1, Skipped: 1 |
| `2026-06-01T11:52:50Z` | `Succeeded` | Tasks Completed: 8 (Failed: 0, Cancelled 0), Skipped: 1 |

**Result:** `Succeeded` — namespace `ci`, started `11:42:03Z`, completed `11:52:50Z` (~10m 47s total).

---

## Known Pitfalls

### 1. EKS MCP Kubernetes Client Auth Failure *(do not use for `shared`)*

`list_k8s_resources` and `read_k8s_resource` persistently fail for the `shared` SSO profile with:

```
Tool call failed due to expired or invalid AWS credentials.
Please refresh your credentials and retry.
```

This failure is **consistent and non-transient** — it occurs even though plain AWS API calls (STS, EKS, CloudWatch) work correctly and the IAM access entry (`AmazonEKSViewPolicy`, cluster scope) is confirmed correct. The root cause appears to be an incompatibility between the EKS MCP Kubernetes client and the `shared` cluster's SSO credential chain.

**Do NOT attempt to use Kubernetes client tools for the `shared` cluster.** Use the CloudWatch audit-log method documented in this skill instead.

---

### 2. Audit-Log Method Limitations

| Limitation | Detail |
|------------|--------|
| No `.status.conditions` | PipelineRun objects are audited at Metadata level — the full `.status` body (conditions, childReferences, steps) is never logged |
| No per-TaskRun names | Individual TaskRun names are not present in the Event `message`; only aggregate task-completion counts are available |
| Checkpoints only | Progress is reported at task-completion boundaries, not per step |
| Skipped task count visible; name is not | The number of skipped tasks appears in the message, but the skipped task's name does not |
| Retention window | Queries must fall within the log group's retention period (90 days for `shared`; 72 hours for `shared-ci`) |
| Session token sensitivity | `get-query-results` must be called in the same session token cycle as `start-query`; a rotation in between causes the result fetch to fail |

---

## Quick Lookup

| Goal | Query / Command |
|------|----------------|
| Discover clusters | `aws eks list-clusters --region eu-central-1` |
| Confirm audit logging is enabled | `aws eks describe-cluster --name shared --region eu-central-1` → check `logging.clusterLogging` for `audit: enabled: true` |
| Confirm log group exists | `aws logs describe-log-groups --log-group-name-prefix /aws/eks/shared/cluster --region eu-central-1` |
| Find a PipelineRun by suffix | `start-query` with `filter @message like /<suffix>/` (Query A) |
| Get final status | Add `AND @message like /Tasks Completed/`, `sort @timestamp desc`, `limit 10` (Query B) |
| Get start time and full timeline | Same filter as Query B, `sort @timestamp asc` (Query C) |
| Search by repo / pipeline name | `filter @message like /<repo>/ AND @message like /<pipeline>/` (Query D) |
| See all recent PipelineRun activity | `filter @message like /pipelineruns/ AND @message like /"verb":"update"/` (Query D variant) |
| Parse a result row | Use `extract_tekton_event(row)` Python skeleton (Step 5) |
