---
name: plan-review
description: Perform thorough plan and design reviews using parallel specialized subagents, then verify findings and produce a curated summary. Use when reviewing technical designs, architecture proposals, implementation plans, or when the user asks for plan/design feedback.
---

# Plan Review (Multi-Agent Orchestrator)

Run a comprehensive plan review by launching 5 specialized reviewer subagents in parallel, filtering their findings through a verifier subagent, then producing a curated summary. The user can reply freeform with decisions, ask to walk through blockers one at a time, and optionally hand accepted fixes to an implementer subagent that edits the plan in place.

## Target Resolution

Determine what plan to review:

1. **User specifies file** — read that file
2. **User pastes plan** — use the pasted content
3. **Open markdown/doc file** — review that document
4. **Conversation context** — review the plan discussed in conversation

Gather the full plan text before launching subagents.

## Calibration reference

For what a well-scoped plan looks like, see [.cursor/plans/borrower_info_child_call_ab22f376.plan.md](.cursor/plans/borrower_info_child_call_ab22f376.plan.md) — 103 lines, 5 todos, self-contained Problem / Design / Changes / Non-goals / Validation sections. Reviewers should not flag a plan in that shape as "missing Success Criteria" or "missing Observability" subsections — those concerns are only worth surfacing when something is actually wrong, not when a section header is absent.

## Subagents

Launch **all 5** reviewer subagents in a single message using parallel Task tool calls. Pass each one the full plan text.

The reviewers are native Cursor agents in `.cursor/agents/`:

| Agent | File | Domain |
|-------|------|--------|
| Problem & Scope | `pr-problem-scope.md` | Clarity, success criteria, scope boundaries |
| Technical Feasibility | `pr-feasibility.md` | Soundness, alternatives, hidden prerequisites |
| Risk & Rollback | `pr-risk-rollback.md` | Failure modes, mitigations, blast radius, observability |
| Completeness & Sequencing | `pr-completeness.md` | Step ordering, dependencies, validation checkpoints |
| Adversarial | `pr-adversarial.md` | Premise challenges, hidden assumptions, strategic pre-mortem, steelmanned alternatives |

Each reviewer:
- Has `readonly: true` (cannot modify files)
- Runs on `model: gpt-5.5-extra-high` (set it on the Task call)
- Returns structured findings or a clean verdict
- The Feasibility agent may read codebase files to verify plan assumptions

**CRITICAL: Set the `model` parameter on every Task tool call.** Use `gpt-5.5-extra-high` for reviewer and implementer subagents. Use `claude-fable-5-thinking-max` for the verifier subagent. Omitting `model` makes the subagent inherit the parent chat model, which can silently weaken verification quality. This applies to both the named-subagent path and the `generalPurpose` fallback path described in the Verification and Fix step sections below.

## Model policy

Verifier subagents must use the current best thinking model for coding — the model that performs best across multiple programming benchmarks. That is currently `claude-fable-5-thinking-max`. Non-verifier plan-review subagents use `gpt-5.5-extra-high` unless this policy is intentionally updated. When changing this section, also update the matching agent frontmatter in `.cursor/agents/`.

## Reviewer tiers

The 5 reviewers split into two tiers based on what kind of finding they produce.

| Tier | Reviewers | Nature of findings |
|------|-----------|--------------------|
| **Strategic** (rethink candidates) | Problem & Scope, Adversarial | Premise, approach, and framing. Findings usually require reconsidering the problem or the chosen approach — a human decision, not a rewrite. |
| **Tactical** (fix-in-place candidates) | Technical Feasibility, Risk & Rollback, Completeness & Sequencing | Concrete library swaps, step reorderings, missing rollback plans, missing feature flags, unspecified validation checkpoints — the kind of finding that has an obvious mechanical fix. |

Technical Feasibility is grouped with the tactical tier because the majority of its findings are concrete swaps ("use library X", "add a compat shim") that the implementer can handle. In the rare case where Feasibility surfaces a genuine approach-level concern (e.g. "this architecture cannot work in this stack"), the walkthrough mode lets the user accept, reject, or defer the finding before any fix is applied.

## Verification

After the 5 reviewers complete (but before synthesis), launch the **Plan Review Verifier** as a single subagent to filter the reviewer findings. The verifier tags each finding as `confirmed`, `false_positive`, or `needs_rephrase`.

**CRITICAL: Set `model: "claude-fable-5-thinking-max"` on the Task tool call for the verifier.** The verifier must run on the current best thinking model for coding (see Model policy).

**Subagent dispatch**: prefer `subagent_type: "Plan Review Verifier"`. If Cursor's session enum does not yet contain that type (e.g. the agent file was added since session start), fall back to `subagent_type: generalPurpose` with the full `pr-verifier.md` body inlined as the prompt. For either dispatch path, pass `model: "claude-fable-5-thinking-max"`.

**Input to pass**: the full plan text, plus all 5 reviewer outputs concatenated with source attribution (e.g. `[Problem & Scope]`, `[Adversarial]`, ...).

**Process the verifier output per finding**:
- `confirmed` → include in the curated summary
- `needs_rephrase` → apply the rephrased text, then include
- `false_positive` → list in a "Findings rejected by verifier" section with the verifier's reason (the user can override if they disagree)

All findings (blockers, suggestions, nits) go through the verifier — not just blockers.

## Synthesis

After the reviewers and the verifier have returned:

1. **Collect** confirmed and rephrased findings from the verifier output.
2. **Deduplicate** — if two agents flag the same gap from different angles, merge into one finding (attribute to both).
3. **Tier** — each finding inherits its reviewer's tier.
4. **Rank** within each tier — blockers, suggestions, nits.
5. **Produce the curated summary** (see "Output Format" below). Do NOT pass reviewer outputs verbatim.

## Output Format

Produce a single curated summary. Omit any section that would be empty.

```
## Plan review summary

**Verdict**: Approve | Request Changes | Questions Only
**Counts**: N blockers (strategic: A, tactical: B) • N suggestions • N nits • N findings rejected by verifier

## The big picture
<1-2 sentence framing of where the plan stands and why>

## Strategic problems (only if any)
N. **<title>**. <2-3 sentence explanation; what decision the user has to make>

## Tactical problems (only if any)
N. **<title>**. <1-2 sentences + the mechanical fix>

## Verifiable factual errors (only if any)
- <bullet with file:line, quote, or specific concrete error>

## Findings rejected by verifier (only if any)
- <terse one-liner per rejected finding + verifier's reason>

## Questions for the user (only if any)
**Q1: <decision title>?** <options a/b/c laid out with trade-offs>
**Q2: ...**

---
Reply with your decisions, or say "walk me through" to discuss each blocker one at a time.
```

## Walkthrough mode

Triggered by the user saying "walk me through" or close variant after the summary. Start at **blocker #1** (no pre-prompt). For each blocker in order:

1. **Re-state in full detail**: title, file:line / quote / code excerpt, what the verifier confirmed.
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

- **yes** → launch the implementer subagent with the plan text + the list of approved fixes. The implementer edits the plan file in place and returns a summary. Report the summary back to the user.
- **no** → end. The user applies fixes manually.
- **edit list** → let the user toggle the apply-list (drop an accepted fix, add a deferred one) and re-prompt.

After fixes apply, suggest: "If you want to verify nothing regressed, re-invoke `/plan-review`."

**CRITICAL: Set `model: "gpt-5.5-extra-high"` on the Task tool call for the implementer.** The implementer is a non-verifier review agent and uses the same preferred model as the reviewers.

**Subagent dispatch**: prefer `subagent_type: "Plan Review Implementer"`. If not in the session enum, fall back to `subagent_type: generalPurpose` with the full `pr-implementer.md` body inlined as the prompt. For either dispatch path, pass `model: "gpt-5.5-extra-high"`.

## Verdict Guidelines

- **Approve** — no blockers in either tier; suggestions are optional improvements; plan is executable as-is.
- **Request Changes** — has blockers or critical gaps that must be addressed before execution. If any strategic blockers exist, the natural path is "walk me through" for per-blocker decisions, or freeform reply if the user already knows what they want.
- **Questions Only** — no blockers, but unanswered questions need clarification before proceeding.

## Communication Style

- Use "this plan" or "the approach" instead of "you".
- Use reviewer output as input; produce a curated summary that's scannable in one read. Drop into per-blocker mode when asked. Hand off accepted fixes to the implementer subagent.
- Assume positive intent.
