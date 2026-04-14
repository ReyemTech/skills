---
name: draft-email
description: Draft professional HTML emails via Gmail or MS365 MCP. Use this skill whenever the user wants to compose, draft, write, reply to, or forward an email. Handles new emails, replies, and forwards with full thread context. Gathers contact and company intelligence from HubSpot, vault files, and Linear before composing. Routes emails through the appropriate MCP channel based on configurable rules. Always includes the correct HTML signature. Use this skill even when the user says something casual like "email John about the project" or "reply to that message" or "send a follow-up".
allowed-tools: AskUserQuestion, Read, Glob, Grep, Bash, Write, Edit, mcp__google-workspace__draft_gmail_message, mcp__google-workspace__send_gmail_message, mcp__google-workspace__search_gmail_messages, mcp__google-workspace__get_gmail_message_content, mcp__google-workspace__get_gmail_messages_content_batch, mcp__google-workspace__get_gmail_thread_content, mcp__ms365__create-draft-email, mcp__ms365__create-reply-draft, mcp__ms365__create-reply-all-draft, mcp__ms365__create-forward-draft, mcp__ms365__send-draft-message, mcp__ms365__send-mail, mcp__ms365__list-mail-messages, mcp__ms365__list-mail-folder-messages, mcp__ms365__get-mail-message, mcp__ms365__add-mail-attachment, mcp__claude_ai_HubSpot__search_crm_objects, mcp__claude_ai_HubSpot__get_crm_objects, mcp__claude_ai_Linear__list_issues, mcp__claude_ai_Linear__get_issue
---

# Draft Email Skill

Draft professional, context-rich HTML emails through the appropriate MCP channel.

## Configuration

Before using this skill, configure the following:

| Setting | Description |
|---------|-------------|
| `{{user_email}}` | Your primary email address (used for Gmail MCP) |
| `{{secondary_email}}` | Your secondary/org email address (used for MS365 MCP, if applicable) |
| `{{booking_link}}` | Your scheduling/booking link URL |
| **Email signatures** | Store HTML signatures in a config file accessible to the skill (e.g., `_resources/docs/integrations.md` or a dedicated `signatures.yaml`). The skill reads signatures at runtime -- do not hardcode them. |
| **Routing rules** | Define which recipients route through Gmail vs MS365 (see Step 7 below). |

## Core Principles

- **ONLY interact with the user via AskUserQuestion** -- never output free-form questions
- **Preview before acting** -- always show a preview and let the user choose to save as draft or send
- **Context is king** -- gather as much intel about the recipient and their org before writing
- **Personal and connected** -- every email should feel like it comes from someone who knows the recipient and their situation
- **Truth over convenience** -- if gathered context contradicts the user's premise (e.g. "report is ready" but Linear shows it's in Backlog), flag the discrepancy to the user before composing

## Workflow

### Step 1: Determine Email Type and Recipient

Use AskUserQuestion if the user's intent is unclear. Determine:

1. **Type**: New email, reply, reply-all, or forward?
2. **Recipient**: Who is this going to? (name, email, or reference to a recent message)
3. **Purpose**: What's the goal of this email?

If the user says "reply to [person/subject]", search for the email first, then ask for confirmation if multiple matches exist.

**Resolving temporal references:** When the user says "last week", "recently", "the other day", etc., resolve to a concrete date range based on today's date before searching. If multiple matches fall within that range, surface all candidates to the user via AskUserQuestion.

**Recipient discovery:** Search vault people files first, then HubSpot. If the recipient isn't found directly but appears in CC lists, calendar invites, or other side-channels, always confirm with the user via AskUserQuestion before proceeding -- never silently use a side-channel discovery.

### Step 2: Gather Thread Context (Replies Only)

For replies and forwards, retrieve the full conversation thread before composing:

**Gmail threads:**
1. Search with `mcp__google-workspace__search_gmail_messages` to find the message
2. Use `mcp__google-workspace__get_gmail_thread_content` with the thread_id to get the full thread
3. Note the `thread_id`, `in_reply_to` (Message-ID header), and `references` chain for proper threading

**MS365 threads:**
1. Search with `mcp__ms365__list-mail-messages` or `mcp__ms365__list-mail-folder-messages` to find the message
2. Use `mcp__ms365__get-mail-message` to read the full message and get `conversationId`
3. For replies, use `mcp__ms365__create-reply-draft` with the original `messageId`

**Disambiguating "last email":** When searching for the "last email from [person]", distinguish between:
- The most recent **inbound** message from them (what they actually sent)
- The most recent **thread** involving them (which may be an unanswered outbound from you)

If these differ significantly (e.g. last inbound is months old but there's a recent unanswered thread), surface both options to the user via AskUserQuestion and let them pick which one to reply to.

**Already-replied detection:** After retrieving the thread, check if your most recent message in the thread is the last one (i.e. you already replied and are awaiting a response). If so, flag this to the user: "You already replied to this thread on [date]. Do you want to send another follow-up, or did you mean a different thread?"

**Replying to your own thread:** If all recent messages in the thread are from you (no reply from the other party), note this in the preview -- the user may want to consolidate into a single follow-up rather than stacking unanswered messages.

### Step 3: Gather Contact & Company Intelligence

Before composing, build a mental picture of the recipient. Check these sources in parallel where possible:

1. **Vault people files**: Search `references/people/` for the recipient's profile -- role, preferences, past interactions
2. **Vault org files**: Search `organizations/` for the recipient's company -- status, projects, relationship health, and `internal-notes.md` for sensitive context
3. **HubSpot**: Search contacts and companies for CRM context -- deals, recent activity, engagement history
4. **Linear**: Check for active issues related to the recipient's org -- ongoing projects, blockers, recent updates

**Create missing vault profiles:** After searching all sources, if the recipient still has no vault people file (`references/people/First Last.md`), create one using information gathered from HubSpot, email headers, and the user's context. Populate what you know and leave the rest blank:

```yaml
---
categories:
  - "[[People]]"
organizations:
  - "[[OrgName]]"
name: Full Name
role: Role if known
company: Company Name if known
company-role: Role if known
relationship-type: Lead|Colleague|Partner Contact
relationship-strength: New
first-met: YYYY-MM-DD  # today's date
last-contact: YYYY-MM-DD  # today's date
status: Active
email: their@email.com
tags:
  - tag based on context
last-updated: YYYY-MM-DD
---

# Full Name -- Role

**Role:** Role if known
**Company:** Company Name if known

---

## Quick Profile

- Context from email/HubSpot (one or two bullets)

---

## Key Interactions

<!-- Will be populated as interactions occur -->
```

Similarly, if the recipient's **company has no vault org directory** (`organizations/[org-name]/`), create the org using the `_template/` structure:
1. Copy `organizations/_template/` to `organizations/[org-name]/`
2. Fill in the README with what you know (company name, industry, relationship type)
3. Set `organization-type` based on context (`"[[Lead]]"` for prospects, `"[[Client]]"` for active clients, `"[[Partnership]]"` for partners)
4. Leave `status.md` and `internal-notes.md` as templates

Do NOT create org directories for well-known external companies (Google, AWS, LinkedIn, etc.) that are not actual clients/leads/partners. Only create orgs for companies you have a direct business relationship with.

Notify the user in the preview step that new vault files were created (e.g. "Created profile for John Smith and org for Acme Corp").

**Email address conflicts:** If HubSpot and Gmail/MS365 show different email addresses for the same person, prefer the address from the most recent actual email exchange. Flag the discrepancy in the preview so the user can correct it.

Use this intelligence to:
- Reference recent meetings or conversations naturally
- Acknowledge ongoing projects or milestones
- Match the appropriate tone (formal for new contacts, warm for established relationships)
- Avoid topics that are sensitive or contentious (check internal-notes.md)
- Handle out-of-email conversations (WhatsApp, calls) with care -- reference the topic obliquely without quoting private channel content directly

### Step 4: Validate & Clarify

**Contradiction check:** Before composing, verify the user's stated premise against gathered context. Common contradictions to catch:
- "Report is ready" but Linear shows the task is in Backlog/Todo
- "Follow up on our meeting" but no meeting found in the date range
- "Reply to their email" but the last message was from you, not them

If a contradiction exists, flag it to the user via AskUserQuestion with the evidence found, and ask how to proceed.

**Content clarification:** If the user gave a vague instruction like "email John about the thing", use AskUserQuestion to clarify:
- What specific topic or request?
- Any key points to include?
- Desired tone (formal, friendly, urgent)?
- Any attachments to include?

Skip clarification if the user already provided clear intent and content direction.

### Step 5: Compose the Email

Write the email body as HTML, following these guidelines:

**Tone & Style:**
- Professional but human, not corporate boilerplate
- Reference shared context naturally (don't force it)
- Match the formality level of the existing thread (for replies)
- Be concise, respect the recipient's time
- Use the recipient's first name unless formality demands otherwise
- When the recipient's role is unknown, default to a warm-professional tone
- **No AI-sounding patterns.** Write like a real person, not a language model. Avoid:
  - Em dashes and en dashes except when separating a list item from its description
  - Filler openers like "Hope this finds you well", "I wanted to reach out", "Just circling back"
  - Overly hedged language ("I was wondering if perhaps...", "Would it be possible to maybe...")
  - Stiff transitions ("Additionally", "Furthermore", "In light of")
  - Exclamation marks on every sentence
  - Generic compliments ("Great work!", "Exciting times!")
  - The word "delighted"
- Prefer commas, periods, and natural sentence breaks over em dashes
- Match your actual writing style from prior emails in the thread when available
- **NEVER sign the email body twice.** The signature block already contains a closing and your name. Do NOT add a separate sign-off (e.g. "Talk soon, Name", "Cheers, Name") in the email body. End the body content naturally and let the signature block handle the closing.

**HTML Structure:**
Always wrap the email in this template:
```html
<html><body>
<div style="font-family:Aptos,Arial,Helvetica,sans-serif; font-size:12pt; color:#000;">
  [EMAIL BODY CONTENT HERE]
</div>
[SIGNATURE BLOCK - see routing rules below]
</body></html>
```

Use semantic HTML for formatting -- `<p>` for paragraphs, `<ul>`/`<ol>` for lists, `<b>` for emphasis. Do not use markdown.

**Booking link:** Whenever the email involves scheduling a meeting, call, or chat (e.g. "let's find a time", "happy to connect", "open to a call"), include your booking link (`{{booking_link}}`). Weave it naturally into the message (e.g. "Here's my booking link if you'd like to grab a time: [link]") rather than asking the recipient to propose times.

### Step 6: Preview the Email

**IMPORTANT: Do NOT create the draft or send the email yet.** This step is preview-only. The MCP draft/send calls happen in Step 7 after the user approves.

**HTML preview first:** Write the full HTML email (including signature) to `/tmp/email-preview.html` and open it in the browser:
```bash
open /tmp/email-preview.html
```
If the `open` command fails (headless environment), skip the browser preview.

**Then use AskUserQuestion** to show a text summary alongside the browser preview:
- **To / CC / BCC**: recipient(s)
- **Subject**: the subject line
- **From**: which account (Gmail or MS365)
- **Body preview**: the plain-text version of the email body (strip HTML tags for readability)
- **Thread note** (if applicable): "Replying in thread -- last message was from [you/them] on [date]"

Ask the user what to do next:

| Option | Description |
|--------|-------------|
| **Send now** | Send the email immediately |
| **Save as draft** | Save to drafts for later review/editing |
| **Edit** | User wants to change something -- ask what to modify, then re-preview |
| **Discard** | Cancel without saving |

If the user chooses **Edit**, make the requested changes to the HTML in `/tmp/email-preview.html`, re-open the browser preview, and show the updated text summary via AskUserQuestion again. Repeat until the user approves.

**For attachments:** If the user mentioned files to attach, confirm the file paths before proceeding. Use the Glob tool to find files if the user gave a partial name.

### Step 7: Route and Execute

Only after the user approves in Step 6, create the draft or send via MCP.

**Routing Rules:**

Configure routing based on your setup. Example routing table:

| Recipient Context | MCP Channel | From Address |
|-------------------|-------------|--------------|
| Org-specific people (e.g., internal team, clients of that org) | MS365 | `{{secondary_email}}` |
| Everyone else (default) | Gmail | `{{user_email}}` |

Determine routing by checking:
1. The recipient's email domain
2. The recipient's org in vault files
3. Whether the original thread is in Gmail or MS365 (for replies, use the same channel)

**Gmail -- Save as Draft:**
Use `mcp__google-workspace__draft_gmail_message` with:
- `user_google_email`: `{{user_email}}`
- `body_format`: "html"
- `body`: Full HTML including signature
- `include_signature`: false (skill appends signature manually for consistency)
- For replies: include `thread_id`, `in_reply_to`, and `references`
- For attachments: use the `attachments` parameter with file `path`

**Gmail -- Send Now:**
Use `mcp__google-workspace__send_gmail_message` with the same parameters as draft but via the send tool.

Append the **primary signature** (from your signature config file) at the end of the body, inside the `</body>` tag but after the content div.

**MS365 -- Save as Draft:**
- For new emails: use `mcp__ms365__create-draft-email`
- For replies: use `mcp__ms365__create-reply-draft` with the original `messageId`
- For reply-all: use `mcp__ms365__create-reply-all-draft`
- For forwards: use `mcp__ms365__create-forward-draft`
- Set body `contentType` to "html"

**MS365 -- Send Now:**
- For new emails: use `mcp__ms365__send-mail`
- For drafts that were already created (replies/forwards): first create the draft, then send it with `mcp__ms365__send-draft-message` using the draft's `messageId`
- Set body `contentType` to "html"

Append the **secondary signature** (from your signature config file) at the end of the body.

**Threading parameters (Gmail replies):**
- `thread_id`: The Gmail thread ID (e.g., `19ca0dc4656a9b34`) -- this groups messages visually
- `in_reply_to`: The RFC 2822 `Message-ID` header of the specific message being replied to (e.g., `<CAExample123@mail.gmail.com>`) -- NOT the Gmail message ID
- `references`: Space-separated chain of all `Message-ID` headers in the thread, oldest first

### Step 8: Confirm Completion

After sending or saving, confirm to the user via AskUserQuestion:
- If sent: "Email sent to [recipient] -- subject: [subject]"
- If drafted: "Draft saved -- you can find it in your [Gmail/Outlook] drafts"
- Ask if they need anything else (reply to another message, etc.)

## Signature Reference

Store your full HTML signatures in a configuration file (e.g., `_resources/docs/integrations.md` or a dedicated config file). The skill reads this file at runtime -- do not hardcode signatures, as they may be updated.

## Edge Cases

- **Unknown recipient**: If no vault/HubSpot match, create a minimal vault people file (and org if applicable) from whatever context is available (email headers, user input, email signature). Proceed with drafting and note in the preview that new vault files were created.
- **Missing email address**: If the recipient is identified by name but no email address can be determined from any source, ask the user directly via AskUserQuestion. Do not guess.
- **Ambiguous routing**: If unclear whether Gmail or MS365, ask the user via AskUserQuestion.
- **Multiple recipients**: Support CC and BCC fields. Route based on the primary recipient (TO field).
- **Forwarding with context**: When forwarding, preserve the original message and add a forwarding note above it.
- **Replying to unanswered thread**: If all recent messages are from you with no reply, suggest consolidating into a single follow-up and note the last send date.
- **Multiple deliverables match**: When the email subject (e.g. "Q1 report") could refer to multiple deliverables in the org context, ask the user to disambiguate.
- **Sensitive out-of-email context**: When internal-notes.md or people files reference WhatsApp/call conversations, reference the topic naturally but never quote or directly reference the private channel (e.g. say "following up on our recent discussion about X" not "per our WhatsApp on March 6").
