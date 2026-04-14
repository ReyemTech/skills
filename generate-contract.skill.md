---
name: generate-contract
description: Generate a contract from templates. Creates Google Docs from contract templates with auto-filled client details and deal terms. Use when the user wants to create a new contract, agreement, NDA, SOW, or amendment.
allowed-tools: AskUserQuestion, Read, Glob, Grep, Write, Edit, Bash, mcp__google-workspace__search_drive_files, mcp__google-workspace__list_drive_items, mcp__google-workspace__copy_drive_file, mcp__google-workspace__get_doc_content, mcp__google-workspace__get_doc_as_markdown, mcp__google-workspace__batch_update_doc, mcp__google-workspace__find_and_replace_doc, mcp__google-workspace__update_drive_file, mcp__google-workspace__create_drive_folder, mcp__google-workspace__export_doc_to_pdf, mcp__claude_ai_HubSpot__search_crm_objects
---

# Generate Contract

Automates contract assembly from Google Docs templates. Copies a template, fills `{{VARIABLE}}` placeholders with client/deal data, and places the output in the correct Drive folder.

## Configuration

Before using this skill, you need:

1. **Contract templates** as Google Docs with `{{VARIABLE}}` placeholders
2. **A contracts config file** (e.g., `_resources/config/contracts.yaml`) containing:
   - Google Drive folder IDs for clients and subcontractors
   - Template document IDs for each contract type
   - Default deal terms
   - Your company details (name, address, signer name/title)
3. **Google Workspace MCP** connected with `{{user_email}}`

---

## Phase A -- Contract Type Selection

Present this numbered list and ask the user to select:

```
Contract Types:
  1a. MNDA (Client) -- Mutual NDA for client engagements
  1b. NDA (Subcontractor) -- One-way NDA for subcontractors
  2.  Fractional CTO/CIO Agreement -- Monthly retainer
  3.  CTO-Led Software Development Agreement -- Monthly retainer
  4.  Advisory Agreement -- Hourly (ad-hoc)
  5.  DMAP Assessment Agreement -- Fixed fee + milestones
  6.  Subcontractor Agreement -- Per Schedule A
  7.  SOW Addendum -- Project scope under existing agreement
  8.  Contract Amendment -- Modify terms of existing agreement
```

Use `AskUserQuestion` to collect the selection.

---

## Phase B -- Entity Lookup

### For client contracts (1a, 2, 3, 4, 5, 7, 8):

1. Ask for the client/organization name via `AskUserQuestion`
2. Search vault `organizations/*/` for a matching org folder using `Glob`
3. Read the org's main markdown file for legal name and address
4. Search HubSpot for the company: `mcp__claude_ai_HubSpot__search_crm_objects` with `object_type: "companies"` and the org name as filter
5. Search vault `references/people/*.md` for a signer -- look for someone with a senior role (CEO, President, Founder, Owner) associated with this org
6. Present the found details to the user via `AskUserQuestion` and ask them to confirm or edit:
   - Client legal name
   - Client address
   - Signer name
   - Signer title

### For subcontractor contracts (1b, 6):

1. Ask for the contractor's name via `AskUserQuestion`
2. Search vault `references/people/*.md` for the person using `Glob` and `Read`
3. Present found details (name, address) and ask user to confirm or edit via `AskUserQuestion`

---

## Phase C -- Deal Terms

1. Read your contracts config file for defaults
2. Based on the selected template type, present the relevant deal-specific variables with defaults pre-filled
3. Use `AskUserQuestion` to collect overrides (user can accept defaults by pressing enter)

### Variables per template type:

| Type | Variables to collect |
|------|---------------------|
| 1a | `EFFECTIVE_DATE` only (no deal terms) |
| 1b | `EFFECTIVE_DATE`, `NON_SOLICIT_YEARS` (default: 2) |
| 2 | `EFFECTIVE_DATE`, `CURRENCY` (CAD), `MONTHLY_RATE`, `HOURS_PER_WEEK`, `RETAINER_DEPOSIT`, `MIN_TERM_MONTHS` (3), `NOTICE_PERIOD_DAYS` (60), `ANNUAL_INCREASE_PCT` (5) |
| 3 | `EFFECTIVE_DATE`, `CURRENCY` (CAD), `MONTHLY_RATE`, `RETAINER_DEPOSIT`, `MIN_TERM_MONTHS` (3), `NOTICE_PERIOD_DAYS` (60), `ANNUAL_INCREASE_PCT` (5) |
| 4 | `EFFECTIVE_DATE`, `CURRENCY` (CAD), `HOURLY_RATE` (275), `MIN_TERM_MONTHS` (3), `NOTICE_PERIOD_DAYS` (60), `ANNUAL_INCREASE_PCT` (5) |
| 5 | `EFFECTIVE_DATE`, `CURRENCY` (CAD), `FLAT_FEE` (30000). Auto-calculate: `DMAP_DEPOSIT_AMOUNT` = 50% of fee, `DMAP_MONTHLY_GROSS` = (fee - deposit) / 3, `DMAP_MONTHLY_NET` = gross - (deposit / 3) |
| 6 | `EFFECTIVE_DATE`, `CURRENCY` (CAD), `HOURLY_RATE` (275), `MIN_TERM_MONTHS` (6), `NOTICE_PERIOD_DAYS` (14), `NON_SOLICIT_YEARS` (2), `CONTRACTOR_EXPERTISE` |
| 7 | `EFFECTIVE_DATE`, `MASTER_AGREEMENT_TYPE`, `MASTER_AGREEMENT_DATE`, `SOW_NUMBER`, `PROJECT_NAME`, `SOW_SCOPE`, `SOW_DELIVERABLES`, `SOW_TIMELINE`, `SOW_FEES` |
| 8 | `EFFECTIVE_DATE`, `ORIGINAL_AGREEMENT_TYPE`, `ORIGINAL_AGREEMENT_DATE`, `AMENDMENT_DESCRIPTION`, `AMENDMENT_CLAUSES` |

**Currency formatting:** When `CURRENCY` is "CAD", use "CAD (Canadian Dollars)" in the doc. When "USD", use "USD (United States Dollars)".

**DMAP auto-calculation example** (fee = $30,000, deposit = 50%):
- `DMAP_DEPOSIT_AMOUNT` = $15,000
- `DMAP_MONTHLY_GROSS` = $10,000
- `DMAP_MONTHLY_NET` = $5,000

---

## Phase D -- Document Number

1. Read your contracts config for Drive folder IDs
2. Get the current year (YYYY)
3. Use `mcp__google-workspace__search_drive_files` to find existing contracts for the current year
4. Parse the highest NNN number found
5. Increment by 1 and format as `YYYY/NNN` (zero-padded to 3 digits)
6. If no contracts exist for the current year, start at `YYYY/001`

---

## Phase E -- Preview & Confirm

Display a summary of ALL values that will be used, organized clearly:

```
Contract Generation Preview
============================
Type: [Contract Type Name]
Document #: [YYYY/NNN]
Effective Date: [date]

Entity:
  Name: [client/contractor name]
  Address: [address]
  Signer: [name], [title]

Deal Terms:
  [list all deal-specific values]

Your Company (from config):
  Name: [Your Company Name]
  Signer: [Your Name], [Your Title]
```

Ask user to confirm via `AskUserQuestion`. If they want changes, loop back to the relevant phase.

---

## Phase F -- Generate

1. Read template doc ID from your contracts config for the selected type
2. Determine target folder:
   - **Client contracts (1a, 2-5, 7, 8):** Find or create `{CLIENT_NAME}/Contracts/` on the Clients shared drive
   - **Subcontractor contracts (1b, 6):** Find or create `{CONTRACTOR_NAME}/` inside the Contractors folder
3. Copy template to target folder:
   ```
   mcp__google-workspace__copy_drive_file
     file_id: [template doc ID]
     new_name: "[YYYY/NNN] {Contract Type} - {Entity Name}"
     parent_folder_id: [target folder ID]
   ```
4. Build variable map -- all `{{VAR}}` to value pairs including:
   - Universal: `EFFECTIVE_DATE`, `DOCUMENT_NUMBER`, `COMPANY_NAME`, `COMPANY_ADDRESS`, `COMPANY_SIGNER_NAME`, `COMPANY_SIGNER_TITLE`
   - Entity: `CLIENT_NAME`/`CLIENT_ADDRESS`/`CLIENT_SIGNER_NAME`/`CLIENT_SIGNER_TITLE` or `CONTRACTOR_PERSON_NAME`/`CONTRACTOR_ADDRESS`
   - Deal-specific: all collected in Phase C
5. Use `mcp__google-workspace__batch_update_doc` with `find_replace` operations:
   ```json
   [
     {"type": "find_replace", "find_text": "{{EFFECTIVE_DATE}}", "replace_text": "March 20, 2026", "match_case": true},
     {"type": "find_replace", "find_text": "{{CLIENT_NAME}}", "replace_text": "Acme Corp", "match_case": true}
   ]
   ```
6. If batch fails, fall back to individual `mcp__google-workspace__find_and_replace_doc` calls per variable
7. Output the Google Docs link to the user

**Structured content note:** Variables like `{{SOW_SCOPE}}`, `{{SOW_DELIVERABLES}}`, `{{SOW_TIMELINE}}`, `{{SOW_FEES}}`, `{{AMENDMENT_DESCRIPTION}}`, and `{{AMENDMENT_CLAUSES}}` may contain multi-line content. Google Docs find-and-replace handles plain text only. For these fields:
- If the user provides a short single-line value, use find-and-replace directly
- If the content is complex/multi-line, insert a brief summary via find-and-replace and note in the output that the user should format detailed content directly in the generated Google Doc

---

## Phase G -- MNDA Bundling (service agreements only)

If the contract type is 2, 3, 4, or 5 (service agreements):

1. Ask via `AskUserQuestion`: "Would you also like to generate an MNDA (Template 1a) to bundle with this agreement?"
2. If yes:
   - Use the next sequential document number (current + 1)
   - Same client details and effective date
   - Repeat Phase F for template 1a
   - Both documents are logged in Phase H

---

## Phase H -- Log to Vault

1. Determine the org folder path: `organizations/{org-slug}/`
2. Check if `contracts.md` exists in that folder using `Glob`
3. If not, create it:

```markdown
---
last-updated: YYYY-MM-DD
---

# Contracts -- {Org Name}

| Doc # | Type | Effective Date | Status | Drive Link |
|-------|------|---------------|--------|------------|
```

4. Append a new row for each generated contract:
   ```
   | YYYY/NNN | {Contract Type} | {Effective Date} | Draft | [View](https://docs.google.com/document/d/{DOC_ID}/edit) |
   ```
5. Update `last-updated` in frontmatter to today's date
6. If this was a bundled generation (service agreement + MNDA), append both rows

---

## Final Output

After all phases complete, display:

```
Contract generated successfully!

{Contract Type}
   Doc #: {YYYY/NNN}
   Link: {Google Docs URL}

Location: Clients/{CLIENT_NAME}/Contracts/
Logged to: organizations/{org}/contracts.md

[If bundled:]
MNDA
   Doc #: {YYYY/NNN+1}
   Link: {Google Docs URL}
```
