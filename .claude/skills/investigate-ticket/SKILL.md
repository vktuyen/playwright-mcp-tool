---
name: investigate-ticket
description: Full ticket investigation workflow — collects data from a TrackerGo support ticket AND produces a root cause analysis in one go. Use when the user provides a TrackerGo URL (https://trackergo.joblogic.com/Task/Detail/<id>) or ticket ID and wants to investigate, debug, find the root cause, or understand a customer issue end-to-end. This is the recommended entry point — it runs collect-ticket then analyze-ticket automatically.
---

# Investigate Ticket — End-to-End Workflow

You are running the full investigation pipeline for a TrackerGo support ticket: collect → analyze.

## Input
Ticket URL or ID: $ARGUMENTS

If no argument is provided, ask the user for the ticket URL or ID.

## Workflow

### Phase 1: Collect ticket data

Invoke the **`collect-ticket`** skill via the Skill tool, passing the ticket URL as args.

This will:
- Navigate to the ticket via Playwright MCP (handle login if needed)
- Extract all ticket data (header, task details, client info, summary)
- Expand and collect all event notes
- Open all checklists and extract structured form data
- Download all attachments (zips, DB backups, videos)
- Save everything to `tickets/<id>-<YYYYMMDD>/`

Wait for the collection phase to complete. Verify that `tickets/<id>-<YYYYMMDD>/ticket-data.json` exists before proceeding.

### Phase 2: Analyze the collected data

Invoke the **`analyze-ticket`** skill via the Skill tool, passing the ticket ID as args.

This will:
- Read the collected JSON
- Identify technical clues, affected entities, error patterns
- Inspect attachments (especially DB backups)
- Formulate root cause hypotheses
- Build an investigation plan (source code targets, DB queries, reproduction steps)
- Save the analysis to `tickets/<id>-<YYYYMMDD>/analysis.md`

### Phase 3: Final summary

After both phases complete, print a concise summary to the user:

```
✓ Ticket <id> investigation complete

Collected:  tickets/<id>-<YYYYMMDD>/ticket-data.json
Analysis:   tickets/<id>-<YYYYMMDD>/analysis.md
Attachments: tickets/<id>-<YYYYMMDD>/attachments/

Issue: <one-line summary>
Most likely root cause: <top hypothesis>
Next step: <top investigation action>
```

## Important
- Always run BOTH phases in sequence — do not stop after collection
- If Phase 1 fails (e.g., login expired and user can't re-auth), report the failure and STOP; do not attempt analysis on missing data
- If the ticket data already exists from a previous collection (same day), you can skip Phase 1 and go straight to Phase 2 — but ask the user first whether to refresh the data
