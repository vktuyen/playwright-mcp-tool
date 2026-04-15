# Paywright-MCP

Claude Code + Playwright MCP agent for collecting and analyzing customer support tickets from [TrackerGo](https://trackergo.joblogic.com). Navigates ticket pages, extracts all information, downloads attachments (DB backups, videos, logs), and produces root cause analysis — all from a single command.

## Quick Start

Open Claude Code in this directory and run:

```
/investigate-ticket https://trackergo.joblogic.com/Task/Detail/555718
```

That's it. Claude will:
1. Open a browser, log you into TrackerGo (Azure SSO + MFA the first time)
2. Collect everything from the ticket — header, notes, checklists, attachments
3. Analyze the issue and produce a root cause report

Output lands in `tickets/<id>-<date>/` (data, screenshot, attachments, analysis.md).

> Tip: you can also just say *"Hey Claude, investigate https://trackergo.joblogic.com/Task/Detail/555718"* — no slash command needed. The skill auto-triggers when it sees a TrackerGo URL.

## How It Works

Claude Code uses the [Playwright MCP server](https://github.com/microsoft/playwright-mcp) to control a real browser. Three custom skills handle the workflow:

| Skill | Purpose |
|---|---|
| **`/investigate-ticket <url>`** | **(recommended)** End-to-end: collects data + analyzes the issue in one go |
| `/collect-ticket <url>` | Just collect data, save to `tickets/<id>-<date>/ticket-data.json` |
| `/analyze-ticket <id>` | Just analyze previously-collected data, save to `analysis.md` |

The skills live in `.claude/skills/<name>/SKILL.md` (proper Claude Code agent skills with YAML frontmatter and auto-discovery). There is no custom code — Claude reads the SKILL.md instructions and uses MCP browser tools (`browser_navigate`, `browser_click`, `browser_snapshot`) plus standard tools (`Bash`, `Write`) to do the work.

## Prerequisites

- **Node.js** 18+ (`brew install node`) — required for the Playwright MCP server
- **Claude Code** — with MCP support

## Setup

```bash
# 1. Install the Playwright browser (Chromium) — one-time
npx playwright install chromium

# 2. Copy the .env template and fill in your credentials
cp .env.example .env
```

Edit `.env` and add:
- `TRACKERGO_EMAIL` — your Microsoft account email
- `TRACKERGO_PASSWORD` — your Microsoft account password

That's it. The `.mcp.json` configures the Playwright MCP server automatically when you open Claude Code in this directory.

### Why isn't 2FA fully automated?

We tried. It's blocked by org policy.

The plan was: add a TOTP secret to `.env` and let Claude generate 6-digit codes via `oathtool`, completing the Microsoft sign-in unattended. But the JobLogic Microsoft Entra tenant has the **Authentication Methods policy** restricted to **Microsoft Authenticator (push notifications) only**. Third-party software OATH tokens / "different authenticator app" are blocked at the tenant level — they don't appear as an option in <https://mysignins.microsoft.com/security-info>, so there's no Base32 secret to extract.

**In practice this is fine:**
- The browser session in `browser-data/` lasts **~8 hours** after you click "Yes" on "Stay signed in?"
- So you only do MFA **once per workday**
- When MFA is needed, Claude auto-fills email + password and just asks you to approve the push notification on your phone (~5 seconds)
- The rest (collection + analysis + attachment downloads) happens fully unattended

If JobLogic's IT policy ever changes and allows third-party authenticator apps, we can revisit.

## Usage

### One command (recommended)

```
/investigate-ticket https://trackergo.joblogic.com/Task/Detail/555718
```

This runs the full pipeline:
1. **Collect** — navigates to the ticket, extracts everything (header, task details, client info, summary, all notes, checklists, attachments, screenshot)
2. **Analyze** — reads the collected data and produces root cause analysis with investigation plan
3. **Summary** — prints a concise overview with the top hypothesis and next steps

You can also just say "Hey Claude, can you investigate https://trackergo.joblogic.com/Task/Detail/555718?" — Claude will pick up the right skill from the URL.

### Or run phases individually

```
/collect-ticket https://trackergo.joblogic.com/Task/Detail/555718    # just collect
/analyze-ticket 555718                                                # just analyze
```

### Login Flow (Azure AD SSO)

On first run (or when the session expires after ~8 hours):

1. Claude navigates to TrackerGo and clicks "Login with Azure"
2. **Email + password** — auto-filled from `.env`
3. **MFA** — Claude shows you the push-notification number, you approve in Microsoft Authenticator on your phone
4. Click "Yes" on "Stay signed in?" — Claude does this automatically
5. Browser session is saved to `browser-data/` for ~8 hours

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
├── .mcp.json                              # Playwright MCP server config
├── .claude/skills/
│   ├── investigate-ticket/SKILL.md        # /investigate-ticket — umbrella
│   ├── collect-ticket/SKILL.md            # /collect-ticket — data collection
│   └── analyze-ticket/SKILL.md            # /analyze-ticket — root cause
├── CLAUDE.md                              # Project context for Claude
├── tickets/                               # Per-ticket output (gitignored)
│   └── <id>-<YYYYMMDD>/
│       ├── ticket-data.json
│       ├── screenshot.png
│       ├── analysis.md
│       └── attachments/
├── browser-data/                          # Persistent browser profile (gitignored)
└── .mcp-output/                           # Temp MCP artifacts (auto-cleaned)
```
