| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-05 |
| Parent Epic | ARC-1348 — OpenCode Agent Tools |
| Jira | ARC-1351 |

# ARC-1364: Implement `generate_issue_file` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/generate_issue_file.ts` that takes structured issue data as input and emits a fully conformant issue markdown file — matching the format defined by the `create-issue` skill and established in `issues/agent-tools/`.

Agents call this tool instead of writing issue markdown by hand, guaranteeing consistent, correctly-formatted output every time — exactly the same guarantee `generate_runbook` provides for runbook files.

## Problem

The `create-issue` skill currently instructs agents to write issue markdown manually. This has two failure modes:

1. Agents produce subtly non-conformant files (wrong section order, missing frontmatter table, missing Completion Tracking table) that pass visual inspection but create inconsistencies across the issue corpus.
2. When the issue format changes, both the skill prompt AND every future issue must be updated manually — there is no single source of truth.

A generator tool encodes the format rules once. Agents supply structured data; the tool guarantees conformant output.

## Acceptance Criteria

**Given** valid input with `key`, `title`, `epic_key`, `goal`, at least one acceptance criterion, at least one in-scope item, and `output_path`,
**When** `generate_issue_file({... })` is called,
**Then** it writes a markdown file to `output_path` and returns `{ success: true, path: "..." }`.

---

**Given** a required input field (`key`, `title`, `epic_key`, `goal`, `acceptance_criteria`, `in_scope`, `output_path`) is missing or empty,
**When** `generate_issue_file` is called,
**Then** it returns `{ success: false, errors: [...] }` without writing any file.

---

**Given** each acceptance criterion is provided as a `{ given, when, then }` object,
**When** the tool renders criteria,
**Then** each criterion is rendered as a `**Given** … **When** … **Then** …` block separated by horizontal rules, matching the format used in the existing issue corpus.

---

**Given** the output directory does not exist,
**When** the tool writes the file,
**Then** it creates the directory recursively and writes the file successfully.

---

**Given** a file already exists at `output_path`,
**When** `generate_issue_file` is called with `overwrite: false` (the default),
**Then** it returns `{ success: false, errors: ["File already exists: <path>. Pass overwrite: true to replace."] }` without modifying the existing file.

## In Scope

- TypeScript tool at `~/.config/opencode/tools/generate_issue_file.ts`
- Generates all required issue sections: frontmatter table, H1 title, Goal, Acceptance Criteria, In Scope, Out of Scope, Constraints, Dependencies, Completion Tracking
- Encodes format rules from the `create-issue` skill and existing issue corpus in `issues/agent-tools/`
- Validates required inputs and returns structured errors for missing fields
- Guards against overwriting existing files unless `overwrite: true` is passed
- Returns structured result: `{ success: true, path }` or `{ success: false, errors }`

## Out of Scope

- Creating Jira stories (belongs to `jira-specialist`)
- Reading or parsing existing issue files
- Modifying existing issue files
- Generating implementation plans or completion summaries
- Semantic validation of goal wording, acceptance criteria quality, or scope correctness
- Integration with the Jira API

## Constraints

- Tool placed at `~/.config/opencode/tools/generate_issue_file.ts` (global, not in any application repo)
- Uses `@opencode-ai/plugin` `tool()` helper (same pattern as `generate_runbook.ts`)
- No runtime dependencies beyond Node.js built-ins and `@opencode-ai/plugin`
- Output format must match the format established in `issues/agent-tools/` (frontmatter table + H1 + sections)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `create-issue` skill format spec (`~/.config/opencode/skills/create-issue/SKILL.md`) defines required sections, order, and quality checklist rules

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1364-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1364-COMPLETION-SUMMARY.md` | TBD |
