---
description: Standard steps that every implementation plan must include.
alwaysApply: true
---

Every implementation plan must include these steps in this order. Add them as both plan sections and frontmatter todos. Plans must also include an **Acceptance Criteria** section in the body (checklist of verifiable outcomes) and a **Verify acceptance criteria** end step (section + frontmatter todo) that the executor completes before declaring done.

### Beginning of every plan

- **Branch setup**: Create fresh feature branches from the repository's default branch in every repo that will be changed. Branch naming convention: `username/TICKET-ID-short-description` (e.g., `phillip/TICKET-1234-add-retry-logic`).
- **Acceptance Criteria**: Include an Acceptance Criteria section in the plan body with a short checklist of verifiable outcomes (`- [ ]` items). Each item must be checkable yes/no by the implementer against the finished work (observable behavior, file contents, or skill behavior) — not vague goals. Keep the list short (roughly ≤8 items when possible); if more are needed, the plan is probably too large. This is plan-level AC for the implementer, distinct from Linear ticket AC (they may overlap).

### Middle of every plan (implementation steps)

- **Commit organization**: If the work requires multiple commits, organize the implementation steps into logical commits. Each commit should be a coherent, reviewable unit with its own plan section and matching frontmatter todo(s).
- **Per-commit CI lint and test**: After completing the code changes for each commit, run the `ci-lint-test` skill for each changed repo included in that commit. Fix failures and re-run until clean.
- **Per-commit commit**: Actually commit the verified changes before moving on to the next commit's implementation steps.

### End of every plan (after all implementation steps)

- **Verify acceptance criteria**: Walk every Acceptance Criteria checkbox and confirm it is met against the actual changes. If any AC is unmet, fix the work (or update the plan with user approval) before proceeding to commit/MR/done. Report which AC were verified in the chat when finishing. Always include this section and frontmatter todo, even when Final CI / Pre-MR / Push are opted out — run it before declaring done.
- **Final CI lint and test**: Run the `ci-lint-test` skill for each changed repo as a final full verification. Fix failures and re-run until clean.
- **Pre-MR checklist**: Run the `pre-mr-checklist` skill on each changed repo. Fix all blockers.
- **Push and create MRs**: For single-commit plans, commit the verified changes before pushing. For multi-commit plans, use the commits created during implementation. Push and create GitLab MRs for each changed repo. Title must include the ticket ID (`TICKET-ID: Short description`). Assign self, add reviewer. MR descriptions must follow the merge-requests rule.
