| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Parent Epic | ARC-1348 — OpenCode Agent Tools |
| Jira | ARC-1366 |

# ARC-1366: Implement `get_checkpoint_context` and `send_checkpoint_notification` OpenCode tools

## Goal

Create two focused OpenCode tools that together enable checkpoint resume with supervisor instruction. `get_checkpoint_context` (at `~/.config/opencode/tools/get_checkpoint_context.ts`) loads the live runbook and story state for a paused checkpoint and returns a structured context package — including any written implementation plan and the supervisor's instruction — so the agent resumes with full awareness of what was done and what to do next. `send_checkpoint_notification` (at `~/.config/opencode/tools/send_checkpoint_notification.ts`) emits a structured completion notification once the instruction has been executed. Together they close the gap where a plan is written at a checkpoint but there is no mechanism to receive further instructions or confirm completion without restarting the session from scratch.

## Problem

After a HIGH story checkpoint, the agent writes an implementation plan and stops. There is currently no way for a supervisor to send an instruction (e.g., "push the plan and open a PR") that the agent can receive and act on while retaining checkpoint context. Without this, the supervisor must restart the session from the beginning, causing the agent to re-read all context and risk duplicating work. Additionally, when execution resumes and completes, there is no notification mechanism to confirm what was accomplished.

## Acceptance Criteria

**Given** a `runbook_path` and `story_key` identifying an active (unchecked, unclosed) checkpoint, and an `instruction` string from the supervisor,
**When** `get_checkpoint_context({ runbook_path, story_key, instruction })` is called,
**Then** the tool returns `{ success: true, context: { story_key, title, lane, last_step_reached, implementation_plan_exists, implementation_plan_path, instruction } }` — populated from live runbook and filesystem state.

---

**Given** the implementation plan file exists at the expected path under `implementation_plans/`,
**When** `get_checkpoint_context` is called for that story,
**Then** `context.implementation_plan_exists` is `true` and `context.implementation_plan_path` is the resolved absolute path to the plan file.

---

**Given** the `story_key` does not match any story in the runbook, or the story is already checked off (checkpoint closed),
**When** `get_checkpoint_context` is called,
**Then** the tool returns `{ success: false, error: "No active checkpoint found for <story_key>" }` without modifying any file.

---

**Given** any required parameter is missing or empty (`runbook_path`, `story_key`, or `instruction`),
**When** `get_checkpoint_context` is called,
**Then** the tool returns `{ success: false, errors: ["<field> is required"] }` without performing any file reads.

---

**Given** a `story_key`, the original `instruction`, and a `summary` of what was accomplished,
**When** `send_checkpoint_notification({ story_key, instruction, summary })` is called,
**Then** the tool returns `{ success: true, notification: { story_key, instruction, summary, timestamp } }` and logs the structured notification to stdout so the session UI can surface it.

---

**Given** any required parameter is missing or empty (`story_key`, `instruction`, or `summary`),
**When** `send_checkpoint_notification` is called,
**Then** the tool returns `{ success: false, errors: ["<field> is required"] }` without emitting any notification.

## In Scope

- TypeScript tool `~/.config/opencode/tools/get_checkpoint_context.ts` — reads runbook via `parseRunbook()`, finds story by key, determines last step reached (checked vs unchecked sub-items), resolves implementation plan path, embeds supervisor instruction in returned context
- TypeScript tool `~/.config/opencode/tools/send_checkpoint_notification.ts` — accepts story_key, instruction, summary; emits structured completion notification to stdout; returns confirmation object
- Input validation with structured error objects for both tools
- Parser-first principle for both tools — all runbook reads via `parseRunbook()` from `./lib/parser.ts`
- Test file `~/.config/opencode/tools/tests/get_checkpoint_context.test.ts` covering context-loaded, plan-exists, not-found, and missing-param cases
- Test file `~/.config/opencode/tools/tests/send_checkpoint_notification.test.ts` covering success, missing-param cases
- Both tools added to the build/senior-coder allowlist in `pa.aid.wsl-setup.sh`

## Out of Scope

- Pushing git commits or opening Bitbucket PRs — those are agent actions triggered by the instruction, not by these tools
- Storing checkpoint state persistently — both tools read live runbook and filesystem state on each call
- UI changes to the session card or Resume button (covered by ARC-1302 / ARC-1365)
- Modifying the runbook or checking off steps — use `runbook_check_step` / `runbook_check_story` for that
- Automatic re-execution of the supervisor's instruction — the tools provide context; the agent executes
- A single tool with an `action` discriminator — each operation is a separate focused tool per the agent-tools pattern

## Constraints

- Each tool in its own file: `get_checkpoint_context.ts` and `send_checkpoint_notification.ts` (one-tool-per-file pattern)
- Both tools use `export default tool({...})` from `@opencode-ai/plugin`
- All runbook reads must use `parseRunbook()` from `./lib/parser.ts` — no raw regex scanning
- No runtime dependencies beyond `@opencode-ai/plugin` and the shared `./lib/` parser modules
- Test files must live in `~/.config/opencode/tools/tests/` — never in the tools root directory

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `./lib/parser.ts` — `parseRunbook()` and `ParsedRunbook` type (shared with pa.aid.conductor.ts)
- ARC-1302 (checkpoint detection and [Resume] card) — defines the checkpoint lifecycle these tools plug into

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1366-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1366-COMPLETION-SUMMARY.md` | TBD |