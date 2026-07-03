# ARC-1362 Implement `validate_runbook` OpenCode tool - Implementation Plan

**Issue:** `issues/agent-tools/ARC-1362-tool-validate-runbook.md`
**Completion Summary:** `task-completions/ARC-1362-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Parser-first — validate via ParsedRunbook AST + raw mdast for sub-parser-level checks
**Owner:** build
**Date:** 2026-07-03

---

## Scope & Alignment

This plan covers creating `~/.config/opencode/tools/validate_runbook.ts` — a read-only OpenCode custom tool that validates a runbook markdown file by parsing it through `parseRunbook()` from `./lib/parser.ts` (same shared parser used by `pa.aid.conductor.ts`) and additionally walking the raw mdast for checks below the `ParsedRunbook` surface.

AC mapping:
- AC 1 (valid runbook → `{ valid: true, errors: [], warnings: [] }`) → Steps 2–5
- AC 2 (missing `**Lane:**` field → error `metadata.lane`) → Step 2
- AC 3 (H2 not matching `## Wave N` → warning `wave.heading`) → Step 3
- AC 4 (duplicate `ARC-XXXX` key → error `step.duplicate_id`) → Step 3
- AC 5 (unchecked story missing `🔒 Claimed:` → error `story.missing_claimed`) → Step 3
- AC 6 (HIGH story missing INDIVIDUAL PLAN CHECKPOINT → warning `story.missing_checkpoint`) → Step 3
- AC 7 (summary table missing/count mismatch → warning `summary.count_mismatch`) → Step 4
- AC 8 (file not found → `{ valid: false, errors: [{ rule: "file", ... }] }`) → Step 1

---

## Assumptions & Dependencies

- `@opencode-ai/plugin` is available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` (confirmed ARC-1348)
- `parseRunbook()` is available at `~/.config/opencode/tools/lib/parser.ts` and returns a `ParsedRunbook` (confirmed ARC-1348)
- `unified` and `remark-parse` are installed in `~/.config/opencode/tools/node_modules/` (confirmed ARC-1348)
- The tool imports `parseRunbook` from `"./lib/parser.ts"` and `type { ParsedRunbook }` from `"./lib/types.ts"`
- Allowlist entry for `validate_runbook` must be added to `opencode-sync-config.sh` for `plan`, `build`, and `senior-coder` agents (per technical-context.md §2b)
- The tool is read-only — no file writes, no git operations

---

## Implementation Steps

### Step 1: Create `validate_runbook.ts` tool scaffold

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

Create the tool file following the standard pattern (`export default tool({...})`). Add imports:
- `import { parseRunbook } from "./lib/parser.ts"`
- `import type { ParsedRunbook } from "./lib/types.ts"`
- `import { unified } from "npm:unified"`
- `import remarkParse from "npm:remark-parse"`

Define the `RuleViolation` interface: `{ rule: string; message: string; line?: number }`.
Define the `ValidationResult` interface: `{ valid: boolean; errors: RuleViolation[]; warnings: RuleViolation[]; summary: string }`.

The tool's `args` schema has one field: `runbook_path: tool.schema.string()`.

The `execute` function:
1. Reads the file with `Bun.file(runbook_path).text()`. Wraps in try/catch — on failure returns `{ valid: false, errors: [{ rule: "file", message: "File not found: {runbook_path}" }], warnings: [], summary: "❌ File not found." }`.
2. Delegates to `validateRunbook(content)` helper (exported named function defined in the same file).
3. Returns `JSON.stringify(result, null, 2)`.

Also exports `validateRunbook` as a named export for unit test access.

**Verification:** File exists; no TypeScript parse errors on load.

---

### Step 2: Implement metadata validation using ParsedRunbook (errors)

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper `validateMetadata(parsed: ParsedRunbook, raw: string): RuleViolation[]`:

Validate using `parsed.metadata.*` fields populated by `parseRunbook()`:

- `metadata.title`: Check that `parsed.metadata.title` is non-empty and matches `/ARC-\d+/i`. If absent or no ARC key, push error. Line number: scan raw string for first `# ` line.
- `metadata.epic`: Check that `parsed.metadata.epic` is non-null and non-empty. If absent, push error.
- `metadata.lane`: Check that `parsed.metadata.lane` is non-null and non-empty. If absent, push error.
- `metadata.depends_on`: Check that `parsed.metadata.depends_on` is not undefined (an empty array `[]` is valid — the field must be present in source). If undefined, push error.

The `parsed.metadata` object is the source of truth for these checks. Do NOT re-scan the raw text with regex for fields that are already in the `ParsedRunbook` model.

**Verification:** Unit test: runbook missing `**Lane:**` field → `parseRunbook()` returns `parsed.metadata.lane === null` → exactly one error with `rule: "metadata.lane"`.

---

### Step 3: Implement wave, step, and story validation (errors + warnings)

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper `validateWavesAndSteps(parsed: ParsedRunbook, mdast: any): { errors: RuleViolation[]; warnings: RuleViolation[] }`:

**From `ParsedRunbook.waves`:**

- `wave.exists` (error): Check `parsed.waves.length > 0`. If zero, push error.
- `step.duplicate_id` (error): Collect all `step.id` values by iterating `parsed.waves[].steps[]`. Track in a `Set<string>`; on second occurrence push error naming the duplicate ID.
- `story.missing_claimed` (error): For each step across all waves where `step.checked === false`, check whether `step.subSteps` (or equivalent sub-item list from `ParsedRunbook`) contains an entry whose label matches `/claimed/i`. If none, push error with the step ID.
- `story.missing_checkpoint` (warning): For each step that is tagged HIGH (check step metadata or label for `🔴`), check whether its sub-items contain an entry matching `/INDIVIDUAL PLAN CHECKPOINT/i`. If none, push warning.
- `wave.missing_gate` (warning): For each wave in `parsed.waves`, check whether any step in that wave has a label or type matching `/gate/i`. If none, push warning for that wave.

**From raw mdast (for sub-parser-level structural checks):**

- `wave.heading` (warning): Walk all `heading` nodes with `depth === 2` in the mdast. For each H2 whose text does NOT match `/^wave\s+\d+/i` AND does NOT match `/^(on.hold|on hold|checkpoint|summary)/i`, push warning with the heading text. These H2s are silently skipped by the parser and may indicate structural errors.

**Verification:** Unit tests — duplicate step ID, unchecked story without claimed sub-item, H2 heading that is neither a wave nor a known structural section triggers `wave.heading` warning.

---

### Step 4: Implement quality/structure warnings using mdast

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper `validateStructure(parsed: ParsedRunbook, mdast: any): RuleViolation[]`:

These fields are NOT present in `ParsedRunbook.metadata` — detect them by scanning paragraph nodes in the raw mdast near the top of the document:

- `metadata.feature_goal` (warning): Walk paragraph nodes in the mdast; check whether any paragraph contains a bold node with text matching `/feature goal/i`. If absent, push warning.
- `metadata.repos` (warning): Same approach — look for bold text matching `/^repos$/i` in paragraph nodes. If absent, push warning.
- `metadata.skills` (warning): Look for bold text matching `/skills to load/i`. If absent, push warning.

Structural checks using mdast H2 headings:

- `structure.missing_onhold` (warning): Walk H2 nodes; check for any matching `/on.hold/i`. If none, push warning.
- `structure.missing_summary` (warning): Walk H2 nodes; check for any matching `/checkpoint summary/i`. If none, push warning.

Count-based check:

- `summary.count_mismatch` (warning): If a checkpoint summary section exists (found above), extract numeric counts from any markdown table rows in that section. Compare against actual counts from `parsed.waves[].steps[]` — count HIGH steps (label contains `🔴`), checkpoint steps (step type/label matches `/checkpoint|gate/i`). If counts differ, push warning.

**Verification:** Unit test: runbook with summary table reporting 3 HIGH steps but only 2 actual HIGH steps → `summary.count_mismatch` warning emitted.

---

### Step 5: Compose `validateRunbook` and wire to tool

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

Implement the exported `validateRunbook(content: string): ValidationResult` function:
1. Call `parseRunbook(content)` → `parsed: ParsedRunbook`. Wrap in try/catch — if the parser throws, return `{ valid: false, errors: [{ rule: "parse_error", message: err.message }], warnings: [], summary: "❌ Parse error." }` and store in `metadata.parse_error`.
2. Call `unified().use(remarkParse).parse(content)` → `mdast`.
3. Call `validateMetadata(parsed, content)` → metadata errors array.
4. Call `validateWavesAndSteps(parsed, mdast)` → `{ errors: waveErrors, warnings: waveWarnings }`.
5. Call `validateStructure(parsed, mdast)` → structure warnings array.
6. Combine: `errors = [...metadataErrors, ...waveErrors]`, `warnings = [...waveWarnings, ...structureWarnings]`.
7. Compute `valid = errors.length === 0`.
8. Compute `summary`: if valid, `"✅ Runbook is valid."`, else `"❌ {errors.length} error(s), {warnings.length} warning(s)."`.
9. Return `{ valid, errors, warnings, summary }`.

**Verification:** Call with a fully valid runbook fixture → `{ valid: true, errors: [], warnings: [] }`.

---

### Step 6: Write unit tests

**File:** `~/.config/opencode/tools/tests/validate_runbook.test.ts`

**Action:**

Create test file following the pattern from `tests/runbook_find_next_story.test.ts`:
- Import `node:test` and `node:assert/strict`
- Import `{ validateRunbook }` (named export) from `"../validate_runbook.ts"`
- Write one test per acceptance criterion (8 tests total):
  1. Valid runbook → `valid: true`, `errors: []`, `warnings: []`
  2. Missing `**Lane:**` field → one error with `rule: "metadata.lane"`
  3. H2 not matching `## Wave N` → one warning with `rule: "wave.heading"`
  4. Duplicate ARC key → one error with `rule: "step.duplicate_id"` naming the key
  5. Unchecked story without `🔒 Claimed:` → one error with `rule: "story.missing_claimed"`
  6. HIGH story without INDIVIDUAL PLAN CHECKPOINT → one warning with `rule: "story.missing_checkpoint"`
  7. Summary table count mismatch → one warning with `rule: "summary.count_mismatch"`
  8. Fixture string that triggers `metadata.parse_error` → `valid: false`, `errors[0].rule === "parse_error"`

Use inline fixture strings (not file reads) for all tests. Each fixture must be minimal — only the content needed to trigger the specific rule.

Run command: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/validate_runbook.test.ts`

**Verification:** All 8 tests pass.

---

### Step 7: Add `validate_runbook` to agent allowlists

**Files:**
- `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
- `~/.config/opencode/opencode.json`

**Action:**

In `opencode-sync-config.sh`, in the embedded Python `"agent"` dict, add `"validate_runbook"` to the tool lists for `build`, `senior-coder`, and `plan` agents.

Also update `~/.config/opencode/opencode.json` directly with the same addition for immediate use.

**Verification:** `grep -r "validate_runbook" /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` shows the new entry; `grep "validate_runbook" ~/.config/opencode/opencode.json` shows it in all three agent allowlists.

---

## Testing & Validation

Run the full test suite:

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/validate_runbook.test.ts
```

Additionally, run a smoke test against a real runbook file by invoking the tool in an OpenCode session:

```
validate_runbook({ runbook_path: "/repos/pa.aid.runbook-executor/docs/plans/runbook-core-infrastructure.md" })
```

Expected: returns JSON with `valid`, `errors`, `warnings`, `summary` fields; no exceptions thrown.

Run existing tool tests to verify no regression:

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_find_next_story.test.ts
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts
```

---

## Risks & Open Questions

- **ParsedRunbook interface for sub-steps**: Verify the exact field names for sub-steps/sub-items in `ParsedRunbook` by reading `~/.config/opencode/tools/lib/types.ts` before implementing step validation. If `subSteps` is named differently (e.g. `items`, `children`), adjust Step 3 accordingly.
- **`metadata.depends_on` presence check**: `parseRunbook()` may return `depends_on: []` for both "field present with empty value" and "field absent". If the parser cannot distinguish these cases, fall back to a targeted mdast paragraph scan for the `**Depends on:**` bold key specifically.
- **Summary table format variability**: Runbooks may format the checkpoint summary table differently. The `summary.count_mismatch` check must be lenient — if the table is present but its format cannot be parsed, emit a warning about unparseable format rather than throwing.
- **`wave.heading` expected exclusions**: The warning suppresses headings matching `/^(on.hold|on hold|checkpoint|summary)/i` — this list may need expansion if runbooks use other structural H2s. If so, update the exclusion pattern without changing the rule logic.
