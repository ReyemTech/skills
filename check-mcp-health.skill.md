---
name: check-mcp-health
description: Verify all MCP servers are connected; auto-open auth URLs on failure without blocking session.
disable-model-invocation: true
allowed-tools: mcp__google-workspace__search_drive_files, mcp__ms365__get-calendar-view, mcp__claude_ai_Linear__list_issues, mcp__google-calendar__list-events, mcp__claude_ai_HubSpot__get_user_details, Bash(*)
---

# Check MCP Health

Verify connectivity of all MCP servers and report status. Auto-opens auth URLs in the browser when a server returns an auth error.

## Configuration

Configure which MCP servers to test in the server reference section below. Add or remove servers as needed for your setup.

## Instructions

When the user invokes `/check-mcp-health`, execute the following steps:

### Step 1: Run all server tests in parallel

Reference the Server Test Calls table below for the exact test call and parameters for each server. Call all simultaneously -- do NOT wait for one before starting the next.

### Step 2: Handle failures

For each server that fails, classify the error:

- **Auth error with URL**: Run `open "URL"` via Bash immediately. Tell user: "Opened [server] auth in browser -- complete sign-in then confirm."
- **Schema validation error** (error text contains `oneOf`, `allOf`, or `anyOf`): Log as "Schema issue -- MCP registration problem, not auth." Suggest adding `"skipSchemaValidation": true` to that server's entry in `.mcp.json`. Continue without blocking.
- **All servers fail simultaneously with the same error**: Flag as "Likely schema validation issue affecting all MCP -- not an auth problem."
- **Down/unavailable**: Log as "Unavailable" and continue.

### Step 3: Return status table

Present the results as a markdown table:

```
| System           | Status                                              |
|------------------|-----------------------------------------------------|
| Google Workspace | Connected / Auth Required / Schema Issue / Unavailable |
| MS365            | Connected / Auth Required / Schema Issue / Unavailable |
| Linear           | Connected / Auth Required / Schema Issue / Unavailable |
| Google Calendar  | Connected / Auth Required / Schema Issue / Unavailable |
| HubSpot          | Connected / Auth Required / Schema Issue / Unavailable |
```

Use "Connected" when the test call succeeds. Use the specific failure type when it fails.

---

## Server Test Calls (Inlined Reference)

| Server | Test Call | Parameters | Auth Fix |
|--------|-----------|------------|----------|
| Google Workspace | `mcp__google-workspace__search_drive_files` | `query: "test"` | OAuth URL in error -> `open "URL"` |
| MS365 | `mcp__ms365__get-calendar-view` | `startDateTime: "<today>T00:00:00Z"`, `endDateTime: "<today>T23:59:59Z"` | OAuth URL in error -> `open "URL"` |
| Linear | `mcp__claude_ai_Linear__list_issues` | `assignee: "me"`, `limit: 1` | OAuth URL in error -> `open "URL"` |
| Google Calendar | `mcp__google-calendar__list-events` | `calendarId: "primary"`, `timeMin: "<today>T00:00:00Z"`, `timeMax: "<today>T23:59:59Z"` | OAuth URL in error -> `open "URL"` |
| HubSpot | `mcp__claude_ai_HubSpot__get_user_details` | (no params) | OAuth URL in error -> `open "URL"` |

Replace `<today>` with the current date in `YYYY-MM-DD` format at runtime.

## Troubleshooting

### Schema Validation Fix

If a server fails with an error containing `oneOf`, `allOf`, or `anyOf`, it is a schema validation issue -- not an auth problem.

**Fix:** Add `"skipSchemaValidation": true` to the affected server's entry in `.mcp.json`:

```json
{
  "mcpServers": {
    "server-name": {
      "url": "...",
      "skipSchemaValidation": true
    }
  }
}
```

### All-Fail Pattern

If all servers fail with the same error, it is likely one server's schema blocking all MCP registration. Check `.mcp.json` for recently added or modified server entries.

### Checking Registered Tools

Run `/mcp` in your Claude Code session to see which MCP servers are registered and their status.
