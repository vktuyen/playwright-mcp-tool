---
name: analyze-ticket
description: Analyze a previously collected TrackerGo ticket and produce a root cause analysis. Use when the user wants to investigate, debug, or find the root cause of a ticket whose data has already been collected (a tickets/<id>-*/ticket-data.json file exists). Reads the structured data, identifies technical clues, formulates root cause hypotheses, and writes an investigation plan with source code targets and DB queries to tickets/<id>-*/analysis.md.
---

# Analyze Ticket — Root Cause Investigation

You are analyzing a collected TrackerGo support ticket to identify the root cause and propose a solution.

## Input
Ticket ID or folder path: $ARGUMENTS

If no argument provided, find the most recent folder under `tickets/` (use `ls -td tickets/*/`).

## Step 1: Load collected ticket data

Find the ticket data:
1. If argument is a ticket ID (e.g., `555718`), glob for `tickets/555718-*/ticket-data.json` and use the most recent match
2. If argument is a path, use it directly
3. If no argument, use the most recent `tickets/*/ticket-data.json`

If the file doesn't exist, tell the user to run `/collect-ticket <url>` first.

## Step 2: Summarize the issue

Create a concise issue summary covering:
- **What**: What is the user experiencing?
- **Where**: Which product/feature/screen is affected?
- **When**: When did the issue start? Any patterns?
- **Who**: Which customer, what account type?
- **Platform**: Android/iOS/Web? App version?
- **Reproduction**: Can it be consistently reproduced? What are the exact steps?

## Step 3: Extract technical clues

From the ticket data, identify:
- **Error patterns**: Any error messages, codes, or stack traces mentioned
- **Affected entities**: Job IDs, Asset IDs, Account IDs, Tenant IDs mentioned
- **Data points**: Shard key, tenant ID, app version, device info
- **Attachments**: What files were attached? Are there DB backups or recordings?

## Step 4: Check for DB backup

If a JobLogicDB.zip was included in the attachments:
1. List its contents: `unzip -l tickets/<folder>/attachments/*.zip`
2. Note the DB file for potential restoration/investigation
3. Suggest what tables/queries to check based on the issue

## Step 5: Formulate investigation plan

Based on the analysis, create an actionable investigation plan:

### Source Code Investigation
- Which codebase to look at (Mobile app, API, Web)
- Which components/modules are likely involved
- Specific files, classes, or functions to examine
- What to search for in the code (keywords, function names)

### Database Investigation (if DB backup available)
- Which tables to query
- What data to look for
- Queries to run

### Reproduction Steps
- Exact steps to reproduce the issue
- Test accounts/data to use
- Which devices/platforms to test on

## Step 6: Output the analysis

Save the analysis to the same ticket folder: `tickets/<folder>/analysis.md` with:
- Issue Summary
- Technical Clues
- Root Cause Hypothesis (ranked by likelihood)
- Investigation Plan
- Suggested Fix (if the root cause is clear enough)

## Important Notes
- Always cross-reference the checklist data (it has structured reproduce steps and device info)
- The tenant ID and shard key from Client Info are essential for DB investigation
- Look for patterns: is this a known issue? Has it happened before?
- Consider both client-side (mobile app) and server-side (API) causes
- If video recordings are in the attachments, mention them as evidence to review
