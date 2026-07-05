### Implementation Plan: ARC-1364 — `generate_issue_file`

Metadata header:
Issue: ARC-1364
Lane: agent-tools
Date: 2026-07-05

---

## Goal

Generate a conformant issue markdown file from structured input. Mirrors `generate_runbook` pattern.

---

## Phase 0: Exploration Findings

| Question | Answer |
| --- | --- |
| What is the purpose of the tool? | Generate a conformant issue markdown file from structured input. |
| What are the inputs to the tool? | key: string, title: string, epic_key: string, epic_summary: string, output_path: string, priority?: string, goal: string, problem?: string, acceptance_criteria: string, in_scope: string, out_of_scope: string, constraints?: string, dependencies?: string, overwrite?: boolean |
| What is the return type? | success: true, path: string or success: false, errors: string[] |

---

## Files

| Action | File |
| --- | --- |
| CREATE | /repos/pa.aid.runbook-executor/implementation_plans/agent-tools/ARC-1364-implementation-plan.md |
| EDIT | ~/.config/opencode/opencode.json |
| EDIT | /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh |

---

## Return Type

```typescript
export type GenerateIssueFileResult =
  | { success: true; path: string }
  | { success: false; errors: string[]; }
```

---

## Exported Helpers

- `renderFrontmatter(input)` — Generates markdown frontmatter table.
- `renderAC(criteria: Array<{given,when,then} | string>)` — Renders each `{given,when,then}` object as `**Given** {given},
**When** {when},
**Then** {then}.` block; string items rendered as-is; blocks joined by `

---

## Implementation Steps

### Step 1: Parse Inputs

- Parse JSON args: `acceptance_criteria`, `in_scope`, `out_of_scope`; optionally `constraints`, `dependencies`.

### Step 2: Validate Required Fields

- Validate required fields: `key`, `title`, `epic_key`, `goal` non-empty; `acceptance_criteria` array non-empty; `in_scope` array non-empty; `output_path` non-empty → collect all errors, return `{ success: false, errors }` if any.

### Step 3: Check File Existence

- Check `existsSync(args.output_path) &&!args.overwrite` → return `{ success: false, errors: ["File already exists: {path}. Pass overwrite: true to replace."] }` if file exists and `overwrite` is not set to `true`.

### Step 4: Build Markdown

- Build markdown:
  - Frontmatter table
  - `# {key}: {title}
  `
  - `## Goal
  
{goal}
  `
  (if `problem`) `## Problem
  
{problem}
  `
  - `## Acceptance Criteria
  
{renderAC(criteria)}
  `
  - `## In Scope
  
{renderBullets(in_scope)}
  `
  - `## Out of Scope
  
{renderBullets(out_of_scope)}
  `
  (if constraints non-empty) `## Constraints
  
{renderBullets(constraints)}
  `
  (if dependencies non-empty) `## Dependencies
  
{renderBullets(dependencies?? ["None."])}
  `
  - `## Completion Tracking
  
{renderCompletionTracking(key)}
  `

### Step 5: Create Output Directory

- `mkdirSync(dirname(output_path), { recursive: true })`

### Step 6: Write File

- `writeFileSync(output_path, markdown)`

### Step 7: Return Result

- Return `{ success: true, path: output_path }`.

---

## Testing & Validation

```bash
npm test ~/.config/opencode/tools/tests/generate_issue_file.test.ts
```