---
description: Rename the chat when starting an MR review.
alwaysApply: true
---

# MR Review Chat Title

When running the `mr-review` skill (`/mr-review`), rename the current chat as soon as the MR IID is known:

- Use the `rename_chat` tool from the `cursor-app-control` MCP server.
- Title format: `<repo> MR <iid> Review - <author>` — e.g., `api MR 42 Review - Jane`.
- `<repo>` is the repository/project name extracted from the GitLab URL or user input.
- `<iid>` is the merge request IID extracted from the GitLab URL or user input.
- `<author>` is the first name of the MR creator, from the MR details.
- Do this immediately after identifying the MR, before fetching details or diffs. If the author's name isn't known yet, rename once with the repo and IID, then rename again with the author once the MR details are fetched.
