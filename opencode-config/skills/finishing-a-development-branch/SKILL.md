---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and you need to decide how to integrate the work
---

# Finishing a Development Branch

## Purpose

Guide completion of development work: verify tests → detect environment → present merge/PR options → execute choice. Enforces local-code-review gate and no-direct-commit rule.

## When to Use

- Implementation complete, all tests passing, ready to integrate
- Before merging to main or creating PR
- After local-code-review exits clean

---

> **Rule: Never commit directly to `master`. All changes via branch → PR.**

> **Note:** Runbook-workflow artifacts (runbook updates, completion summaries) are committed directly to `main`, not to feature branches.

## Overview

Guide completion of development work by presenting clear options and handling chosen workflow.

**Core principle:** Verify tests → Detect environment → Present options → Execute choice → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 0: Local Review Gate

**Before anything else, confirm `local-code-review` has been run and exited clean.**

If `local-code-review` has NOT been run yet:
> "Local code review has not been run. Running it now before proceeding."

Invoke `local-code-review`. Wait for it to exit clean (no BLOCKER, no ISSUE).

- If it exits clean → continue to Step 1
- If it escalates to human (3 iterations, unresolved findings) → **STOP. Do not present merge/PR options.** Present the escalation output and wait for human instruction.

**Do NOT skip Step 0.** The PR review (Bitbucket) is not a substitute for local review.

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:
[Show failures]
Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

| State | Menu | Cleanup |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON` (normal repo) | Standard 4 options | No worktree to clean up |
| `GIT_DIR != GIT_COMMON`, named branch | Standard 4 options | Provenance-based (see Step 6) |
| `GIT_DIR != GIT_COMMON`, detached HEAD | Reduced 3 options (no merge) | No cleanup (externally managed) |

### Step 3: Determine Base Branch

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 4: Present Options

**Normal repo and named-branch worktree — present exactly these 4 options:**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD — present exactly these 3 options:**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

### Step 5: Execute Choice

#### Option 1: Merge Locally

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

git checkout <base-branch>
git pull
git merge <feature-branch>

# Verify tests on merged result
<test command>
```

Then: Cleanup worktree (Step 6), then delete branch:

```bash
git branch -d <feature-branch>
```

#### Option 2: Push and Create PR

**Branch name check:** Must be exactly `<issueid>` (e.g. `ARC-123`). No prefix (`fix/`, `feature/`), no suffix/description. If current branch has a prefix or suffix, rename it first:
```bash
git branch -m <issueid>
```

```bash
git push -u origin <feature-branch>

gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

**Do NOT clean up worktree** — user needs it alive to iterate on PR feedback.

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

Don't cleanup worktree.

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

Then: Cleanup worktree (Step 6), then force-delete branch:
```bash
git branch -D <feature-branch>
```

### Step 6: Cleanup Workspace

**Only runs for Options 1 and 4.** Options 2 and 3 always preserve the worktree.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**If `GIT_DIR == GIT_COMMON`:** Normal repo, no worktree to clean up. Done.

**If worktree path is under `.worktrees/` or `worktrees/`:** We own cleanup. **Only remove THIS lane's worktree — never touch sibling worktrees.**

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# Safety: confirm we are only removing THIS worktree, not a sibling
echo "Removing worktree: $WORKTREE_PATH"
git worktree list   # show remaining worktrees before removal

git worktree remove "$WORKTREE_PATH"
git worktree prune
```

After removal, report remaining active worktrees:
```bash
git worktree list
```

**Otherwise:** Host environment owns this workspace. Do NOT remove it.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | yes | - | - | yes |
| 2. Create PR | - | yes | yes | - |
| 3. Keep as-is | - | - | yes | - |
| 4. Discard | - | - | - | yes (force) |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skipping test verification | Always verify tests before offering options |
| Open-ended "what should I do?" | Present exactly 4 structured options |
| Cleaning up worktree for Option 2 | Only cleanup for Options 1 and 4 |
| Deleting branch before removing worktree | Remove worktree first |
| Running `git worktree remove` from inside worktree | cd to main repo root first |
| No confirmation for discard | Require typed "discard" |
| Branch name has prefix or suffix | Rename: `git branch -m <issueid>` — only bare issue ID allowed |
| Skipping local-code-review before presenting options | Step 0 is mandatory — run local-code-review first, always |
| Removing sibling worktree during cleanup | Only remove `$WORKTREE_PATH` — other lanes may be active |

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Delete work without confirmation
- Force-push without explicit request
- Remove worktree before confirming merge success
- Run `git worktree remove` from inside the worktree
- Push branch with type prefix (`fix/`, `feature/`, `bugfix/`, `hotfix/`) — breaks CI pipelines
- Present merge/PR options without local-code-review having exited clean
- Run `git worktree remove` on any path other than the current lane's `$WORKTREE_PATH`

**Always:**
- Verify tests before offering options
- Present exactly 4 options (or 3 for detached HEAD)
- Get typed confirmation for Option 4
- Cleanup worktree for Options 1 & 4 only
- `cd` to main repo root before worktree removal
- Run `git worktree prune` after removal
