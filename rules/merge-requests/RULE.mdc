---
description: "Merge request conventions: titles, descriptions, and when to ask for clarification."
alwaysApply: true
---

- **Title must include the ticket identifier** formatted as `TICKET-ID: Short description`.
  - If no ticket was provided, ask the user for it before creating the MR.
  - Never drop an existing ticket reference when updating a title.

```
# ✅ Good
TICKET-1234: Add retry logic for webhook delivery

# ❌ Bad — missing ticket
Add retry logic for webhook delivery
```

- **Description must have `## Problem`, `## Fix` (with `### Changes`), `## Impact`, and `## Test Plan` sections.**
  - `## Problem` describes the problem with concrete symptoms and real-world impact. Be specific — name the services, environments, and failure modes affected.
  - `## Fix` explains the solution approach at a high level, then lists specific changes under a `### Changes` sub-heading. Each bullet should have a **bold scope label** (file, module, or environment) followed by a description of the change in that scope.
  - `## Impact` states the consequences of the change — what improves, what trade-offs exist, and any residual risks.
  - `## Test Plan` lists concrete verification steps (lint, type checks, tests, manual verification, deploy checks).
  - If you don't have enough context to write any section, ask the user to explain before writing the description.

```markdown
# ✅ Good
## Problem

The `general` cluster NodePool in both dev and prod uses the default
`WhenUnderutilized` consolidation policy. This causes the autoscaler to evict
running pods (service-a, service-b, workers) whenever it considers a node
underutilized — even when those pods are the only replicas of a service.

In dev, this has been causing repeated e2e test failures: the autoscaler evicts
service-a/service-b pods mid-test, leading to service outages, dropped calls,
and test timeouts.

## Fix

Set `consolidationPolicy: "WhenEmpty"` on the `general` NodePool in both
environments. This matches the existing `on-demand` pool policy and ensures
the autoscaler only consolidates nodes that have zero running pods.

### Changes

- **dev** (`1-environments/dev/eks.yaml`): Add `consolidationPolicy: "WhenEmpty"`
  to general NodePool (keeps existing `consolidateAfter: "20m"`).
- **prod** (`1-environments/prod/eks.yaml`): Add `disruption` block with
  `consolidationPolicy: "WhenEmpty"` and `consolidateAfter: "20m"` to general
  NodePool (previously had no disruption config, defaulting to immediate
  `WhenUnderutilized`).

## Impact

- The autoscaler will still consolidate truly empty nodes after 20 minutes.
- Services with `minReplicas: 1` will no longer be disrupted by consolidation.
- Slightly higher cluster cost (nodes kept alive longer when underutilized),
  but prevents service disruptions.

## Test Plan

- [ ] Deploy modified dev manifest; verify `general` NodePool has
  `consolidationPolicy: "WhenEmpty"` and `consolidateAfter: "20m"` via
  `your-cluster-cli get nodepool general -o yaml`
- [ ] Run e2e test suite against dev; confirm no mid-test pod evictions for
  services with `minReplicas: 1`
- [ ] Check `your-cluster-cli describe node` and pod events — the autoscaler should not
  evict pods on underutilized nodes, only consolidate truly empty nodes
  after ~20 minutes
- [ ] Deploy modified prod manifest; repeat the `your-cluster-cli` verification

# ❌ Bad — vague problem, no changes list, no impact
## Problem

Pods keep getting evicted.

## Fix

Changed the consolidation policy to WhenEmpty in dev and prod.

## Impact

Should fix the issue.
```

- **Include a test plan** listing what was verified (lint, type checks, tests, manual verification).

## Self-Improvement

After writing an MR description, reflect on whether this rule should be updated:

- **Missing conventions**: Did the MR require structure or sections not covered here (e.g., migration notes, deployment order, rollback plan)?
- **Unclear guidance**: Was the Why/What distinction hard to apply for this type of change?
- **New patterns**: Did you write a description format that worked well and should be captured as an example?

If you identified improvements, ask the user:

> "I noticed [specific observation] while writing this MR description. Would you like me to update the merge-requests rule to cover this?"

Do NOT apply changes automatically — let the user review and decide.
