| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1363 |

# ARC-1363: Implement `generate_runbook` OpenCode tool

## Goal
Create a custom OpenCode tool at `~/.config/opencode/tools/generate_runbook.ts` that takes structured lane/wave/story data as input and emits a fully conformant runbook markdown file — identical in structure to what the `pa.aid.conductor.ts` parser expects. Agents call this tool instead of writing runbook markdown by hand.

## Problem
The `generate-lane-runbooks` skill currently instructs agents to write runbook markdown manually, following a detailed format specification in the skill prompt. This approach has two failure modes:
1. Agents misformat markdown (wrong heading syntax, missing fields, wrong checkbox format) causing silent parser failures downstream.
2. When the parser format changes, both the skill prompt AND every future runbook must be updated — there is no single source of truth.

A generator tool encodes the format rules in code once. Agents supply structured data; the tool guarantees conformant output.

## Acceptance Criteria

1. Given a valid input with `lane_name`, `epic_key`, and at least one wave containing at least one story, when the tool is called, then it writes a markdown file to `output_path` and returns `{ success: true, path: "...", story_count: N, wave_count: N }`.

2. Given a story with `priority: "🔴"`, when the tool generates the story block, then the story block includes a `🔴 INDIVIDUAL PLAN CHECKPOINT` section with the correct checkpoint count sub-item.

3. Given a story with one or more `dependencies`, when the tool generates the story block, then each dependency is rendered as a `  - 🔒 Blocked by: ARC-XXXX` sub-item under the story checkbox.

4. Given an unchecked story (not yet done), when the tool generates the story block, then the story includes a `  - 🔒 Claimed: —` sub-item.

5. Given a `checkpoint_count` is specified for a story, when the tool generates the runbook, then the checkpoint summary table at the top of the file reflects the correct totals.

6. Given a missing required field (`lane_name`, `epic_key`, or empty `waves`), when the tool is called, then it returns `{ success: false, errors: ["..."] }` without writing any file.

7. Given the tool writes a file, when `validate_runbook` (ARC-1362) is run against that file, then it returns `{ valid: true, errors: [] }`.

## Input Schema
```ts
{
  lane_name: string;           // e.g. "Core Infrastructure"
  epic_key: string;            // e.g. "ARC-1284"
  output_path: string;         // absolute path to write the .md file
  waves: Array<{
    wave_number: number;
    stories: Array<{
      key: string;             // e.g. "ARC-1285"
      title: string;
      priority: "🔴" | "🟡" | "🟢";
      status: "pending" | "claimed" | "done";
      claimed_by?: string;     // agent name if status = "claimed"
      dependencies?: string[]; // ARC keys this story is blocked by
      checkpoint_count?: number; // required if priority = "🔴"
      notes?: string;          // optional free-text appended to story block
    }>;
  }>;
}
```

## In Scope
- TypeScript tool at `~/.config/opencode/tools/generate_runbook.ts`
- Generates all required runbook sections: lane header, checkpoint summary table, wave headings, story blocks with correct checkbox/sub-item syntax
- Encodes all structural rules from `astWalker.ts` / `parser.ts` in `pa.aid.conductor.ts`
- Returns structured result: `{ success, path, story_count, wave_count, errors? }`
- Output passes `validate_runbook` (ARC-1362) with zero errors
- Update `generate-lane-runbooks` skill to call this tool instead of writing markdown by hand

## Out of Scope
- Reading or parsing existing runbook files (that is `validate_runbook`'s job)
- Modifying existing runbook files (use `tool-runbook-check-step`, etc.)
- Generating issue files or implementation plans
- Semantic validation of story content
- UI or CLI interface

## Constraints
- Must use `@opencode-ai/plugin` `tool()` helper (same as all other OpenCode tools)
- Output markdown must be byte-for-byte parseable by `astWalker.ts` — keep format rules in sync
- If `validate_runbook` tool exists (ARC-1362), its acceptance test is the conformance oracle for this tool's output

## Dependencies
- OpenCode custom tools API (`@opencode-ai/plugin`)
- Parser source reference: `pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts` and `parser.ts` (format rules source of truth)
- `validate_runbook` tool (ARC-1362) — used as conformance oracle in tests; recommend ARC-1362 be implemented first or in parallel
- Skill to update after implementation: `generate-lane-runbooks` (`~/.config/opencode/skills/generate-lane-runbooks/SKILL.md`)

## Completion Tracking
| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1363-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1363-COMPLETION-SUMMARY.md` | TBD |
