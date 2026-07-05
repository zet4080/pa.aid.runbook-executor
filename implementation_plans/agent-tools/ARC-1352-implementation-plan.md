### Implementation Plan: ARC-1352 — `runbook_check_dependencies`

Metadata:
- Issue: ARC-1352
- Lane: agent-tools
- Date: 2026-07-05

**Goal:**
Read a runbook file and return all unmet cross-lane dependency gate checkboxes above a given story section.

**Phase 0 Exploration**
| Purpose | Details |
|---|---|
| Scan a runbook for dependency gate checkbox lines above a target story section and report blocked/unmet status. |
| Inputs | `runbook_path: string, story_key: string` |
| Return type | `{ blocked: boolean, unmet: string[] }` |

**Files**
| Action | Path |
|---|---|
| CREATE | `/repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1352-implementation-plan.md` |
| CREATE | `~/.config/opencode/tools/runbook_check_dependencies.ts` |
| EDIT | `~/.config/opencode/opencode.json` |
| EDIT | `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` |

**Return Type**

```typescript
export interface RunbookCheckDependenciesResult {
  blocked: boolean;
  unmet: string[];
}
```

**Exported Helpers**

- `extractDependenciesAboveStory(content: string, storyKey: string): string[]` — finds the heading for the story (line matching `###.*{storyKey}` or containing the key in a checkbox), collects all lines above that position matching `/\[ ]\.*(?:must be merged|pre-check|dependency|gate)/i`, returns their text content.

**Implementation Steps:**

- Step 1: Read runbook file — read full content of `runbook_path` using `fs.readFileSync`.
- Step 2: Locate story section — find the first line containing `storyKey` to mark the boundary.
- Step 3: Collect dependency gate lines — extract all `[ ]` checkbox lines above that boundary that match gate language patterns.
- Step 4: Build result — if unmet lines found return `{ blocked: true, unmet: [...] }`, else `{ blocked: false, unmet: [] }`.

**Testing**

```bash
npm test ~/.config/opencode/tools/tests/runbook_check_dependencies.test.ts
```