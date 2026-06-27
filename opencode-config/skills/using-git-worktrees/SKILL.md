---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans
---

# Using Git Worktrees

## Purpose

Set up isolated workspaces for feature work: detect existing isolation, prefer platform native tools, fall back to git worktrees. Prevents work collision on parallel lanes.

## When to Use

- Starting feature work needing isolation from current workspace
- Before executing implementation plans
- When multiple branches/lanes active
- Multiworktree setup needed

---

## Overview

Ensure work happens in an isolated workspace. Prefer platform's native worktree tools. Fall back to manual git worktrees only when no native tool is available.

**Core principle:** Detect existing isolation first. Then use native tools. Then fall back to git. Never fight the harness.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Workflow

**Before creating anything, check current context.**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Submodule guard:** `GIT_DIR != GIT_COMMON` is also true inside submodules. Verify:

```bash
git rev-parse --show-superproject-working-tree 2>/dev/null
```

If this returns a path, you're in a submodule — treat as normal repo.

**If `GIT_DIR != GIT_COMMON` (and not a submodule):** Already in a linked worktree.

- Is `$BRANCH` the same lane/branch you are about to work on? → Skip to Step 3.
- Is `$BRANCH` a **different** lane? → You are in a sibling worktree. Proceed to Step 1 to create a **new** worktree for this lane. Do NOT re-use the current worktree.

**If `GIT_DIR == GIT_COMMON`:** Normal repo checkout. Check live worktrees first:

```bash
git worktree list
```

Report active worktrees to the user:
> "Active worktrees: {list}. Creating new worktree for lane {BRANCH_NAME}."

Ask for consent if no declared preference:
> "Would you like me to set up an isolated worktree? It protects your current branch from changes."

Honor any existing declared preference without asking. If declined, work in place and skip to Step 3.

## Step 1: Create Isolated Workspace

### 1a. Native Worktree Tools (preferred)

Do you have a tool named like `EnterWorktree`, `WorktreeCreate`, `/worktree`, or `--worktree`? If yes, use it and skip to Step 3.

**Only proceed to 1b if no native worktree tool exists.**

### 1b. Git Worktree Fallback

#### Directory Selection (priority order)

1. Check instructions for a declared worktree directory preference — use it if specified
2. Check for existing project-local worktree directory:
   ```bash
   ls -d .worktrees 2>/dev/null
   ls -d worktrees 2>/dev/null
   ```
   If found, use it. `.worktrees` beats `worktrees`.
3. Default to `/worktrees/` at filesystem root

#### Safety Verification

**Conditional `git check-ignore` — only for project-local directories:**

```bash
# Determine if directory is inside repo root
REPO_ROOT=$(git rev-parse --show-toplevel)
BASE_DIR_ABS=$(cd "$BASE_DIR" 2>/dev/null && pwd -P)

if [[ "$BASE_DIR_ABS" == "$REPO_ROOT"* ]]; then
  # Project-local (inside repo): MUST verify ignored
  git check-ignore -q "$BASE_DIR" 2>/dev/null
  if [ $? -ne 0 ]; then
    # NOT ignored: Add to .gitignore, commit, then proceed
    echo "$BASE_DIR" >> .gitignore
    git add .gitignore
    git commit -m "chore: gitignore worktree directory"
  fi
else
  # External path (outside repo): Skip check
  # No gitignore check needed — path is outside repo
  true
fi
```

**Why critical for project-local:** Prevents accidentally committing worktree contents.

**Why skip for external paths:** Worktrees outside the repo root cannot be tracked by gitignore — irrelevant.

#### Branch Naming — REQUIRED

Branch name MUST be exactly `<issueid>` — no prefix, no suffix, no description. Just the raw issue ID.

**NO type prefixes.** `fix/`, `feature/`, `bugfix/`, `hotfix/` etc. break CI pipelines.
**NO description suffix.** `ARC-123-add-login` is wrong. `ARC-123` is correct.

Example: `git worktree add ../ARC-123 -b ARC-123`

| ✅ Correct | ❌ Wrong |
|-----------|---------|
| `ARC-123` | `ARC-123-add-login` |
| `PROJ-42` | `fix/ARC-123-add-login` |
| `ARC-123` | `feature/PROJ-42-dead-code` |

**Runbook update commits** (checking off steps in a planning repo) go directly to `main`, not to the worktree branch.

#### Create the Worktree

Each lane gets its own subdirectory — prevents name collisions between parallel lanes:

```bash
git worktree add "$BASE_DIR/$BRANCH_NAME" -b "$BRANCH_NAME"
cd "$BASE_DIR/$BRANCH_NAME"
```

**Default example (if `/worktrees` becomes base directory):**
```bash
git worktree add "/worktrees/$BRANCH_NAME" -b "$BRANCH_NAME"
cd "/worktrees/$BRANCH_NAME"
```

**Project-local example (if `.worktrees` found in repo):**
```bash
git worktree add ".worktrees/$BRANCH_NAME" -b "$BRANCH_NAME"
cd ".worktrees/$BRANCH_NAME"
```

**Parallel lanes:** Multiple worktrees coexist safely under the base directory as long as each uses a unique branch name. e.g.:
```
/worktrees/ARC-10/
/worktrees/ARC-20/
```
or
```
.worktrees/ARC-10/
.worktrees/ARC-20/
```

**Sandbox fallback:** If `git worktree add` fails with permission error, report it and work in place.

## Step 3: Project Setup

Auto-detect and run appropriate setup:

```bash
if [ -f package.json ]; then npm install; fi
if [ -f Cargo.toml ]; then cargo build; fi
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi
if [ -f go.mod ]; then go mod download; fi
```

## Step 4: Verify Clean Baseline

```bash
npm test / cargo test / pytest / go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### Report

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Already in linked worktree, same lane | Skip creation (Step 0) |
| Already in linked worktree, different lane | Create new worktree for this lane (Step 1) |
| Multiple parallel lanes active | Each lane has own `$BASE_DIR/<BRANCH_NAME>/` (project-local or `/worktrees/` by default) |
| In a submodule | Treat as normal repo (Step 0 guard) |
| Native worktree tool available | Use it (Step 1a) |
| No native tool | Git worktree fallback (Step 1b) |
| `.worktrees/` exists | Use it (verify ignored) |
| `worktrees/` exists | Use it (verify ignored) |
| Both exist | Use `.worktrees/` |
| Neither exists | Check instruction file, then default `/worktrees/` at filesystem root |
| Directory not ignored | Add to .gitignore + commit |
| Permission error on create | Sandbox fallback, work in place |
| Tests fail during baseline | Report failures + ask |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Fighting the harness | Use native tools (Step 1a). Skip 1b when native tool exists. |
| Skipping detection | Always run Step 0 before creating anything |
| Skipping ignore verification | Always `git check-ignore` before project-local worktree; skip for external paths |
| Proceeding with failing tests | Report failures, get explicit permission |
| Using `fix/` or `feature/` prefix | Remove prefix. Branch name MUST be exactly `<issueid>` (e.g. `ARC-123`). |
| Adding description to branch name | Remove suffix. `ARC-123-add-login` is wrong. `ARC-123` is correct. |

## Red Flags

**Never:**
- Re-use an existing worktree for a different lane
- Create worktree without checking `git worktree list` first
- Use `git worktree add` when native worktree tool exists
- Create worktree without verifying it's ignored (project-local)
- Proceed with failing tests without asking
- Use branch name prefixes like `fix/`, `feature/`, `bugfix/` — breaks CI
- Append description to branch name — branch MUST be exactly `<issueid>` (e.g. `ARC-123`)

**Always:**
- Run Step 0 detection first
- Prefer native tools over git fallback
- Verify directory is ignored for project-local
- Verify clean test baseline
