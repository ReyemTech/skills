---
name: hubspot-email
description: Log an email engagement to HubSpot for actionable emails.
disable-model-invocation: true
allowed-tools: Task
---

# HubSpot Email

When the user invokes `/hubspot-email`, spawn the `hubspot-email` agent via Task() with the user's arguments. Pass `mode: "manual"` so the agent shows a preview before writing to HubSpot.

Use Task() to run the `hubspot-email` agent with the following inputs from the user's invocation:
- `org_name`: the organization name or slug
- `email_subject`: the email subject line
- `email_body`: the email body or summary
- `email_classification`: must be "Needs Reply" or "Time-Sensitive" -- agent exits early for any other value
- `email_direction`: "INCOMING" or "OUTGOING" (optional -- defaults to "INCOMING")
- `sender`: sender name or email (optional)
- `recipient`: recipient name or email (optional)
- `mode`: always `"manual"` for direct user invocation (agent will show preview + ask for confirmation before writing)

The agent gates on email classification first, then applies the org type gate, smart-matches company and contact, finds any active deal, and creates the email engagement. Only "Needs Reply" and "Time-Sensitive" emails are logged.

Wait for the agent to complete and return its output directly to the user.
