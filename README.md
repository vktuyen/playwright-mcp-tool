# Paywright-MCP

Claude Code + Playwright MCP agent for collecting customer support ticket data from [TrackerGo](https://trackergo.joblogic.com). Navigates ticket pages, extracts all information, downloads attachments (DB backups, videos, logs), and produces root cause analysis.

## How It Works

Claude Code uses the [Playwright MCP server](https://github.com/microsoft/playwright-mcp) to control a real browser. Two custom skills (slash commands) handle the workflow:

- **`/collect-ticket <url>`** — Navigates to the ticket, extracts all data, downloads attachments
- **`/analyze-ticket <id>`** — Reads collected data, produces root cause analysis

There is no custom code. Claude reads the skill instructions and uses MCP browser tools (`browser_navigate`, `browser_click`, `browser_snapshot`) to interact with the page, plus standard tools (`Bash`, `Write`) for file operations.

## Prerequisites

- **Node.js** 18+ (`brew install node`) — required for the Playwright MCP server
- **Claude Code** — with MCP support

## Setup

```bash
# Install Playwright browser (Chromium) — one-time
npx playwright install chromium

# Optional: store credentials so Claude can auto-fill the login form when sessions expire
cp .env.example .env
# then edit .env with your TrackerGo email + password
```

The `.mcp.json` configures the Playwright MCP server automatically when you open Claude Code in this directory. The `.env` file is optional — without it, you'll need to type your credentials directly into the browser window when prompted.

## Usage

### 1. Collect ticket data

```
/collect-ticket https://trackergo.joblogic.com/Task/Detail/555718
```

On first run, Claude will navigate to TrackerGo and help you log in via Azure SSO + MFA. The browser session persists in `browser-data/` so you only log in once (~8 hours).

The skill collects:
- Ticket header (ID, title, status, priority, SLA)
- Task details (owner, department, product, call type, etc.)
- Client info (tenant ID, shard key, contacts)
- AI-generated summary
- All event notes (expanded full text)
- Checklist form data (reproduce steps, device info, app version)
- All attachments (downloaded)
- Full-page screenshot

Everything is saved to `tickets/<id>-<YYYYMMDD>/`.

### 2. Analyze the issue

```
/analyze-ticket 555718
```

Reads the collected data and produces:
- Issue summary (what/where/when/who/platform)
- Technical clues from notes and checklists
- Root cause hypotheses
- Investigation plan (source code, DB queries, reproduction steps)

Saved to `tickets/<id>-<YYYYMMDD>/analysis.md`.

### Login Flow (Azure AD SSO)

1. Navigate to trackergo.joblogic.com → "Login with Azure" button
2. Redirects to Microsoft login → Enter email → Enter password
3. Approve MFA in Authenticator app (enter the displayed number)
4. "Stay signed in?" → Yes
5. Redirected back to TrackerGo dashboard

## Output Structure

Each ticket gets its own folder:

```
tickets/555718-20260415/
  ticket-data.json          # All structured data
  screenshot.png            # Full-page screenshot
  analysis.md               # Root cause analysis
  attachments/
    Ticket-data.zip          # Downloaded files (videos, DB backups)
```

## Security

- `browser-data/` — browser session with login cookies (gitignored)
- `tickets/` — collected data with customer info (gitignored)
- `.env` — credentials (gitignored)
- **Never commit these files**

## Project Structure

```
Paywright-MCP/
├── .mcp.json                    # Playwright MCP server config
├── .claude/commands/
│   ├── collect-ticket.md        # /collect-ticket skill
│   └── analyze-ticket.md        # /analyze-ticket skill
├── CLAUDE.md                    # Project context for Claude
├── tickets/                     # Per-ticket output (gitignored)
│   └── <id>-<YYYYMMDD>/
│       ├── ticket-data.json
│       ├── screenshot.png
│       ├── analysis.md
│       └── attachments/
├── browser-data/                # Persistent browser profile (gitignored)
└── .mcp-output/                 # Temp MCP artifacts (auto-cleaned)
```
