# ARC-1348 Implement `runbook_find_next_story` OpenCode tool — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1348-tool-runbook-find-next-story.md`
**Completion Summary:** `task-completions/ARC-1348-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single standalone TypeScript tool file registered as an OpenCode plugin, reusing the existing AST-based runbook parser from `pa.aid.conductor.ts`
**Owner:** agent
**Date:** 2026-07-02

---

## Scope & Alignment

This plan delivers a single TypeScript file at `~/.config/opencode/tools/runbook-tools.ts` that is loadable by OpenCode as a plugin, exposing one tool named `runbook_find_next_story`. The tool parses a runbook markdown file and returns the first unchecked, unclaimed story in wave order.

The tool MUST reuse the existing AST-based parser from `/repos/pa.aid.conductor.ts/packages/server/src/runbook/` rather than re-implementing wave, story, risk, or claim detection from scratch.

AC mapping:

| AC | Covered by |
|----|-----------|
| Returns first unchecked AND unclaimed story | Steps 3–4: use `parseRunbook()` + navigate structure |
| Claimed stories (`[x] 🔒 Claimed:`) are skipped | Step 4: claim-detection via `subSteps` |
| All stories checked/claimed → returns null | Step 4: exhaustion case |
| Runbook file not found → returns error | Step 3: `parseRunbook()` error handling |

---

## Assumptions & Dependencies

- OpenCode version 1.17.11 with `@opencode-ai/plugin` 1.17.11 is installed globally. The package is already present at `~/.config/opencode/node_modules/@opencode-ai/plugin/`.
- OpenCode loads plugins declared in the `"plugin"` array of `~/.config/opencode/opencode.json`. A plugin module must export a named `server` export of type `Plugin` (async function returning `Hooks`).
- The tool file is a TypeScript ES module. OpenCode uses Bun under the hood and can execute `.ts` files directly without a separate compile step.
- The existing parser at `/repos/pa.aid.conductor.ts/packages/server/src/runbook/parser.ts` provides `parseRunbook(filePath: string): ParsedRunbook` which reads the file from disk and returns a fully typed structure. The tool does not need to re-implement any parsing logic.
- The `~/.config/opencode/tools/` directory does not yet exist; it must be created as part of deployment.
- After the tool file is placed and opencode.json is updated, OpenCode must be restarted for the plugin to load.

---

## Decisions & Assumptions

| Decision | Resolution |
|----------|-----------|
| Plugin registration mechanism | Assume `opencode.json` `"plugin"` array; test manually and adjust if OpenCode uses a `tools/` directory scan instead |
| Tilde path in `opencode.json` | Assume tilde works; if not, use absolute path `/home/zimmermann/.config/opencode/tools/runbook-tools.ts` |
| Import strategy for parser | **Copy** `parser.ts`, `astWalker.ts`, `types.ts` from `/repos/pa.aid.conductor.ts/packages/server/src/runbook/` into `~/.config/opencode/tools/lib/`. Cross-repo imports are fragile in the OpenCode plugin runtime. |
| Dependency installation | Run `bun add unified remark-parse remark-gfm` inside `~/.config/opencode/tools/` |
| `lineNumber` in output | Omit or return `null` — AST-based parser does not track line numbers per step |
| Tool file name | `runbook-tools.ts` (kebab-case, matches tools directory convention) |

---

## Implementation Steps

### Step 1: Confirm OpenCode plugin/tool registration mechanism

**Files:** `~/.config/opencode/opencode.json`, `~/.config/opencode/node_modules/@opencode-ai/plugin/dist/index.d.ts`
**Action:** Verify how OpenCode loads custom tools — confirm whether it uses the `"plugin"` array in `opencode.json`, a `tools/` directory scan, or both. Confirm the `Plugin` export shape: `export const server: Plugin = async (_input) => ({ tool: { ... } })`.

**Verification:** Read `opencode.json` and the plugin type definitions. Document the confirmed mechanism so the correct wiring step is used in Step 4.

---

### Step 2: Set up parser dependency — copy files and install dependencies

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/runbook/parser.ts`, `/repos/pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts`, `/repos/pa.aid.conductor.ts/packages/server/src/runbook/types.ts`
**Action:** Copy the three parser source files from `/repos/pa.aid.conductor.ts/packages/server/src/runbook/` into `~/.config/opencode/tools/lib/`. Then run `bun add unified remark-parse remark-gfm` in `~/.config/opencode/tools/` to install the required markdown parsing dependencies.

Cross-repo imports (absolute path imports pointing into `/repos/pa.aid.conductor.ts/`) are fragile in the OpenCode plugin runtime — the copy strategy is the decided approach.

Document the copy operation in the completion summary.

**Verification:** A minimal Bun script `import { parseRunbook } from './lib/parser'; console.log(typeof parseRunbook)` exits 0 and prints `"function"`.

---

### Step 3: Implement `runbook-tools.ts` using `parseRunbook()`

**Files:** `~/.config/opencode/tools/runbook-tools.ts`
**Action:** Implement the tool file. The `execute` function must:

1. Call `parseRunbook(runbook_path)` to get `{ metadata, waves }`. If the file does not exist, `parseRunbook` will throw — catch the error and return `"Error: runbook file not found: {runbook_path}"`.
2. Iterate `waves` in order.
3. For each wave, iterate `wave.steps` in order and find the first step where `step.checked === false`.
4. For that step, check `step.subSteps`: a step is **claimed** if any subStep has a label matching `/claimed/i` AND `subStep.checked === true`.
5. Skip claimed steps and continue to the next unchecked step.
6. Return the first unchecked AND unclaimed step as:
   ```json
   {
     "storyKey": "ARC-NNNN",
     "title": "...",
     "wave": "Wave N — Title",
     "risk": "HIGH|MEDIUM|LOW|BATCH|UNKNOWN",
     "lineNumber": null
   }
   ```
   where `risk` comes from `step.checkpointLevel` (the `CheckpointLevel` enum from the parser types).
7. If no unchecked unclaimed step exists, return:
   ```json
   { "result": null, "message": "No unclaimed work remains" }
   ```

The full module structure:
- Import `tool` from `"@opencode-ai/plugin/tool"`.
- Import `parseRunbook` from `./lib/parser`.
- Define `runbook_find_next_story` using `tool({ description, args, execute })`:
  - `description`: `"Parse a runbook markdown file and return the first unchecked, unclaimed story in wave order. Returns null if no unclaimed work remains."`
  - `args`: one field — `runbook_path: tool.schema.string().describe("Absolute path to the runbook markdown file")`
  - `execute`: calls `parseRunbook` and applies the selection logic above
- Export: `export const server = async (_input) => ({ tool: { runbook_find_next_story: runbook_find_next_story } })`.

**Verification:** File is syntactically valid TypeScript with no import errors when loaded in a Bun environment (`bun check ~/.config/opencode/tools/runbook-tools.ts` exits 0).

---

### Step 4: Wire the tool into OpenCode

**Files:** `~/.config/opencode/opencode.json`
**Action:** Based on the confirmed mechanism from Step 1, register the tool:

- **If `"plugin"` array:** Add the tool file path to the `"plugin"` array in `opencode.json`:
  ```json
  "plugin": ["~/.config/opencode/tools/runbook-tools.ts"]
  ```
  Use absolute path `/home/zimmermann/.config/opencode/tools/runbook-tools.ts` if tilde expansion is not supported.

- **If tools directory scan:** The file at `~/.config/opencode/tools/runbook-tools.ts` is already in the correct location — no `opencode.json` change needed.

**Verification:** `opencode.json` is valid JSON (`python3 -m json.tool ~/.config/opencode/opencode.json` exits 0).

---

### Step 5: Write unit tests (`bun test`)

**Files:** `~/.config/opencode/tools/runbook-tools.test.ts`
**Action:** Write a test file using Bun's built-in test runner (`import { test, expect } from "bun:test"`). Tests must cover 7 scenarios using fixture runbook markdown strings (not live files):

1. **Empty runbook / no waves** — assert returns `{ result: null }`.
2. **All stories checked** — all `[x]` steps; assert returns `{ result: null }`.
3. **First unchecked, no claim sub-item** — assert correct `storyKey`, `title`, `wave`, `risk`.
4. **First unchecked is claimed, second is not** — assert second story is returned (claimed step skipped).
5. **BATCH risk level** — step with `checkpointLevel: "BATCH"`; assert `risk: "BATCH"` in output.
6. **No unchecked unclaimed step exists (all claimed)** — assert returns `{ result: null, message: "No unclaimed work remains" }`.
7. **Claim detected via subStep `checked: true` with label matching `/claimed/i`** — assert story is skipped.

For tests that need `parseRunbook` to process fixture markdown without real files, either:
- Write fixture markdown to a temp file using `Bun.write(tmpFile, fixtureMarkdown)` and call `parseRunbook(tmpFile)`, or
- Extract `findNextStory(parsedRunbook: ParsedRunbook)` as a pure function and test it directly.

**Verification:** `bun test ~/.config/opencode/tools/runbook-tools.test.ts` exits 0 with all 7 tests passing.

---

### Step 6: End-to-end smoke test

**Files:** Live runbook at `/repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md`
**Action:** Invoke the tool against the live runbook:
```
opencode run --pure=false "Call runbook_find_next_story with runbook_path /repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md"
```
Confirm the tool is invoked without a "tool not found" error and returns a JSON result. The expected `storyKey` is `ARC-1349` (the first unclaimed story after ARC-1348 is claimed).

**Verification:** OpenCode invokes the tool; response includes JSON with `storyKey: "ARC-1349"`, `wave`, and `risk` fields.

---

### Step 7: Update `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh`

**Files:** `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
**Action:** In the directory symlink loop (around line 108), add `tools` to the list of directories that are symlinked from `~/.config-src/pa.aid.config.md/` to `~/.config/opencode/`.

Change:
```bash
for _dir in agents bin rules; do
```
To:
```bash
for _dir in agents bin rules tools; do
```

This ensures that when `opencode-sync-config.sh` runs, it creates the symlink `~/.config/opencode/tools/` → `~/.config-src/pa.aid.config.md/tools/` using the same backup-and-symlink pattern already used for `agents`, `bin`, and `rules`.

**Verification:** Diff shows only the one-line change in the `for` loop. Script syntax is valid (`bash -n /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` exits 0).

---

### Step 8: Commit both changes

**Action:** Commit the tool file and the sync script change with appropriate messages:

```bash
# Commit the tool file to pa.aid.config.md or directly to opencode config
git -C /repos/pa.aid.conductor.ts add ... && git -C /repos/pa.aid.conductor.ts commit -m "feat(agent-tools): add runbook_find_next_story OpenCode tool (ARC-1348)"

# Commit the wsl-setup.sh change
git -C /repos/pa.aid.wsl-setup.sh add components/opencode/opencode-sync-config.sh
git -C /repos/pa.aid.wsl-setup.sh commit -m "feat(opencode): add tools/ directory to opencode-sync-config symlink loop"
```

---

## Testing & Validation

**Unit tests (`runbook-tools.test.ts`):**
Run `bun test ~/.config/opencode/tools/runbook-tools.test.ts`. All 7 scenarios must pass.

**Type check:**
Run `bun check ~/.config/opencode/tools/runbook-tools.ts` (or `tsc --noEmit`). No type errors.

**JSON validity:**
Run `python3 -m json.tool ~/.config/opencode/opencode.json` to confirm the config is valid after Step 4.

**Plugin load smoke test:**
Run `opencode run --pure=false "List available tools"` (or equivalent) and confirm `runbook_find_next_story` appears in the tool list.

**End-to-end call:**
Call the tool against the live runbook file and confirm it returns `storyKey: "ARC-1349"`.

---

## Risks & Open Questions

1. **Parser file staleness:** The copied `parser.ts`, `astWalker.ts`, and `types.ts` files in `~/.config/opencode/tools/lib/` will not automatically stay in sync with upstream changes in `pa.aid.conductor.ts`. If the parser interface changes, the tool files must be re-copied manually.

2. **Plugin path resolution:** It is not yet confirmed whether the `"plugin"` array in `opencode.json` supports tilde (`~`) in paths. If not, use the absolute path. Verify at Step 4.

3. **Bun TypeScript execution:** OpenCode uses Bun, which can execute `.ts` files natively. If the binary requires compilation, the tool file must be pre-compiled to `.js`. Verify by checking whether `opencode` binary is a Bun executable.

4. **`CheckpointLevel` enum values:** The `risk` field in the output uses `step.checkpointLevel` from the parser types. Confirm the enum values (`HIGH`, `MEDIUM`, `LOW`, `BATCH`) match what the runbook format produces. If the parser returns different strings, map them in the tool's execute function.

5. **Re-registration on restart:** After updating `opencode.json`, any live OpenCode session must be restarted to pick up the new plugin. This is expected behavior — document in the completion summary.

---

## WSL Setup Integration

This section describes deployment changes needed in `/repos/pa.aid.wsl-setup.sh` to make the `tools/` directory a first-class citizen of the OpenCode configuration setup, alongside `agents/`, `bin/`, and `rules/`.

### Context

The `opencode-sync-config.sh` script symlinks subdirectories from `~/.config-src/pa.aid.config.md/` into `~/.config/opencode/`. Currently it handles `agents`, `bin`, and `rules`. Adding `tools` to this loop means:

- Source location: `~/.config-src/pa.aid.config.md/tools/runbook-tools.ts`
- Symlink target: `~/.config/opencode/tools/` → `~/.config-src/pa.aid.config.md/tools/`

### For this story (ARC-1348)

The tool file `runbook-tools.ts` will be created directly at `~/.config/opencode/tools/runbook-tools.ts` during development and testing. The `pa.aid.config.md` integration is a deployment concern — the tool should be moved to `pa.aid.config.md/tools/` after it is validated, as a separate housekeeping step.

### Change: `opencode-sync-config.sh`

**File:** `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
**Line ~108 — change:**

```bash
# Before
for _dir in agents bin rules; do

# After
for _dir in agents bin rules tools; do
```

This uses the existing backup-and-symlink pattern (already implemented in the loop body) — no other changes to the script are needed.

### Dependency installation

**Decided strategy (copy):** Copy `parser.ts`, `astWalker.ts`, `types.ts` from `/repos/pa.aid.conductor.ts/packages/server/src/runbook/` into `~/.config/opencode/tools/lib/`, then install dependencies inside the tools directory:
```bash
cd ~/.config/opencode/tools && bun add unified remark-parse remark-gfm
```
This can be added to `opencode-install.sh` or a new `opencode-tools-install.sh` component in `pa.aid.wsl-setup.sh`.

### Summary of files to change in `pa.aid.wsl-setup.sh`

| File | Change |
|------|--------|
| `components/opencode/opencode-sync-config.sh` | Add `tools` to the `for _dir in` loop (one word, one line) |
| `components/opencode/opencode-install.sh` | Add `bun add` step for unified/remark-parse/remark-gfm |

---

After writing the file, commit it:
```bash
git -C /repos/pa.aid.runbook-executor add implementation_plans/agent-tools/ARC-1348-implementation-plan.md
git -C /repos/pa.aid.runbook-executor commit -m "docs(agent-tools): update ARC-1348 plan — reuse conductor parser, add wsl-setup integration"
```

Return:
- Confirmation the file was written
- The commit hash from the git commit output