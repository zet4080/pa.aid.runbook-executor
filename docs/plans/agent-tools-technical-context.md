# Agent Tools Technical Context

## 1. Overview

This document contains standing technical decisions and patterns for the agent-tools lane. It is loaded at the start of every agent-tools session.

## 2. Tool Registration — Auto-Discovery (No Registration Required)

Tools placed in `~/.config/opencode/tools/` are **automatically discovered** by OpenCode. No `opencode.json` `"plugin"` array entry is needed.

**How it works:**
- File name = tool name: `runbook_find_next_story.ts` → tool `runbook_find_next_story`
- `export default tool({...})` → single tool per file (preferred pattern for this lane)
- Named exports: `export const foo = tool({...})` → tool named `<filename>_foo`
- OpenCode scans the directory at startup — no restart config change needed (restart still needed after adding a new file)

**Plugin array in opencode.json:** NOT needed. Remove any existing `"plugin"` entries for tools in `~/.config/opencode/tools/`.

## 2b. Agent Tool Allowlists — managed in `pa.aid.wsl-setup.sh`

Custom tools are auto-discovered from `~/.config/opencode/tools/` but access is controlled per-agent. Tool allowlists are managed in the **Python config generator**, not in `opencode.json` directly.

**Source of truth:** `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
- The `opencode.json` is generated from scratch by embedded Python in this script
- Do NOT commit `opencode.json` to `pa.aid.config.md` — it is generated, not stored
- To add a tool to an agent's allowlist: edit the `"agent"` dict in `opencode-sync-config.sh`

| Agent | Tools allowed | Reason |
|---|---|---|
| **build** | All 11 runbook tools | Primary execution agent |
| **senior-coder** | All 11 runbook tools | Fallback execution agent |
| **plan** | `validate_runbook`, `get_current_issue` | Planning workflow support |
| **debug** | `run_tests`, `run_lint` | Reproducing failures |

**When adding a new tool:**
1. Create `~/.config/opencode/tools/{tool_name}.ts`
2. Edit `opencode-sync-config.sh` — add tool to the `"agent"` dict for relevant agents
3. For immediate use on current machine: update `~/.config/opencode/opencode.json` directly with the same Python patch (the wsl-setup script regenerates it on next full setup run)
4. Commit tool file to `pa.aid.config.md` tools/ directory
5. Commit `opencode-sync-config.sh` change to `pa.aid.wsl-setup.sh`
6. Restart OpenCode

## 3. Markdown Parser — Zero-Dependency Inline Strategy

The original plan was to copy `parser.ts`, `astWalker.ts`, `types.ts` from `pa.aid.conductor.ts` and install `unified`/`remark-parse`/`remark-gfm` via npm.

**This approach FAILS** in OpenCode's embedded Bun runtime: packages installed in `~/.config/opencode/tools/node_modules/` are silently not resolved, causing `parseRunbook` to return `undefined`.

**Decided strategy:** Each tool file must be **fully self-contained** — zero npm dependencies, zero cross-file imports from a `lib/` directory.

- Implement a line-based markdown parser inlined directly in the tool file. No `unified`, no `remark-*`, no external packages.
- All types (RunbookStep, RunbookWave, etc.) are also inlined in the tool file.
- **Do NOT use `lib/` imports or npm packages in any agent tool.**

**OpenCode's embedded Bun runtime does NOT resolve packages from `~/.config/opencode/tools/node_modules/`. All tool files must be zero-dependency and self-contained.**

Removed references:
- ❌ `bun add unified remark-parse remark-gfm`
- ❌ Copying `parser.ts` / `astWalker.ts` / `types.ts` to `lib/`
- ❌ `~/.config/opencode/tools/lib/` directory
- ❌ Installing dependencies in the tools directory

## 4. Standard Tool Pattern

Every tool in this lane follows this pattern:

```typescript
import { tool } from "@opencode-ai/plugin"

// Internal helpers (not exported, or exported for testing)
function myHelper(...) { ... }

export default tool({
  description: "What the tool does",
  args: {
    paramName: tool.schema.string().describe("Description of param"),
  },
  async execute(args, context) {
    // context.directory — session working directory
    // context.worktree  — git worktree root
    // context.sessionID, context.messageID, context.agent
    const result = myHelper(args.paramName);
    return JSON.stringify(result);
  },
})
```

**Rules:**
- One tool per file, file named after the tool (`tool_name.ts`)
- `export default tool({...})` — not named export, not `server` export
- All helper logic inlined in the same file (zero external imports except `@opencode-ai/plugin`)
- Return JSON strings from `execute`
- Use `tool.schema` (Zod) for arg validation
- Use `Bun.$\`command\`` for shell execution if needed

### Shell commands in tools

- Use `Bun.$\`command\`` (Bun shell utility, available in the OpenCode runtime)
- Do NOT use `child_process.exec` or `execSync` — use Bun's built-in shell

### ⚠️ Test File Location — Critical

- Test files must live in `~/.config/opencode/tools/tests/` — **never** in the plugin root
- OpenCode's Bun runtime scans and loads all `.ts` files in the tools root at startup
- A test file in the root crashes every session with `"Cannot use test outside of the test runner"`
- Rule: `runbook_find_next_story.test.ts` → `tests/runbook_find_next_story.test.ts`

## 5. WSL Integration

- All paths must use Unix-style forward slashes (`/`) — WSL does not resolve Windows-style backslashes (`\`).
- Use absolute paths only — relative paths may not resolve correctly.
- Test paths in WSL before committing.

## 6. TypeScript Interfaces

```typescript
interface RunbookStep {
  id: string;
  name: string;
  description: string;
  parameters: {
    [key: string]: any;
  };
}

interface RunbookWave {
  id: string;
  name: string;
  steps: RunbookStep[];
}
```

## 7. Key Assumptions

| Assumption | Status | How to verify |
|---|---|---|
| Tilde in plugin array path resolves correctly | ❌ **Confirmed does NOT work** — use absolute path `/home/zimmermann/.config/opencode/tools/runbook_find_next_story.ts` | Confirmed during ARC-1348 |
| npm packages in `tools/node_modules/` resolve in OpenCode Bun runtime | ❌ **Confirmed does NOT work** — use zero-dependency inline code only | Confirmed during ARC-1348 |
| Test files can live in plugin root directory | ❌ **Confirmed does NOT work** — tests must go in `tests/` subdirectory | Confirmed during ARC-1348 |
| `bun` is the TypeScript runtime for OpenCode plugins | ✅ **Confirmed** — OpenCode embeds Bun; it is NOT a standalone CLI (`bun` command not available in PATH) | Confirmed during ARC-1348 |
| `@opencode-ai/plugin` is the correct import for tool registration | ✅ **Confirmed** — available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` | Confirmed during ARC-1348 |
| All 4 Wave 1 tools should live in one file (`runbook-tools.ts`) vs separate files | ❌ **Revised** — one file per tool, named after the tool (e.g. `runbook_find_next_story.ts`) | Confirmed during ARC-1348 (refactored) |
| `"plugin"` array in opencode.json required | ❌ **NOT needed** — tools in `~/.config/opencode/tools/` are auto-discovered | Confirmed from official docs |

## 8. Standing Decisions

- All agent tools must be zero-dependency and self-contained
- No npm packages or external dependencies
- No cross-file imports from `lib/` directory
- Each tool lives in its own file named exactly after the tool: `{tool_name}.ts`
- Tools in `~/.config/opencode/tools/` are auto-discovered — **no `"plugin"` array entry in opencode.json needed**
- `export default tool({...})` is the required export pattern (single tool per file)
- All tool files must live in `~/.config/opencode/tools/`
- Test files must live in `~/.config/opencode/tools/tests/`, named `{tool_name}.test.ts`
- Use absolute paths only
- Use Unix-style forward slashes
- Test paths in WSL before committing

## 9. Verification Status

| Decision | Verified | Verification Method |
|---|---|---|
| Zero-dependency tools | ✅ | Confirmed during ARC-1348 |
| Absolute paths only | ✅ | Confirmed during ARC-1348 |
| Unix-style forward slashes | ✅ | Confirmed during ARC-1348 |
| Test files in `tests/` subdirectory | ✅ | Confirmed during ARC-1348 |
| Single file approach | ❌ Revised — one file per tool | Refactored during ARC-1348 |
| Auto-discovery from `tools/` dir, no `"plugin"` array needed | ✅ | Confirmed from official OpenCode docs |
| `export default tool({...})` is correct export pattern | ✅ | Confirmed from official OpenCode docs |
