# ARC-1383: End-to-end lifecycle regression tests - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1383-end-to-end-lifecycle-regression-tests.md`  
**Completion Summary:** `task-completions/ARC-1383-COMPLETION-SUMMARY.md` (TBD)  
**Approach:** Single approach — integration test suite covering full state machine  
**Owner:** build agent  
**Date:** 2026-07-15

## Scope & Alignment

Write integration tests exercising the full claim-through-archive story lifecycle across all StoryLifecycleState states, including retry loops, all three review gates (plan/implementation/decision) with merged and comment-loop branches, and both escalation-routing paths (worktree failure, archive-guard failure). Verify no regressions against existing laneRunner, claimGit, and checkpointGit test suites.

**Acceptance criteria mapping:**
- AC1 (every state transition exercised at least once) → Step 1: full lifecycle integration test
- AC2 (existing tests still pass) → Step 4: regression verification
- AC3 (all three gates, both branches per gate) → Step 2: gate-specific integration tests

## Assumptions & Dependencies

- All prior ARC-1373 stories completed: ARC-1374 through ARC-1382
- Test infrastructure exists at `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/`
- Mock sidecar, git operations, and PR creation available for test isolation
- Vitest or Jest test runner configured

## Implementation Steps

### Step 1: Write full lifecycle happy-path integration test

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/lifecycle.integration.test.ts` (new file)

**Action:** Create integration test exercising the complete state machine from unclaimed to archived, verifying every state transition:

**Test scenario:**
1. Story starts in `unclaimed`
2. Claim → `claimed`
3. Provision worktree → `worktree_provisioning` → `agent_executing`
4. Execute plan → transitions through execution substates
5. Hit plan checkpoint → `awaiting_plan_review`
6. Merge plan PR → `agent_executing` (resume)
7. Self-review passes → `agent_self_review` → `awaiting_implementation_review`
8. Merge implementation PR → `completion_summary_drafting`
9. Write completion summary → `archiving` → `archived`

**State transitions covered:**
- `unclaimed` → `claimed`
- `claimed` → `worktree_provisioning`
- `worktree_provisioning` → `agent_executing`
- `agent_executing` → `awaiting_plan_review`
- `awaiting_plan_review` → `addressing_plan_comments` (comment loop branch)
- `addressing_plan_comments` → `agent_executing` (after addressing)
- `agent_executing` → `agent_self_review`
- `agent_self_review` → `awaiting_implementation_review`
- `awaiting_implementation_review` → `addressing_implementation_comments` (comment loop)
- `addressing_implementation_comments` → `agent_self_review` (after addressing)
- `agent_self_review` → `completion_summary_drafting` (clean path)
- `completion_summary_drafting` → `archiving`
- `archiving` → `archived`

**Verification:** Assertion on sidecar state after each transition; confirm all 14 core states visited.

---

### Step 2: Write gate-specific integration tests for all three review gates

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/lifecycle.integration.test.ts`

**Action:** Add three test cases, one per gate (plan, implementation, decision), each covering both the "merged → advance" and "not merged + comments → address loop" branches:

**Test: Plan review gate (ARC-1376)**
- Story reaches `awaiting_plan_review`
- **Branch A (merged):** Supervisor merges plan PR → story advances to `agent_executing`
- **Branch B (comments):** Supervisor adds PR comments → story transitions to `addressing_plan_comments` → agent addresses → loop back to `awaiting_plan_review`

**Test: Implementation review gate (ARC-1378)**
- Story reaches `awaiting_implementation_review`
- **Branch A (merged):** Supervisor merges implementation PR → story advances to `completion_summary_drafting`
- **Branch B (comments):** Supervisor adds PR comments → story transitions to `addressing_implementation_comments` → agent addresses → loop back to `awaiting_implementation_review`

**Test: Decision review gate (ARC-1375)**
- Story reaches `awaiting_decision_review` (via approach-decision document)
- **Branch A (merged):** Supervisor merges decision PR → story advances to `agent_executing`
- **Branch B (comments):** Supervisor adds PR comments → story transitions to `addressing_decision_comments` → agent addresses → loop back

**Verification:** Each test asserts correct state transitions, PR creation, and resume behavior for both branches.

---

### Step 3: Write escalation-routing integration tests for both failure paths

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/lifecycle.integration.test.ts`

**Action:** Add two test cases for escalation checkpoint routing:

**Test: Worktree provisioning failure (ARC-1379)**
- Story in `worktree_provisioning` encounters git worktree add failure
- Expected transition: `worktree_provisioning` → `awaiting_escalation_checkpoint`
- Expected behavior: Lane pauses, checkpoint queue entry created with `checkpointLevel: 'high'`
- Supervisor resolves manually → story resumes from last known good state

**Test: Archive-guard failure (ARC-1380)**
- Story in `archiving` encounters preflight check failure (missing implementation plan or completion summary)
- Expected transition: `archiving` → `awaiting_escalation_checkpoint`
- Expected behavior: Lane pauses, checkpoint queue entry created with `checkpointLevel: 'high'`
- Supervisor resolves manually → story resumes archiving

**Verification:** Assertions on sidecar state, checkpoint queue contents, and lane pause behavior.

---

### Step 4: Run full regression suite and update existing tests as needed

**Files:**
- `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/laneRunner.test.ts`
- `/repos/pa.aid.conductor.ts/packages/server/src/git/claimGit.test.ts`
- `/repos/pa.aid.conductor.ts/packages/server/src/git/checkpointGit.test.ts`
- `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/checkpoints.test.ts`

**Action:**
1. Run full test suite: `npm test` in `/repos/pa.aid.conductor.ts/packages/server/`
2. Identify tests broken by state machine migration (e.g., tests asserting on old `laneStatusMap` instead of sidecar state)
3. Update assertions to reference new canonical StoryLifecycleState and computed rollup values
4. Remove tests for legacy behavior superseded by new state machine (if any)

**Verification:** All existing tests pass with zero failures.

---

### Step 5: Add retry-loop integration tests

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/lifecycle.integration.test.ts`

**Action:** Add two test cases for retry logic:

**Test: Plan-phase retry loop**
- Story executes plan → fails (e.g., test failure)
- Expected transition: `agent_executing` → `retrying` (phase: 'plan')
- Agent retries with updated approach → succeeds → advances to next phase
- Expected max retries: verify exhaustion triggers escalation checkpoint

**Test: Code-phase retry loop**
- Story in self-review phase → local-code-review finds BLOCKER findings
- Expected transition: `agent_self_review` → `retrying` (phase: 'code')
- Agent fixes issues → re-runs self-review → passes → advances to implementation-review gate

**Verification:** Assertions on retry count, phase tracking, and escalation after max retries exhausted.

---

### Step 6: Add rollup computation integration test (ARC-1381)

**Files:** `/repos/pa.aid.conductor.ts/packages/server/src/__tests__/lifecycle.integration.test.ts`

**Action:** Add test verifying computed LaneStatus rollup:

**Test: Lane status rollup from story states**
- Lane with 3 stories: one `archived`, one `awaiting_plan_review`, one `agent_executing`
- Expected `computeLaneStatus()` result: `'paused'` (any gate state present)
- Change gate story to `archived` → expected result: `'running'` (no gate, one active)
- Change active story to `archived` → expected result: `'done'` (all archived)

**Verification:** Assertions on `computeLaneStatus()`, `computeRunbookStatus()`, and `computeClosedSessionStatus()` outputs for various story-state combinations.

---

## Testing & Validation

1. **New integration test file:** `lifecycle.integration.test.ts` with 10+ test cases covering all scenarios above
2. **Regression verification:** All existing tests in laneRunner, claimGit, checkpointGit, checkpoints test files pass
3. **Coverage report:** Run `npm run test:coverage` — verify all new state machine code paths covered
4. **Manual verification:** Run conductor against test runbook; observe state transitions in logs match expected sequences

## Risks & Open Questions

**Risk:** Mock sidecar implementation may not fully replicate production behavior — integration tests should use a realistic in-memory sidecar mock that enforces state machine invariants.

**Risk:** Test execution may be slow due to full lifecycle simulation — consider parallel test execution or isolated unit tests for individual state transitions if runtime exceeds acceptable threshold.

**Risk:** Existing tests may have brittle assumptions about timing or sequencing — refactor flaky tests to use deterministic mocks and explicit state assertions rather than timing-based waits.

**Open question:** Should the integration test suite include negative test cases (e.g., invalid state transitions)? Current scope assumes state machine enforcement prevents invalid transitions; verify if explicit negative testing is needed.

**Open question:** Should tests cover wave-level lifecycle (WaveLifecycleState)? Issue AC focuses on story-level states; wave lifecycle may warrant separate test story if not already covered.
