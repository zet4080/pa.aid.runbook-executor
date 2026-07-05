### Implementation Plan: ARC-1357 — `get_branch_diff`

Metadata header:
Issue: ARC-1357
Lane: agent-tools
Date: 2026-07-05

---

## Goal

Returns unified diff of current branch vs merge-base with main/master.

---

## Phase 0: Exploration Findings

| Question | Answer |
| --- | --- |
| What is the purpose of the tool? | Returns unified diff of current branch vs merge-base with main/master. |
| What are the inputs to the tool? | directory?: string |
| What is the return type? | diff: string, changed_files: string[], base_branch: string, error?: string |

---

## Files

| Action | File |
| --- | --- |
| CREATE | /repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1357-implementation-plan.md |
| EDIT | ~/.config/opencode/opencode.json |
| EDIT | /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh |

---

## Return Type

```typescript
export interface GetBranchDiffResult {
  diff: string;
  changed_files: string[];
  base_branch: string;
  error?: string;
}
```

---

## Exported Helpers

- `detectBaseBranch(directory: string): Promise<string>` — runs `git -C {directory} show-ref --verify --quiet refs/heads/main`; exit 0 → 