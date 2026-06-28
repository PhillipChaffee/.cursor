---
name: code-review
description: Perform thorough code reviews using parallel specialized subagents, then verify findings and produce a curated summary. Use when reviewing pull requests, git diffs, code changes, or when the user asks for a code review.
---

# Code Review (Multi-Agent Orchestrator)

Run a comprehensive code review by launching 7 specialized reviewer subagents in parallel, filtering their findings through a verifier subagent, then producing a curated summary. The user can reply freeform with decisions, ask to walk through blockers one at a time, and optionally hand accepted fixes to an implementer subagent that edits the source files in place. Includes a **review-only mode** for callers (e.g. `mr-review`) that want just the curated summary.

## Scope Resolution

Determine what code to review using this priority:

1. **User specifies scope** — branch name, commit SHA, PR number/URL, or file paths
2. **On a feature branch** — all changes vs main/master (`git diff main...HEAD`)
3. **Staged changes** — `git diff --staged`
4. **Unstaged changes** — `git diff`
5. **Latest commit** — `git show HEAD`

Gather the diff and changed file paths before launching subagents.

## Subagents

Launch **all 7** reviewer subagents in a single message using parallel Task tool calls. Pass each one the diff and changed file paths.

The reviewers are native Cursor agents in `.cursor/agents/`:

| Agent | File | Domain |
|-------|------|--------|
| Security | `cr-security.md` | Injection, auth, secrets, data exposure |
| Correctness | `cr-correctness.md` | Logic bugs, edge cases, None handling |
| Performance | `cr-performance.md` | N+1 queries, blocking ops, memory, hot paths |
| Architecture | `cr-architecture.md` | SRP, coupling, layering, cross-service contracts |
| Test Quality | `cr-test-quality.md` | Coverage gaps, anti-patterns, test ROI |
| Deployment Safety | `cr-deployment-safety.md` | Migrations, deploy order, feature flags, rollback |
| Simplification | `cr-simplification.md` | Over-engineering, duplication, change atomicity |

Each reviewer:
- Has `readonly: true` (cannot modify files)
- Has `model: inherit` (uses the same model as this chat)
- Returns structured findings or an exact "no issues" string

**CRITICAL: Do NOT set the `model` parameter on any Task tool call** — for the reviewers, the verifier, or the implementer. Omitting `model` makes the subagent inherit the parent chat model. Setting `model: "fast"` (or any other value) downgrades to a weaker model and produces lower-quality output. This applies to both the named-subagent path and the `generalPurpose` fallback path described in the Verification and Fix step sections below.

## Verification

After the 7 reviewers complete (but before synthesis), launch the **Code Review Verifier** as a single subagent to filter the reviewer findings. The verifier tags each finding as `confirmed`, `false_positive`, or `needs_rephrase`.

**CRITICAL: Do NOT set the `model` parameter on the Task tool call for the verifier.** It must inherit the parent chat model (same rule as the reviewers).

**Subagent dispatch**: prefer `subagent_type: "Code Review Verifier"`. If Cursor's session enum does not yet contain that type (e.g. the agent file was added since session start), fall back to `subagent_type: generalPurpose` with the full `cr-verifier.md` body inlined as the prompt.

**Input to pass**: the git diff + changed file paths, plus all 7 reviewer outputs concatenated with source attribution (e.g. `[Security]`, `[Correctness]`, ...).

**Process the verifier output per finding**:
- `confirmed` → include in the curated summary
- `needs_rephrase` → apply the rephrased text, then include
- `false_positive` → list in a "Findings rejected by verifier" section with the verifier's reason (the user can override if they disagree)

All findings (blockers, suggestions, nits) go through the verifier — not just blockers.

## Synthesis

After the reviewers and the verifier have returned:

1. **Collect** confirmed and rephrased findings from the verifier output.
2. **Deduplicate** — if two agents flag the same issue from different angles (e.g., Security and Correctness both flag a missing None check on auth), merge into one finding and note both perspectives.
3. **Categorize** — separate into show-stopper bugs (must fix), architectural concerns (design rethink), smaller suggestions, and nits.
4. **Rank** within each category by severity.
5. **Collapse clean reviewers** — reviewers that returned "no issues" get a one-line summary in the All Clear section.
6. **Produce the curated summary** (see "Output Format" below). Do NOT pass reviewer outputs verbatim.

## Output Format

Produce a single curated summary. Omit any section that would be empty.

```
## Code review summary

**Verdict**: Ready to Merge | Needs Attention | Needs Work
**Counts**: N blockers • N suggestions • N nits • N findings rejected by verifier

## The big picture
<1-2 sentence framing>

## Show-stopper bugs (only if any)
N. **<title>** — `file:line`. <explanation + suggested fix>

## Architectural concerns (only if any)
N. **<title>**. <explanation + approach>

## Smaller suggestions (only if any)
- <terse one-liner per finding with `file:line`>

## Nits (only if any)
- <one-liner per finding>

## Findings rejected by verifier (only if any)
- <terse one-liner per rejected finding + verifier's reason>

## All Clear (only if any reviewers returned clean)
- <reviewer>: <one-line summary>

## Questions to clarify intent (only if any)
N. **<question>** — context

---
Reply with your decisions, or say "walk me through" to step through each blocker.
```

## Walkthrough mode

Triggered by the user saying "walk me through" or close variant after the summary. Start at **blocker #1** (no pre-prompt). For each blocker in order:

1. **Re-state in full detail**: title, `file:line`, code excerpt from the diff, what the verifier confirmed.
2. **Explain why it's a blocker**: concrete impact, what breaks.
3. **Propose 1-3 specific fixes** with trade-offs.
4. **Ask**: "Accept fix [N] / Reject the finding (give reason) / Discuss further / Move to next".
5. **Record the decision in chat** (visible to the user).
6. **Move to the next blocker**.

When all blockers are done, also offer to walk suggestions if there are any (user can decline).

At the end of the walkthrough, summarize all captured decisions and transition to the **Fix step** (see next section).

## Fix step (final, opt-in)

After the walkthrough (or after a freeform decisions reply), surface a "ready to apply" confirmation:

```
Decisions captured:
- Accept fix #1: <title>
- Accept fix #3: <title>
- Accept fix #5: <title>
- Reject #2 (reason: <user's reason>)
- Defer #4

Apply 3 approved fixes now via the implementer subagent? [yes / no / edit list]
```

- **yes** → launch the implementer subagent with the diff + the list of approved fixes. The implementer edits source files in place and returns a summary. Report the summary back to the user.
- **no** → end. The user applies fixes manually.
- **edit list** → let the user toggle the apply-list (drop an accepted fix, add a deferred one) and re-prompt.

After fixes apply, suggest: "If you want to verify nothing regressed, re-invoke `/code-review` and re-run your tests."

**CRITICAL: Do NOT set the `model` parameter on the Task tool call for the implementer.** Same rule as the reviewers and the verifier — omit `model` to inherit the parent chat model.

**Subagent dispatch**: prefer `subagent_type: "Code Review Implementer"`. If not in the session enum, fall back to `subagent_type: generalPurpose` with the full `cr-implementer.md` body inlined as the prompt.

## Review-only mode

Some callers want only the curated summary as a result, not an interactive walkthrough/fix flow. Most notably `mr-review` invokes `code-review` to surface findings as GitLab MR comments; the walkthrough and fix step are inappropriate for an external MR you don't own.

Enter review-only mode when **either**:

- The user message explicitly requests it (e.g. "review only — don't walk through or fix"), OR
- Another skill invokes `code-review` and signals review-only intent (the caller's wording should make this explicit, e.g. "Follow code-review/SKILL.md in review-only mode").

In review-only mode:

1. Run the 7 reviewers in parallel (same as default mode).
2. Run the verifier (same).
3. Emit the curated summary in the same format — **but omit** the trailing `--- Reply with your decisions, or say "walk me through" ...` line.
4. **Halt immediately** after emitting the summary. Do NOT prompt for walkthrough. Do NOT prompt for fixes. Do NOT launch the implementer subagent.

Callers take the summary findings and do whatever they want with them downstream (e.g. post as MR comments via the GitLab MCP).

## Verdict Guidelines

- **Ready to Merge** — all reviewers clean or only nits; no blockers or suggestions.
- **Needs Attention** — has medium-severity issues or important suggestions worth addressing.
- **Needs Work** — has critical/high blockers that must be fixed.

## Scope Boundaries

Only review files in the changeset. If an agent flags an issue outside the diff, include it as an `out-of-scope:` note suggesting a follow-up.

## Communication Style

- Use "we" or "this code" instead of "you".
- Explain the *why* for every finding.
- Assume positive intent.
- Use reviewer output as input; produce a curated summary that's scannable in one read. Drop into per-blocker mode when asked. Hand off accepted fixes to the implementer subagent.
