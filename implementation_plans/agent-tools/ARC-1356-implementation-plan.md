### Implementation Plan: ARC-1356 — `run_typecheck`

Metadata:
- Issue: ARC-1356
- Lane: agent-tools
- Date: 2026-07-05

**Goal:**
Auto-detect and run the type checker for a project directory, returning pass/fail and output.

**Phase 0 Exploration**
| Purpose | Details |
|---|---|
| Detect project type (TypeScript, Python, Rust) by file presence and run the appropriate type checker. |
| Inputs | `directory?: string` |
| Return type | `{ passed: boolean | null, output: string, checker: string | null, error?: string }` |

**Files**
| Action | Path |
|---|---|
| CREATE | `/repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1356-implementation-plan.md` |
| CREATE | `~/.config/opencode/tools/run_typecheck.ts` |
| EDIT | `~/.config/opencode/opencode.json` |
| EDIT | `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` |

**Return Type**

```typescript
export interface RunTypecheckResult {
  passed: boolean | null;
  output: string;
  checker: string | null;
  error?: string;
}
```

**Exported Helpers**

- `detectTypeChecker(directory: string): { checker: string; command: string[] } | null` — checks file existence in order: `tsconfig.json` → `{ checker: 