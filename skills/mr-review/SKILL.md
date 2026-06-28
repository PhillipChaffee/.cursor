---
name: mr-review
description: Review a GitLab merge request end-to-end. Fetches MR details and diffs via the GitLab MCP, checks out the branch locally, runs a code review focused on Pydantic usage, DRY/SOLID, code placement, naming, alternative approaches, and preferring stdlib/libraries over custom code, then surfaces all findings to the user for selective posting. Use when the user asks to review a GitLab MR, provides a GitLab MR link, or mentions MR review.
---

# GitLab MR Review

Review a GitLab MR, surface findings to the user for approval, then post selected comments as draft notes (in your team's review voice if you keep one).

## Dependencies

This skill composes one other skill and one optional rule:

- **Code review skill** (`~/.cursor/skills/code-review/SKILL.md`) — provides the review methodology, severity labels, checklists, and feedback format. Used in **review-only mode** (see Step 5) so it produces the curated summary without prompting for walkthrough or fix application. Read it before reviewing.
- **MR comment voice rule** (optional, e.g. `~/.cursor/rules/mr-comment-voice/RULE.mdc`) — if you keep a rule defining your team's writing voice for review comments, load it before writing comments so they match that style. Not required; skip if you don't have one.

## Review Priorities

These are the highest-priority areas. The code-review skill covers general correctness/security/performance, but **weight these six areas most heavily** — they're the areas most worth calling out explicitly:

### 1. Pydantic over raw containers

- New structured data should use `pydantic.BaseModel`, not raw dicts, `TypedDict`, or `dataclasses`.
- `dataclasses.dataclass` is acceptable only for small internal containers with trusted inputs and no validation needs.
- Flag any new `TypedDict`, `NamedTuple`, or plain dict used where a Pydantic model would be clearer.

### 2. Use existing libraries / builtins instead of writing code

- If a standard library function, a Pydantic validator, or an already-imported third-party library can do the job, prefer that over hand-rolled logic.
- Common misses: reimplementing `itertools`, `collections`, `functools`, date math, URL parsing, retry logic that `tenacity` already provides, manual JSON schema that Pydantic generates for free.
- Flag custom utility functions that duplicate what a dependency already exposes.

### 3. DRY

- Same logic in multiple places with minor variations.
- Copy-pasted blocks (even with small tweaks).
- Magic strings/numbers repeated across files.
- Configuration values hardcoded in multiple locations.
- Similar conditionals checking the same thing in different ways.

### 4. SOLID

- **SRP**: Classes/functions doing multiple unrelated things.
- **OCP**: Adding features requires modifying existing code instead of extending.
- **DIP**: High-level modules directly instantiating low-level concrete classes.
- Focus on SRP and DIP — they're the most common violations.

### 5. Code placement

- New functions, classes, and methods should live in the module/file where they logically belong, not just wherever the author happened to be editing.
- Flag functions added to a file when they don't relate to that file's core responsibility. Ask: "Does this belong in `X` or would `Y` be a better home?"
- Common smells:
  - A utility/helper added to a domain-specific service file instead of a shared utils module.
  - A model method added to a service layer or vice versa.
  - Business logic stuffed into a route handler, callback, or serializer.
  - A function that only operates on data from module B but lives in module A.
- When the codebase already has a clear module structure, new code should follow it. If there's an existing `utils/`, `models/`, `services/` breakdown, new code should land in the right bucket.
- Consider cohesion: functions in the same file should be related. If a new function has no relationship to its neighbors, it probably belongs elsewhere.

### 6. Naming

- Names must be descriptive, intuitive, and self-documenting.
- Flag: single-letter vars outside short loops, type suffixes (`user_list`, `data_dict`), vague names (`handle`, `process`, `data`, `info`, `manager`, `utils`), abbreviations that aren't universally understood.
- Variable names should tell you what the value *represents*, not what type it is.
- Function names should tell you what they *do*, not how they do it.

## Workflow

### Step 1: Fetch the MR and understand product purpose

Extract `project_id` and `merge_request_iid` from the GitLab URL or user input.

```
GitLab URL pattern: https://gitlab.com/{group}/{project}/-/merge_requests/{iid}
→ project_id = "{group}/{project}", merge_request_iid = "{iid}"
```

Fetch in parallel:

1. **MR details** — `get_merge_request` tool (need `diff_refs` for inline comments, and the **description** for product purpose)
2. **MR diffs** — `get_merge_request_diffs` tool (exclude `poetry.lock`, `package-lock.json`, etc.)
3. **Existing discussions** — `mr_discussions` tool (avoid duplicating existing feedback)

Save `diff_refs.base_sha`, `diff_refs.head_sha`, `diff_refs.start_sha` — required for inline comments.

#### Understand product purpose

Read the MR title and description carefully. Extract:

- **What product/user problem is this solving?** Look for Problem/Why sections, linked tickets, or the title.
- **What is the intended behavior change?** Look for Fix/What/How sections or infer from the description.
- **What constraints or context shaped the approach?** Look for mentions of deadlines, backward compatibility, deployment order, or technical debt.

Summarize the product purpose in 1-2 sentences. This understanding drives the alternative-approaches analysis in Step 3 and helps distinguish intentional design decisions from accidental complexity. Keep this summary — you will present it to the user in Step 3.

If the MR description is empty or unclear, note this as a finding (the MR should document its purpose). Still infer the purpose from the code changes as best you can.

### Step 2: Check out the MR branch locally

Before reading code, check out the MR's source branch so all file reads reflect the actual changes:

1. **Identify the submodule** — Determine which submodule(s) the MR belongs to based on the `project_id` (e.g., the Django service, the FastAPI service, or `shared/`).
2. **Fetch and checkout** — In the relevant submodule directory:
   ```
   git fetch origin <source_branch>
   git checkout <source_branch>
   ```
   The source branch name is available from the MR details (`source_branch` field).
3. **Verify** — Confirm the checkout succeeded and HEAD matches the MR's `head_sha`.

If the checkout fails (e.g., branch not found locally), fall back to fetching by the head SHA:
```
git fetch origin <head_sha>
git checkout <head_sha>
```

After the review is complete, remind the user that the submodule is now on the MR branch and they may want to switch back.

### Step 3: Evaluate alternative approaches (gate)

**This step runs before the detailed code review.** If the overall approach is wrong, a line-by-line review is wasted effort.

#### 3a. Read changed files at a high level

Read the full changed files from the local repo (now on the MR branch) to understand *what* the MR does and *how* it accomplishes it. You don't need to trace every caller yet — focus on grasping the shape of the change: which modules are touched, what abstractions are introduced or modified, and what the end-to-end flow looks like.

#### 3b. Analyze alternative approaches

Using the **product purpose** from Step 1 and the high-level understanding from 3a, consider whether the same goal could have been achieved differently. Evaluate each of these lenses:

- **Configuration over code** — Could the behavior change be driven by a config/feature flag, database setting, or environment variable instead of a code change?
- **Existing abstractions** — Does the codebase already have a mechanism that could handle this (an existing hook, plugin system, event handler, middleware, etc.)?
- **Simpler implementation** — Could fewer files be touched, fewer abstractions introduced, or a more straightforward approach accomplish the same thing?
- **Different layer** — Would this be better handled at a different layer (e.g., database constraint instead of application validation, API gateway instead of per-service logic, frontend instead of backend)?
- **Avoiding the change entirely** — Is there a reason the existing behavior is actually correct and the change is unnecessary?

Only surface alternatives that are **concretely better** (simpler, safer, more maintainable, or more consistent with existing patterns). Do not raise alternatives just for the sake of it.

#### 3c. Present to the user and wait

Present the product purpose summary and any alternative approaches to the user. Format:

```
### Product purpose
<1-2 sentence summary from Step 1>

### Alternative approaches
<If none: "No better alternatives identified — the approach looks reasonable. Proceeding with detailed review.">

<If alternatives exist, list each one:>
**Alternative [N]: <short title>**
<What the alternative is, why it's better, and any trade-offs.>
```

Then ask the user how to proceed:

- **Continue with review** — the current approach is fine, proceed to the detailed code review (Step 4+)
- **Stop — will request a rewrite** — the user will ask the MR author to take a different approach; skip the rest of the review
- **Continue but include alternative as a comment** — proceed with the detailed review and include the alternative approach as a general MR comment in the findings

**Do not proceed to Step 4 until the user responds.** If the user chooses to stop, skip to Step 11 (report) and remind them which submodule is on the MR branch.

### Step 4: Gather deep context

For each changed file, read as much surrounding code as needed to **fully understand the changes and verify correctness**. The goal is to catch bugs, broken flows, and incorrect assumptions — not just style issues. Since the MR branch is checked out locally (Step 2), all file reads will reflect the MR's actual state.

#### 4a. Read the full changed files

If you already read the full files in Step 3a, you can skip re-reading them. Otherwise, read the **entire** version of each changed file from the local repo (now on the MR branch), not just the diff hunks. You need the full picture to understand how the changes fit in.

#### 4b. Trace callers and callees

For every function/method/class that was **modified, added, or has a changed signature**:

- **Search for all callers** of that function across the codebase. Verify each call site is still correct after the change (argument order, new required params, changed return type, removed fields, etc.).
- **Read the functions it calls** to understand downstream effects. If the change alters what data flows into a downstream function, read that function to verify it still works.
- **Check imports** — if the MR adds/removes/renames exports, search for every file that imports them.

#### 4c. Trace data flow for changed models/types

If the MR changes a Pydantic model, dataclass, TypedDict, dict shape, or function return type:

- Search for every place that type is constructed, consumed, serialized, or deserialized.
- Check that all consumers handle the new shape (new fields, removed fields, type changes, Optional→required or vice versa).
- Pay special attention to API boundaries — if data crosses service boundaries, check the cross-service contract rules.

#### 4d. Understand the broader flow

Read enough of the codebase to understand the end-to-end flow the changes touch. If a function is part of a request handler chain, read the full chain. If it's part of a background task pipeline, read the pipeline. The point is to verify:

- No existing code paths break due to the changes.
- Edge cases in the existing flow are still handled.
- The change is consistent with how the rest of the system works.

### Step 5: Review using code-review in review-only mode

Follow `~/.cursor/skills/code-review/SKILL.md` in **review-only mode**: run the full reviewer + verifier + curated summary flow, then halt before any walkthrough or fix prompt. mr-review takes those findings and posts them as MR comments rather than applying them.

Signal review-only mode explicitly when invoking code-review so the orchestrator picks the halt-after-summary path:

1. Pass the MR diff and changed file paths as the review target, plus the **Review Priorities** above (Pydantic, builtins/libraries, DRY, SOLID, code placement, naming) and the **product purpose summary** from Step 1 as invocation context.
2. If the user chose "continue but include alternative as a comment" in Step 3c, include the alternative approach as a general finding in the results.
3. The code-review skill returns the curated summary with findings under Show-stopper bugs / Architectural concerns / Smaller suggestions / Nits. Map these to severity labels for posting: Show-stopper bugs → `blocker`, Architectural concerns → `blocker` or `suggestion` per impact, Smaller suggestions → `suggestion`, Nits → `nit`. Questions to clarify intent → `question`. **Do not collect or post `praise` comments** — skip any "this looks good" or positive-only findings.
4. For each finding, classify it as either:
   - **Line-specific**: tied to a particular file and line number → will become an inline diff comment.
   - **General**: applies to the MR as a whole (e.g., architectural concern, alternative approach, missing test coverage across files, cross-cutting pattern issue) → will become a non-positional MR note.

### Step 6: Check for existing comments

Before presenting findings, compare them against existing MR discussions. **Do not** surface issues already covered by existing threads (from you or others). Search for `username: "<your-gitlab-username>"` to find your prior comments.

### Step 7: Write comments in your review voice

If you keep a review-voice rule (e.g. `~/.cursor/rules/mr-comment-voice/RULE.mdc`), load it and match that style. Draft every comment in that voice so they're ready to post if the user approves them. If you don't have one, write clear, direct review comments.

### Step 8: Present findings to the user for selection

**Do not post anything to GitLab yet.** Instead, present all findings to the user and let them decide which to post.

#### Presentation format

Display each finding with:
- A **number** for easy reference (e.g., `#1`, `#2`, ...)
- The **severity label** (`blocker`, `suggestion`, `nit`, `question`)
- The **file and line** (for line-specific findings) or `[general]` for MR-level findings
- A **brief summary** of the issue
- The **full draft comment text** (in your review voice, exactly as it would be posted)

Group findings by severity:

```
### Blockers
#1 [blocker] `src/service.py:42` — SQL injection in user query
> draft: "this is concatenating user input directly into the query string. needs to use parameterized queries"

### Suggestions
#2 [suggestion] `src/models.py:15` — Use Pydantic instead of TypedDict
> draft: "i think this should be a pydantic model instead of a TypedDict. we get validation for free and it matches what we do everywhere else"

#3 [suggestion] [general] — Could use feature flag instead of code change
> draft: "i feel like this whole behavior change could be driven by a feature flag instead of hardcoding it. that way we can toggle it per customer without a deploy"

### Nits
#4 [nit] `src/utils.py:88` — Rename `data` to `user_profile`
> draft: "data is pretty vague here, maybe user_profile?"

### Questions
#5 [question] `src/handler.py:23` — Why not use existing retry middleware?
> draft: "why not use the existing retry middleware for this? seems like it already handles this case"
```

#### Ask for selection

After presenting, ask the user which comments to post. Offer these options:

- **Post all** — post every finding as draft notes
- **Post by severity** — e.g., "post blockers and suggestions only"
- **Cherry-pick** — user lists specific numbers to post (e.g., "#1, #3, #5")
- **Edit and post** — user can modify a comment's text before posting
- **Post none** — skip posting entirely

Also ask the user for the **review state**:
- **Request changes** (default if any blockers or suggestions are selected for posting)
- **Comment** (default if only nits are selected for posting)
- **Approve**

**Do not proceed to Step 9 until the user responds.**

### Step 9: Post selected comments as draft notes

Post only the user-approved comments using `create_draft_note`.

- **Inline diff comments** (with position): Post one for each line-specific finding the user selected.
- **Overall MR note** (no position): Post **only** if the user selected any general (non-line-specific) findings. Do **not** post a summary-only note that just recaps inline comments.

If the user asked to edit a comment, use the edited text instead of the original draft.

For inline comments on **added lines**:

```json
{
  "position": {
    "base_sha": "<from diff_refs>",
    "head_sha": "<from diff_refs>",
    "start_sha": "<from diff_refs>",
    "position_type": "text",
    "new_path": "<file path>",
    "old_path": "<file path>",
    "new_line": <line number in new file>,
    "old_line": null
  }
}
```

For comments on **deleted or context lines**, use `old_line` instead and set `new_line` to null.

Calculate `new_line` from the diff hunk header: `@@ -old_start,old_count +new_start,new_count @@`. Count lines from `new_start`, incrementing for context lines (no prefix) and added lines (`+` prefix), skipping deleted lines (`-` prefix).

### Step 10: Bulk publish with review state

Use the review state the user chose in Step 8.

If the `bulk_publish_draft_notes` tool supports a `state` parameter, pass it directly:

- `"requested_changes"` for Request changes
- `"reviewed"` for Comment
- `"approved"` for Approve

If the tool does not support a `state` parameter, publish the drafts first with `bulk_publish_draft_notes`, then set the review state by calling the GitLab API directly:

```
PUT /projects/:project_id/merge_requests/:merge_request_iid/reviewers
```

with the appropriate reviewer state, or use `unapprove_merge_request` if the MR was previously approved.

### Step 11: Report to user

Summarize what was posted:

- How many comments posted out of how many total findings
- Brief description of each posted finding
- Review state submitted
- Link to the MR
- If no findings were posted, tell the user the MR looks clean
- **Reminder**: the submodule is now on the MR branch — mention which submodule and branch so the user can switch back if needed

## Iteration (optional)

If the user asks to re-review after the author pushes fixes:

1. Pull latest on the MR branch (`git pull origin <source_branch>`)
2. Re-fetch diffs (head_sha will have changed)
3. Check which previous findings are resolved
4. Review only new/changed code
5. Present new findings for user selection (same Step 8 flow)
6. Post selected comments, resolve fixed threads if possible

## Line Number Calculation

Given a diff hunk `@@ -old_start,old_count +new_start,new_count @@`:

```
new_line_counter = new_start

For each diff line:
  if line starts with ' ' (context): new_line_counter += 1
  if line starts with '+' (added):   new_line_counter += 1  ← use this value for new_line
  if line starts with '-' (deleted): skip (don't increment new_line_counter)
```

## Notes

- Always exclude lock files from diffs (`poetry.lock`, `package-lock.json`, `yarn.lock`)
- Check for your existing comments (`username: "<your-gitlab-username>"`) before posting to avoid duplicates
- When the MR is in a monorepo submodule, read existing code from the submodule path (which is on the MR branch after Step 2)
- If the MR touches API response shapes, check the cross-service contract rules
- After the review, the submodule will be on the MR branch — always remind the user in Step 11
