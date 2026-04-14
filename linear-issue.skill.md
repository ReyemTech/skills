---
name: linear-issue
description: Create, update, or close a Linear issue.
disable-model-invocation: true
allowed-tools: Task
---

# Linear Issue

When the user invokes `/linear-issue`, spawn the `linear-issue` agent via Task() with the user's arguments. Pass `mode: "manual"` so the agent shows a preview before writing to Linear.

Use Task() to run the `linear-issue` agent with the following inputs from the user's invocation:
- `title`: the issue title (required)
- `org_name`: organization name for project routing (optional)
- `project`: direct Linear project name override (optional -- skips org routing)
- `action`: "create", "update", or "close" (optional -- defaults to smart match)
- `description`: issue description in Markdown (optional)
- `priority`: 0=None, 1=Urgent, 2=High, 3=Medium, 4=Low (optional)
- `labels`: array of label names, e.g., ["Strategic", "Delegated"] (optional)
- `assignee`: user name, email, or "me" (optional)
- `state`: explicit state name for transitions, e.g., "In Progress" (optional)
- `dueDate`: due date in YYYY-MM-DD format (optional)
- `issue_id`: existing issue identifier like "PROJ-109" for direct update/close (optional)
- `mode`: always `"manual"` for direct user invocation (agent shows preview + asks for confirmation before writing)

The agent handles smart match automatically -- it searches for an existing issue before creating to prevent duplicates. It also handles project routing from org_name and close via runtime state lookup (no hard-coded state names).

Wait for the agent to complete and return its output directly to the user.
