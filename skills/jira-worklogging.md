---
name: jira-worklogging
description: Use when logging work time to a Jira ticket — adding hours spent, viewing existing worklogs, adding work descriptions, tracking time after completing a task
user-invocable: false
---

# Logging Work Time

> **ABSOLUTE HARD RULE — NO ATLASSIAN MCP. EVER.**
> NEVER use `mcp__claude_ai_Atlassian__*` or ANY Atlassian MCP tool. ALL Jira operations MUST use ACLI (`acli`), curl + REST API, or `gh` CLI. No exceptions. No fallbacks to MCP. Ever.

## Step 0: Parse Arguments

If arguments include time and/or date (passed from finish-task or user):
- Parse time: `10m`, `1h`, `30m`, `2h30m`, `2h 30m`
- Parse date: `YYYY-MM-DD` or `today` (default)
- If both provided: skip Steps 2 and 5 (time and date prompts), go directly to Step 3 validation then Step 4/6
- If only time provided: use today's date, skip Step 2
- If neither: run full interactive flow

## Overview

Add a worklog entry to a Jira ticket to track time spent. Supports viewing existing worklogs, adding descriptions, and date overrides. Uses curl + Jira REST API v3 directly (ACLI does not support worklog operations).

### Prerequisites
Verify ACLI is authenticated and config exists:
```bash
acli jira auth status 2>&1 && test -f ~/.jira-config && source ~/.jira-config && test -n "$JIRA_TOKEN"
```
If any check fails: **REQUIRED:** Auto-run the `android:jira-setup` skill inline — do not tell the user to run it manually. After setup completes, continue with the original task.

## Step 1: Show Existing Worklogs

Before adding a new entry, show what's already been logged to avoid duplicates.

Fetch worklogs via REST API (do NOT use ACLI for worklog operations — its flags are unreliable):
```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>?fields=worklog,timetracking"
```

From the JSON output:
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

## Step 3: Validate and Convert Input

Accept flexible time formats and convert to seconds (REST API v3 requires `timeSpentSeconds`):
- `2h` → 7200
- `30m` → 1800
- `4h 30m` → 16200
- `1d` → 28800 (8 working hours)
- `1d 2h` → 36000
- `1d 2h 30m` → 37800
- `0.5h` → 1800

**Conversion formula:**
- Days: `× 28800` (8 working hours per day — Jira convention)
- Hours: `× 3600`
- Minutes: `× 60`

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
PR: https://github.com/OWNER/REPO/pull/NNN
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

Format for API: Use the **local machine timestamp** with local timezone offset. Generate with:
```bash
date +"%Y-%m-%dT%H:%M:%S.000%z"
```
Example output: `"started": "2026-03-18T09:00:00.000+0530"` — NEVER use UTC (`+0000`). Always use the local timezone offset.

## Step 6: Add Worklog via REST API

The Jira REST API v3 requires ADF (Atlassian Document Format) for the comment field.

**MANDATORY: Always generate and include the local timestamp.** Never omit `started` — Jira defaults to server time which may be a different timezone.

```bash
source ~/.jira-config
LOCAL_TS=$(date +"%Y-%m-%dT%H:%M:%S.000%z")
curl -s -w "\n%{http_code}" -X POST \
  -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
  -H "Content-Type: application/json" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>/worklog" \
  -d "{
    \"timeSpentSeconds\": <SECONDS>,
    \"comment\": {
      \"version\": 1,
      \"type\": \"doc\",
      \"content\": [{\"type\": \"paragraph\", \"content\": [{\"type\": \"text\", \"text\": \"<DESCRIPTION>\"}]}]
    },
    \"started\": \"$LOCAL_TS\"
  }"
```

**Timestamp rules (MANDATORY — no exceptions):**
- ALWAYS include `started` field — NEVER omit it
- ALWAYS use local machine time: `date +"%Y-%m-%dT%H:%M:%S.000%z"`
- NEVER use UTC (`+0000`) — always use the local timezone offset
- For date overrides, append the local timezone offset: `2026-03-10T09:00:00.000+0530` (use `date +%z` to get offset)

**ADF comment format notes:**
- The comment MUST use ADF format for REST API v3 — plain strings will be rejected
- The template above supports single-paragraph text only
- For multi-line descriptions, create multiple paragraph nodes in the `content` array:
  ```json
  "content": [
    {"type": "paragraph", "content": [{"type": "text", "text": "Line 1"}]},
    {"type": "paragraph", "content": [{"type": "text", "text": "Line 2"}]}
  ]
  ```

**Parse the response:**
- Extract the HTTP status code from the `-w "\n%{http_code}"` output
- HTTP 201 → success
- HTTP 400 → "Failed to log work. Time tracking may be disabled for this project."
- HTTP 403 → "No permission to log work. Run `/android:jira-setup` to check credentials."
- HTTP 404 → "Ticket not found."
- HTTP 400 with date error → "Invalid date. Ensure format is correct and date is not in the future."

**On success:**
> Logged [time] to [TICKET-ID].
> [If description: Description: "[description]"]
> [If date override: Date: [date]]

## Step 7: Show Updated Totals

After logging, fetch updated time tracking:

```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>?fields=timetracking"
```

Present:

> **Time Tracking Updated:**
> - Original estimate: [Xh]
> - Total time spent: [new total] (was [previous])
> - Remaining: [Xh]
> [If spent > estimate: "Total time ([X]) now exceeds the estimate ([Y])"]

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not showing existing worklogs | Always show what's already logged to prevent duplicates |
| Using plain string for comment | REST API v3 requires ADF format — use the JSON template above |
| Not converting time to seconds | REST API requires `timeSpentSeconds` as integer |
| Logging without asking | Always confirm with user first |
| Ignoring duplicate detection | Warn if same-day entry already exists |
| Skipping the updated totals | Always show how the entry changed the time tracking |
| Generic worklog description | Auto-generate from git commits since last worklog — be specific about what was done |
| Omitting PR URL from worklog | Always append `PR: <URL>` to description if a PR was created in this session |
| Forgetting to source ~/.jira-config | Always source before curl calls to get credentials |
| Telling user to run jira-setup manually | Auto-run the setup skill inline — user should never see "run /android:jira-setup" |
| Using UTC or omitting `started` field | ALWAYS include `started` with local timestamp (`date +"%Y-%m-%dT%H:%M:%S.000%z"`) — never UTC, never omit |
