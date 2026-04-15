# Paywright-MCP

Claude Code + Playwright MCP agent for collecting customer support ticket data from TrackerGo (trackergo.joblogic.com).

## Architecture

Claude uses the `@playwright/mcp` server (configured in `.mcp.json`) to control a real Chromium browser. No custom code — just MCP browser tools and two Claude skills:

- `/collect-ticket <url>` — Navigates ticket page, extracts all data, downloads attachments
- `/analyze-ticket <id>` — Reads collected data, produces root cause analysis

## Login Flow (TrackerGo + Azure AD)
1. Navigate to trackergo.joblogic.com → redirects to `/Account/Login`
2. Click "Login with Azure" button
3. Redirects to `login.microsoftonline.com` (Azure AD)
4. Enter email → Enter password → Approve MFA (Authenticator app, enter number)
5. "Stay signed in?" → Yes
6. Redirects back to TrackerGo dashboard
7. Session persists in `browser-data/` (~8 hours)

## Ticket Page Data Model
A TrackerGo ticket (`/Task/Detail/<id>`) contains:

### Header
- Ticket ID, Title, Status (Outstanding/Resolved/etc.), Priority (P1-P5), SLA

### Task Details (right sidebar)
- Owner, Department, Product, Call Type, Domain, Type of Work, Sub Category
- Requested By, Customer Persona, Source
- Contact info (Type, Telephone, Email)
- Time Since Logged, Completed Count, Logged date, Logged By

### Client Info (sidebar tab)
- Client name, Group, Account Manager, Team
- Main Telephone, Main Email, Primary User
- **Tenant ID**, **Country**, **Shard Key** (essential for DB investigation)

### Summary (sidebar tab)
- AI-generated summary of the ticket

### Events (main content, click "Expand All Text" to see full notes)
- **Notes**: Status changes, author, department, date, full text, note type
- **Attachments**: Downloadable files (zip, DB backups, videos, screenshots)
- **Checklists**: Clickable links that open modals with structured forms (Customer Problem, Reproduce Steps, App Version, Device info, DB Backup reference)
- **Jira Links**, **Email Sent**, **Title Changes**, etc.

### Attachments
- `.zip` files may contain: video recordings (.mp4), JobLogicDB backups (.zip), logs, screenshots
- Download URL pattern: `/Task/DownloadTaskAttachment?attachmentId=<id>&taskId=<id>`

## Key Paths
- `browser-data/` — Persistent MCP browser profile with login session (gitignored)
- `.mcp-output/` — Temporary MCP artifacts (gitignored, cleaned after collection)
- `tickets/<id>-<YYYYMMDD>/` — Per-ticket output folder (gitignored)
  - `ticket-data.json` — Structured ticket data
  - `screenshot.png` — Full-page screenshot
  - `attachments/` — Downloaded files (zips, DBs, videos)
  - `analysis.md` — Root cause analysis (created by /analyze-ticket)
- `.claude/commands/` — Custom Claude skills

## Workflow
1. `/collect-ticket <ticket-url>` — Collect all ticket data (handles login if needed)
2. `/analyze-ticket <id>` — Produce root cause analysis
3. Cross-reference analysis with source code to find the fix
