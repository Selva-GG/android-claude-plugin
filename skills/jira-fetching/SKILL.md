---
name: jira-fetching
description: Use when fetching Jira task details, reading ticket descriptions, checking acceptance criteria, validating task readiness, or needing context about a Jira issue before implementation
---

# Fetching Jira Task Details

## Overview

Retrieve a Jira issue's full details, parse its description, and validate whether it contains enough information for implementation.

## Step 1: Get Cloud ID

Call `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` to get the `cloudId`.

**Error handling:**
- No resources returned → "No Atlassian cloud found. Check your Atlassian MCP connection in `~/.claude/mcp.json`."
- Multiple clouds → present list with names, ask user which one
- Cache the `cloudId` for all subsequent Atlassian calls in this session — do not re-fetch

## Step 2: Fetch the Issue

Call `mcp__claude_ai_Atlassian__getJiraIssue` with:
- `cloudId`: from Step 1
- `issueIdOrKey`: the ticket ID (e.g., `MA-3353`)

**Error handling:**
- 404 → "Ticket [ID] not found. Check the ID and try again."
- 403 → "No permission to access [ID]. You may need to be added to the project."
- 401 → "Authentication failed. Your Atlassian MCP token may have expired."
- Network/timeout → "Failed to reach Jira. Check your internet connection."
- Empty response → "Jira returned an empty response. The ticket may have been deleted."

## Step 3: Extract and Present

Parse the response and display a comprehensive summary:

```
## [TICKET-ID]: [Summary]

**Status:** [Current status]          **Type:** [Story / Bug / Task / Sub-task]
**Assignee:** [Name or Unassigned]    **Reporter:** [Name]
**Priority:** [Priority level]        **Sprint:** [Sprint name or None]
**Labels:** [label1, label2 or None]  **Components:** [comp1, comp2 or None]
**Fix Version:** [version or None]    **Epic:** [Epic key - Epic name or None]

**Time Tracking:**
- Original estimate: [Xh or Not set]
- Time spent: [Xh or None]
- Remaining: [Xh or N/A]

### Description
[Full description rendered as markdown — see ADF parsing below]

### Acceptance Criteria
[Extracted from description if present, or "None specified"]

### Subtasks
- [SUBTASK-KEY] [Summary] — [Status]
- [or "None"]

### Linked Issues
- blocks [OTHER-KEY]: [Summary]
- is blocked by [OTHER-KEY]: [Summary]
- relates to [OTHER-KEY]: [Summary]
- [or "None"]

### Recent Comments (last 3)
> **[Author]** ([date]):
> [Comment body]
[or "No comments"]

### Linked Pull Requests
- [PR title] — [status] ([repo])
[or "None"]

### Design References
- 🎨 **Figma:** [Figma link title](url)
- 📎 **Attachments:** [screenshot.png], [mockup.jpg]
[or "No design references found"]
```

## Field Mapping Reference

| Display | Jira API Path | Notes |
|---------|--------------|-------|
| Summary | `fields.summary` | Title |
| Status | `fields.status.name` | Current workflow state |
| Type | `fields.issuetype.name` | Story, Bug, Task, Sub-task |
| Assignee | `fields.assignee.displayName` | May be null → "Unassigned" |
| Reporter | `fields.reporter.displayName` | May be null |
| Priority | `fields.priority.name` | |
| Sprint | `fields.sprint.name` or `fields.customfield_10020[0].name` | Varies by Jira config |
| Labels | `fields.labels[]` | Array of strings |
| Components | `fields.components[].name` | Array of objects |
| Fix Version | `fields.fixVersions[].name` | Array of objects |
| Epic | `fields.parent.key` + `fields.parent.fields.summary` | For subtasks/stories in epics |
| Estimate | `fields.timeoriginalestimate` | Seconds → hours (÷ 3600) |
| Time Spent | `fields.timetracking.timeSpent` | Human-readable string |
| Remaining | `fields.timetracking.remainingEstimate` | Human-readable string |
| Description | `fields.description` | ADF or plain text — see below |
| Subtasks | `fields.subtasks[]` | Array of {key, fields.summary, fields.status.name} |
| Links | `fields.issuelinks[]` | See link parsing below |
| Comments | `fields.comment.comments[]` | Array, take last 3 |
| Attachments | `fields.attachment[]` | Array of {filename, mimeType, content (URL), thumbnail} |

## Parsing Atlassian Document Format (ADF)

Jira Cloud returns descriptions in ADF (a JSON structure). Convert to readable markdown:

| ADF Node Type | Render As |
|---------------|-----------|
| `paragraph` | Plain text with newline |
| `heading` (level 1-6) | `#` through `######` |
| `bulletList` / `listItem` | `- item` |
| `orderedList` / `listItem` | `1. item` |
| `codeBlock` | ` ```language\ncode\n``` ` |
| `blockquote` | `> text` |
| `table` / `tableRow` / `tableCell` | Markdown table |
| `mediaSingle` / `media` | `[attachment: filename]` |
| `mention` | `@displayName` |
| `emoji` | Render the shortName |
| `hardBreak` | Newline |
| `rule` | `---` |
| `text` with marks | `**bold**`, `*italic*`, `` `code` ``, `~~strikethrough~~`, `[link text](url)` |

**If ADF parsing fails or description is plain text:** Display as-is.

## Parsing Issue Links

```
fields.issuelinks[] → each has:
  - type.name: "Blocks", "Relates", "Clones", etc.
  - type.inward: "is blocked by"
  - type.outward: "blocks"
  - inwardIssue or outwardIssue: { key, fields.summary, fields.status.name }
```

Display format:
- If `outwardIssue` exists: `[type.outward] [outwardIssue.key]: [summary] — [status]`
- If `inwardIssue` exists: `[type.inward] [inwardIssue.key]: [summary] — [status]`

## Fetching Comments

Comments are in `fields.comment.comments[]`. Take the last 3 (most recent).

Each comment has:
- `author.displayName` — who wrote it
- `created` — when (format as relative: "2 days ago" or date)
- `body` — ADF format, parse same as description

## Fetching Remote Links (PRs)

Call `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks` with:
- `cloudId`, `issueIdOrKey`

This returns linked PRs, external links, etc. Display PR title, status, and repo.

**Skip if call fails** — remote links are supplementary, not critical.

## Step 4: Validate Requirements

Assess the description for implementation readiness.

**Sufficient — ALL of:**
- Clear definition of what to build, fix, or change
- Expected behavior or acceptance criteria (explicit or inferable)
- Enough context to start: which parts of the system are affected
- For bugs: reproduction steps or clear description of the problem

**Insufficient — ANY of:**
- Description is empty, just a title, or only "See attached"
- No acceptance criteria and expected behavior is ambiguous
- References external documents (Figma, Confluence, Slack) without inline context
- Bug report with no reproduction steps and unclear expected behavior
- Technical task with no technical details (which API, which screen, which data)
- Contradictory requirements

**If insufficient, be specific about gaps:**

> The task description needs more detail before implementation:
> - **Missing:** [e.g., "No acceptance criteria — what does 'done' look like?"]
> - **Unclear:** [e.g., "Description says 'update the UI' but doesn't specify which screen"]
> - **Missing context:** [e.g., "References Figma design but no link or description of changes"]
> - **Ambiguous:** [e.g., "Should this apply to all users or only premium?"]
>
> Please provide more context, or should we add details to the Jira description?

**Do not proceed until requirements are clear.** Ask follow-up questions if the user's initial response is still ambiguous.

**If sufficient:**
> Requirements look clear. Ready to proceed.

## Detecting Design References

Scan three sources for design assets:

### 1. Figma Links in Description

When parsing the ADF description, detect URLs matching:
- `https://www.figma.com/*`
- `https://figma.com/*`
- `https://*.figma.com/*`

Extract the link text and URL. Present each as a design reference.

### 2. Image Attachments

From `fields.attachment[]`, filter for image types:
- `mimeType` starting with `image/` (png, jpg, gif, svg, webp)
- Common screenshot filenames: `*screenshot*`, `*mockup*`, `*design*`, `*screen*`

Present attachment name and thumbnail URL.

### 3. Remote Links

From `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks`, filter for:
- URLs containing `figma.com`
- URLs containing `zeplin.io`, `sketch.com`, `invisionapp.com`
- Any link with title containing "design", "mockup", "figma"

### Surfacing Design References

If ANY design references are found:

> **Design references found for this ticket:**
> - Figma: [link title](url)
> - Attachments: [screenshot.png] (image), [mockup.jpg] (image)
>
> Would you like to view the designs? (yes / no — I'll check them manually)

If user says yes:
**REQUIRED:** Use the `android:design-reference` skill to open and capture the designs.

If NO design references found and the task involves UI changes:
> ⚠ This task appears to involve UI changes but no design references were found.
> Is there a Figma link or design spec? (paste URL / no design needed / will check later)

## Custom Fields

Jira projects may have custom fields (e.g., story points, team, environment). Handle gracefully:
- Known custom fields (story points): extract if present
- Unknown custom fields: skip silently — do not error on unexpected data
- If a field is null or missing: display "Not set" or omit the row

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Re-fetching cloudId every time | Cache it after first call in session |
| Crashing on null fields | Every field can be null — always use safe access |
| Displaying raw ADF JSON | Parse ADF to markdown for readability |
| Ignoring comments | Comments often contain critical context and decisions |
| Skipping linked issues | Blockers reveal dependencies that affect implementation |
| Proceeding with ambiguous requirements | Always validate — unclear requirements waste more time than asking |
