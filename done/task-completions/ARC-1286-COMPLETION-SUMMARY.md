# ARC-1286 Parse runbook markdown into typed step graph — Completion Summary

**Issue:** `issues/core-infrastructure/ARC-1286-parse-runbook-markdown.md`
**Implementation Plan:** `implementation_plans/core-infrastructure/ARC-1286-implementation-plan.md`
**Completed:** 2026-06-28
**Branch:** `ARC-1286` in `pa.aid.conductor.ts`
**Commit:** `4f495cd`

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given a runbook with 🔴🟡🟢 markers and checkboxes, when parser runs, then each step has: type (agent/manual/checkpoint), checkbox state, and checkpoint level | Passed | Tests: "steps under 🔴 HIGH H3 have type checkpoint", "steps under 🔴 HIGH H3 have checkpointLevel high", "checked list item (- [x]) has checked === true", "unchecked list item (- [ ]) has checked === false" — all pass in `src/runbook/parser.test.ts`. `npm test`: 21/21 passed. |
| 2 | Given a step with no explicit type field, when parsed, then classified as manual by default | Passed | Tests: "step under unmarked H3 has type manual", "step under unmarked H3 has checkpointLevel null" — both pass in `src/runbook/parser.test.ts`. `npm test`: 21/21 passed. |

## Implementation Summary

Implemented a server-side markdown parser using the `unified`/`remark-parse`/`remark-gfm` AST pipeline to convert `docs/plans/runbook-*.md` files into typed in-memory step graphs. The server package was migrated from CommonJS to ESM output (tsconfig `NodeNext`, `"type":"module"`) to support `unified` v11+ which is ESM-only.

**Files created:**
- `packages/server/src/runbook/types.ts` — `ParsedRunbook` type contract: `StepType`, `CheckpointLevel`, `RunbookStep`, `RunbookSubStep`, `RunbookWave`, `RunbookMetadata`, `ParsedRunbook`
- `packages/server/src/runbook/astWalker.ts` — `extractMetadata` (H1 title + bold key-value frontmatter pairs) and `extractWaves` (H2→wave, H3→step group with emoji-derived type/level, GFM `checked` boolean, nested sub-steps)
- `packages/server/src/runbook/parser.ts` — synchronous `parseRunbook(filePath: string): ParsedRunbook` entry point
- `packages/server/src/routes/runbooks.ts` — `GET /api/runbooks` Express route returning `ParsedRunbook[]` for all `runbook-*.md` files in `docs/plans/`
- `packages/server/src/runbook/__fixtures__/sample-runbook.md` — minimal fixture for unit tests
- `packages/server/src/runbook/parser.test.ts` — 20 Vitest tests

**Files modified:**
- `packages/server/tsconfig.json` — `module`/`moduleResolution` → `NodeNext`, `target` → `ES2022`
- `packages/server/package.json` — added `"type":"module"`, added `unified`, `remark-parse`, `remark-gfm` runtime deps, `@types/mdast` dev dep
- `packages/server/src/index.ts` — replaced `__dirname` with `fileURLToPath(import.meta.url)`, added `.js` import extensions, registered runbooks router
- `packages/server/src/__tests__/health.test.ts` — fixed import extension to `.js` for ESM

**Downstream contract:** `ParsedRunbook` type in `src/runbook/types.ts` is the shared data contract consumed by ARC-1313 (session-setup dashboard) and ARC-1291 (parallel lane step executor).

**Type derivation:** H3 heading text containing `HIGH` → `type:"checkpoint"`, `MEDIUM` → `type:"agent"`, `LOW`/absent → `type:"manual"`. Emoji prefix (🔴🟡🟢) matched best-effort with text-only fallback for resilience across editors.

## Verification Steps

```bash
# Run from worktree (app repo feature branch)
cd /repos/ARC-1286/runbook-executor

# All tests pass
npm test --workspace=packages/server
# Expected: Test Files 2 passed (2), Tests 21 passed (21)

# Build passes clean
npm run build --workspace=packages/server
# Expected: no output (tsc exits 0)

# API endpoint (start server from pa.aid.runbook-executor repo root)
cd /repos/pa.aid.runbook-executor
node /repos/ARC-1286/packages/server/dist/index.js &
curl -s http://localhost:4000/api/runbooks | node -e "
  const d = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
  console.log('count:', d.length);
  d.forEach(r => console.log(r.metadata.lane, '| waves:', r.waves.length));
"
# Expected: 7 runbooks, each with metadata.lane set
```

## Tests Added/Modified

- `packages/server/src/runbook/parser.test.ts` *(new)* — 20 tests across 5 groups:
  - **Metadata extraction** (4 tests): title from H1, lane/epic from bold frontmatter, dependsOn as string array
  - **Wave extraction** (2 tests): wave count from H2 sections, wave title content
  - **Step typing from 🔴 HIGH marker — AC 1** (2 tests): `type:"checkpoint"`, `checkpointLevel:"high"` on HIGH-marked steps
  - **Default type for unmarked steps — AC 2** (2 tests): `type:"manual"`, `checkpointLevel:null` on unmarked steps
  - **Checkbox state — AC 1** (2 tests): `- [x]` → `checked:true`, `- [ ]` → `checked:false`
  - **Sub-steps** (3 tests): nested list items populate `subSteps`, checked state preserved
  - **Step id extraction** (1 test): ARC key extracted from H3 heading text
  - **Idempotency** (1 test): two calls on same file return identical JSON
  - **Real runbook smoke test** (3 tests): parses `runbook-core-infrastructure.md`, asserts `lane:"core-infrastructure"`, `waves.length > 0`, total steps > 0

- `packages/server/src/__tests__/health.test.ts` *(modified)* — updated import path to `.js` extension (ESM migration)

## Rollback Notes

The ESM migration (`"type":"module"` in package.json, `NodeNext` tsconfig) is a breaking change for any CJS-style `require()` consumers. No such consumers exist in this codebase. To roll back the ESM switch: revert `tsconfig.json` `module`/`moduleResolution` to `commonjs`, remove `"type":"module"` from `package.json`, restore `__dirname` in `index.ts`, remove `.js` extensions from imports, and pin `unified` to v10.

## Next Steps

- None within this story. `ParsedRunbook` contract is ready for ARC-1313 (session-setup dashboard) and ARC-1291 (step executor) to consume.
