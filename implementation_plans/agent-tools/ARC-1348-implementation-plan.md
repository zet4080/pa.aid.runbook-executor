# ARC-1348 Implement `runbook_find_next_story` OpenCode tool — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1348-tool-runbook-find-next-story.md`
**Completion Summary:** `task-completions/ARC-1348-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single standalone TypeScript tool file registered as an OpenCode plugin
**Owner:** agent
**Date:** 2026-07-02

---

## Scope & Alignment

This plan delivers a single TypeScript file at `~/.config/opencode/tools/runbook_find_next_story.ts` that is loadable by OpenCode as a plugin, exposing one tool named `runbook_find_next_story`. The tool parses a runbook markdown file and returns the first unchecked, unclaimed story in wave order.

AC mapping:

| AC | Covered by |
|----|-----------|
| Returns first unchecked AND unclaimed story | Steps 3–4: wave/story parsing + claim detection |
| Claimed stories (`[x] 🔒 Claimed:`) are skipped | Step 4: claim-detection logic |
| All stories checked/claimed → returns null | Step 4: exhaustion case |
| Runbook file not found → returns error | Step 3: file read with error handling |

---

## Assumptions & Dependencies

- OpenCode version 1.17.11 with `@opencode-ai/plugin` 1.17.11 is installed globally. The package is already present at `~/.config/opencode/node_modules/@opencode-ai/plugin/`.
- OpenCode loads plugins declared in the `"plugin"` array of `~/.config/opencode/opencode.json`. A plugin module must export a named `server` export of type `Plugin` (async function returning `Hooks`).
- The tool file is a TypeScript ES module. OpenCode uses Bun under the hood and can execute `.ts` files directly without a separate compile step.
- The runbook markdown format is the format used in `pa.aid.runbook-executor/docs/plans/runbook-*.md`: wave sections introduced by `## Wave N —`, story checkboxes as `- [ ] **KEY**` or `- [x] **KEY**`, and `🔒 Claimed:` sub-items as `  - [ ] 🔒 Claimed:` or `  - [x] 🔒 Claimed:`.
- The `~/.config/opencode/tools/` directory does not yet exist; it must be created as part of deployment.
- After the tool file is placed and opencode.json is updated, OpenCode must be restarted for the plugin to load.

---

## Implementation Steps

### Step 1: Create the `~/.config/opencode/tools/` directory

**Files:** `~/.config/opencode/tools/` (new directory)
**Action:** Create the directory that will hold all custom tool files for this epic.
**Verification:** `ls ~/.config/opencode/tools/` shows the directory.

---

### Step 2: Understand the plugin registration contract

**Files:** `~/.config/opencode/opencode.json`, `~/.config/opencode/node_modules/@opencode-ai/plugin/dist/index.d.ts`
**Action:** Confirm the plugin loading mechanism. The `opencode.json` `"plugin"` array accepts module paths. Each module must have a named export `server: Plugin` where `Plugin = (input: PluginInput, options?: PluginOptions) => Promise<Hooks>`. The `Hooks` object has a `tool` property mapping tool names to `ToolDefinition` objects (created via `tool()` from `@opencode-ai/plugin/tool`).

The tool file must:
1. Be an ES module with `export const server: Plugin = async (_input) => ({ tool: { runbook_find_next_story: tool({...}) } })`.
2. Import `tool` from `@opencode-ai/plugin/tool` (the subpath export, not the root).
3. Be referenced in `opencode.json` as a path relative to `~/.config/opencode/` or as an absolute path.

**Verification:** Read `opencode.json` to confirm the `"plugin"` field accepts string module paths; read the `Plugin` type to confirm `server` export shape.

---

### Step 3: Implement the runbook file reader and wave/story parser

**Files:** `~/.config/opencode/tools/runbook_find_next_story.ts`
**Action:** Write the parser logic that operates on the raw markdown string. The parser must:

1. Read the file at `runbook_path` using `fs/promises readFile`. If the file does not exist, catch the error and return a `ToolResult` string: `"Error: runbook file not found: {runbook_path}"`.
2. Split the content into lines.
3. Identify wave sections: lines matching `/^## Wave (\d+)/` mark the start of a wave. Track the current wave number.
4. Identify story lines: lines matching `/^- \[( |x)\] \*\*([A-Z]+-\d+)\*\*/` where:
   - `[ ]` = unchecked
   - `[x]` = checked (done)
   - The captured group 2 is the story KEY
5. Extract the story title from the same line (text after the `**KEY**` part, up to end of line, trimmed, stripping a leading em-dash `—` if present).
6. Extract the risk level by scanning forward from the story line for the `### 🔴 HIGH —` or `### 🟡 MEDIUM —` or `### 🟢 LOW —` section header that precedes the story line (the section header immediately before the story checkbox).
7. Record the 1-based line number of the story checkbox line.

**Verification:** Unit-testable by passing known markdown strings and asserting the correct wave/story/key/risk is extracted.

---

### Step 4: Implement the claim-status detector and story selector

**Files:** `~/.config/opencode/tools/runbook_find_next_story.ts`
**Action:** After parsing all story lines, apply the selection logic:

1. For each unchecked story (`[ ]`), scan the immediately following lines (indented sub-items) for a `🔒 Claimed:` sub-item. A line is a sub-item if it starts with 2 or more spaces followed by `- `.
2. A `🔒 Claimed:` sub-item is present if a sub-item line contains the text `🔒 Claimed:`.
3. The claim status is:
   - **unclaimed**: the `🔒 Claimed:` sub-item is absent, OR the sub-item line matches `- [ ] 🔒 Claimed:`.
   - **claimed**: the sub-item line matches `- [x] 🔒 Claimed:`.
4. Sub-item scanning stops at the next story checkbox line (`- [ ]` or `- [x]`) or the next wave header.
5. Return the **first** unchecked AND unclaimed story (wave order, line order within wave).
6. If no such story exists, return the JSON string: `{ "result": null, "message": "No unclaimed work remains" }`.
7. If a story is found, return the JSON string: `{ "storyKey": "ARC-NNNN", "title": "...", "wave": N, "risk": "HIGH|MEDIUM|LOW", "lineNumber": N }`.

**Verification:** Parser returns correct story given a runbook with mixed checked/claimed/unchecked stories. Returns null when all stories are done or claimed.

---

### Step 5: Wire the parser into the `tool()` definition and `server` export

**Files:** `~/.config/opencode/tools/runbook_find_next_story.ts`
**Action:** Define the final tool module structure:

- Import `tool` from `"@opencode-ai/plugin/tool"`.
- Import `readFile` from `"node:fs/promises"`.
- Define `runbook_find_next_story` using `tool({ description, args, execute })`:
  - `description`: `"Parse a runbook markdown file and return the first unchecked, unclaimed story in wave order. Returns null if no unclaimed work remains."`
  - `args`: one field — `runbook_path: tool.schema.string().describe("Absolute path to the runbook markdown file")`
  - `execute`: calls the parser from Steps 3–4, returns `ToolResult`
- Export: `export const server = async (_input) => ({ tool: { runbook_find_next_story: ... } })`.

**Verification:** The file is syntactically valid TypeScript with no import errors when loaded in a Bun environment.

---

### Step 6: Register the plugin in `opencode.json`

**Files:** `~/.config/opencode/opencode.json`
**Action:** Add the `"plugin"` array to `opencode.json` (it does not currently exist). The array entry must be the path to the tool file. Use `"~/.config/opencode/tools/runbook_find_next_story.ts"` or the absolute resolved path `/home/zimmermann/.config/opencode/tools/runbook_find_next_story.ts`.

The resulting `opencode.json` must have:
```json
"plugin": ["~/.config/opencode/tools/runbook_find_next_story.ts"]
```

Or as an absolute path if tilde expansion is not supported by the plugin loader.

**Verification:** `opencode.json` contains a `"plugin"` array with the tool file path; JSON is valid (`python3 -m json.tool ~/.config/opencode/opencode.json` exits 0).

---

### Step 7: Write the test suite for the parser

**Files:** `~/.config/opencode/tools/runbook_find_next_story.test.ts`
**Action:** Write a test file using Node's built-in `node:test` module (no external test framework required, compatible with Bun). Tests must cover:

1. **First unchecked unclaimed story returned** — runbook with Wave 1 containing one checked story, one unclaimed story; assert correct KEY, title, wave, risk, lineNumber.
2. **Claimed story skipped** — runbook where the first unchecked story has `[x] 🔒 Claimed:`; assert the second unchecked story is returned.
3. **Absent Claimed sub-item treated as unclaimed** — runbook where a story has no `🔒 Claimed:` line at all; assert it is returned (not skipped).
4. **All stories done or claimed → null** — assert `result === null`.
5. **Wave ordering respected** — Wave 2 story not returned when Wave 1 has unclaimed story.
6. **File not found → error string** — assert the result starts with `"Error: runbook file not found"`.
7. **Risk level extracted correctly** — one test per risk level (🔴 HIGH, 🟡 MEDIUM, 🟢 LOW).

**Verification:** `bun test ~/.config/opencode/tools/runbook_find_next_story.test.ts` exits 0 with all tests passing.

---

### Step 8: Validate end-to-end plugin loading

**Files:** `~/.config/opencode/opencode.json`, `~/.config/opencode/tools/runbook_find_next_story.ts`
**Action:** Verify the tool registers correctly by running:
```
opencode run --pure=false "Call runbook_find_next_story with runbook_path /repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md"
```
and confirming the tool is invoked and returns structured output (the next unclaimed story after ARC-1348 which is now claimed).

**Verification:** opencode invokes the tool without a "tool not found" error; the response includes a JSON result with `storyKey`, `title`, `wave`, `risk`, `lineNumber`.

---

## Testing & Validation

**Unit tests (`runbook_find_next_story.test.ts`):**  
Run `bun test ~/.config/opencode/tools/runbook_find_next_story.test.ts`. All 7+ scenarios must pass.

**JSON validity:**  
Run `python3 -m json.tool ~/.config/opencode/opencode.json` to confirm the config is valid after Step 6.

**Plugin load smoke test:**  
Run `opencode run --pure=false "List available tools"` (or equivalent) and confirm `runbook_find_next_story` appears in the tool list.

**End-to-end call:**  
Call the tool against the live runbook file and confirm it returns the next unclaimed story (ARC-1349 after ARC-1348 is claimed).

---

## Risks & Open Questions

1. **Plugin path resolution:** It is not yet confirmed whether the `"plugin"` array in `opencode.json` supports tilde (`~`) in paths. If not, the absolute path `/home/zimmermann/.config/opencode/tools/runbook_find_next_story.ts` must be used. Verify at Step 6.

2. **Bun TypeScript execution:** OpenCode uses Bun, which can execute `.ts` files natively. If the binary is not Bun-based and requires compilation, the tool file must be pre-compiled to `.js`. Verify by checking whether `opencode` binary is a Bun executable (`file ~/.nvm/versions/node/v24.18.0/bin/opencode`).

3. **Risk level parsing fragility:** The risk level (`🔴 HIGH`, `🟡 MEDIUM`, `🟢 LOW`) is derived from the `### ` section header before the story line. This requires backward-scanning or a two-pass approach. If a runbook section header pattern changes, the risk level will default to `"UNKNOWN"`. Document this as a known limitation.

4. **Sub-item indentation variance:** The runbook format uses 2-space indentation for sub-items (`  - [ ] 🔒 Claimed:`). If any runbook uses 4 spaces or tabs, the sub-item scanner may miss the claim. The scanner should match `/^\s+- /` (one or more leading spaces) rather than fixed 2-space indent.

5. **Re-registration on restart:** After updating `opencode.json`, any live OpenCode session must be restarted to pick up the new plugin. This is expected behavior — document in the completion summary.
