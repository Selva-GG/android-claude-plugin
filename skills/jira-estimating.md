---
name: jira-estimating
description: Use when reviewing or updating a Jira ticket's time estimate or story points — checking accuracy, comparing with time spent, updating based on task complexity
user-invocable: false
---

# Reviewing & Updating Time Estimates

## Overview

Review the current time estimate and story points on a Jira ticket. Compare with actual time spent. Update if the user wants to change them.

## Step 1: Present Current Tracking

From the fetched issue data (use `android:jira-fetching` if not already fetched):

**Time tracking fields:**
- `fields.timeoriginalestimate` → original estimate in seconds
- `fields.timetracking.originalEstimate` → human-readable original (e.g., "4h")
- `fields.timetracking.remainingEstimate` → remaining estimate
- `fields.timetracking.timeSpent` → time already logged
- `fields.timetracking.timeSpentSeconds` → logged time in seconds

**Story points:**
- `fields.customfield_10016` → story points (common Jira Cloud field)
- May also be `fields.story_points` or another custom field — check what's available

Convert seconds to human-readable:
- `3600` → 1h
- `7200` → 2h
- `14400` → 4h
- `28800` → 1d (8 working hours)
- `null` → "Not set"

Present:

> **Time Tracking:**
> | | Value |
> |---|---|
> | Original estimate | [Xh or Not set] |
> | Time spent | [Xh or None] |
> | Remaining | [Xh or N/A] |
> | Story points | [N or Not set] |
>
> **Analysis:**
> - [If time spent > 0 and estimate exists: "[X]h spent of [Y]h estimated ([Z]% complete)"]
> - [If time spent > estimate: "⚠ Time spent ([X]h) exceeds estimate ([Y]h) by [Z]h"]
> - [If no estimate: "No estimate set — consider adding one"]
>
> Does this estimate still look accurate? (yes / update to Xh / update story points)

## Step 2: Parse User Input

### Time Estimates

Accept flexible time formats:
- `2h` → 7200 seconds
- `30m` → 1800 seconds
- `4h 30m` → 16200 seconds
- `1d` → 28800 seconds (8 working hours)
- `1d 2h` → 36000 seconds
- `2d 4h 30m` → 72600 seconds
- `0.5h` → 1800 seconds
- `1.5d` → 43200 seconds

**Conversion formula:**
- Days: `× 28800` (8 working hours per day — Jira convention)
- Hours: `× 3600`
- Minutes: `× 60`

**Validation:**
- Zero or negative → "Estimate must be positive"
- > 30d → "That's over 30 working days. Did you mean [X]?"
- Decimal days/hours are OK (e.g., `1.5h` = 5400s)

### Story Points

Accept:
- Integer: `1`, `2`, `3`, `5`, `8`, `13`, `21`
- Fibonacci or custom scales — accept any positive number
- `0` → valid (trivial task)

**Validation:**
- Negative → "Story points must be 0 or positive"
- > 100 → "That seems unusually high. Did you mean [X]?"

## Step 3: Update the Estimate

### Update Time Estimate

Call `mcp__claude_ai_Atlassian__editJiraIssue` with:
- `cloudId`: cached from session
- `issueIdOrKey`: the ticket ID
- `fields`: `{ "timeoriginalestimate": <seconds_as_integer> }`

### Update Story Points

Call `mcp__claude_ai_Atlassian__editJiraIssue` with:
- `cloudId`: cached from session
- `issueIdOrKey`: the ticket ID
- `fields`: `{ "customfield_10016": <points_as_number> }`

**Note:** The custom field ID for story points varies per Jira instance. If `customfield_10016` fails with 400, try `story_points` or ask the user for the correct field name.

### Update Both

If user wants to update both estimate and story points, make a single API call:
- `fields`: `{ "timeoriginalestimate": <seconds>, "customfield_10016": <points> }`

**Error handling:**
- 400 → "Failed to update. The field may not be editable in this project, or the field ID may be wrong."
- 400 on story points → "Story points field not found with default ID. What is your Jira's story points field name?"
- 403 → "No permission to edit this ticket."

**On success:**
> Updated [TICKET-ID]:
> - Estimate: [old] → [new]
> - Story points: [old] → [new]

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Sending time as string instead of seconds | Always convert to integer seconds for `timeoriginalestimate` |
| Forgetting that 1d = 8h (not 24h) | Use Jira's working day convention: 8 hours per day |
| Updating without asking | Always confirm with user first |
| Assuming story points field ID | `customfield_10016` is common but not universal — handle gracefully |
| Not showing spent vs estimate comparison | Always show how much time has been used — helps user decide |
| Ignoring negative remaining time | If spent > estimate, explicitly warn the user |
