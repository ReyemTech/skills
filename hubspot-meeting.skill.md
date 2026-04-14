---
name: hubspot-meeting
description: Log a meeting to HubSpot with dedup and org-type filtering.
disable-model-invocation: true
allowed-tools: Task
---

# HubSpot Meeting

When the user invokes `/hubspot-meeting`, spawn the `hubspot-meeting` agent via Task() with the user's arguments. Pass `mode: "manual"` so the agent shows a preview table before writing to HubSpot.

Use Task() to run the `hubspot-meeting` agent with the following inputs from the user's invocation:
- `org_name`: the organization name or slug
- `meeting_title`: the meeting title
- `meeting_date`: the date and start time of the meeting
- `meeting_duration`: duration in minutes (or pass `meeting_end_time` if that was provided instead)
- `meeting_content`: meeting notes, summary, decisions, and action items from the vault note or calendar event
- `calendar_url`: calendar event URL for deduplication (optional -- pass if provided)
- `attendees`: list of attendee names to look up as HubSpot contacts (optional -- pass if provided)
- `mode`: always `"manual"` for direct user invocation (agent will show preview + ask for confirmation before writing)

The agent handles deduplication automatically -- if a meeting with the same calendar URL (or same title/date) already exists in HubSpot, it will update rather than create. Wait for the agent to complete and return its output directly to the user.
