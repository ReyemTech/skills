---
name: linear-label
description: Add or remove labels on a Linear issue.
disable-model-invocation: true
allowed-tools: Task
---

# Linear Label

When the user invokes `/linear-label`, spawn the `linear-label` agent via Task() with the user's arguments.

Use Task() to run the `linear-label` agent with the following inputs from the user's invocation:
- `issue_id`: the Linear issue ID or identifier (e.g., "PROJ-109")
- `action`: "add" or "remove"
- `label`: the label name to add or remove (e.g., "Delegated", "Technical", "Waiting")
- `team`: team name (optional -- defaults to your primary team in the agent)

The agent validates that the label exists for the team, reads current labels before updating (to preserve existing labels on add), and computes the correct full label set before calling update_issue.

Wait for the agent to complete and return its output directly to the user.
