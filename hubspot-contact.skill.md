---
name: hubspot-contact
description: Create or update a HubSpot contact and company.
disable-model-invocation: true
allowed-tools: Task
---

# HubSpot Contact

When the user invokes `/hubspot-contact`, spawn the `hubspot-contact` agent via Task() with the user's arguments. Pass `mode: "manual"` so the agent shows a preview table before writing to HubSpot.

Use Task() to run the `hubspot-contact` agent with the following inputs from the user's invocation:
- `person_name`: the person's full name
- `org_name`: the organization name
- `email`: email address (optional -- pass if provided)
- `mode`: always `"manual"` for direct user invocation (agent will show preview + ask for confirmation before writing)

The agent handles company lookup automatically as a prerequisite step -- no need to invoke `/hubspot-company` separately. If the company doesn't exist in HubSpot, the agent creates it before creating or updating the contact.

Wait for the agent to complete and return its output directly to the user.
