---
name: hubspot-deal
description: Suggest a HubSpot deal from context signals (always confirms before creating).
disable-model-invocation: true
allowed-tools: Task
---

# HubSpot Deal

When the user invokes `/hubspot-deal`, spawn the `hubspot-deal` agent via Task() with the user's arguments. This skill ALWAYS requires user confirmation -- deals are never auto-created under any circumstances.

Use Task() to run the `hubspot-deal` agent with the following inputs from the user's invocation:
- `org_name`: the organization name or slug
- `context_content`: meeting notes, email, or manual description to scan for deal signals
- `person_name`: name of the primary contact to associate with the deal (optional -- pass if provided)

The agent will scan for deal-worthy signals, fetch the account's pipeline configuration at runtime, show a preview table, and wait for explicit confirmation before creating anything. If no signals are detected, it returns immediately without creating a deal. Wait for the agent to complete and return its output directly to the user.
