# .cursor

A personal collection of Cursor skills, subagents, and rules for research, code
review, plan review, MR review, refactor planning, ticket shipping, exhaustive
multi-agent orchestration, and merge/CI hygiene. Python/Django + GitLab-flavored,
but the review and planning workflows are generic.

This repository's Git worktree **is** `~/.cursor`. Skills, agents, and rules
here are the live global Cursor configuration — no copy/symlink install step is
required on this machine.

Feel free to borrow anything useful. Use it as a starting point for your own
setup rather than a polished, general-purpose plugin.

## Install (primary)

This repository's Git worktree **is** meant to be `~/.cursor`. Cursor already
loads `~/.cursor/skills/`, `~/.cursor/agents/`, and `~/.cursor/rules/` as
user-level configuration.

**Cherry-pick User Rules in Cursor** — open **Cursor Settings → Rules**, then add
individual User Rules by pasting the contents of the files you want from
`rules/` if you prefer UI-managed always-on conventions.

### Live home worktree (`~/.cursor`)

**Empty or missing `~/.cursor`:** clone is fine.

```bash
git clone https://github.com/PhillipChaffee/.cursor.git ~/.cursor
```

**Existing non-empty `~/.cursor` (usual case):** attach git — do **not**
`git clone` into the directory, and never `git reset --hard` / `git clean -xfd`
here (those can wipe Cursor app data).

```bash
cd ~/.cursor

# 1) Preserve any local kit customizations before first sync (timestamped; never overwrite)
BACKUP="$HOME/.cursor-kit-backup-$(date +%Y%m%d-%H%M%S)"
mkdir "$BACKUP"   # fails if path exists — do not reuse
for item in skills agents rules; do
  [ -e "$item" ] && cp -a "$item" "$BACKUP/"
done
echo "Backup written to $BACKUP"

# 2) Attach remote without destroying untracked app data
git init
git remote add origin https://github.com/PhillipChaffee/.cursor.git 2>/dev/null || \
  git remote set-url origin https://github.com/PhillipChaffee/.cursor.git
git fetch origin
git checkout -B main origin/main   # or: git checkout -B <branch> origin/<branch>
# If checkout refuses (usually untracked local skills/agents/rules), path-limited restore only:
#   git checkout origin/main -- .gitignore README.md LICENSE skills agents rules

# 3) Verify ignore allowlist (see Notes → Git safety)
```

### Secondary install (another machine / project copy)

```bash
git clone https://github.com/PhillipChaffee/.cursor.git
# Copy selected skills/agents/rules into a project .cursor/ (or merge into ~/.cursor/)
```

Cursor picks up project and user-level skills, agents, and rules on the next
session. No install script — take only what looks useful.

### Duplicates with project skills

When a project also has `.cursor/skills/` (for example a monorepo team catalog),
Cursor discovers **both** project and user roots. Same-named skills can appear
as duplicates. That is expected and intentional when you want global availability
outside the project.

These are **not** supported ways to hide project copies for yourself only:

- `.cursorignore` / `.cursorindexingignore` (indexing and file-access only; skill
  / rule / subagent loaders do not honor them)
- Same-name precedence (not documented for skills)
- Symlinks into `~/.cursor/skills/` (discovery does not follow them)

Keep team project skills checked in for coworkers; keep this kit globally for
personal use across projects.

## What's here

### `skills/`

- `ship` — orchestrate Linear ticket work from intake (or a problem paste)
  through research, plan, implement, verify, and GitLab MRs; optional handoff
  to Cursor's `/autopilot`
- `deep-research` — tiered research workflow that scales effort to the task
  (direct answer, parallel researchers, or a full planner → research →
  synthesis pipeline)
- `code-review` — multi-reviewer code review that dispatches specialized
  subagents and synthesizes findings (ships `checklists.md` + `examples.md`)
- `looping-code-review` — iteratively review, apply minimal verified fixes,
  test, push, and re-review within a fixed growth budget
- `plan-review` — multi-reviewer plan/design review pipeline (planner +
  structural and critical reviewers)
- `looping-plan-review` — architecture alignment, then implementation
  convergence until Approve (drives `/plan-review`)
- `mr-review` — review a GitLab MR end-to-end via the GitLab MCP, surface
  findings for approval, then post selected comments as draft notes
- `refactor-planner` — design a behavior-preserving refactor before touching
  code (ships a `references/` catalog of refactor patterns and placement guides)
- `clean-plan` — tidy an implementation plan so an agent can execute it cleanly
- `pre-mr-checklist` — pre-merge checklist (inline imports, type annotations,
  logging, test coverage, secrets, feature-flag checks, etc.)
- `ci-lint-test` — run a project's CI lint/test steps locally before pushing
- `ultracode` — standing exhaustive orchestration for one substantive request
  (slash-only; nests the specialist skills above rather than replacing them)

### `agents/`

Subagents the skills dispatch to:

- Code-review reviewers: `cr-security`, `cr-correctness`, `cr-performance`,
  `cr-architecture`, `cr-organization`, `cr-test-quality`, `cr-deployment-safety`,
  `cr-simplification`, plus `cr-planner`, `cr-verifier`, and `cr-implementer`
- Plan-review reviewers: `pr-planner`, `pr-problem-scope`, `pr-feasibility`,
  `pr-risk-rollback`, `pr-completeness`, `pr-adversarial`, `pr-architecture`,
  `pr-organization`, `pr-naming`, `pr-simplification`, `pr-verifier`,
  `pr-implementer`
- Refactor scouts: `refactor-code-scout`, `refactor-placement-scout`
- Research agents: `research-planner`, `research-synthesizer`,
  `researcher-lite`, `researcher-mid`, `researcher-deep`

### `rules/`

Always-applied conventions (unless noted):

- Python style, pytest, Google-style docstrings, class-section headers
  (`python`, `python-pytest`, `python-docstrings-google`, `python-class-sections`)
- Workflow conventions: `engineering`, `minimal-changes`, `plan-steps`,
  `merge-requests`, `linear-tickets`
- Writing style and process: `comment-style`, `subagents`, `skill-creation`,
  `code-organization`, `mr-review-chat-title`, `autopilot`, `look-it-up`
- Tooling: `django-migrations`, `github-vs-gitlab-mcp`
- Optional: `design-docs` (`alwaysApply: false`), `writing-voice` (template;
  `alwaysApply: false`)

## Personalizing the writing-voice rule

`rules/writing-voice.mdc` ships as a **fill-in template**. It is disabled by
default (`alwaysApply: false`). Before enabling it:

1. Fill each section with your own capitalization, slang, tone, and examples.
2. Replace the fictional worked example with your real voice samples (redacted).
3. Set `alwaysApply: true` only after the template no longer contains placeholders.

## How I use it (the workflow this is built for)

I run a **fast, pretty-smart model as my main-window agent** — the one I'm
iterating in all day — instead of the slowest, most expensive model. The
expensive model would do fine in the main window, but it's slower and costs
more, and for most turn-by-turn work the fast one is enough.

What makes that viable is **delegating the heavy thinking to subagents** so it
happens in isolated context windows rather than the main one:

- **Research and code review get kicked off as subagents as I go.** The main
  agent stays cheap and responsive while a `researcher-*` or `cr-*` agent spins
  up in its own context to do the in-depth investigation or review, then returns
  just the synthesis. The always-on `subagents` rule is the source of truth for
  delegation, model selection, and dispatch behavior.
- **When something genuinely needs the smartest agent and real deep thinking**
  (architecture calls, ambiguous investigations, deep research), I dispatch it
  specifically to a slower, smarter subagent — e.g. `researcher-deep` or the
  top-tier plan-review agents. I get the heavy model's reasoning where it
  matters without paying for it on every turn.

The second reason I do this, beyond cost and speed, is **keeping the main
context window clean.** Because the deep work happens in subagent contexts and
only the distilled result comes back, the main window doesn't fill up with
transcripts, large reads, or intermediate reasoning. That lets me keep going in
one agent for a long time without ever having to summarize context to keep
working.

### Normal workflow path

Shipping a ticket, in order. `/ship` runs this whole path for you; I also
often run the steps myself:

1. `deep-research` — investigate before planning
2. draft a plan (plan-steps rule; not a skill)
3. `looping-plan-review` — architecture alignment, then `/plan-review` until Approve
4. `clean-plan` — make the plan agent-executable
5. implement
6. `ci-lint-test` + `pre-mr-checklist` — verify before push
7. open MRs
8. `looping-code-review` — review → fix → re-review on the branch
9. `/autopilot` — keep the MR merge-ready (Cursor personal skill; not vendored here)

Side paths: `mr-review` for reviewing someone else's GitLab MR; `code-review`
when I want a multi-reviewer pass on a diff without the loop; `refactor-planner`
before a behavior-preserving refactor; `ultracode` when I want standing
exhaustive orchestration around those other skills.

### Most used skills

What I actually reach for day to day (full catalog under
[What's here](#whats-here)):

- `ci-lint-test` — local CI / lint / test before a push
- `looping-code-review` — review → fix → re-review
- `deep-research` — when a question needs a real investigation
- `mr-review` — GitLab MR review end to end
- `clean-plan` — tidy a plan before an agent executes it
- `looping-plan-review` / `plan-review` / `code-review` — fuller multi-reviewer
  pass when the change warrants it

Less often as slash commands: `pre-mr-checklist` (shipping hygiene), `ship` and
`refactor-planner` (larger ticket / refactor flows).

## Notes

### Git safety (live `~/.cursor` worktree)

This directory also holds Cursor app data (projects, plugins, built-ins,
credentials, plans, etc.). The repo `.gitignore` keeps Cursor's managed block,
then appends a **trailing allowlist** that re-ignores everything and un-ignores
only:

- `.gitignore`, `README.md`, `LICENSE`
- `skills/**`, `agents/**`, `rules/**`

**Always use path-limited staging** (`git add -- path…`). Never `git add .` or
`git add -A`. Never `git clean` or `git reset --hard` here.

**Recovery:** if a path-limited checkout overwrote kit trees you still want,
restore from the timestamped backup first
(`cp -a "$BACKUP"/skills "$BACKUP"/agents "$BACKUP"/rules ~/.cursor/` —
use the path printed in step 1). For git-only mistakes, use `git reflog` and
restore allowlisted paths from a known-good SHA
(`git checkout <sha> -- .gitignore README.md LICENSE skills agents rules`).
Ignored Cursor app data is **not** in git — restore it from Time Machine / your
OS backup if damaged.

After Cursor updates (or before pushing kit changes), re-check that the personal
allowlist still appears **after** the managed block and still wins. From this
directory:

```bash
set -euo pipefail

assert_no_matches() {
  # rg: 0=matches (fail), 1=clean, 2+=tool error (fail)
  local pattern=$1; shift
  set +e
  rg -n -- "$pattern" "$@"
  local st=$?
  set -e
  if [ "$st" -eq 0 ]; then
    echo "FAIL: forbidden pattern: $pattern"
    exit 1
  fi
  if [ "$st" -ne 1 ]; then
    echo "FAIL: rg error ($st) for: $pattern"
    exit 1
  fi
}

# Sensitive / managed paths must stay ignored
git check-ignore -q --no-index mcp.json \
  || { echo 'FAIL: mcp.json should be ignored'; exit 1; }
git check-ignore -q --no-index projects/example/mcps/x.json \
  || { echo 'FAIL: projects mcps should be ignored'; exit 1; }
git check-ignore -q --no-index projects/example/agent-transcripts/x.jsonl \
  || { echo 'FAIL: transcripts should be ignored'; exit 1; }
git check-ignore -q --no-index plugins/cache/x \
  || { echo 'FAIL: plugins should be ignored'; exit 1; }
git check-ignore -q --no-index skills-cursor/x \
  || { echo 'FAIL: skills-cursor should be ignored'; exit 1; }

# Kit roots must stay trackable (use --no-index so tracked files aren't a false pass)
for p in README.md skills/ship/SKILL.md agents/pr-planner.md rules/autopilot.mdc; do
  git check-ignore -q --no-index "$p" \
    && { echo "FAIL: $p unexpectedly ignored"; exit 1; }
done

# Contract smoke (model slug + phase tokens)
assert_no_matches 'thinking-xhigh' skills agents rules
assert_no_matches 'claude-fable-5-thinking-max' skills agents rules
# Hyphenated / legacy tokens must not appear as live skip_phases or phase: values
assert_no_matches 'skip_phases.*(plan-review|create-ticket|babysit_opt)' skills/ship/SKILL.md
assert_no_matches 'phase: (plan-review|create-ticket|babysit_opt)\b' skills/ship/SKILL.md

echo 'Git safety + contract smoke OK'
```

### Skill / agent coupling

- `mr-review` can optionally load a review-voice rule if you keep one in
  `rules/`; it's not required. Use `writing-voice` as a starting template if you
  want one.
- Some skills/agents reference each other (e.g. `code-review` and `plan-review`
  dispatch to the `cr-*` / `pr-*` agents); keep the matching `agents/` files so
  the cross-references resolve.
- `/ship` may hand off to Cursor's `/autopilot` if present; that skill is not
  vendored in this repo.

## License

MIT — borrow freely.
