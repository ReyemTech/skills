---
name: hubspot-note
description: Log an HTML note to HubSpot with org-type filtering.
disable-model-invocation: true
allowed-tools: Task
---

# HubSpot Note

When the user invokes `/hubspot-note`, spawn the `hubspot-note` agent via Task() with the user's arguments. Pass `mode: "manual"` so the agent shows a preview before writing to HubSpot.

Use Task() to run the `hubspot-note` agent with the following inputs from the user's invocation:
- `org_name`: the organization name or slug
- `note_content`: the note content (meeting summary, intelligence capture, or manual note)
- `mode`: always `"manual"` for direct user invocation (agent will show preview + ask for confirmation before writing)

The agent handles org type detection automatically (reads vault frontmatter), smart-matches company and contact, finds any active deal, and formats the note body as HTML. Client orgs are filtered by signal keywords.

Wait for the agent to complete and return its output directly to the user.
