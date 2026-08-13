---
description: Constrain /autopilot with a size budget and minimal-change triage.
alwaysApply: true
---

# Autopilot

When running the `autopilot` skill (`/autopilot`), or when `/ship` hands off to autopilot, apply this discipline on top of the built-in skill. Do not vendor or edit the Cursor-managed autopilot skill.

## Goal

Get the PR merge-ready (conflicts, valid comments/Bugbot, in-scope CI) **without ballooning the diff**. Hold or reduce size when possible. Prefer deletion or replacement over new abstractions.

## Size budget

Record once at autopilot start; never reset for the run:

```
baseline_changed_lines = insertions + deletions (PR vs target)
growth_budget = min(200, max(50, ceil(baseline_changed_lines * 0.20)))
maximum_changed_lines = baseline_changed_lines + growth_budget
```

Also track net code growth:

```
(current_insertions - current_deletions) - (baseline_insertions - baseline_deletions)
```

- This is a ceiling, not a quota. Unrelated deletion never creates credit for more work.
- Tests count toward the budget and must not be omitted to fit it.
- Over ceiling: stop and get explicit user approval for a **named blocker** and estimated final size before continuing. Never commit or push an unapproved over-budget diff.

## Scope

**Implement during autopilot only:**

- Merge conflicts (preserve branch + base intent; if intents conflict, abort and ask)
- Items triage marks `fix-now` (comments, Bugbot, in-scope CI)
- CI failures caused by this PR’s scope — never change CI workflows just to pass, never unrelated code

**Allowed as `fix-now` when small and in budget:**

- Style nits that match Google style / existing style rules (e.g. `python-docstrings-google`, `comment-style`, `python` line length and imports)
- Preference comments that are a **good opinion** — infer from active rules and skills (`minimal-changes`, `engineering`, `code-organization`, Naming, Subagent Use, etc.). Weak taste with no backing rule → `reject` or `nit-skip`

**Restructure / “could be faster” / broad architecture** can be real and worth doing, but **not inside autopilot**. Use `pause-plan`: stop the loop, surface the finding, wait for the user to plan. Do not silently redesign under merge-ready pressure.

**Reject:** invalid Bugbot, CI-workflow hacks, unrelated code, preferences unsupported by rules/skills, style that conflicts with project/Google rules.

User-declared out-of-scope stays out, unless the PR’s own behavior is independently broken (then fix only that side). Every commit must map to a triaged `fix-now` item.

## Triage before acting

Before implementing any comment, Bugbot, or CI fix (pure intent-preserving conflict resolution may proceed without triage, but still counts against the budget if the diff grows):

1. Launch **one** `generalPurpose` triage subagent on `cursor-grok-4.6-high-fast`.
2. Pass: current PR diff paths, candidate items, this scope contract, baseline / current size / remaining budget, and pointers to active rules for style / good-opinion judgment.
3. For each item, exactly one decision:

| Decision | Meaning |
|----------|---------|
| `fix-now` | Correctness, in-scope CI, Google-aligned style, or good-opinion preference; include optimal minimal change set |
| `pause-plan` | Real restructure / perf / architecture — pause autopilot and ask the user to plan; do not implement in this loop |
| `defer` | Real but not merge-blocking and not worth pausing for (follow-up later) |
| `nit-skip` | Tiny / not worth a commit this pass (may batch with another `fix-now` on the same file) |
| `reject` | Invalid, out of scope, or preference/style that conflicts with rules |

For every `fix-now`: state the **optimal minimal change set** (file, function, what to edit) and estimated line impact — not a rewrite plan. Prefer fewest lines, zero or negative net growth when practical, no new abstractions. Shared root cause → one fix.

Implement only the selected `fix-now` set. On any `pause-plan`, stop and ask the user before continuing. If a fix expands beyond triage, stop and escalate — do not silently widen scope.

## Anti-stall

Two consecutive iterations with the same blocker and no viable minimal in-budget fix → stop and escalate with the finding, why the minimal fix failed, estimated final size, and options. Do not silently widen scope to force merge-ready.

## Exit / wrap-up

Exit when mergeable + green + comments triaged. “Triaged” includes `reject` / `defer` / `nit-skip` with reasons, and any `pause-plan` items surfaced and either planned+done or explicitly left for the user.

Report: baseline → final changed lines, net code growth, budget used, rejected items, and any paused plan asks.
