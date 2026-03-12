---
name: jira-transitioning
description: Use when moving a Jira ticket between statuses — To Do to In Progress, In Progress to In Review, In Review to Done, or any other workflow transition including those requiring resolution fields
user-invocable: false
---

# Transitioning Jira Status

## Overview

Move a Jira ticket to a new workflow status. Always discovers transition IDs dynamically — never hardcode them. Handles transitions that require additional fields (e.g., resolution).

## Step 1: Get Cloud ID

Call `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` to get the `cloudId`.

Skip if already cached from a previous skill call in this session.

## Step 2: Check Current Status

Before attempting a transition, confirm the current status by reading `fields.status.name` from the issue data.

If the issue is already in the target status:
> [TICKET-ID] is already in "[Target Status]". No transition needed.

Skip remaining steps.

## Step 3: Discover Available Transitions

Call `mcp__claude_ai_Atlassian__getTransitionsForJiraIssue` with:
- `cloudId`: from Step 1
- `issueIdOrKey`: the ticket ID

This returns the list of valid transitions from the current status.

**CRITICAL:** Never hardcode transition IDs. They vary per Jira project, workflow configuration, and current status. Always discover them dynamically.

## Step 4: Match the Target Status

From the transitions list, find the one matching the target status:

**Matching strategy:**
1. Exact match on `transition.name` (case-insensitive)
2. If no exact match, try `transition.to.name` (the destination status name)
3. If still no match, try partial/fuzzy matching:
   - "in progress" matches "In Progress", "Start Progress", "Begin Work"
   - "in review" matches "In Review", "Code Review", "Peer Review"
   - "done" matches "Done", "Closed", "Resolved", "Complete"

**If no matching transition found:**
> Cannot transition [TICKET-ID] from "[Current Status]" to "[Target Status]".
>
> Available transitions from "[Current Status]":
> - [transition.name] → [transition.to.name]
> - [transition.name] → [transition.to.name]
> - ...
>
> Which transition would you like to use?

Let the user pick from the available options.

## Step 5: Check for Required Fields

Some transitions require additional fields to be set (e.g., "Done" often requires a `resolution`).

The transition response includes `fields` that indicate required inputs:
- `transition.fields.resolution` → needs a resolution value
- `transition.fields.comment` → may need a comment
- Other screen fields

**If resolution is required:**
> This transition requires a resolution. Common options:
> - Done
> - Won't Do
> - Duplicate
> - Cannot Reproduce
>
> Which resolution? (or type a custom one)

**If other fields are required:**
> This transition requires: [field names]
> Please provide values for each.

## Step 6: Confirm with User

Present the full transition:

> Move [TICKET-ID] from "[Current Status]" to "[Target Status]"?
> [If resolution: Resolution: [value]]
> (yes/no)

**Never transition without user confirmation.**

## Step 7: Execute Transition

Call `mcp__claude_ai_Atlassian__transitionJiraIssue` with:
- `cloudId`: from Step 1
- `issueIdOrKey`: the ticket ID
- `transitionId`: the `transition.id` from Step 4
- Additional fields if required (e.g., `resolution`)

**Error handling:**
- 400 → "Transition failed. [Parse error message]. The ticket may require fields that weren't provided."
- 403 → "No permission to transition this ticket. Check your Jira project role."
- 404 → "Ticket not found. It may have been deleted or moved."
- 409 → "Conflict — the ticket's status may have been changed by someone else. Let me re-fetch."
- 422 → "Validation error. A required field may be missing or invalid."

**On 409 conflict:** Re-fetch the issue, check current status, discover transitions again, and retry.

## Step 8: Verify Transition

After successful transition, re-fetch the issue to confirm:

Call `mcp__claude_ai_Atlassian__getJiraIssue` and check `fields.status.name`.

> [TICKET-ID] successfully moved to "[New Status]".

If the status didn't change (rare edge case):
> Warning: Transition was accepted but status appears unchanged. Check Jira directly.

## Common Workflow Paths

| From | To | Typical Transition Name | Notes |
|------|----|------------------------|-------|
| To Do | In Progress | "Start Progress", "In Progress" | start-task |
| In Progress | In Review | "Submit for Review", "In Review" | finish-task |
| In Review | Done | "Done", "Resolve" | Usually needs resolution |
| Any | To Do | "Reopen", "Back to To Do" | Reopening |
| In Review | In Progress | "Request Changes", "Back to Progress" | PR feedback |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoding transition IDs | Always discover via getTransitionsForJiraIssue |
| Not checking current status first | Avoid unnecessary API calls and confusing errors |
| Ignoring required fields | Check transition.fields and prompt for values |
| Not verifying after transition | Re-fetch to confirm the status actually changed |
| Assuming transition names are universal | Names vary per project — always match dynamically |
| Transitioning without user consent | Always confirm, even if the command requested it |
