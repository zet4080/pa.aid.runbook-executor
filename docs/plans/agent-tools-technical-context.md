# Agent Tools Lane — Technical Context

This document captures standing technical decisions, architecture patterns, and setup instructions for the `agent-tools` lane. Load this at the start of every future execution session.

---

## 1. Lane Overview

- **Lane name:** `agent-tools`
- **Purpose:** OpenCode tools that automate planning-repo operations (finding next story, claiming stories, checking off steps, running tests/lint, etc.)
- **Tools live at:** `~/.config/opencode/tools/`

### Story List (15 stories across 2 waves)

**Wave 1 — Runbook Navigation Tools**
- ARC-1348: `runbook_find_next_story` — 🔴 HIGH
- ARC-1349: `runbook_claim_story` — 🔴 HIGH
- ARC-1350: `runbook_check_step` — 🔴 HIGH
- ARC-1351: `runbook_check_story` — 🔴 HIGH

**Wave 2 — Code Quality & Issue Tools**
- ARC-1353: `get_current_issue` — 🔴 HIGH
- ARC-1354: `run_tests` — 🔴 HIGH
- ARC-1355: `run_lint` — 🔴 HIGH
- ARC-1358: `update_issue_status` — 🔴 HIGH
- ARC-1360: `archive_issue` — 🔴 HIGH
- ARC-1362: `validate_runbook` — 🔴 HIGH
- ARC-1352: `runbook_check_dependencies` — 🟡 BATCH

---

## 2. Tool Registration

- Tools are registered via the `"plugin"` array in `opencode.json`
- Each tool file exports a `server` async function
- Plugin path format: `~/.config/opencode/tools/runbook-tools.ts` (tilde assumed to work; fallback: absolute path `$HOME/.config/opencode/tools/runbook-tools.ts`)
- OpenCode must be restarted after plugin registration changes
- **Assumption:** tilde paths work in plugin array — test manually and adjust to absolute path if not

---

## 3. Markdown Parser — Copy Strategy

The existing AST-based parser from `pa.aid.conductor.ts` is **COPIED** (not imported cross-repo) for use by all agent tools.

- **Source:** `/repos/pa.aid.conductor.ts/packages/server/src/runbook/`
  - `parser.ts` — `parseRunbook(filePath: string): ParsedRunbook`
  - `astWalker.ts` — wave/story/risk/claim AST traversal
  - `types.ts` — all TypeScript interfaces
- **Destination:** `~/.config/opencode/tools/lib/` (parser.ts, astWalker.ts, types.ts copied here)
- **Why copy, not import:** Cross-repo absolute path imports are fragile in the OpenCode plugin runtime; copying ensures hermetic operation
- **Sync policy:** When `pa.aid.conductor.ts` parser is updated, manually re-copy the three files to `lib/`
- **Dependencies:** `unified ^11.0.5`, `remark-parse`, `remark-gfm` — installed via `bun add unified remark-parse remark-gfm` in `~/.config/opencode/tools/`

---

## 4. ParsedRunbook Structure

```typescript
export type StepType = 'agent' | 'checkpoint';
export type CheckpointLevel = 'high' | 'medium' | 'low' | null;

export interface RunbookSubStep {
  label: string;
  checked: boolean;
}

export interface RunbookStep {
  id: string;
  label: string;
  type: StepType;
  checked: boolean;
  checkpointLevel: CheckpointLevel;
  subSteps: RunbookSubStep[];
}

export interface RunbookWave {
  title: string;
  steps: RunbookStep[];
}

export interface RunbookMetadata {
  title: string;
  epic: string | null;
  lane: string | null;
  dependsOn: string[];
}

export interface ParsedRunbook {
  metadata: RunbookMetadata;
  waves: RunbookWave[];
}
```

---

## 5. Standard Tool Pattern

Every tool in this lane follows this pattern:

```typescript
import { tool, server as createServer } from '@opencode-ai/plugin';
import { parseRunbook } from './lib/parser.js';

const myTool = tool({
  name: 'tool_name',
  description: 'What it does',
  args: { /* zod schema */ },
  execute: async (args) => {
    // implementation
    return JSON.stringify(result);
  }
});

export const server = async () => createServer({ tools: [myTool] });
```

- All tools in one file OR split per tool — decide at ARC-1348 implementation time
- Tool names use snake_case
- Return JSON strings from execute

---

## 6. Finding Next Story Logic

The algorithm for `runbook_find_next_story` is the foundation other tools build on:

1. Call `parseRunbook(runbookPath)` → `{ metadata, waves }`
2. Iterate `waves` in order
3. For each wave, find first `step` where `step.checked === false`
4. A step is **claimed** if any `subStep.label.match(/claimed/i)` AND `subStep.checked === true`
5. Skip claimed steps; continue to next
6. Return first unchecked + unclaimed step
7. Return `null` if no unchecked unclaimed step found

### Output Shape

**Found:**
```json
{
  "storyKey": "ARC-NNNN",
  "title": "...",
  "wave": "Wave N — Title",
  "risk": "high|medium|low|null",
  "lineNumber": null
}
```

**Not Found:**
```json
{
  "result": null,
  "message": "No unclaimed work remains in this runbook"
}
```

---

## 7. WSL Setup Integration

All tools are deployed through the existing `opencode` component in `pa.aid.wsl-setup.sh`.

### opencode-sync-config.sh Change (already planned, implement in ARC-1348)

- **File:** `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
- **Change:** `for _dir in agents bin rules; do` → `for _dir in agents bin rules tools; do`
- This symlinks `~/.config-src/pa.aid.config.md/tools/` → `~/.config/opencode/tools/`

### Source Repo for Tools

- **Source repo:** `pa.aid.config.md` (git repo at `~/.config-src/pa.aid.config.md/`) 
- Tool files eventually committed to `pa.aid.config.md/tools/`
- During development: tool files placed directly in `~/.config/opencode/tools/` for testing

### Dependency Install

Add to `opencode-install.sh` or a new `opencode-tools-install.sh`:

```bash
cd ~/.config/opencode/tools && bun install
```

(requires a `package.json` in `~/.config/opencode/tools/`)

---

## 8. Key Assumptions

| Assumption | Status | How to verify |
|------------|--------|---------------|
| Tilde in plugin array path resolves correctly | Unverified | Test after ARC-1348 implementation; use absolute path if tilde fails |
| `bun` is the TypeScript runtime for OpenCode plugins | Assumed | Confirmed by ARC-1348 testing |
| `@opencode-ai/plugin` is the correct import for `tool()` | Unverified | Check OpenCode docs / installed packages after ARC-1348 |
| All 4 Wave 1 tools should live in one file (`runbook-tools.ts`) vs separate files | TBD | Decide at ARC-1348 implementation — document outcome here |

---

## 9. Runbook Paths Used by Tools

- **Planning repo runbooks:** `/repos/pa.aid.runbook-executor/docs/plans/runbook-{lane}.md`
- **Default runbook for agent-tools lane:** `/repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md`
- **Issue files:** `/repos/pa.aid.runbook-executor/issues/{epic}/{KEY}-*.md`
- **Implementation plans:** `/repos/pa.aid.runbook-executor/implementation_plans/{lane}/{KEY}-implementation-plan.md`
- **Completion summaries:** `/repos/pa.aid.runbook-executor/task-completions/{KEY}-COMPLETION-SUMMARY.md`