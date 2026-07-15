# ARC-1382: Add implementation-review checkpoint recognition to runbook template/docs/parser - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1382-add-implementation-review-checkpoint-to-template.md`  
**Completion Summary:** `task-completions/ARC-1382-COMPLETION-SUMMARY.md` (TBD)  
**Approach:** Single approach — extend existing keyword-based checkpoint recognition pattern  
**Owner:** build agent  
**Date:** 2026-07-15

## Scope & Alignment

Extend the runbook template, HOW-THIS-WORKS.md documentation, and astWalker.ts parser to recognize and classify the new "IMPLEMENTATION REVIEW CHECKPOINT" gate type, so future-generated runbooks can express the ARC-1378 implementation-review gate using the same keyword-sniffing convention already established for plan and batch checkpoints.

**Acceptance criteria mapping:**
- AC1 (astWalker.ts correctly classifies implementation-review checkpoint) → Step 2: extend keyword recognition in deriveStepTypeFromLabel
- AC2 (HOW-THIS-WORKS.md documents the new checkpoint) → Step 3: update documentation
- AC3 (generate_runbook tool includes the new checkpoint in HIGH story template) → Step 1: update story-block template in skill file

## Assumptions & Dependencies

- ARC-1378: Implementation-review gate implemented in laneRunner.ts, routes/checkpoints.ts
- astWalker.ts already implements keyword-based checkpoint classification (`deriveStepTypeFromLabel` at lines 32–68)
- generate-lane-runbooks skill exists at `/repos/pa.aid.runbook-executor/opencode-config/skills/generate-lane-runbooks/SKILL.md`
- Parser copy strategy (agent-tools-technical-context.md §3) — parser changes in pa.aid.conductor.ts may need re-copy to `~/.config/opencode/tools/lib/` for tools to pick up new keyword

## Implementation Steps

### Step 1: Add implementation-review checkpoint to HIGH story template in generate-lane-runbooks skill

**Files:** `/repos/pa.aid.runbook-executor/opencode-config/skills/generate-lane-runbooks/SKILL.md`

**Action:** Modify the HIGH story-block template (lines 126–136) to include the new checkpoint step between "Run `local-code-review`" and "Write completion summary":

```markdown
### 🔴 HIGH stories
- [ ] **{KEY}** — {summary}
  - [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/{feature}/{KEY}-{title}.md`
  - [ ] Write `implementation_plans/{feature}/{KEY}-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] 🔴 IMPLEMENTATION REVIEW CHECKPOINT
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/{KEY}-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat({feature}): {short description}`
```

**Verification:** Generate a test runbook with one HIGH story using the skill; confirm the template output includes the new checkpoint line.

---

### Step 2: Extend astWalker.ts keyword recognition to classify implementation-review checkpoint

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts`

**Action:** Update `deriveStepTypeFromLabel()` function (lines 32–68) to recognize "IMPLEMENTATION REVIEW CHECKPOINT" as a distinct checkpoint level.

**Current logic:**
- Keyword `HIGH` → `checkpointLevel: 'high'`
- Keywords `MEDIUM` or `BATCH` → `checkpointLevel: 'medium'`
- Keywords `LOW` or `WAVE` → `checkpointLevel: 'low'`
- Fallback → `checkpointLevel: 'high'`

**New logic (no change to checkpointLevel, but recognize the keyword):**
- Add explicit recognition for "IMPLEMENTATION REVIEW" substring in label
- Classification: continue using `checkpointLevel: 'high'` (implementation-review gates are per-story, same level as plan checkpoints)
- Rationale: astWalker.ts only needs to recognize this as a valid checkpoint; the specific artifact kind (`'implementation'`) is already handled by `ArtifactKind` mapping in checkpointGit.ts (ARC-1375)

**Code change:**
No functional change needed to `deriveStepTypeFromLabel` — the existing fallback logic already classifies any step containing "CHECKPOINT" as `type: 'checkpoint'`. The keyword "IMPLEMENTATION REVIEW CHECKPOINT" will be recognized by the substring test at line ~36.

**Optional improvement:** Add explicit keyword check for clarity:
```typescript
const upper = label.toUpperCase();
if (upper.includes('CHECKPOINT') || upper.includes('GATE')) {
  result.type = 'checkpoint';
  if (upper.includes('IMPLEMENTATION REVIEW')) result.checkpointLevel = 'high';
  else if (upper.includes('HIGH')) result.checkpointLevel = 'high';
  // ... rest of keyword checks
}
```

**Verification:** Add test case to `astWalker.test.ts` (if it exists) or `parser.test.ts` verifying a step labeled "🔴 IMPLEMENTATION REVIEW CHECKPOINT" returns `{ type: 'checkpoint', checkpointLevel: 'high' }`.

---

### Step 3: Document implementation-review checkpoint in HOW-THIS-WORKS.md

**Files:** `/repos/pa.aid.runbook-executor/docs/plans/HOW-THIS-WORKS.md`

**Action:** Update the 9-step agent workflow description (Section 7, lines ~89–95) to include the new checkpoint step in its correct position:

**Current 9-step HIGH story workflow:**
1. Read issue file
2. Write implementation plan
3. 🔴 CHECKPOINT — human reviews plan
4. Execute plan
5. Lint / tests pass
6. Write completion summary
7. Commit + ☑

**New 10-step HIGH story workflow:**
1. Read issue file
2. Write implementation plan
3. 🔴 INDIVIDUAL PLAN CHECKPOINT — human reviews plan before execution
4. Execute plan
5. Run `local-code-review` — agent self-review with automated lint/tests/type-check
6. 🔴 IMPLEMENTATION REVIEW CHECKPOINT — human reviews implementation before archiving
7. Lint / tests pass (final verification after addressing review comments)
8. Write completion summary
9. Commit + ☑

**Also update:** Checkpoint model table (if present) to clarify that both plan and implementation checkpoints are `checkpointLevel: 'high'`.

**Verification:** Read the updated section; confirm it clearly distinguishes plan-review (before execution) from implementation-review (after self-review, before completion).

---

### Step 4: Update batch workflow documentation to note implementation-review applies per-story in batches

**Files:** `/repos/pa.aid.runbook-executor/docs/plans/HOW-THIS-WORKS.md`

**Action:** If HOW-THIS-WORKS.md documents the MEDIUM/LOW batch workflow, add a note clarifying that implementation-review checkpoints apply per-story within a batch (i.e., after each story's self-review passes, a per-story implementation-review checkpoint is created, but they may be reviewed together in one pass by the supervisor).

**Verification:** Documentation clearly states that batch plan checkpoints gate all stories before execution, while implementation-review checkpoints occur per-story during execution.

---

### Step 5 (optional): Re-copy parser files to agent tools lib/ if needed

**Files:**
- `/repos/pa.aid.conductor.ts/packages/server/src/runbook/astWalker.ts` → `~/.config/opencode/tools/lib/astWalker.ts`

**Action:** If Step 2 modifies astWalker.ts logic, re-copy the updated file to the agent tools lib/ directory per the parser copy strategy (agent-tools-technical-context.md §3).

**Verification:** Agent tools that use `parseRunbook()` (e.g., `validate_runbook`) correctly recognize implementation-review checkpoints in test runbooks.

---

## Testing & Validation

1. **Unit test:** Add test case to astWalker or parser test suite verifying implementation-review checkpoint classification
2. **Skill output test:** Generate a runbook using the updated skill template; verify new checkpoint line appears
3. **Parser integration test:** Parse a runbook with implementation-review checkpoint; verify no parsing errors, `checkpointLevel` correct
4. **Documentation review:** Read HOW-THIS-WORKS.md end-to-end; confirm new checkpoint is documented in correct position

## Risks & Open Questions

**Risk:** If Step 2 changes are backported to agent tools lib/, coordination with other in-flight agent-tools stories may be required to avoid parser copy conflicts.

**Risk:** Existing runbooks generated before this change will not have implementation-review checkpoint lines — they will need manual insertion or regeneration. Out of scope per AC, but document this as a migration note.

**Open question:** Should implementation-review checkpoints have a distinct `checkpointLevel` value (e.g., `'implementation'`) separate from `'high'`, or continue using `'high'`? Current approach assumes `'high'` is sufficient; verify with supervisor if ambiguous.
