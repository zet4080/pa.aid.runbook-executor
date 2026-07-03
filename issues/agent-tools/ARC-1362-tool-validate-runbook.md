| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1362 |

# ARC-1362: Implement `validate_runbook` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/validate_runbook.ts` that validates a runbook markdown file against the exact structural rules required by the `pa.aid.conductor.ts` parser (`packages/server/src/runbook/parser.ts` + `astWalker.ts`) and the `generate-lane-runbooks` skill quality checklist — so the agent writing or editing a runbook can verify it is parse-ready before committing.

## Problem

Runbooks are written by agents following the `generate-lane-runbooks` skill, then consumed by the `pa.aid.conductor.ts` parser at runtime. A malformed runbook causes silent failures: wrong step IDs, missing waves, uncounted checkpoints, or stories that never become claimable. Agents have no way to verify format correctness short of running the full server.

The parser's requirements are non-obvious and derived from `astWalker.ts` source logic:
- Metadata is extracted from a **bold paragraph block** (not frontmatter), using regex patterns like `/\*{0,2}Epic:\*{0,2}\s*(.+)/i`
- Wave sections must start with `## Wave N` (matched by `/^wave\s+\d+/i`)
- Checkpoint classification is determined by **item label keywords** (`CHECKPOINT`, `GATE`) — not by the H3 heading
- Step IDs are derived from `**ARC-XXXX**` keys in item labels; duplicates fall back to positional IDs, corrupting step state
- The `🔒 Claimed:` sub-item must exist on every unchecked story for `runbook_claim_story` to work

## Acceptance Criteria

**Given** a fully valid runbook file,
**When** `validate_runbook({ runbook_path })` is called,
**Then** the tool returns `{ valid: true, errors: [], warnings: [], summary: "✅ Runbook is valid." }`.

---

**Given** a runbook missing the `**Lane:**` bold paragraph field,
**When** `validate_runbook` is called,
**Then** `errors` contains `{ rule: "metadata.lane", message: "**Lane:** field missing from metadata block" }` and `valid` is `false`.

---

**Given** a runbook with an H2 heading that does not match `## Wave N`,
**When** `validate_runbook` is called,
**Then** `warnings` contains `{ rule: "wave.heading", message: "H2 heading '...' will not be parsed as a wave — must match '## Wave N'" }`.

---

**Given** a runbook with two top-level story items sharing the same `ARC-XXXX` key,
**When** `validate_runbook` is called,
**Then** `errors` contains `{ rule: "step.duplicate_id", message: "Duplicate step key ARC-XXXX — second occurrence will get a positional fallback ID, corrupting step state" }`.

---

**Given** an unchecked story item (`- [ ] **ARC-XXXX**`) with no `🔒 Claimed:` sub-item,
**When** `validate_runbook` is called,
**Then** `errors` contains `{ rule: "story.missing_claimed", message: "Story ARC-XXXX has no '🔒 Claimed:' sub-item — runbook_claim_story tool will fail" }`.

---

**Given** a 🔴 HIGH story missing the `🔴 INDIVIDUAL PLAN CHECKPOINT` sub-item,
**When** `validate_runbook` is called,
**Then** `warnings` contains `{ rule: "story.missing_checkpoint", message: "HIGH story ARC-XXXX has no INDIVIDUAL PLAN CHECKPOINT step" }`.

---

**Given** the checkpoint summary table at the bottom is missing or counts do not match the actual body,
**When** `validate_runbook` is called,
**Then** `warnings` contains `{ rule: "summary.count_mismatch", message: "Checkpoint summary table missing or counts don't match body" }`.

---

**Given** the runbook file does not exist,
**When** `validate_runbook` is called,
**Then** returns `{ valid: false, errors: [{ rule: "file", message: "File not found: ..." }], warnings: [], summary: "❌ File not found." }`.

## Validation Rules

### Errors (parser will fail or produce wrong results)

| Rule ID | What is checked |
|---------|----------------|
| `metadata.title` | H1 heading exists and contains an ARC key |
| `metadata.epic` | `**Epic:**` bold field present and non-empty in paragraph block |
| `metadata.lane` | `**Lane:**` bold field present and non-empty in paragraph block |
| `metadata.depends_on` | `**Depends on:**` bold field present (value can be "none") |
| `wave.exists` | At least one `## Wave N` H2 heading present |
| `wave.heading` | Every H2 heading either matches `## Wave N` or is an On-Hold/Summary section — anything else is unexpected |
| `step.duplicate_id` | No two top-level story items share the same `**ARC-XXXX**` key across the whole runbook |
| `story.missing_claimed` | Every unchecked story (`- [ ] **ARC-XXXX**`) has at least one `🔒 Claimed:` sub-item |

### Warnings (skill quality violations, non-fatal to parser)

| Rule ID | What is checked |
|---------|----------------|
| `metadata.feature_goal` | `**Feature goal:**` field present |
| `metadata.repos` | `**Repos:**` field present |
| `metadata.skills` | `**Skills to load at start:**` field present |
| `story.missing_checkpoint` | Every story whose H3 heading contains 🔴 HIGH has an `INDIVIDUAL PLAN CHECKPOINT` sub-item |
| `wave.missing_gate` | Each wave section ends with a Wave gate sub-section |
| `structure.missing_onhold` | On-Hold Register table present |
| `structure.missing_summary` | Checkpoint summary table present at end of file |
| `summary.count_mismatch` | If summary table is present, HIGH/batch/gate counts match actual counts in body |

## Implementation Note

Validation operates on the `ParsedRunbook` data model (metadata, waves, steps, sub-steps) and NOT on raw text. The tool imports `parseRunbook()` from `~/.config/opencode/tools/lib/parser.ts` which returns a `ParsedRunbook` containing extracted metadata fields and parsed wave/step structures. Raw regex scanning of runbook text is used only as a supplement for structural checks below the `ParsedRunbook` surface level (e.g., detecting non-wave H2 headings that the AST walker skips).

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/validate_runbook.ts`
- Validates against parser rules derived from `pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts`
- **Approach:** Use `parseRunbook()` from `./lib/parser.ts` to produce a `ParsedRunbook` AST. Validate the structured AST fields against the rule table. For rules that check markdown elements the parser silently ignores (e.g. non-wave H2 headings), use the mdast directly via `unified().use(remarkParse).parse(content)` alongside the structured parse.
- Returns: `{ valid: boolean, errors: RuleViolation[], warnings: RuleViolation[], summary: string }`
  where `RuleViolation = { rule: string, message: string, line?: number }`
- Reports line numbers where possible

## Out of Scope

- Running the full `pa.aid.conductor.ts` server (runtime server dependency)
- Validating issue file content or implementation plans
- Fixing the runbook (tool is read-only)
- Validating semantic correctness of commit messages or feature goal wording

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Input: `runbook_path` (absolute path to runbook `*.md` file)
- Read-only — no side effects
- Must align with parser logic in `pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts` — if the parser changes, this tool must be updated to match

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Parser source reference: `pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts` and `parser.ts`
- Skill reference: `generate-lane-runbooks` quality checklist (last section of `~/.config/opencode/skills/generate-lane-runbooks/SKILL.md`)
- Used by: `generate-lane-runbooks` skill (after generation), `write-implementation-plan` skill (before execution begins)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1362-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1362-COMPLETION-SUMMARY.md` | TBD |
