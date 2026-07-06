# Workorder

**Repo:** pa.aid.runbook-executor
**Generated:** 2026-06-27
**Stories:** 47 across 8 epics

---

## Execution Model

| Role | Responsibility |
|------|---------------|
| Coding agent | Implements each story end-to-end on the story branch |
| Review agent | Reviews diff after each story; approves or returns with findings |
| Human supervisor | Approves plans, unblocks cross-lane decisions, merges PRs |

Workflow per story:
1. Load `write-implementation-plan` skill → produce `implementation_plans/{KEY}-plan.md`
2. Implement on branch `{KEY}`
3. Run `local-code-review` skill; iterate until clean
4. Write `task-completions/{KEY}-COMPLETION-SUMMARY.md`
5. Load `close-issue` skill; archive artifacts
6. Merge; update runbook checkbox

---

## Feature Lane Overview

| Lane | Feature | Epic Key | Stories | Est. Total | Start condition |
|------|---------|----------|---------|------------|-----------------|
| core-infrastructure | Core Infrastructure | ARC-1284 | 4 | 18h | immediately |
| session-setup | Session Setup & Runbook Selection | ARC-1289 | 4 | 12h | after ARC-1286 merged |
| parallel-lane-execution | Parallel Lane Execution | ARC-1290 | 8 | 28h | after ARC-1288 merged |
| checkpoint-management | Checkpoint Management | ARC-1295 | 7 | 24h | after ARC-1291 merged |
| queue-scheduling-policy | Queue & Scheduling Policy | ARC-1296 | 3 | 10h | after ARC-1303 merged |
| failure-handling | Failure Handling & Escalation | ARC-1300 | 3 | 10h | after ARC-1288 merged |
| session-history | Session History | ARC-1309 | 3 | 9h | after ARC-1287 merged |
| agent-tools | OpenCode Agent Tools | ARC-1348-epic | 16 | 40h | immediately |

```
T=0       T=8       T=15      T=19      T=25      T=31      T=37
│         │         │         │         │         │         │
├─── core-infrastructure ─────────────────────────────────────┤  18h
          ├─── session-setup ──────────┤                          12h (done T=20)
                    ├─── parallel-lane-execution ────────────┤    16h (done T=31)
                              ├─── checkpoint-management ──────────────┤  17h (done T=36)
                                        ├─── queue-scheduling-policy ──┤  10h (done T=37)
                    ├─── failure-handling ──────────┤               10h (done T=25)
          ├─── session-history ──────────┤                           9h (done T=20)
```

**Critical path:** core-infrastructure → parallel-lane-execution → checkpoint-management → queue-scheduling-policy
**Wall-clock estimate:** 37h

Each lane is independently executable. Start conditions are story-level prerequisites, not global wave gates.

---

## Lane: core-infrastructure

**Epic:** ARC-1284 | **Repo:** pa.aid.runbook-executor | **Start condition:** immediately

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1285 | Bootstrap web server and serve local UI | 4h | — | 🔴 HIGH |
| ARC-1286 | Parse runbook markdown into typed step graph | 4h | — | 🔴 HIGH |
| ARC-1287 | Implement sidecar state file | 3h | — | 🟡 MEDIUM |
| ARC-1288 | Integrate opencode CLI subprocess with JSON event stream | 4h | — | 🔴 HIGH |

Recommended execution order: ARC-1285 → ARC-1286 → ARC-1287 → ARC-1288 (unblocks cross-lane dependents earliest).

**🔴 HIGH stories (individual checkpoints):**
- ARC-1285 (new web server + API skeleton — architectural foundation; sets serving model for all UI)
- ARC-1286 (foundational parser; 3 lanes blocked on this story)
- ARC-1288 (subprocess integration with JSON event stream; 2 lanes blocked on this story; cross-system)

---

## Lane: session-setup

**Epic:** ARC-1289 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1286 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1313 | Display runbook dashboard at startup | 4h | — | 🔴 HIGH |
| ARC-1314 | Expand runbook detail on demand | 2h | ARC-1313 | 🟡 MEDIUM |
| ARC-1315 | Select runbooks to include in session | 2h | ARC-1313 | 🟡 MEDIUM |
| ARC-1316 | Configure session parameters before start | 2h | — | 🟡 MEDIUM |

**Pre-check:** verify ARC-1286 (runbook parser) is merged before starting this lane.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1313 (new dashboard integrating parser output, runbook scanning, status derivation; architectural UI entry point)

---

## Lane: parallel-lane-execution

**Epic:** ARC-1290 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1288 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1291 | Execute runbook steps sequentially within a lane | 4h | — | 🔴 HIGH |
| ARC-1292 | Run multiple lanes concurrently up to concurrency limit | 4h | ARC-1291 | 🔴 HIGH |
| ARC-1293 | Stream live agent output per lane in the UI | 3h | ARC-1291 | 🔴 HIGH |
| ARC-1294 | Update runbook checkbox state in real time | 2h | ARC-1291 | 🟡 MEDIUM |

**Pre-check:** verify ARC-1288 (opencode CLI subprocess) is merged before starting this lane.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1291 (step execution engine; routes agent/manual/checkpoint types; blocks checkpoint-management lane)
- ARC-1292 (concurrency management; architectural scheduling decision)
- ARC-1293 (real-time streaming UI; WebSocket/SSE architecture decision)

**Wave 2 — Conductor Automation (ARC-1367, ARC-1368, ARC-1369, ARC-1370):**

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1367 | Conductor: auto-claim story in runbook before dispatching agent | 3h | ARC-1349 (agent-tools) | 🔴 HIGH |
| ARC-1368 | Conductor: auto-provision feature git worktree for HIGH stories | 3h | ARC-1367 | 🔴 HIGH |
| ARC-1369 | Conductor: auto-commit implementation plans and completion summaries to planning repo | 3h | ARC-1359 (agent-tools) | 🔴 HIGH |
| ARC-1370 | Conductor: auto-update issue status and archive story artifacts on completion | 5h | ARC-1358, ARC-1360, ARC-1369 | 🟡 MEDIUM |

**Wave 2 start condition:** Wave 1 gate passed; ARC-1349 and ARC-1359 (agent-tools lane) merged.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1367 (eliminates manual claim ceremony; must fire before worktree creation)
- ARC-1368 (eliminates manual git worktree setup; injects worktree path into agent prompt)
- ARC-1369 (eliminates manual planning_commit calls; post-step auto-commit hook)

---

## Lane: checkpoint-management

**Epic:** ARC-1295 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1291 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1301 | Detect checkpoint markers and pause lane execution | 3h | — | 🔴 HIGH |
| ARC-1302 | Deliver browser notification within 5 seconds of checkpoint hit | 2h | ARC-1301 | 🟡 MEDIUM |
| ARC-1303 | Display checkpoint queue with automatic priority scoring | 3h | ARC-1301 | 🟡 MEDIUM |
| ARC-1304 | Inline-edit checkpoint artifacts and persist handoff file to disk | 4h | ARC-1301 | 🔴 HIGH |
| ARC-1305 | Approve or reject checkpoint inline with feedback | 2h | ARC-1303, ARC-1304 | 🟡 MEDIUM |
| ARC-1365 | Display agent last message on checkpoint Resume card | 3h | ARC-1302, ARC-1305 | 🟡 MEDIUM |

**Pre-check:** verify ARC-1291 (sequential step execution) is merged before starting this lane.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1301 (checkpoint detection + lane pause; blocks queue-scheduling-policy lane via ARC-1303)
- ARC-1304 (inline edit + crash-safe handoff file persistence; wide artifact surface)

**Wave 5 — PR Approval Gate for Planning Artifacts (ARC-1371):**

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1371 | Conductor: open PR for implementation plans and decision documents as approval gate | 4h | ARC-1366, ARC-1369 | 🔴 HIGH |

**Wave 5 start condition:** Wave 4 gate passed; ARC-1366 (resume_checkpoint, agent-tools lane) and ARC-1369 (parallel-lane-execution Wave 2) both merged.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1371 (changes approval workflow for plans and decisions; PR-merge as resume signal; Bitbucket API dependency)

---

## Lane: queue-scheduling-policy

**Epic:** ARC-1296 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1303 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1297 | Display ordered checkpoint queue with priority scores | 3h | — | 🟡 MEDIUM |
| ARC-1298 | Reorder checkpoint queue manually | 2h | ARC-1297 | 🟡 MEDIUM |
| ARC-1299 | Select and switch queue policy mid-session | 3h | ARC-1297 | 🟡 MEDIUM |

**Pre-check:** verify ARC-1303 (checkpoint queue with automatic priority scoring) is merged before starting this lane.

**🔴 HIGH stories (individual checkpoints):**
None. All three stories extend an established queue UI with incremental scope.

---

## Lane: failure-handling

**Epic:** ARC-1300 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1288 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1306 | Retry failed agent steps up to configurable limit | 3h | — | 🔴 HIGH |
| ARC-1307 | Escalate agent level on each retry | 2h | ARC-1306 | 🟡 MEDIUM |
| ARC-1308 | Surface exhausted retries as supervisor checkpoint notification | 3h | ARC-1306 | 🔴 HIGH |

**Pre-check:** verify ARC-1288 (opencode CLI subprocess) is merged before starting this lane.
**Note:** ARC-1308 also depends on ARC-1302 (browser notification) from checkpoint-management lane. Do not begin ARC-1308 until ARC-1302 is merged.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1306 (retry mechanism; architectural; cross-system subprocess re-invocation)
- ARC-1308 (cross-lane dep on ARC-1302; failure-as-checkpoint; wide blast — lane state, queue, notification)

---

## Lane: session-history

**Epic:** ARC-1309 | **Repo:** pa.aid.runbook-executor | **Start condition:** after ARC-1287 merged

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1310 | Persist active session state across restarts | 3h | — | 🔴 HIGH |
| ARC-1311 | Browse past sessions list | 2h | ARC-1310 | 🟡 MEDIUM |
| ARC-1312 | View full step output history for any past session | 2h | ARC-1311 | 🟡 MEDIUM |

**Pre-check:** verify ARC-1287 (sidecar state file) is merged before starting this lane.

**🔴 HIGH stories (individual checkpoints):**
- ARC-1310 (session persistence + restart recovery + single-session-per-repo enforcement; crash-safe writes)

---

## Effort & Schedule Summary

| Lane | Stories | Wall-clock | Agent-hours total | Supervision |
|------|---------|------------|------------------|-------------|
| core-infrastructure | 4 | 18h | 18h | 4 × 15 min = 1h |
| session-setup | 4 | 12h | 12h | 4 × 15 min = 1h |
| parallel-lane-execution | 8 | 28h | 30h | 8 × 15 min = 2h |
| checkpoint-management | 7 | 24h | 24h | 7 × 15 min = 1.75h |
| queue-scheduling-policy | 3 | 10h | 10h | 3 × 15 min = 0.75h |
| failure-handling | 3 | 10h | 10h | 3 × 15 min = 0.75h |
| session-history | 3 | 9h | 9h | 3 × 15 min = 0.75h |
| agent-tools | 16 | 40h | 39h | 16 × 15 min = 4h |
| **Total** | **47** | **38h** (parallel) | **152h** | **11.75h** |

Agent-hours include 20% coordination overhead per lane.

---

## Supervision Budget

- 47 story checkpoints × 15 min = **11.75h supervisor time** across 38h wall-clock
- ~18% supervision ratio (healthy; no wave gates)
- No global sync gates — supervisor handles per-story reviews only

---

## Cross-Lane Dependencies

| Story | Blocks | Type | Notes |
|-------|--------|------|-------|
| ARC-1286 | ARC-1313 (session-setup) | hard data | Dashboard requires parsed runbook step graph |
| ARC-1287 | ARC-1310 (session-history) | hard data | Session history extends sidecar state file |
| ARC-1288 | ARC-1291 (parallel-lane-exec) | hard data | Step executor invokes opencode CLI subprocess |
| ARC-1288 | ARC-1306 (failure-handling) | hard data | Retry mechanism re-invokes opencode CLI subprocess |
| ARC-1291 | ARC-1301 (checkpoint-mgmt) | hard data | Checkpoint detection hooks into step execution flow |
| ARC-1302 | ARC-1308 (failure-handling) | hard data | Exhausted-retry notification reuses browser notification logic |
| ARC-1303 | ARC-1297 (queue-scheduling) | hard data | Queue panel UI builds on checkpoint priority scoring model |
| ARC-1349 | ARC-1367 (parallel-lane-execution) | hard data | Auto-claim reuses runbook_claim_story markdown-writer pattern |
| ARC-1358 | ARC-1370 (parallel-lane-execution) | hard data | Auto-archive reuses update_issue_status frontmatter logic |
| ARC-1359 | ARC-1369 (parallel-lane-execution) | hard data | Auto-commit reuses planning_commit git pattern |
| ARC-1360 | ARC-1370 (parallel-lane-execution) | hard data | Auto-archive reuses archive_issue file-move logic |
| ARC-1366 | ARC-1371 (checkpoint-management) | hard data | PR-merge detection builds on resume_checkpoint mechanism |
| ARC-1369 | ARC-1371 (checkpoint-management) | hard data | PR branch+commit infrastructure reused for plan/decision PRs |

---

## Lane: agent-tools

**Epic:** ARC-1348-epic | **Repo:** pa.aid.runbook-executor | **Start condition:** immediately

| Story | Summary | Est. | Intra-lane depends on | Risk |
|-------|---------|------|----------------------|------|
| ARC-1348 | Implement `runbook_find_next_story` OpenCode tool | 3h | — | 🔴 HIGH |
| ARC-1349 | Implement `runbook_claim_story` OpenCode tool | 3h | ARC-1348 | 🔴 HIGH |
| ARC-1350 | Implement `runbook_check_step` OpenCode tool | 3h | ARC-1349 | 🔴 HIGH |
| ARC-1351 | Implement `runbook_check_story` OpenCode tool | 2h | ARC-1349 | 🔴 HIGH |
| ARC-1352 | Implement `runbook_check_dependencies` OpenCode tool | 2h | — | 🟡 MEDIUM |
| ARC-1353 | Implement `get_current_issue` OpenCode tool | 2h | — | 🔴 HIGH |
| ARC-1354 | Implement `run_tests` OpenCode tool | 3h | — | 🔴 HIGH |
| ARC-1355 | Implement `run_lint` OpenCode tool | 2h | — | 🔴 HIGH |
| ARC-1356 | Implement `run_typecheck` OpenCode tool | 2h | — | 🟡 MEDIUM |
| ARC-1357 | Implement `get_branch_diff` OpenCode tool | 1h | — | 🟡 MEDIUM |
| ARC-1358 | Implement `update_issue_status` OpenCode tool | 3h | — | 🔴 HIGH |
| ARC-1359 | Implement `planning_commit` OpenCode tool | 2h | — | 🟡 MEDIUM |
| ARC-1360 | Implement `archive_issue` OpenCode tool | 4h | ARC-1351, ARC-1358 | 🔴 HIGH |
| ARC-1361 | Implement `preflight_check` OpenCode tool | 2h | — | 🟡 MEDIUM |
| ARC-1362 | Implement `validate_runbook` OpenCode tool | 3h | — | 🔴 HIGH |
| ARC-1364 | Implement `generate_issue_file` OpenCode tool | 2h | — | 🟡 MEDIUM |

**🔴 HIGH stories (individual checkpoints):**
- ARC-1348 (foundational — all runbook navigation tools depend on this parser)
- ARC-1349 (claim workflow — ARC-1350, ARC-1351, ARC-1360 depend on this)
- ARC-1350 (step-level runbook write + commit)
- ARC-1351 (story-level runbook write + commit; gates ARC-1360)
- ARC-1353 (cross-cutting — used at start of every local-code-review)
- ARC-1354 (framework detection + subprocess; used in every review)
- ARC-1355 (framework detection + subprocess; used in every review)
- ARC-1358 (markdown patch; gates ARC-1360)
- ARC-1360 (atomic 6-step archive; most error-prone operation in close-issue)
- ARC-1362 (parser-derived validation; must align with astWalker.ts)

---

## On-Hold Items

None. All 47 stories have named prerequisites and can begin once their start conditions are met.

---

## Known Risks

| Risk | Severity | Lane | Mitigation |
|------|----------|------|------------|
| `opencode run --format json` event schema undocumented or subject to change | 🔴 HIGH | core-infrastructure | Integration-test against live opencode CLI before ARC-1288 is marked done; pin CLI version |
| Browser Notification API requires HTTPS or localhost — may behave differently per browser | 🟡 MEDIUM | checkpoint-management | Test in Chrome + Firefox on localhost during ARC-1302; document workarounds |
| Crash-safe markdown checkbox write (ARC-1294) — concurrent writes from multiple lanes may corrupt file | 🔴 HIGH | parallel-lane-execution | Serialize all markdown writes through a single write queue; validate in ARC-1294 acceptance tests |
| Single-session-per-repo enforcement (ARC-1310) — race condition on simultaneous starts | 🟡 MEDIUM | session-history | Use file lock on sidecar file; test with two simultaneous starts |
| Drag-and-drop queue reorder (ARC-1298) — browser API compatibility | 🟢 LOW | queue-scheduling-policy | Use established library (e.g. dnd-kit); no custom DnD implementation |
