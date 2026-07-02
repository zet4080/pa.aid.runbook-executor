# Agent Tools Technical Context

## 1. Overview

This document contains standing technical decisions and patterns for the agent-tools lane. It is loaded at the start of every agent-tools session.

## 2. Tool Registration

- **Plugin path must be absolute** — tilde (`~`) does NOT work in the `"plugin"` array in `opencode.json`
- Use: `"/home/zimmermann/.config/opencode/tools/runbook-tools.ts"`
- Do NOT use: `"~/.config/opencode/tools/runbook-tools.ts"`

## 3. Markdown Parser — Zero-Dependency Inline Strategy

The original plan was to copy `parser.ts`, `astWalker.ts`, `types.ts` from `pa.aid.conductor.ts` and install `unified`/`remark-parse`/`remark-gfm` via npm.

**This approach FAILS** in OpenCode's embedded Bun runtime: packages installed in `~/.config/opencode/tools/node_modules/` are silently not resolved, causing `parseRunbook` to return `undefined`.

**Decided strategy:** Each tool file must be **fully self-contained** — zero npm dependencies, zero cross-file imports from a `lib/` directory.

- Implement a line-based markdown parser inlined directly in `runbook-tools.ts`. No `unified`, no `remark-*`, no external packages.
- All types (RunbookStep, RunbookWave, etc.) are also inlined in the tool file.
- **Do NOT use `lib/` imports or npm packages in any agent tool.**

**OpenCode's embedded Bun runtime does NOT resolve packages from `~/.config/opencode/tools/node_modules/`. All tool files must be zero-dependency and self-contained.**

Removed references:
- ❌ `bun add unified remark-parse remark-gfm`
- ❌ Copying `parser.ts` / `astWalker.ts` / `types.ts` to `lib/`
- ❌ `~/.config/opencode/tools/lib/` directory
- ❌ Installing dependencies in the tools directory

## 4. Standard Tool Pattern

### 4.1 Tool File Structure

```typescript
import { Tool } from "@opencode-ai/plugin";

// Tool 1
const tool1: Tool = {
  name: "tool1",
  description: "Tool 1 description",
  parameters: {
    type: "object",
    properties: {
      param1: { type: "string" }
    },
    required: ["param1"]
  },
  execute: async ({ param1 }) => {
    // Implementation
  }
};

// Tool 2
const tool2: Tool = {
  name: "tool2",
  description: "Tool 2 description",
  parameters: {
    type: "object",
    properties: {
      param2: { type: "string" }
    },
    required: ["param2"]
  },
  execute: async ({ param2 }) => {
    // Implementation
  }
};

// Export all tools
const tools = [tool1, tool2];
export default tools;
```

### 4.2 Tool Registration

```json
{
  "tools": [
    {
      "name": "tool1",
      "description": "Tool 1 description"
    },
    {
      "name": "tool2",
      "description": "Tool 2 description"
    }
  ],
  "plugin": ["/home/zimmermann/.config/opencode/tools/runbook-tools.ts"]
}
```

### 4.3 Tool Testing

- Test files must live in `~/.config/opencode/tools/tests/` — **never** in `~/.config/opencode/tools/` (the plugin root).
- OpenCode's Bun runtime scans the plugin root directory and loads all `.ts` files it finds at tool-registry resolution time.
- A test file in the root will be loaded on every prompt, and any `test()` call outside of `bun test` throws: `"Cannot use test outside of the test runner"` — crashing every request.
- **Rule:** `runbook-tools.test.ts` → `tests/runbook-tools.test.ts`
- **Rule:** All future tool test files go in `tests/` subdirectory.

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
| Tilde in plugin array path resolves correctly | ❌ **Confirmed does NOT work** — use absolute path `/home/zimmermann/.config/opencode/tools/runbook-tools.ts` | Confirmed during ARC-1348 |
| npm packages in `tools/node_modules/` resolve in OpenCode Bun runtime | ❌ **Confirmed does NOT work** — use zero-dependency inline code only | Confirmed during ARC-1348 |
| Test files can live in plugin root directory | ❌ **Confirmed does NOT work** — tests must go in `tests/` subdirectory | Confirmed during ARC-1348 |
| `bun` is the TypeScript runtime for OpenCode plugins | ✅ **Confirmed** — OpenCode embeds Bun; it is NOT a standalone CLI (`bun` command not available in PATH) | Confirmed during ARC-1348 |
| `@opencode-ai/plugin` is the correct import for tool registration | ✅ **Confirmed** — available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` | Confirmed during ARC-1348 |
| All 4 Wave 1 tools should live in one file (`runbook-tools.ts`) vs separate files | ✅ **Confirmed** — single file approach works | Confirmed during ARC-1348 |

## 8. Standing Decisions

- All agent tools must be zero-dependency and self-contained
- No npm packages or external dependencies
- No cross-file imports from `lib/` directory
- All tool files must live in `~/.config/opencode/tools/`
- Test files must live in `~/.config/opencode/tools/tests/`
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
| Single file approach | ✅ | Confirmed during ARC-1348 |
