### Implementation Plan: ARC-1359 — `planning_commit`

Metadata header:
Issue: ARC-1359
Lane: agent-tools
Date: 2026-07-05

---

## Goal

Atomic `git add {files}` + `git commit -m {message}` + `git push` for planning artifacts.

---

## Phase 0: Exploration Findings

| Question | Answer |
| --- | --- |
| What is the purpose of the tool? | Atomic `git add {files}` + `git commit -m {message}` + `git push` for planning artifacts. |
| What are the inputs to the tool? | repo_path: string, files: string[], message: string |
| What is the return type? | hash: string, pushed: boolean, error?: string |

---

## Files

| Action | File |
| --- | --- |
| CREATE | /repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1359-implementation-plan.md |
| EDIT | ~/.config/opencode/opencode.json |
| EDIT | /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh |

---

## Return Type

```typescript
export interface PlanningCommitResult {
  hash: string;
  pushed: boolean;
  error?: string;
}
```

---

## Exported Helpers

- `extractCommitHash(stdout: string): string` — applies regex `/\[(.*?)\] ([a-f0-9]{7,})\` to stdout; returns group 1 or "" 

---

## Implementation Steps

### Step 1: Validate Inputs

- Validate `args.files.length > 0` → else return `{ hash: "", pushed: false, error: "files array must not be empty" }`

### Step 2: Stage Files

- Stage: `await Bun.$\`git -C ${repo_path} add -- ${args.files.join(' ')}\`.quiet()` (note: special case `[".