### Implementation Plan: ARC-1361 — `preflight_check`

Metadata header:
Issue: ARC-1361
Lane: agent-tools
Date: 2026-07-05

---

## Goal

Verify planning artifacts exist before executing an implementation plan.

---

## Phase 0: Exploration Findings

| Question | Answer |
| --- | --- |
| What is the purpose of the tool? | Verify planning artifacts exist before executing an implementation plan. |
| What are the inputs to the tool? | repo_path: string, issue_key: string, epic: string, lane: string |
| What is the return type? | name: string, passed: boolean, path?: string, details?: string[] |

---

## Files

| Action | File |
| --- | --- |
| CREATE | /repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1361-implementation-plan.md |
| EDIT | ~/.config/opencode/opencode.json |
| EDIT | /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh |

---

## Return Type

```typescript
export interface PreflightCheck {
  name: string;
  passed: boolean;
  path?: string;
  details?: string[];
}

export interface PreflightCheckResult {
  passed: boolean;
  checks: PreflightCheck[];
}
```

---

## Exported Helpers

- `findIssueFile(repoPath: string, epic: string, key: string): string | null` — reads `readdirSync(join(repoPath, 'issues', epic))` and finds the first entry where `entry.startsWith(key + '-') && entry.endsWith('.md')` and returns absolute path or `null` if no match found.

---

## Implementation Steps

### Step 1: Find Issue File

- Use `findIssueFile` to locate the issue file.

### Step 2: Check File Existence

- Check if the issue file exists.

### Step 3: Check for Placeholders

- Scan the issue file for any placeholders like `TBD` or `TODO`.

### Step 4: Check for Runbook

- Verify if the runbook file exists.

---

## Testing & Validation

```bash
npm test ~/.config/opencode/tools/tests/preflight_check.test.ts
```