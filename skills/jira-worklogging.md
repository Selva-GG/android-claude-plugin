---
name: jira-worklogging
description: Use when logging work time to a Jira ticket — adding hours spent, viewing existing worklogs, adding work descriptions, tracking time after completing a task
user-invocable: false
---

# Logging Work Time

## Overview

Add a worklog entry to a Jira ticket to track time spent. Supports viewing existing worklogs, adding descriptions, and date overrides.

## Step 1: Show Existing Worklogs

Before adding a new entry, show what's already been logged to avoid duplicates.

From the fetched issue data:
- `fields.worklog.worklogs[]` — existing worklog entries

Present:

> **Existing Worklogs for [TICKET-ID]:**
> | Date | Author | Time | Description |
> |------|--------|------|-------------|
> | [date] | [author.displayName] | [timeSpent] | [comment or "—"] |
> | [date] | [author.displayName] | [timeSpent] | [comment or "—"] |
>
> **Total logged:** [timetracking.timeSpent]
> **Original estimate:** [timetracking.originalEstimate or "Not set"]
> **Remaining:** [timetracking.remainingEstimate or "N/A"]

If no worklogs exist: "No work logged yet."

## Step 2: Ask for Time Spent

> Log work time to Jira? (yes/no)

If no → skip, done.

If yes:

> How much time did you spend? (e.g., 2h, 30m, 4h 30m)

## Step 3: Validate Input

Accept flexible time formats — Jira accepts these directly in the `timeSpent` field (no conversion to seconds needed for worklogs):
- `2h` → "2h"
- `30m` → "30m"
- `4h 30m` → "4h 30m"
- `1d` → "1d"
- `1d 2h` → "1d 2h"
- `1d 2h 30m` → "1d 2h 30m"

**Validation:**
- Empty or zero → "Please enter a valid time (e.g., 2h, 30m)"
- Negative → "Time must be positive"
- > 24h single entry → "That's over 24 hours for a single entry. Did you mean [X]? (yes / correct to [Y])"
- Duplicate detection: If a worklog exists for the same day with similar time → "You already logged [X] today. Add another [Y] entry? (yes/no)"

## Step 4: Auto-Generate Work Description

Do NOT ask the user for a description — generate it automatically from the git history.

### Determine the since-date

If previous worklogs exist, use the most recent worklog's `started` date as the cutoff:
```bash
git log --since="<last-worklog-date>" --oneline --no-merges
```

If no previous worklogs exist, use all commits on the current branch since it diverged from the base:
```bash
git log <base-branch>..HEAD --oneline --no-merges
```

### Build the description

From the commit list:
1. Group commits by theme (e.g., "Added X", "Fixed Y", "Updated Z")
2. Summarize in 2-4 concise bullet points describing WHAT was done and WHY
3. If a PR was created in this session, append: `PR: <PR URL>`

**Example output:**
```
- Implemented SafeWebViewClient with HTTPS enforcement and domain allowlist (OWASP M1/M8)
- Applied SafeWebViewClient in WebViewScreen and WebViewLauncher; tightened initial URL validation to HTTPS-only
- Added 9 unit tests covering all acceptance criteria scenarios
PR: https://github.com/dmdbrands/meApp/pull/1583
```

Present the generated description to the user for confirmation:
> **Generated worklog description:**
> [description]
>
> Use this? (yes / edit)

If user wants to edit, let them type a replacement.

Include the confirmed description as the `comment` field in the worklog.

## Step 5: Optional — Date Override

By default, the worklog is for today. Ask only if relevant:

> Log this for today ([today's date])? (yes / different date)

If different date:
> Which date? (YYYY-MM-DD format, e.g., 2026-03-10)

Validate:
- Must be valid date format
- Cannot be in the future
- Cannot be more than 30 days in the past (Jira may reject it)

Format for API: `"started": "2026-03-10T09:00:00.000+0000"` (ISO 8601)

## Step 6: Get Cloud ID

Call `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` for the `cloudId`.

Skip if already cached from a previous skill call in this session.

## Step 7: Add Worklog

Call `mcp__claude_ai_Atlassian__addWorklogToJiraIssue` with:
- `cloudId`: from Step 6
- `issueIdOrKey`: the ticket ID
- `timeSpent`: the user-provided value (e.g., "2h")
- `comment`: work description (if provided, omit if not)
- `started`: date override (if provided, omit for today)

**Error handling:**
- 400 → "Failed to log work. Time tracking may be disabled for this project, or the time format is invalid."
- 403 → "No permission to log work on this ticket. Check your project permissions."
- 404 → "Ticket not found. It may have been deleted or moved."
- 400 with date error → "Invalid date. Ensure format is correct and date is not in the future."

**On success:**
> Logged [time] to [TICKET-ID].
> [If description: Description: "[description]"]
> [If date override: Date: [date]]

## Step 8: Show Updated Totals

After logging, present the updated time tracking:

> **Time Tracking Updated:**
> - Original estimate: [Xh]
> - Total time spent: [new total] (was [previous])
> - Remaining: [Xh]
> [If spent > estimate: "⚠ Total time ([X]) now exceeds the estimate ([Y])"]

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not showing existing worklogs | Always show what's already logged to prevent duplicates |
| Converting time to seconds for worklogs | Worklogs accept human-readable format directly (unlike estimates) |
| Logging without asking | Always confirm with user first |
| Ignoring duplicate detection | Warn if same-day entry already exists |
| Skipping the updated totals | Always show how the entry changed the time tracking |
| Generic worklog description | Auto-generate from git commits since last worklog — be specific about what was done |
| Omitting PR URL from worklog | Always append `PR: <URL>` to description if a PR was created in this session |
