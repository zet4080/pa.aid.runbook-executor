# ARC-1363 Implement `generate_runbook` OpenCode Tool - Implementation Plan

**Issue:** `issues/agent-tools/ARC-1363-tool-generate-runbook.md`
**Completion Summary:** `task-completions/ARC-1363-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — input schema fully specified in issue; parser-first inverse (generate → validate with `parseRunbook()` before writing)
**Owner:** agent-tools
**Date:** 2026-07-03

---

## Scope & Alignment

This plan implements `~/.config/opencode/tools/generate_runbook.ts` — a tool that accepts structured lane/wave/story data and writes a conformant runbook markdown file. It also updates the `generate-lane-runbooks` skill to call this tool instead of writing markdown by hand.

| Acceptance Criterion | Covered By |
|---------------------|-----------|
| AC1: Valid input writes file, returns `{ success, path, story_count, wave_count }` | Steps 1–4 |
| AC2: HIGH story (🔴) includes INDIVIDUAL PLAN CHECKPOINT sub-item | Step 2 |
| AC3: Stories with dependencies render `🔒 Blocked by:` sub-items | Step 2 |
| AC4: Unchecked stories include `🔒 Claimed: —` sub-item | Step 2 |
| AC5: `checkpoint_count` is reflected correctly in the summary table | Steps 2–3 |
| AC6: Missing required fields return `{ success: false, errors }` without writing | Step 4 |
| AC7: Output passes `validate_runbook` with zero errors | Steps 5–6 |
| Skill update: `generate-lane-runbooks` calls the tool | Step 7 |

---

## Assumptions & Dependencies

- `validate_runbook.ts` (ARC-1362) is complete and its exported `validateRunbook()` function is importable from `../validate_runbook.ts` within the tests directory.
- `parseRunbook()` from `./lib/parser.ts` accepts a file path; to validate in-memory generated content, the generator writes to a temp file, calls `parseRunbook()`, then moves the file to `output_path` — OR validates using `unified().use(remarkParse)` directly (same approach as `validate_runbook.ts`) without round-tripping through disk.
- The parser expects the H1 title to contain an ARC key. The input `epic_key` is used in the title.
- The `tool.schema` Zod helper supports `.array()` and `.object()` for nested input schemas.
- Node's `fs.writeFileSync` / `fs.mkdirSync` are available in the Bun runtime (confirmed for other tools).
- All paths in the tool are Unix-style absolute paths (WSL constraint).
- The `generate-lane-runbooks` skill file is at `~/.config/opencode/skills/generate-lane-runbooks/SKILL.md`.

---

## Implementation Steps

### Step 1: Define the input schema and return types

**Files:** `~/.config/opencode/tools/generate_runbook.ts`

**Action:** Define the Zod schema using `tool.schema` to match the issue's input spec exactly:
- Top-level: `lane_name` (string), `epic_key` (string), `output_path` (string), `waves` (array)
- Each wave: `wave_number` (number), `stories` (array)
- Each story: `key`, `title`, `priority` (enum: `🔴|🟡|🟢`), `status` (enum: `pending|claimed|done`), optional `claimed_by`, optional `dependencies` (string array), optional `checkpoint_count` (number), optional `notes` (string)

Define a TypeScript interface `GenerateRunbookResult` with fields: `success: boolean`, `path?: string`, `story_count?: number`, `wave_count?: number`, `errors?: string[]`.

**Verification:** TypeScript type-checks with no errors when run with `node --experimental-strip-types`.

---

### Step 2: Implement `buildRunbookMarkdown(input)`

**Files:** `~/.config/opencode/tools/generate_runbook.ts`

**Action:** Implement a pure function `buildRunbookMarkdown(input)` → `string` that assembles the runbook in sections. The output must match the exact markdown structure the astWalker parses:

**Section order:**
1. **H1 title**: `# runbook-{kebab(lane_name)} — {epic_key}` — must contain the ARC epic key so `metadata.title` passes the ARC key check in `validateMetadata`.
2. **Metadata block** (paragraph with bold fields, one per line):
   - `**Lane:** {lane_name}`
   - `**Epic:** {epic_key}`
   - `**Depends on:** none`
   - `**Feature goal:** {lane_name} runbook`
   - `**Repos:** pa.aid.runbook-executor`
   - `**Skills to load at start:** execute-implementation-plan`
3. **Checkpoint Summary section** (`## Checkpoint Summary`): markdown table with columns `Type | Count`, rows for `HIGH`, `BATCH`, `WAVE`. Counts are derived from stories: HIGH count = sum of `checkpoint_count` for stories with `priority: "🔴"` (defaulting to 1 if `checkpoint_count` is absent but story is HIGH); BATCH = 0 (no batch stories in this generator's model); WAVE = number of waves (one gate per wave).
4. **Wave sections** (`## Wave {N} — stories`): one per wave entry in `input.waves`, in `wave_number` order.
5. For each wave, for each story: emit an **H3 heading** (`### {priority_emoji} {priority_label} — {key}: {title}`) then a **list** of checkbox items:
   - Story item: `- [ ] **{key}** — {title}` (or `- [x]` if `status: "done"`)
   - Sub-items (indented 2 spaces, `  - [ ]`):
     - `🔒 Claimed: —` if `status: "pending"`, `🔒 Claimed: {claimed_by}` if `status: "claimed"`, `🔒 Claimed: ✅` if `status: "done"`
     - For each dependency: `🔒 Blocked by: {dep_key}`
     - If `priority: "🔴"`: `🔴 INDIVIDUAL PLAN CHECKPOINT` sub-item
     - If `notes` is set: appended as a plain sub-item
6. **On-Hold Register section** (`## On-Hold Register`): fixed content `\nNo items on hold.\n`

**Critical structural rules** (derived from astWalker):
- Wave H2 must match `/^wave\s+\d+/i` — use exact form `Wave {N}` (e.g. `Wave 1 — Stories`).
- Story step label must contain the ARC key as a word boundary match for `extractStepId` to assign the ARC key as step ID.
- Sub-items are nested list items under the story item (indented 2 spaces with `  - [ ]` syntax).
- The H1 must contain an ARC key matching `/ARC-\d+/i` — embed `epic_key` in the title.

**Verification:** Calling `buildRunbookMarkdown` with a minimal valid input produces a string containing `## Wave 1`, a story list item, and a `🔒 Claimed:` sub-item. Manually inspect output in test.

---

### Step 3: Implement `computeCheckpointCounts(waves)`

**Files:** `~/.config/opencode/tools/generate_runbook.ts`

**Action:** Implement a helper `computeCheckpointCounts(waves)` → `{ high: number, batch: number, wave: number }`. Rules:
- `high`: sum over all stories with `priority === "🔴"` of `story.checkpoint_count ?? 1`
- `batch`: always 0 (this generator does not model batch checkpoints)
- `wave`: `waves.length` (one wave gate per wave)

Used by `buildRunbookMarkdown` to populate the `## Checkpoint Summary` table.

**Verification:** Unit test verifies that two HIGH stories (one with `checkpoint_count: 2`, one without) yield `high: 3`, `wave: 1`.

---

### Step 4: Implement `validateInput(input)` and the main `generateRunbook(input)` function

**Files:** `~/.config/opencode/tools/generate_runbook.ts`

**Action:**

`validateInput(input)` → `string[]`: Return a list of error strings for any of the following conditions:
- `lane_name` is missing or empty
- `epic_key` is missing or empty
- `waves` is missing, not an array, or is empty (length 0)

`generateRunbook(input)` → `Promise<GenerateRunbookResult>`:
1. Call `validateInput`; if errors, return `{ success: false, errors }` without writing any file.
2. Call `buildRunbookMarkdown(input)` to get the markdown string.
3. Write the markdown string to a temp file path (`{output_path}.tmp`), then call `parseRunbook(tmpPath)` from `./lib/parser.ts` to confirm parsability. If `parseRunbook` throws, return `{ success: false, errors: ["Generated markdown failed to parse: {msg}"] }` and delete the temp file.
4. Use `fs.mkdirSync(dirname(output_path), { recursive: true })` to ensure the output directory exists.
5. Write the final markdown string to `output_path` using `fs.writeFileSync`.
6. Delete the temp file.
7. Count total stories across all waves and return `{ success: true, path: output_path, story_count, wave_count: input.waves.length }`.

**Verification:** Calling `generateRunbook` with a valid input produces a file on disk. Calling with missing `epic_key` returns `{ success: false }` without creating a file.

---

### Step 5: Wire the tool export

**Files:** `~/.config/opencode/tools/generate_runbook.ts`

**Action:** Export the tool using `export default tool({...})` with:
- `description`: concise description of purpose, input, and return shape
- `args`: full Zod schema from Step 1 (using `tool.schema` builder)
- `execute(args)`: calls `generateRunbook(args)`, returns `JSON.stringify(result, null, 2)`

**Verification:** File type-checks cleanly. The tool name in OpenCode will be `generate_runbook` (filename-derived auto-discovery).

---

### Step 6: Write tests

**Files:** `~/.config/opencode/tools/tests/generate_runbook.test.ts`

**Action:** Write tests using `node:test` + `node:assert/strict` (same pattern as `validate_runbook.test.ts`). Use a `writeTmpDir()` helper to provide a temp output path. Import `generateRunbook` and `buildRunbookMarkdown` from `../generate_runbook.ts`. Import `validateRunbook` from `../validate_runbook.ts` for the conformance oracle test.

Test scenarios (7 tests):

1. **Valid minimal input writes file and returns success** — single wave, single story, `priority: "🟢"`, `status: "pending"`. Assert `result.success === true`, file exists on disk, `story_count === 1`, `wave_count === 1`.

2. **HIGH story includes INDIVIDUAL PLAN CHECKPOINT sub-item** — single story with `priority: "🔴"`, `checkpoint_count: 1`. Call `buildRunbookMarkdown` and assert output string contains `INDIVIDUAL PLAN CHECKPOINT`.

3. **Story with dependencies renders Blocked-by sub-items** — story with `dependencies: ["ARC-0001", "ARC-0002"]`. Assert output contains `🔒 Blocked by: ARC-0001` and `🔒 Blocked by: ARC-0002`.

4. **Unchecked story has Claimed sub-item** — story with `status: "pending"`. Assert output contains `🔒 Claimed: —`.

5. **Checkpoint count in summary table matches input** — two HIGH stories, `checkpoint_count: 2` and `checkpoint_count: 1`. Assert summary table row for HIGH shows count `3`.

6. **Missing required field returns error without writing file** — call with `epic_key: ""`. Assert `result.success === false`, `result.errors` non-empty, file does NOT exist at `output_path`.

7. **Output passes validate_runbook with zero errors** — generate a full valid runbook, then call `validateRunbook(result.path)`. Assert `validation.valid === true` and `validation.errors.length === 0`.

**Verification:** Run all 7 tests with `node --experimental-strip-types --test ~/.config/opencode/tools/tests/generate_runbook.test.ts`; all pass.

---

### Step 7: Update `generate-lane-runbooks` skill

**Files:** `~/.config/opencode/skills/generate-lane-runbooks/SKILL.md`

**Action:** Update the skill's instruction for generating runbook markdown to call the `generate_runbook` tool instead of writing markdown by hand. The skill should:
- Describe the tool's input schema (refer agents to the `generate_runbook` tool with structured data)
- Remove or replace any manual markdown formatting instructions that the tool now handles
- Keep all other skill content (when to use, workflow steps, etc.) intact

**Verification:** Read the updated skill and confirm the manual markdown formatting section is replaced with a call to `generate_runbook`.

---

### Step 8: Add tool to `opencode.json` allowlists

**Files:** `~/.config/opencode/opencode.json`

**Action:** Add `generate_runbook` to the `"tools"` allowlist for the `build` and `senior-coder` agents in `~/.config/opencode/opencode.json`. Follow the same pattern as existing tool entries in the agent sections.

Note: The long-term source of truth is `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh`. The direct `opencode.json` edit is required for immediate use on the current machine (per agent-tools technical context §2b).

**Verification:** After adding, the `generate_runbook` entry appears in the correct agent sections of `~/.config/opencode/opencode.json`. Restart OpenCode to activate.

---

## Testing & Validation

Run the test suite from the WSL shell:

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/generate_runbook.test.ts
```

All 7 tests must pass. The critical conformance test is Test 7 — it calls `validateRunbook()` on the generated file and asserts `valid: true, errors: []`. This is the single oracle that proves the generator output is structurally conformant with the parser.

Additionally, run the existing validate_runbook tests to confirm no regressions:

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/validate_runbook.test.ts
```

---

## Risks & Open Questions

- **`tool.schema` nested object/array support**: The issue input schema requires `waves: Array<{wave_number, stories: Array<{...}>}>`. Zod nested objects via `tool.schema` have not been exercised in this lane yet. If `tool.schema.object()` / `tool.schema.array()` are unavailable, fall back to `tool.schema.string().describe("JSON-encoded waves array")` and `JSON.parse` inside `execute`. This fallback reduces type safety but is functionally equivalent.

- **Temp file strategy for `parseRunbook()` validation**: `parseRunbook()` reads from disk. The plan uses a `.tmp` file to pre-validate before writing to the final path. If the Bun runtime restricts `fs.renameSync` across mount points (unlikely in WSL), write directly to `output_path` and delete on parse failure instead.

- **Skill update scope**: The `generate-lane-runbooks` skill may have extensive manual formatting instructions. If the skill is large, the update in Step 7 should preserve all non-formatting guidance (when to use, which runbooks to generate, wave/story ordering) and only replace the "write the markdown like this" section with "call `generate_runbook` with this schema".
