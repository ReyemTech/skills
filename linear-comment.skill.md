---
name: linear-comment
description: Add a comment to a Linear issue.
disable-model-invocation: true
allowed-tools: Task
---

# Linear Comment

When the user invokes `/linear-comment`, spawn the `linear-comment` agent via Task() with the user's arguments.

Use Task() to run the `linear-comment` agent with the following inputs from the user's invocation:
- `issue_id`: the issue identifier to comment on, e.g., "PROJ-109" (required)
- `body`: comment content in Markdown format (required)
- `parent_id`: parent comment ID for threaded replies (optional)

The agent resolves the issue first to confirm it exists, then appends the comment. Each invocation creates a new comment -- comments are additive by nature, no deduplication.

Wait for the agent to complete and return its output directly to the user.
