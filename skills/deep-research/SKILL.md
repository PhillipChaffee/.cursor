---
name: deep-research
description: >-
  Tiered research and planning workflow that first estimates a task's complexity,
  then runs the matching effort level: a direct answer for trivial asks, a few
  parallel researchers for standard ones, or a full planner -> multi-model
  research -> synthesis pipeline for complex ones. Use when the user asks to
  research, investigate, look into, dig into, figure out, scope, deep dive,
  compare options, or plan something, or otherwise needs information gathered and
  synthesized across the codebase, the web, or connected tools.
disable-model-invocation: false
---

# Deep Research (tiered orchestrator)

Always do **Step 0 (complexity triage)** first, then run exactly one tier. Scale
effort to the task: don't spin up subagents for a trivial question, and don't
hand-answer a broad investigation that deserves the full pipeline.

This skill orchestrates personal subagents (in `~/.cursor/agents/`). Subagents run
in isolated context windows and do **not** see this conversation, so every Task
prompt you send must be **fully self-contained** (restate the goal, the relevant
context, the sources to use, and the exact output you want back).

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
  Choosing Tier 3 **commits you to the full subagent pipeline** (planner +
  researchers + synthesizer). If a dedicated planner and synthesizer subagent feel
  like overkill for this task, it is **Tier 2, not Tier 3** - pick Tier 2 instead.

## Model map (which model does which job)

Pick the researcher whose model matches each subtask's cognitive load. The model is
pinned in each subagent's frontmatter; you also pass the matching `model` on the
Task call (see "Dispatch rules").

- **opus 4.8 max** -> `claude-opus-4-8-thinking-max` - planning, synthesis, and
  high-knowledge / heavy-reasoning research. Agents: `research-planner`,
  `research-synthesizer`, `researcher-deep`.
- **gpt 5.5 xhigh** -> `gpt-5.5-extra-high` - in-between research (moderate
  reasoning, multi-source gathering). Agent: `researcher-mid`.
- **composer 2.5** -> `composer-2.5-fast` - simple, high-volume reads: code
  lookups, fact-finding, "where/what is X". Agent: `researcher-lite`.

## Tier 1 - Direct

1. Answer directly from your own knowledge.
2. If a single fact / file / symbol needs confirming, spawn **one**
   `researcher-lite` for it; otherwise use your own tools inline.
3. Give a concise answer with citations (`file:line` for code, URLs for web). No
   planner, no synthesizer, no canvas.

## Tier 2 - Standard

1. Create a short `TodoWrite` list (one item per research thread + "synthesize").
2. Decompose the request inline into 2-4 subtasks. Mark each as independent or
   dependent on another subtask's output, and assign each a tier (lite vs mid) by
   its difficulty.
3. **Dispatch dependency-aware research**: launch independent researchers in
   parallel with multiple Task calls in a single message (see "Dispatch rules").
   For dependent subtasks, wait for the dependency to return, then launch the next
   researcher with the prior findings included in its self-contained prompt.
4. Collect the findings and **synthesize them yourself** (no synthesizer subagent
   at this tier). Deduplicate overlaps; resolve contradictions or flag them.
5. Present a scannable summary with citations. Use a canvas only if the result is a
   standalone analytical artifact (see "Canvas").

## Tier 3 - Complex (full pipeline)

Tier 3 is a **delegation pipeline, not a solo effort**. Your role is **orchestrator
only**: you triage, dispatch subagents, relay their results, and decide tiers/
sequencing. You **MUST NOT** do the planning, the research, or the synthesis
yourself - each of those three roles runs as a **spawned subagent**. You are fully
capable of doing them inline, and that is exactly the trap: doing so wastes the
dedicated opus reasoning and isolated context that make this tier worthwhile, and it
is the **#1 failure mode of this skill**. Delegate even when it feels faster not to.

**Definition of a valid Tier 3 run.** Before you present anything, you must have
made, in order:

1. exactly **one** `research-planner` Task call,
2. **one or more** researcher Task calls (parallel or dependency-aware sequential), and
3. exactly **one** `research-synthesizer` Task call.

If you are about to present and any of these is missing, **STOP and make the missing
call**. If you genuinely don't want to spawn a planner and a synthesizer, you chose
the wrong tier - this is Tier 2; go run Tier 2 instead. Do not present a "Tier 3"
result that you planned or synthesized inline.

### Steps

1. **Track it**: create a `TodoWrite` list - `plan -> research (N subtasks) ->
   synthesize -> present`.
2. **Plan (MANDATORY subagent - your first action)**: immediately spawn
   `research-planner` (opus 4.8 max). Do **not** decompose the task yourself first -
   that is the planner's job; your first move after triage is this Task call. Pass
   it the full request, all known context, the available source types (codebase via
   Read/Grep/Glob/SemanticSearch, the web via WebSearch/WebFetch, and relevant MCP
   tools), and the roster of researchers (lite/mid/deep with their models). It
   returns a structured plan: subtasks each with `{id, goal, tier (lite|mid|deep),
   sources, expected_output}`, plus any open questions.
3. **Review the plan** briefly. Adjust tiers, merge redundant subtasks, or drop
   out-of-scope ones. If the planner surfaced a blocking ambiguity, ask the user
   before spending research effort.
4. **Dispatch dependency-aware research (MANDATORY subagents)**: use each
   subtask's `depends_on` value from the planner. Launch all subtasks with no unmet
   dependencies in parallel with multiple Task calls in a single message. After each
   wave returns, launch the next subtasks whose dependencies are now complete,
   including the needed dependency findings in each self-contained prompt. Do **not**
   gather the findings yourself with your own tools. If there are many ready
   subtasks, run them in batches (e.g. 4-6 at a time) rather than one giant fan-out.
5. **Synthesize (MANDATORY subagent)**: spawn `research-synthesizer` (opus 4.8 max)
   with the original request, the plan, and all researcher outputs (attributed by
   subtask id). Do **not** write the summary yourself - even though you could, the
   synthesis must run on opus in an isolated context. It returns one curated,
   deduplicated summary with citations preserved, and a canvas-ready structure when
   the deliverable warrants it.
6. **Present**: relay the synthesizer's summary. If it is a standalone analytical
   artifact, build a canvas (see "Canvas"); otherwise post the markdown summary in
   chat. Always surface open questions / gaps and the sources used.

## Dispatch rules (how to launch every subagent)

- **Parallelism**: launch independent subagents in one message with multiple Task
  calls so they run concurrently.
- **Sequential dependencies**: run subtasks sequentially only when a later subtask
  genuinely needs an earlier researcher's output. Feed the relevant prior findings
  into the dependent subagent's prompt, and keep unrelated subtasks parallel.
- **Self-contained prompts**: subagents can't see this chat. Restate the goal,
  paste/point to the needed context, name the sources to use, and specify the exact
  return format.
- **Model selection**: pass `model` on each Task call to match the agent's role:
  - `research-planner`, `research-synthesizer`, `researcher-deep` ->
    `claude-opus-4-8-thinking-max`
  - `researcher-mid` -> `gpt-5.5-extra-high`
  - `researcher-lite` -> `composer-2.5-fast`
  The same model is pinned in each agent's frontmatter, so the two agree. If your
  Task tool rejects the `model` value (some plans only accept `"fast"`), omit
  `model` and rely on the frontmatter pin instead.
- **subagent_type (a missing type is NOT permission to skip the subagent)**: use
  the agent's name (`research-planner`, `researcher-deep`, `researcher-mid`,
  `researcher-lite`, `research-synthesizer`). If a name isn't yet in the Task
  `subagent_type` enum - newly added agents only register after a Cursor reload, so
  this is common right after install - you **MUST still run the subagent via the
  fallback**: `subagent_type: generalPurpose` with that agent's full file body (read
  it from `~/.cursor/agents/<name>.md`) inlined as the prompt, still passing the
  matching `model`. Never substitute doing the step yourself just because the named
  type wasn't selectable - that is exactly how the planner/synthesizer get skipped.
- **readonly**: do **not** pass `readonly: true` on researcher Task calls - they
  need web search/fetch and MCP access. Their no-edit discipline comes from their
  prompts. (`research-planner` and `research-synthesizer` are `readonly: true` in
  frontmatter; that's fine - they don't fetch new external data.)

## Canvas

For Tier 3 (and Tier 2 when it fits), decide per the canvas skill at
`~/.cursor/skills-cursor/canvas/SKILL.md`: build a canvas when the output is a
**standalone analytical artifact** the user would want beside the chat (structured
findings, comparisons, multi-section reports, data tables). Skip the canvas for a
direct answer, a quick summary, or work that's a means to another deliverable. Read
that skill before creating any `.canvas.tsx` file.

## Guardrails

- **Triage out loud**: always state the chosen tier and a one-line reason first.
- **Tier 3 means delegate (do not fake it)**: if you didn't spawn a
  `research-planner` and a `research-synthesizer` (plus researchers), you did not
  run Tier 3 - regardless of how good the inline result looks. Either spawn the
  subagents (use the `generalPurpose` fallback if they're not registered yet) or
  relabel the run as Tier 2. Planning and synthesis done inline is the failure this
  skill exists to prevent.
- **Never dump raw subagent output**: you (or the synthesizer) produce a single
  curated result. Deduplicate overlapping findings; reconcile or flag conflicts.
- **Cite everything**: `file:line` for code, URLs for web, tool/source name for MCP.
- **Right-size effort**: prefer the lowest tier that fully answers the question.
- **Surface gaps**: list what couldn't be confirmed and what would resolve it.
- **Model-resolution caveat**: Cursor has a known bug where a subagent's model can
  be ignored or overridden. You've mitigated it (frontmatter pin + matching `model`
  on the Task call). If a run's quality looks off, note that the intended model may
  not have been used.
