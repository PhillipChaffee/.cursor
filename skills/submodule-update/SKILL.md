---
name: submodule-update
description: Checkout all git submodules to their default branch (main or master) and pull the latest changes. Use when the user wants to update submodules, sync submodules, pull latest submodule changes, or reset submodules to their default branch.
---

# Submodule Update

Checkout every submodule in the monorepo to its default remote branch and pull the latest changes.

## Workflow

```
Task Progress:
- [ ] Step 1: Detect submodules and their default branches
- [ ] Step 2: Check for uncommitted changes
- [ ] Step 3: Checkout and pull each submodule
- [ ] Step 4: Report results
- [ ] Step 5: Self-improvement check
```

## Step 1: Detect Submodules and Default Branches

List all submodules and determine each one's default branch:

```bash
git submodule status
```

```bash
git submodule foreach 'git remote show origin | sed -n "s/.*HEAD branch: //p"'
```

## Step 2: Check for Uncommitted Changes

Before switching branches, check whether any submodules have tracked or staged changes:

```bash
git submodule foreach 'git status --porcelain --untracked-files=no'
```

If any submodule has tracked or staged changes, report which ones are dirty and ask the user how to proceed before updating that submodule. Do not checkout or pull submodules with tracked or staged changes without user confirmation.

Untracked files are not blockers. Continue updating submodules whose only local changes are untracked files. If an untracked file conflicts with checkout or pull, surface the Git error in the final report.

## Step 3: Checkout and Pull

Run a single command that iterates every submodule, skips submodules with tracked or staged changes, detects the default branch, checks it out, and pulls:

```bash
git submodule foreach '
tracked_changes=$(git status --porcelain --untracked-files=no)
if [ -n "$tracked_changes" ]; then
  echo "[SKIP] $name: tracked or staged changes present"
  printf "%s\n" "$tracked_changes"
  exit 0
fi

default_branch=$(git remote show origin | sed -n "s/.*HEAD branch: //p")
if [ -z "$default_branch" ]; then
  echo "[FAIL] $name: could not detect default branch"
elif git checkout "$default_branch" && git pull; then
  echo "[OK]   $name: $default_branch updated"
else
  echo "[FAIL] $name: checkout/pull failed"
fi
'
```

## Step 4: Report Results

Summarize the outcome in a table:

| Submodule | Branch | Status |
|-----------|--------|--------|
| `name` | `main` or `master` | Already up to date / Pulled N commits / **Failed** (reason) |

If any submodule failed, surface the error so the user can address it manually.

## Step 5: Self-Improvement Check

After completing the workflow, review whether this skill's instructions matched reality:

1. **New or removed submodules** — Did `git submodule status` list submodules not mentioned in this skill, or vice versa? If the set of submodules has changed, no update is needed (the skill is submodule-agnostic by design).
2. **Command failures** — Did any command fail for a reason the skill doesn't address (e.g., auth prompt, shallow clone issues, submodule init required)? If so, add a troubleshooting entry.
3. **Missing steps** — Did you have to do something not covered by the skill (e.g., `git submodule init`, `git submodule sync`)? If so, add it as a prerequisite or new step.
4. **Unclear instructions** — Were any instructions ambiguous or misleading during execution?

If any of the above apply, summarize proposed changes and ask the user:
"Do you want me to update this skill now with these improvements?"
If the user confirms, update this `SKILL.md` and tell the user what changed.
If everything worked as documented, skip this step silently.
