---
description: Choose GitHub vs GitLab MCP based on the git remote host.
alwaysApply: true
---

# Choosing GitHub vs GitLab MCP

Before using the GitHub or GitLab MCP for PRs, MRs, issues, reviews, pipelines, notes, or repo metadata, run `git remote -v` to identify the host, then route accordingly:

- **GitHub remote** (e.g. `github.com`) → use the GitHub MCP. Use `gh` CLI as the fallback.
- **GitLab remote** (e.g. `gitlab.com` or any self-hosted GitLab host) → use the **GitLab MCP** (`user-GitLab`) for all GitLab actions, including resolving discussion threads.

## GitLab MCP is mandatory for GitLab work

For MRs, issues, notes, pipelines, search, labels, and work items:

- Always use the GitLab MCP (`user-GitLab`).
- Do **not** use `glab`, `curl`, or the GitLab REST/GraphQL API via shell.
- Do **not** prefer `plugin-gitlab-gitlab` over `user-GitLab`. If both are available, use `user-GitLab` (fuller tool set, including thread resolve).
- If the GitLab MCP is missing, unauthenticated, or cannot perform the requested action, stop and tell the user. Do not improvise with another tool.

If the remote is missing, ambiguous, or the repo is mirrored to both, ask which platform to target instead of guessing.
