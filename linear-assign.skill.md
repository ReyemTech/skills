---
name: linear-assign
description: Assign or reassign a Linear issue.
disable-model-invocation: true
allowed-tools: Task
---

# Linear Assign

When the user invokes `/linear-assign`, spawn the `linear-assign` agent via Task() with the user's arguments.

Use Task() to run the `linear-assign` agent with the following inputs from the user's invocation:
- `issue_id`: the Linear issue ID or identifier (e.g., "PROJ-109")
- `assignee`: user name, email, "me" to assign to yourself, or "none"/"unassign" to clear the assignment

The agent verifies the issue exists, resolves the assignee (handling "me", names, emails, and null for unassign), and falls back to list_users lookup if the direct name/email assignment returns a user-not-found error.

Wait for the agent to complete and return its output directly to the user.
