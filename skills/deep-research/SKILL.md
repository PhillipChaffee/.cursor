---
name: deep-research
description: >-
  Tiered research and planning workflow that first estimates a task's complexity,
  then runs the matching effort level: a direct answer for trivial asks, a few
  parallel researchers for standard ones, or a scoped research -> synthesis
  pipeline for complex ones with optional thinking-only planning. Use when the
  user asks to research, investigate, look into, dig into, figure out, scope, deep
  dive, compare options, or plan something, or otherwise needs information
  gathered and synthesized across the codebase, the web, or connected tools.
disable-model-invocation: false
---

# Deep Research (tiered orchestrator)

Always do **Step 0 (complexity triage)** first, then run exactly one tier. Scale
effort to the task: don't spin up subagents for a trivial question, and don't
hand-answer a broad investigation that deserves the full pipeline.

Subagents run in isolated contexts and do not see this conversation. Every prompt must restate
the goal, relevant context, sources, constraints, and expected output.

## Step 0 - Complexity triage (always)

Score the request across four axes, then pick a tier:

- **Breadth** - how many distinct areas / files / sources are involved?
- **Depth** - how much reasoning, domain knowledge, or synthesis is required?
- **Ambiguity** - is the question well-formed, or does it need scoping first?
- **Stakes** - how costly is a shallow or wrong answer?

State the chosen tier in one short line before proceeding (e.g.
`Triage: Tier 2 (standard) - 3 independent threads, low ambiguity`). Then run that
tier. When genuinely on the fence between two tiers, pick the lower one but say so.

- **Tier 1 - Trivial / Direct**: answerable from what you already know or with one
  quick lookup. Narrow scope, low stakes.
- **Tier 2 - Standard**: a handful of mostly-independent threads; the shape of the
  answer is clear; little upfront scoping needed.
- **Tier 3 - Complex / Deep**: broad, ambiguous, high-stakes, or needs a plan
  before research can even start. Multiple interacting threads, heavy synthesis.
  Choosing Tier 3 **commits you to delegated scoping, research, and synthesis**.
  Roles still default to grok. A Fable upgrade is optional when deeper thinking-only
  reasoning is justified after evidence has been collected (see Models).

## Models

- `researcher-lite`: `composer-2.5-fast`
- `researcher-mid`, `researcher-deep`, `research-planner`, and `research-synthesizer`:
  `cursor-grok-4.5-high-fast`

Pass the matching model explicitly on every subagent call and keep it aligned with the agent
frontmatter.

**Default every role to grok** (lite stays on composer). These roles formerly defaulted to Fable
and may be upgraded to `claude-fable-5-thinking-max` when the criteria below apply:

| Role | Upgrade when |
|------|----------------|
| `researcher-deep` | Heavy architecture/tradeoff, security/performance, or conflicting-source reasoning, and the subtask evidence packet is already complete (thinking-only; no further source inspection). |
| `research-planner` | Difficult decomposition remains after collectors assembled a complete evidence packet — including irreversible schema/deploy sequencing or cross-service contract design. |
| `research-synthesizer` | Large, conflicting, or high-stakes researcher outputs need deeper judgment to curate. |

`researcher-mid` never uses Fable — if thinking-only Fable reasoning is needed, assign
or reassign the subtask to `researcher-deep` instead. Tier 2 never launches
`research-planner` or `research-synthesizer` (the main chat synthesizes). Tier 3 alone
does not justify Fable, but after collectors finish, prefer Fable for `research-planner`
(or `researcher-deep`) when the remaining question is irreversible schema/deploy
sequencing or cross-service contract design. When unsure on other cases, stay on grok.
The main chat applies any upgrade when launching the subagent; this skill does not
switch models itself. Fable prompts must forbid tools and source lookups and must stop
with exact missing evidence rather than gathering anything.

## Tier 1 - Direct

1. Answer directly from your own knowledge.
2. If a single fact / file / symbol needs confirming, spawn **one**
   `researcher-lite` for it; otherwise use your own tools inline. If the parent
   model is Fable, do not use tools inline — always spawn a Composer or Grok
   collector instead.
3. Give a concise answer with citations (`file:line` for code, URLs for web). No
   planner, no synthesizer, no canvas.

## Tier 2 - Standard

1. Create a short `TodoWrite` list (one item per research thread + "synthesize").
2. Decompose the request inline into 2-4 subtasks. Mark each as independent or
   dependent on another subtask's output, and assign each a tier (lite vs mid) by
   its difficulty.
3. **Dispatch dependency-aware research**: launch independent researchers in
   parallel with multiple Task calls in a single message.
   For dependent subtasks, wait for the dependency to return, then launch the next
   researcher with the prior findings included in its self-contained prompt.
4. Collect the findings and **synthesize them yourself** (no synthesizer subagent
   at this tier — do not launch `research-planner` or `research-synthesizer`).
   Deduplicate overlaps; resolve contradictions or flag them.
5. Present a scannable summary with citations. Use a canvas only if the result is a
   standalone analytical artifact (see "Canvas").

## Tier 3 - Complex (full pipeline)

Tier 3 is a **delegation pipeline, not a solo investigation**. Your role is to
scope the work, dispatch collectors and researchers, optionally upgrade a
thinking-only Fable subagent when the Models criteria apply, and relay the
synthesized result. All source inspection belongs to Composer or Grok researchers.

**Definition of a valid Tier 3 run.** Before you present anything, you must have
made, in order:

1. **one or more** Composer or Grok collector calls to gather planning evidence
   (scoping),
2. optionally **one** `research-planner` call (optionally on Fable) when the Models
   upgrade criteria apply and a complete evidence packet exists,
3. **one or more** Composer or Grok researcher calls for the planned subtasks, and
4. exactly **one** `research-synthesizer` call (grok by default; optionally Fable per
   Models).

Do not use Fable merely because the task is Tier 3. Upgrade when the remaining
question is a difficult thinking-only reasoning problem — for example decomposing
a high-stakes investigation or analyzing a complete evidence packet that no longer
needs source lookups. The main chat applies any upgrade when launching the
subagent; this skill does not switch models itself.

### Steps

1. **Track it**: create a `TodoWrite` list - `scope -> plan -> research (N subtasks)
   -> synthesize -> present`.
2. **Collect planning evidence**: use Composer or Grok collectors to gather the
   codebase, web, MCP, log, or command evidence needed to scope concrete research
   subtasks. Launch independent collection in parallel. Do not give this work to
   Fable.
3. **Plan**: normally decompose the work with Grok from the collected evidence.
   Upgrade to `research-planner` on Fable (or elevate a deep subtask to Fable) when
   the Models upgrade criteria apply. Never put Fable on `researcher-mid` — use
   `researcher-deep` instead.
   - Give Fable one discrete planning or reasoning question and a compact, complete
     evidence packet containing the request, constraints, source inventory, relevant
     excerpts, competing findings, and unresolved decisions.
   - Explicitly forbid tools, file reads, repository searches, web or MCP fetches,
     shell commands, tests, builds, and diagnostics.
   - If Fable reports missing evidence, send that exact gap to a Composer or Grok
     collector, then resume or rerun Fable with the completed packet. Never let
     Fable gather the missing evidence.
4. **Review the plan** briefly. Adjust tiers, merge redundant subtasks, or drop
   out-of-scope ones. Resolve blocking user choices before spending more research
   effort.
5. **Dispatch dependency-aware research (MANDATORY subagents)**: launch all
   subtasks with no unmet dependencies in parallel on grok by default. After each
   wave returns, launch the next subtasks whose dependencies are complete and
   include the needed prior findings in each self-contained prompt. Upgrade a
   `researcher-deep` subtask to Fable when deeper thinking-only reasoning is
   justified and its evidence packet is complete — never upgrade mid; reassign to
   deep if Fable is needed. Keep unrelated work parallel and use batches of 4-6
   for large fan-outs.
6. **Synthesize (MANDATORY subagent)**: spawn the `research-synthesizer` on grok by default
   (optionally Fable per Models) with the original request, the plan, and all researcher
   outputs (attributed by subtask id). Do **not** write the summary yourself - even though you
   could, the synthesis must run in an isolated context. It returns one curated, deduplicated
   summary with citations preserved, and a canvas-ready structure when the deliverable warrants
   it.
7. **Present**: relay the synthesizer's summary. If it is a standalone analytical
   artifact, build a canvas (see "Canvas"); otherwise post the markdown summary in
   chat. Always surface open questions / gaps and the sources used.

## Dispatch

- Launch independent researchers in parallel; run dependent work sequentially with prior findings.
- If a named type is unavailable, use `generalPurpose` with its `.cursor/agents/` definition
  inlined and the same model. Never skip a required role.
- Do not pass `readonly: true` to researchers that need web or MCP access; prohibit edits in the
  prompt instead.
- Fable is thinking-only. Its prompt must be self-contained, must forbid every
  tool and source lookup, and must tell it to stop with exact missing evidence
  rather than gathering anything.
- Curate and deduplicate outputs, preserve citations, and surface conflicts or gaps.

## Canvas

For Tier 3 (and Tier 2 when it fits), decide per the canvas skill at
`~/.cursor/skills-cursor/canvas/SKILL.md`: build a canvas when the output is a
**standalone analytical artifact** the user would want beside the chat (structured
findings, comparisons, multi-section reports, data tables). Skip the canvas for a
direct answer, a quick summary, or work that's a means to another deliverable. Read
that skill before creating any `.canvas.tsx` file.

## Guardrails

- **Triage out loud**: always state the chosen tier and a one-line reason first.
- **Tier 3 means delegate**: if Composer or Grok collectors did not gather scoping
  evidence, Composer or Grok researchers did not inspect the sources, and a
  synthesizer did not merge their findings, you did not run Tier 3. Fable upgrades
  remain optional per the Models criteria for deep, planner, and synthesizer (the
  roles that formerly defaulted to Fable).
- **Grok by default, Fable upgrade when criteria apply**: no skill role defaults to
  Fable. Documented upgrade criteria live in this skill for the main chat to apply
  when launching subagents. Upgrade only for difficult thinking-only reasoning after
  the evidence packet is complete, and state why.
- **Fable never gathers evidence**: no file or repository reads, searches, web or
  MCP access, shell commands, tests, builds, or diagnostics. Give it a complete
  evidence packet or do not dispatch it. If the parent model is Fable, delegate
  every source lookup and command to Composer or Grok — including Tier 1 confirmations
  (never use tools inline from a Fable parent).
- **Cite everything**: `file:line` for code, URLs for web, tool/source name for MCP.
- **Right-size effort**: prefer the lowest tier that fully answers the question.
- **Surface gaps**: list what couldn't be confirmed and what would resolve it.
