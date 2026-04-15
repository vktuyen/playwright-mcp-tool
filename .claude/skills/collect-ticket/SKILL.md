---
name: collect-ticket
description: Collect all data from a TrackerGo customer support ticket. Use when the user provides a TrackerGo URL (https://trackergo.joblogic.com/Task/Detail/<id>) or asks to scrape, gather, or download data from a support ticket. Navigates the ticket page via Playwright MCP, extracts header, task details, client info, all event notes (expanded), checklist forms, and downloads all attachments (DB backups, videos, logs). Saves everything to tickets/<id>-<YYYYMMDD>/.
---

# Collect Ticket Data from TrackerGo

You are collecting data from a TrackerGo support ticket using Playwright MCP browser tools.

## Input
Ticket URL: $ARGUMENTS

If no URL is provided, ask the user for the ticket URL.

## Output Directory
All files for this ticket go into a single folder under `tickets/`:
```
tickets/<ticketId>-<YYYYMMDD>/
  ticket-data.json          # structured data
  screenshot.png            # full page screenshot
  attachments/              # downloaded files (zips, DBs, videos)
```

Extract the ticket ID from the URL (e.g., `555718` from `/Task/Detail/555718`).
Use today's date for the folder name (e.g., `tickets/555718-20260415/`).
Create the directory at the start: `mkdir -p tickets/<ticketId>-<YYYYMMDD>/attachments`

## Prerequisites
- The Playwright MCP server must be running (configured in `.mcp.json`)
- The browser session in `browser-data/` should be authenticated (lasts ~8 hours)
- If not authenticated, follow the **Login Flow** below

## Login Flow (when session expires)
If `browser_navigate` to a ticket URL redirects to `/Account/Login` or `login.microsoftonline.com`, the session has expired. Re-authenticate:

### Reading credentials from .env
```bash
set -a; source .env 2>/dev/null; set +a
```
This loads `TRACKERGO_EMAIL` and `TRACKERGO_PASSWORD` as env vars.
**NEVER print the password to the user.** When typing into the browser, use `browser_type` directly.

### Steps

1. Navigate to `https://trackergo.joblogic.com`
2. Click the "Login with Azure" button
3. On the Microsoft SSO email page:
   - If `.env` has `TRACKERGO_EMAIL`, use `browser_type` to fill the email field → click Next
   - Otherwise ask the user to type it in the browser
4. On the password page:
   - If `.env` has `TRACKERGO_PASSWORD`, use `browser_type` to fill the password field → click Sign in
   - Otherwise ask the user to type it in the browser
5. **MFA step** (push notification — TOTP automation is not possible because the org's Authentication Methods policy only allows Microsoft Authenticator push, no third-party software OATH tokens):
   a. The page shows "Approve sign in request" with a number (e.g., "91")
   b. Tell the user the number clearly: *"Please approve the Microsoft Authenticator request on your phone — enter number **91**"*
   c. Wait for the user to confirm they approved
6. On "Stay signed in?" → click **Yes** (this extends the browser session)
7. Verify the URL is back on `trackergo.joblogic.com/` (not `/Account/Login` or `login.microsoftonline.com`)
8. Then navigate to the original ticket URL and continue

## Collection Steps

### Step 1: Navigate and verify access
1. Use `browser_navigate` to go to the ticket URL
2. Take a `browser_snapshot` to verify the page loaded
3. If redirected to a login page (URL contains "login", "Account", or "microsoftonline"), inform the user their session expired and help them re-authenticate

### Step 2: Extract ticket header and details
From the initial snapshot, extract:
- **Ticket ID** (from breadcrumb, e.g., "555718")
- **Title** (the main heading text)
- **Status** (e.g., Outstanding, Resolved)
- **Priority** (e.g., P1, P2, P3)
- **SLA** info (hours left, config)

### Step 3: Extract Task Details sidebar
The right sidebar shows task metadata. Extract ALL fields:
- Owner, Department, Product, Call Type, Domain, Type of Work, Sub Category
- Requested By, Customer Persona, Source
- Contact info (Type, Telephone, Email)
- Time Since Logged, Completed Count, Logged date, Logged By
- Tags (listed under description area)

### Step 4: Click "Client Info" tab and extract
Click the "Client Info" tab in the sidebar and snapshot. Extract:
- Client name, Group, Account Manager, Team
- Main Telephone, Main Email, Primary User
- Tenant ID, Country, Shard Key

### Step 5: Click "Summary" tab and extract
Click the "Summary" tab and snapshot. Extract the AI-generated summary text.

### Step 6: Check "Linked Tasks" tab
Click "Linked Tasks" tab. Note any linked ticket IDs and their titles.

### Step 7: Expand and collect ALL events
1. Click the "Expand All Text" checkbox to expand all notes
2. Take a snapshot and extract EVERY event:
   - For **Notes**: type, status, author, department, date, full text content, note type
   - For **Attachments**: filename, download URL
   - For **Checklists**: checklist name
   - For **Jira Links**: link URL and text
   - For **Email Sent**: recipient and content
3. If the snapshot is too large, read events one by one

### Step 8: Open and extract ALL checklists
For each checklist link found in the events:
1. Click the checklist link to open the modal
2. Snapshot the modal
3. Extract ALL fields and their values (these contain structured investigation data like Customer Problem, Reproduce Steps, App Version, etc.)
4. Close the modal

### Step 9: Download ALL attachments
For each attachment link found in the events:
1. Click the download link
2. Wait for download to complete (use `browser_wait_for` if needed)
3. The file will download to `.mcp-output/` — move it to `tickets/<ticketId>-<YYYYMMDD>/attachments/`
4. Note the filename and local path

After downloading, check the contents of zip files:
```bash
unzip -l tickets/<ticketId>-<YYYYMMDD>/attachments/<filename>
```

### Step 10: Take screenshots
- Take a full-page screenshot of the ticket page
- Move it from `.mcp-output/` to `tickets/<ticketId>-<YYYYMMDD>/screenshot.png`

### Step 11: Save structured output
Save ALL collected data as JSON to `tickets/<ticketId>-<YYYYMMDD>/ticket-data.json` with this structure:
```json
{
  "ticketId": "",
  "url": "",
  "collectedAt": "",
  "header": { "title", "status", "priority", "sla", "slaConfig" },
  "description": "",
  "taskDetails": { "owner", "department", "product", "callType", "domain", "typeOfWork", "subCategory", "requestedBy", "customerPersona", "source", "timeSinceLogged", "completedCount", "logged", "loggedBy" },
  "contact": { "type", "telephone", "email" },
  "clientInfo": { "client", "group", "accountManager", "team", "mainTelephone", "mainEmail", "primaryUser", "tenantId", "country", "shardKey" },
  "tags": [],
  "checklists": [{ "name": "", "fields": {} }],
  "events": [{ "type", "status", "author", "department", "dateLogged", "noteType", "text" }],
  "summary": "",
  "attachments": [{ "filename", "localPath", "sizeBytes", "contents" }],
  "screenshots": ["screenshot.png"]
}
```

### Step 12: Clean up MCP artifacts
After all data is saved, clean up temporary MCP output:
```bash
rm -rf .mcp-output/*.yml .mcp-output/*.log
```

## Important Notes
- Be thorough — do NOT skip any events, attachments, or checklist fields
- Some notes are truncated with "See more" links — always expand them or use "Expand All Text"
- Checklists open as modals — they contain critical structured data (reproduce steps, device info, DB backup references)
- Attachments may be `.zip` files containing video recordings, DB backups (JobLogicDB.zip), logs, etc.
- The Client Info tab has the tenant ID and shard key which are essential for DB investigation
- After collection, print a summary of what was collected and the output folder path
