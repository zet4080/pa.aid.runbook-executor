# ARC-1286 Parse runbook markdown into typed step graph - Implementation Plan

**Issue:** `issues/core-infrastructure/ARC-1286-parse-runbook-markdown.md`
**Completion Summary:** `task-completions/ARC-1286-COMPLETION-SUMMARY.md` (TBD)
**Approach:** unified/remark AST pipeline
**Owner:** build
**Date:** 2026-06-28

## Scope & Alignment

This plan covers the server-side markdown parser that reads `docs/plans/runbook-*.md` files and returns a typed in-memory step graph. It does not cover UI rendering, state persistence, or runbook authoring.

Acceptance criteria mapping:

- **AC 1** (🔴🟡🟢 markers + checkboxes → step with type/checkbox-state/checkpoint-level): covered by Steps 1–4. The remark-gfm AST exposes `checked` on list items natively. Step type is derived from the nearest H3 heading text; checkpoint level from emoji prefix (best-effort). Steps 3–4 build and validate this derivation logic.
- **AC 2** (no explicit type field → classify as `manual`): covered by Step 3. The type derivation function returns `"manual"` as the default when no recognised marker is found.

## Assumptions & Dependencies

- ARC-1285 is merged; the server package at `packages/server/` compiles cleanly with `tsc` and passes `vitest run`.
- `unified`, `remark-parse`, and `remark-gfm` are available on npm and compatible with the server's TypeScript/CommonJS output target (`"module": "commonjs"` in `packages/server/tsconfig.json`).
- Runbook files are located at `docs/plans/runbook-*.md` relative to the repository root of `pa.aid.runbook-executor`. The parser receives an absolute file path as input; callers are responsible for glob-resolving which files to pass.
- Emoji markers (🔴🟡🟢) appear in H3 heading text as a prefix followed by a space and `HIGH`/`MEDIUM`/`LOW`. Their absence does not prevent parsing — the step is typed `manual` and checkpoint level is `null`.
- Emoji markers in checkbox text (🔒🔔) are treated as decorative; the parser strips them when constructing step labels but does not assign semantic meaning.
- The step graph contract defined in Step 1 (types file) is the single source of truth consumed by all downstream lanes (ARC-1313 session-setup, ARC-1291 parallel-lane-execution, etc.).
- No streaming or incremental parsing is required; the parser is synchronous and reads the full file.

## Implementation Steps

---

### Step 1: Install remark dependencies and define step graph types

**Files:**
- `packages/server/package.json`
- `packages/server/src/runbook/types.ts` *(new)*

**Action:**
Add `unified`, `remark-parse`, and `remark-gfm` as runtime dependencies in `packages/server/package.json`. Add `@types/mdast` as a dev dependency for AST node types.

Create `packages/server/src/runbook/types.ts` exporting the following interfaces and union types:

- `StepType`: union `"agent" | "manual" | "checkpoint"`
- `CheckpointLevel`: union `"high" | "medium" | "low" | null`
- `RunbookStep`: object with fields `id` (string, e.g. `"ARC-1285"`), `label` (string, cleaned heading text), `type` (StepType), `checked` (boolean), `checkpointLevel` (CheckpointLevel), `subSteps` (array of `RunbookSubStep`)
- `RunbookSubStep`: object with fields `label` (string), `checked` (boolean)
- `RunbookWave`: object with fields `title` (string), `steps` (array of `RunbookStep`)
- `RunbookMetadata`: object with fields `title` (string), `epic` (string | null), `lane` (string | null), `dependsOn` (string[])
- `ParsedRunbook`: object with fields `metadata` (RunbookMetadata), `waves` (array of `RunbookWave`)

**Verification:** `tsc --noEmit` passes with the new types file in place and no import errors.

---

### Step 2: Implement the AST walker utility

**Files:**
- `packages/server/src/runbook/astWalker.ts` *(new)*

**Action:**
Create `packages/server/src/runbook/astWalker.ts` exporting two pure functions:

- `extractMetadata(root: Root): RunbookMetadata` — walks the AST to find the H1 node (runbook title) and bold key-value pairs in paragraph nodes matching the pattern `**Key:** value`. Extracts `epic`, `lane`, and `dependsOn` (split on commas from `Depends on` field). Returns `RunbookMetadata`.
- `extractWaves(root: Root): RunbookWave[]` — walks the AST section by section. Each H2 node starts a new wave; each H3 node starts a new step group within the current wave. List items at the top level of a bullet list under an H3 become `RunbookStep` entries; nested list items (one level deeper) become `RunbookSubStep` entries. Uses the GFM `checked` boolean on list items for checkbox state.

`Root` is the `mdast` root node type from `@types/mdast`.

The step type derivation logic (used inside `extractWaves`): examine the H3 heading text for the pattern `🔴 HIGH`, `🟡 MEDIUM`, or `🟢 LOW` (case-insensitive fallback: `HIGH`/`MEDIUM`/`LOW` text alone if emoji absent). Map: HIGH → `"checkpoint"`, MEDIUM → `"agent"`, LOW → `"manual"`. If none match, return `"manual"`. This satisfies AC 2.

The checkpoint level derivation: HIGH → `"high"`, MEDIUM → `"medium"`, LOW → `"low"`, none → `null`.

Step `id` extraction: scan H3 heading text for a token matching `/ARC-\d+/`; use it as `id`. If absent, derive id from position index as `"step-{waveIndex}-{stepIndex}"`.

Step `label` extraction: strip leading emoji characters and the risk keyword prefix from H3 heading text; trim whitespace.

Sub-step `label` extraction: strip leading emoji characters (🔒🔔 etc.) from list item text; trim whitespace.

**Verification:** Unit test in Step 5 exercises this module directly with a fixture file. At this stage: `tsc --noEmit` passes.

---

### Step 3: Implement the parser entry point

**Files:**
- `packages/server/src/runbook/parser.ts` *(new)*

**Action:**
Create `packages/server/src/runbook/parser.ts` exporting one function:

`parseRunbook(filePath: string): ParsedRunbook`

The function: reads the file synchronously using `fs.readFileSync`; passes the content through `unified().use(remarkParse).use(remarkGfm).parse()` to produce an `mdast` Root node; calls `extractMetadata` and `extractWaves` from `astWalker.ts`; returns a `ParsedRunbook` object.

No async variant is needed for this story; the function is intentionally synchronous because callers (ARC-1313, ARC-1291) will invoke it at session-start time when blocking is acceptable.

**Verification:** Importing `parseRunbook` from the module resolves without TypeScript errors. Manual smoke test: call `parseRunbook` against `docs/plans/runbook-core-infrastructure.md` in a short script; confirm the returned object has a non-empty `waves` array and `metadata.lane === "core-infrastructure"`.

---

### Step 4: Expose a parser API route

**Files:**
- `packages/server/src/routes/runbooks.ts` *(new)*
- `packages/server/src/index.ts` *(edit)*

**Action:**
Create `packages/server/src/routes/runbooks.ts` exporting an Express router with one endpoint:

`GET /api/runbooks` — globs `docs/plans/runbook-*.md` relative to `process.cwd()` (the `pa.aid.runbook-executor` repo root), calls `parseRunbook` for each, and returns a JSON array of `ParsedRunbook` objects. Returns HTTP 200 on success; HTTP 500 with `{ error: string }` if any file fails to parse.

Register the router in `packages/server/src/index.ts` under the existing `app.use('/api', ...)` block, alongside the health router.

The glob for runbook files uses Node `fs.readdirSync` filtered by the pattern `runbook-*.md` within the resolved `docs/plans/` directory, or `fast-glob` if already present in the project; no new glob dependency is added unless `fast-glob` is already a transitive dep.

**Verification:** `curl http://localhost:4000/api/runbooks` returns a JSON array. The array contains one entry per `runbook-*.md` file in `docs/plans/`. Each entry has `metadata.lane` set and a non-empty `waves` array.

---

### Step 5: Write Vitest tests

**Files:**
- `packages/server/src/__tests__/runbook-parser.test.ts` *(new)*
- `packages/server/src/__tests__/fixtures/runbook-minimal.md` *(new)*

**Action:**
Create a minimal fixture file at `packages/server/src/__tests__/fixtures/runbook-minimal.md`. The fixture must contain:
- An H1 title
- Bold frontmatter with `**Epic:**`, `**Lane:**`, `**Depends on:**` fields
- One H2 wave section
- Two H3 step-group headings: one with a 🔴 HIGH marker and an ARC key, one with no marker
- Three top-level checkbox list items under the first H3 (mix of checked/unchecked), each with one nested sub-item
- One top-level checkbox list item under the second H3 (no marker)

Write tests covering:

- **Metadata extraction:** parsed `metadata.lane` matches the fixture value; `metadata.epic` matches; `metadata.dependsOn` is an array.
- **Wave extraction:** `waves` array has length 1; wave `title` matches the H2 text.
- **Step typing from 🔴 HIGH marker:** first step under the HIGH H3 has `type === "checkpoint"` and `checkpointLevel === "high"`.
- **Default type (AC 2):** step under the unmarked H3 has `type === "manual"` and `checkpointLevel === null`.
- **Checkbox state:** a `- [x]` item has `checked === true`; a `- [ ]` item has `checked === false`.
- **Sub-steps:** first step's `subSteps` array length equals the number of nested items in fixture.
- **Idempotency:** calling `parseRunbook` twice on the same file returns structurally identical results.
- **Real runbook smoke test:** call `parseRunbook` on the actual `docs/plans/runbook-core-infrastructure.md` file (resolved via `path.resolve` from repo root); assert `metadata.lane === "core-infrastructure"` and `waves.length > 0`.

**Verification:** `npm run test --workspace=packages/server` exits 0. All named scenarios pass.

---

### Step 6: Wire up and verify end-to-end

**Files:**
- No new files — integration and lint pass verification only

**Action:**
Run `npm run build --workspace=packages/server` to confirm TypeScript compilation succeeds with no errors. Run `npm run test --workspace=packages/server` to confirm all tests pass. Start the server in dev mode (`npm run dev --workspace=packages/server`) and manually call `GET /api/runbooks`; verify JSON structure matches `ParsedRunbook[]` contract.

Confirm that the `parseRunbook` export from `packages/server/src/runbook/parser.ts` is importable by path `@runbook-executor/server/runbook/parser` if the monorepo workspace alias is configured, or by relative path from sibling modules — whichever pattern ARC-1285 established.

**Verification:** `tsc --noEmit`, `vitest run`, and `GET /api/runbooks` all succeed. No `any` casts in the parser or AST walker without accompanying type assertion comment.

---

## Testing & Validation

All tests run via `npm run test --workspace=packages/server` (Vitest).

**Unit scenarios (fixture-based):**
- Metadata fields extracted correctly from bold key-value pairs in H1/paragraph nodes
- Wave title matches H2 heading text after stripping wave number prefix
- Step under 🔴 HIGH H3 gets `type: "checkpoint"`, `checkpointLevel: "high"`
- Step under 🟡 MEDIUM H3 gets `type: "agent"`, `checkpointLevel: "medium"`
- Step under unmarked H3 gets `type: "manual"`, `checkpointLevel: null` (AC 2 direct coverage)
- `- [x]` list item → `checked: true`; `- [ ]` list item → `checked: false` (AC 1 direct coverage)
- Nested list items → `subSteps` array; top-level items → `steps` array
- Step `id` contains `ARC-XXXX` token when present in H3 text
- Emoji characters stripped from `label` fields
- Parser is idempotent across repeated calls

**Integration scenario (real file):**
- `parseRunbook` on `docs/plans/runbook-core-infrastructure.md` returns a `ParsedRunbook` with `metadata.lane === "core-infrastructure"` and at least one wave containing at least one step

**API scenario:**
- `GET /api/runbooks` returns HTTP 200 and a JSON array with one entry per `runbook-*.md` file

## Risks & Open Questions

- **remark CommonJS compatibility:** `unified` v11+ ships as ESM-only. The server uses `"module": "commonjs"` in tsconfig. If ESM-only imports fail at runtime, the mitigation is to pin to `unified@10` and matching `remark-parse@10`/`remark-gfm@3` which ship dual CJS/ESM builds. This must be verified during Step 1 before any other steps proceed; if the pin is needed, update `package.json` accordingly. This is the highest-risk item in the plan.
- **Emoji normalization across platforms:** emoji bytes differ across editors (variation selector U+FE0F may or may not be present). The type derivation in Step 2 should also match the text strings `HIGH`, `MEDIUM`, `LOW` without emoji to be resilient. Already accounted for in Step 2 action.
- **`docs/plans/` path resolution at runtime:** the server process must be started from the `pa.aid.runbook-executor` repo root for `process.cwd()` to resolve the glob correctly. This is a deployment constraint, not a code defect; document it in a comment in `routes/runbooks.ts`.
- **Future emoji-to-structured-syntax migration:** the workorder notes that emoji markers may be replaced by parseable structured syntax in a later story. The type derivation function in `astWalker.ts` is intentionally isolated so that a future story can swap the derivation strategy without touching `parser.ts` or the API route.
