---
name: ultracode
description: >-
  Runs a substantive request as an exhaustive, staged multi-agent effort.
  Specialist skills such as ship or code-review run as nested Ultracode
  steps; remaining Ultracode phases and verification still apply. Use when
  the user explicitly invokes /ultracode.
disable-model-invocation: true
---

# Ultracode

Ultracode is on: optimize for the most exhaustive, correct answer — not the
fastest or cheapest. Token cost is not a constraint. See the quality patterns.
Solo only on conversational or trivial turns.

A workflow is a deterministic driver in which Task is the only
nondeterministic call. Control flow — loops, fan-out, dedup, thresholds,
early exit, synthesis — is real structure you write. Only the LLM steps vary
between runs. That is what the three goals are for:

| Goal | How the structure delivers it |
|---|---|
| **Comprehensiveness** | Decompose the problem, cover the pieces in parallel |
| **Confidence** | Independent perspectives + adversarial checks before committing to a finding |
| **Scale** | Work that one context window can't hold: migrations, audits, broad sweeps |

Cursor has no Workflow JS harness. Do not call `agent()`, `pipeline()`,
`parallel()`, `schema`, `budget`, or prefix-cache `resume`. Treat each Task as
`agent()`. An item-local sequence in one subagent is the pipeline analog.
Waiting for the complete prior result set is the barrier analog.

The parent stays in the loop throughout: it establishes scope, selects phases,
reads every result, makes decisions, owns edits and verification, and writes
the final response.

When a specialist skill owns part of the work, run that skill as a nested
Ultracode step. Then continue this skill: remaining phases, section 7
verification, stop conditions, and the parent report.

## Precedence

Ultracode is lower priority than:

1. Explicit user instructions and scope limits.
2. The active interaction mode.
3. Workspace and repository rules.
4. Security, production-data, authentication, and secrets boundaries.
5. Normal authorization requirements for writes and destructive actions.

Ultracode raises thoroughness. It never raises authority or expands scope.
An explicit "do not fan out" wins over standing opt-in.

## 1. Check whether the request is substantive

**Ultracode.** `/ultracode` is standing opt-in for this turn: author and run a
fan-out for every substantive task by default. The goal is the most exhaustive,
correct answer you can produce — token cost is not a constraint. For
multi-phase work (understand → design → implement → review), that often means
several fan-outs in sequence — one per phase — so you stay in the loop between
them. The quality patterns below (adversarial verify, multi-modal sweep,
completeness critic, loop-until-dry) are the tools; pick what fits the task.
Lean toward orchestrating with Task waves and adversarially verifying your
findings — unless the work is trivial or already verified. Solo only on
conversational turns or trivial mechanical edits.

State one line:

`Ultracode: substantive`

or:

`Ultracode: trivial - answering solo`

Answer solo when the turn is conversational, a single-fact lookup, a trivial
mechanical edit, or a follow-up whose result was already verified in this
Ultracode run. Completing a nested specialist skill does not make the request
trivial or already verified. Exhaustive orchestration on trivial work is a
failure, not thoroughness.

## 2. Nest a specialist skill as an Ultracode step

A matching specialist skill is a nested Ultracode phase, not a substitute for
Ultracode. Detecting an owner is not Ultracode done. Sections 3 through 9 stay
in force after and around the child.

When an available skill matches the request:

1. **Detect** the owner. These are common matches rather than a complete list,
   so check the available skills for a closer owner:
   - Research, investigation, or scoping: `deep-research`
   - Reviewing a diff, branch, PR, or code change: `code-review`
   - Reviewing a plan or design: `plan-review`
   - Iterative code review and repair: `looping-code-review`
   - Iterative plan review and alignment: `looping-plan-review`
   - GitLab merge request review: `mr-review`
   - Linear ENG work through implementation and MRs: `ship`
   - Behavior-preserving refactor planning: `refactor-planner`
   - Pre-MR checks: `pre-mr-checklist`

2. **Run it as a named Ultracode phase.** The parent reads and executes the
   child skill directly. Never invoke a child skill inside a subagent.
   Preserve the child's required agents, phases, models, and exit criteria for
   that nested step. Do not rewrite or replace that rubric. Child exit
   criteria end the nested step only; they are not Ultracode stop conditions.

3. **Map coverage.** After reading the child skill, state which Ultracode
   phases it covers and which it does not. Continue Ultracode for every phase
   the child does not cover. Never map a child's own verify, review, test, or
   exit step as covering section 7.

4. **Keep Ultracode verification.** Child checks do not replace section 7 on
   the overall request. Still run completeness critic and adversarial
   verification of consequential findings, including claims the child treated
   as done. A child "verify" phase is not Ultracode section 7.

5. **Do not duplicate the child's panel.** Do not add a second copy of the
   child's identical reviewer agents for the same phase. Do add Ultracode
   waves the child never runs.

Examples:

- `/ship` covers ticket, plan, implement, and MR. Ultracode still scouts,
  still fans out for research and review as needed, still verifies, and still
  reports. Ship does not run a completeness critic or adversarial refutation
  of "AC is met"; Ultracode still does.
- `/code-review` covers the Review phase rubric. Ultracode still does any
  phases before and after, plus completeness critic and adversarial
  verification of the overall request.

## 3. Scout and declare the phase plan

When you do fan out, the right move is often **hybrid**: scout inline first
(list the files, find the channels, scope the diff) to discover the work-list,
then pipeline over it. You don't need to know the shape before the *task* —
only before the *orchestration step*.

Inline scouting is appropriate only for a quick, bounded lookup. Delegate
unfamiliar, broad, or multi-file discovery to the lowest-cost collector
allowed by the active Subagent Use rule.

Common single-phase workflows you can chain across turns:

- **Understand** — parallel readers over relevant subsystems → structured map
- **Design** — judge panel of N independent approaches → scored synthesis
- **Review** — dimensions → find → adversarially verify
- **Research** — multi-modal sweep → deep-read → synthesize
- **Migrate** — discover sites → transform each → verify
- **Implement** — bounded, independent change sets; parent applies edits

For larger work, run several in sequence — read each result before deciding
the next phase. You stay in the loop; each fan-out is one well-scoped wave.

```
scout inline  →  fan-out (understand)  →  read result
              →  fan-out (design)      →  read result
              →  fan-out (implement)   →  read result
              →  fan-out (review)      →  report
```

One fan-out per phase. Not one mega-workflow that tries to do everything —
that forfeits your judgment at exactly the points where it matters most.

Scout and declare this plan even when a specialist skill will own later
phases. A nested child skill is one declared phase, not the whole plan.

Apply the section 2 nest per phase as well as per request. If a selected
phase is owned by a specialist skill, run that skill as the named Ultracode
phase. Do not also build a generic wave that copies the child's reviewers for
that same phase. After the child finishes, continue remaining declared phases
and section 7.

Use `TodoWrite` for one todo per selected phase plus `synthesize`. Include the
nested child as one todo, every Ultracode phase it does not cover, a verify
todo for section 7, and `synthesize`. State the selected phases, planned wave
size, and stopping criterion before the first wave. Never add a phase
silently.

Run phases in sequence. The parent reads and evaluates one phase before
starting the next. Do not collapse a multi-phase run into one mega-wave.

## 4. Run bounded, dependency-aware waves

DEFAULT TO pipeline(). Only reach for a barrier when you genuinely need ALL
prior-stage results together.

A barrier is correct ONLY when stage N needs cross-item context from all of
stage N-1:

- Dedup/merge across the full result set before expensive downstream work
- Early-exit if the total count is zero ("0 bugs found → skip verification entirely")
- Stage N's prompt references "the other findings" for comparison

A barrier is NOT justified by:

- "I need to flatten/map/filter first" — do it inside the item-local chain
- "The stages are conceptually separate" — that's what pipeline models.
  Separate stages ≠ synchronized stages.
- "It's cleaner" — barrier latency is real. If 5 finders run and the slowest
  takes 3× the fastest, a barrier wastes 2/3 of the fast finders' idle time.

Smell test: if you would write

```
const a = await parallel(...)
const b = transform(a)        // flatten, map, filter — no cross-item dependency
const c = await parallel(b.map(...))
```

that middle transform doesn't need the barrier. Rewrite as a pipeline with
the transform inside a stage. When in doubt: pipeline.

One-line test: does stage N need to see *other items'* stage N-1 results?
No → pipeline.

Cursor mapping: launch independent Tasks in one message. An item-local
sequence in one subagent prompt is pipeline — "map this module's public
interface, then list every caller you find." Waiting for the complete prior
result set is the only barrier. Never fold finding and verifying into one
agent; a verifier has to be independent of the agent whose work it checks,
so run the verifier as a later Task. That later Task may start per item
without waiting for sibling items.

Scale to what the user asked for. "find any bugs" → a few finders, single-vote
verify. "thoroughly audit this" or "be comprehensive" → larger finder pool,
3–5 vote adversarial pass, synthesis stage. When unsure, lean toward
thoroughness for research/review/audit requests and toward brevity for quick
checks.

Default to 3-6 subagents per wave. Increase toward 10 only when the scouted
work list justifies the coverage. Guideline, not a hard cap: keep a wave
under ~15 agents unless the user's prompt calls for a different scale.
Declare the expected run total and explain any increase.

- Launch all independent subagents in one message.
- Run dependent work in a later wave and pass curated prior evidence forward.
- Multi-modal sweep: parallel agents each searching a different way
  (by-container, by-content, by-entity, by-time). Each is blind to what the
  others surface; useful when one search angle won't find everything. Do not
  seed them with each other's results.
- Give every subagent a self-contained prompt with the goal, sources,
  constraints, allowed mutations, and exact return format.
- Follow the active Subagent Use rule and current Subagent tool schema for
  agent types and model routing. Do not hard-code or substitute an
  unavailable model.
- Default to omitting a model override — inherit the session model, which is
  almost always correct. Only set it when you're highly confident a different
  tier fits the task; when unsure, omit. Use the default Subagent Use tier
  for cheap mechanical stages and a stronger reasoning model only for the
  hardest verify, judge, and synthesis steps, within what that rule allows.
- Honor an explicit user model choice when that listed model is available.
- Keep the parent active. Never delegate the entire request and merely relay
  the result. Relaying a nested child skill's output is the same failure.

"Cleaner" and "conceptually separate" do not justify holding a wave.

For background subagents, rely on completion notifications. Do not poll,
predict, or report a pending result.

## 5. Enforce the subagent return contract

Subagents are told their final text IS the return value (not a human-facing
message), so they return raw data. Don't ask a subagent to explain nicely —
ask it for data, and shape prose in the synthesis step or the main context.

Require:

- exact fields or one line per finding
- `file:line` citations for code and URLs for web sources
- a defined empty result such as `no issues`
- uncertainty and uninspected scope stated explicitly

If a result is missing or violates the contract, re-dispatch with the contract
restated or report that item as a gap. Never treat a missing return as a clean
result.

No silent caps: if a wave bounds coverage (top-N, no-retry, sampling), state
what was dropped — silent truncation reads as "covered everything" when it
didn't. State every sample, skipped system, or single-round bound in both
progress updates and the final response.

## 6. Keep mutation isolated

The parent owns edits. Serialize every change through the parent and never run
concurrent writers against the same working tree. A subagent may propose a
diff; the parent applies it.

Ultracode does not authorize commits, pushes, merges, deployments,
destructive commands, or external writes.

## 7. Verify before believing

Verify consequential findings before presenting them as facts. Run this
section on the overall request even when a nested child skill already did its
own checks. A child "verify" phase is not Ultracode section 7.

Quality patterns — common shapes; pick by task and compose freely:

- **Adversarial verify**: spawn N independent skeptics per finding, each
  prompted to REFUTE. Kill if ≥majority refute. Prevents plausible-but-wrong
  findings from surviving. Prompt:
  `Try to refute: ${claim}. Default to refuted=true if uncertain.`
  Bounded check: one skeptic. High-stakes or exhaustive: three. Survives only
  if a majority do not refute.

- **Perspective-diverse verify**: when a finding can fail in more than one
  way, give each verifier a distinct lens (correctness, security, perf,
  does-it-reproduce) instead of N identical refuters — diversity catches
  failure modes redundancy can't. It survives only if a majority of lenses
  judges it real.

- **Judge panel**: generate N independent attempts from different angles
  (e.g. MVP-first, risk-first, user-first), score with parallel judges,
  synthesize from the winner while grafting the best ideas from runners-up.
  Beats one-attempt-iterated when the solution space is wide. The judges must
  not be the authors.

- **Loop-until-dry**: for unknown-size discovery (bugs, issues, edge cases),
  keep spawning finders until K consecutive rounds return nothing new.
  Simple counters (while count < N) miss the tail. Dedup against everything
  *seen*, not everything *confirmed* — else judge-rejected findings reappear
  every round and it never converges.

- **Multi-modal sweep**: parallel agents each searching a different way
  (by-container, by-content, by-entity, by-time). Each is blind to what the
  others surface.

- **Completeness critic**: a final agent that asks "what's missing —
  modality not run, claim unverified, source unread?" What it finds becomes
  the next round of work.

- **No silent caps**: if a wave bounds coverage (top-N, no-retry, sampling),
  state what was dropped.

These patterns aren't exhaustive — compose novel harnesses when the task
calls for it (tournament brackets, self-repair loops, staged escalation,
whatever fits).

| Situation | Shape |
|---|---|
| N independent items, multiple stages each | pipeline (item-local, no sibling wait) |
| Need all of stage N-1 together (dedup, early exit, cross-comparison) | barrier |
| Unknown-size discovery | loop-until-dry (K empty rounds) |
| Fixed target count | loop-until-count |
| Wide solution space | judge panel |
| Findings that could be plausible-but-wrong | adversarial verify (majority-refute kill) |
| Findings that can fail in several distinct ways | perspective-diverse verify |

For implemented changes, run verification proportional to risk and check
diagnostics for edited files. A critic's new in-scope finding becomes another
declared wave; an out-of-scope finding becomes a stated gap.

## 8. Stop on convergence or a real boundary

Stop when any condition holds:

- All declared phases are complete and the completeness critic finds no new
  in-scope work.
- Two consecutive waves hit the same blocker without a viable new approach.
- The next step requires a user decision or authorization.
- A rule, security boundary, or explicit instruction blocks the next step.

A nested child skill finishing is not a stop condition. Stop only when one of
the conditions above holds for the overall request.

Do not widen scope to manufacture completeness. State why the run stopped.

## 9. Report from the parent

The parent writes the final response and never pastes raw subagent output.

Report:

1. The direct answer or completed outcome.
2. Verified findings with citations, ordered by importance.
3. Findings rejected by verification when they affected the investigation.
4. Coverage caps, skipped scope, and unresolved gaps.
5. Open questions or required user decisions.

Mention phases and subagent count only when the scale matters to the user.
