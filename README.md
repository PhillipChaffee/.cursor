# .cursor

A personal collection of Cursor skills, subagents, and rules I use day to day
for research, code review, plan review, MR review, refactor planning, and
merge/CI hygiene. Python/Django + GitLab-flavored, but the review and planning
workflows are generic.

Feel free to borrow anything useful. Use it as a starting point for your own
setup rather than a polished, general-purpose plugin.

## What's here

### `skills/`

- `deep-research` — tiered research workflow that scales effort to the task
  (direct answer, parallel researchers, or a full planner → research →
  synthesis pipeline)
- `code-review` — multi-reviewer code review that dispatches specialized
  subagents and synthesizes findings (ships `checklists.md` + `examples.md`)
- `plan-review` — multi-reviewer plan/design review pipeline
- `mr-review` — review a GitLab MR end-to-end via the GitLab MCP, surface
  findings for approval, then post selected comments as draft notes
- `refactor-planner` — design a behavior-preserving refactor before touching
  code (ships a `references/` catalog of refactor patterns and placement guides)
- `clean-plan` — tidy an implementation plan so an agent can execute it cleanly
- `pre-mr-checklist` — pre-merge checklist (inline imports, type annotations,
  logging, test coverage, secrets, etc.)
- `ci-lint-test` — run a project's CI lint/test steps locally before pushing
- `submodule-update` — sync git submodules to their default branch

### `agents/`

Subagents the skills dispatch to:

- Code-review reviewers: `cr-security`, `cr-correctness`, `cr-performance`,
  `cr-architecture`, `cr-test-quality`, `cr-deployment-safety`,
  `cr-simplification`, `cr-verifier`, `cr-implementer`
- Plan-review reviewers: `pr-problem-scope`, `pr-feasibility`,
  `pr-risk-rollback`, `pr-completeness`, `pr-adversarial`, `pr-architecture`,
  `pr-simplification`, `pr-verifier`, `pr-implementer`
- Refactor scouts: `refactor-code-scout`, `refactor-placement-scout`
- Research agents: `research-planner`, `research-synthesizer`,
  `researcher-lite`, `researcher-mid`, `researcher-deep`

### `rules/`

Always-applied conventions:

- Python style, pytest, Google-style docstrings, class-section headers
  (`python`, `python-pytest`, `python-docstrings-google`, `python-class-sections`)
- Workflow conventions: `engineering`, `minimal-changes`, `plan-steps`,
  `merge-requests`, `linear-tickets`
- Writing style and process: `comment-style`, `delegate-research-to-subagents`
- Tooling: `django-migrations`, `github-vs-gitlab-mcp`

## Try it out

Copy whichever pieces you want into your own `~/.cursor/`:

    git clone https://github.com/phillipchaffee/.cursor.git
    cp -R .cursor/skills/<name> ~/.cursor/skills/
    cp -R .cursor/agents/*.md  ~/.cursor/agents/
    cp -R .cursor/rules/<name> ~/.cursor/rules/

Cursor picks up user-level skills, agents, and rules from `~/.cursor/` on the
next session. No install script — just copy what looks useful.

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
  just the synthesis. This is what `delegate-research-to-subagents` codifies.
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

## Notes

- `mr-review` can optionally load a review-voice rule (e.g.
  `mr-comment-voice`) if you keep one in `~/.cursor/rules/`; it's not required
  and isn't included here.
- Some skills/agents reference each other (e.g. `code-review` and `plan-review`
  dispatch to the `cr-*` / `pr-*` agents); copy the matching `agents/` files so
  the cross-references resolve.

## License

MIT — borrow freely.
