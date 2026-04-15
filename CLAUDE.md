# Paywright-MCP

Claude Code + Playwright MCP agent for collecting customer support ticket data from TrackerGo (trackergo.joblogic.com).

## Architecture

Claude uses the `@playwright/mcp` server (configured in `.mcp.json`) to control a real Chromium browser. No custom code — just MCP browser tools and three Claude agent skills:

- `/investigate-ticket <url>` — **Umbrella skill (recommended)**: runs collect + analyze in one go
- `/collect-ticket <url>` — Navigates ticket page, extracts all data, downloads attachments
- `/analyze-ticket <id>` — Reads collected data, produces root cause analysis

Skills live at `.claude/skills/<name>/SKILL.md` (proper agent-skill format with YAML frontmatter and auto-discovery). Each has a `description` so Claude can also auto-invoke them when it sees a TrackerGo URL in the conversation — no slash command required.

## Login Flow (TrackerGo + Azure AD)
1. Navigate to trackergo.joblogic.com → redirects to `/Account/Login`
2. Click "Login with Azure" button
3. Redirects to `login.microsoftonline.com` (Azure AD)
4. Email + password (auto-filled from `.env`)
5. **MFA — push notification only**: ask user to approve number in Microsoft Authenticator on their phone
6. "Stay signed in?" → Yes
7. Redirects back to TrackerGo dashboard
8. Session persists in `browser-data/` (~8 hours)

### Why no TOTP automation?
We tried — it's blocked by the JobLogic Microsoft Entra Authentication Methods policy, which only permits Microsoft Authenticator push (no third-party software OATH tokens). The "different authenticator app" option doesn't appear in `https://mysignins.microsoft.com/security-info`. So 2FA stays manual; everything else is automated.

## Credentials (`.env`)
- `TRACKERGO_EMAIL` — Microsoft account email
- `TRACKERGO_PASSWORD` — Microsoft account password
- File is gitignored. Read with `set -a; source .env; set +a` so vars are available to subprocesses.

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
- `.claude/skills/<name>/SKILL.md` — Custom Claude agent skills

## Workflow
1. `/collect-ticket <ticket-url>` — Collect all ticket data (handles login if needed)
2. `/analyze-ticket <id>` — Produce root cause analysis
3. Cross-reference analysis with source code to find the fix
