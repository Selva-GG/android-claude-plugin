---
name: jira-transitioning
description: Use when moving a Jira ticket between statuses — To Do to In Progress, In Progress to In Review, In Review to Done, or any other workflow transition including those requiring resolution fields
user-invocable: false
---

# Transitioning Jira Status

## Overview

Move a Jira ticket to a new workflow status. ACLI handles transition discovery internally — no need to fetch transition IDs manually. Always confirms with the user before executing.

### Prerequisites
Check if ACLI is available and `~/.jira-config` exists:
```bash
which acli && test -f ~/.jira-config
```
If either check fails: **REQUIRED:** Use the `android:jira-setup` skill.

## Step 1: Check Current Status

Fetch the current status before attempting a transition:

```bash
acli jira workitem view <TICKET-ID> --json --fields "status"
```

Read `fields.status.name` from the JSON output.

If the issue is already in the target status:
> [TICKET-ID] is already in "[Target Status]". No transition needed.

Skip remaining steps.

## Step 2: Confirm with User

Present the full transition:

> Move [TICKET-ID] from "[Current Status]" to "[Target Status]"?
> (yes/no)

**Never transition without user confirmation.**

## Step 3: Execute Transition

```bash
acli jira workitem transition --key <TICKET-ID> --status "TARGET_STATUS"
```

**Status name matching:**
ACLI performs its own matching, but if the user provides a shorthand:
- "in progress" matches "In Progress", "Start Progress", "Begin Work"
- "in review" matches "In Review", "Code Review", "Peer Review"
- "done" matches "Done", "Closed", "Resolved", "Complete"

Pass the user's target status as-is to ACLI — it handles fuzzy matching.

**If ACLI transition fails with unrecognized status:**

Try to list available transitions:
```bash
source ~/.jira-config
curl -s -X GET \
  -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
  -H "Content-Type: application/json" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>/transitions" | jq '.transitions[] | {id: .id, name: .name, to: .to.name}'
```

Present available options:
> Cannot transition [TICKET-ID] from "[Current Status]" to "[Target Status]".
>
> Available transitions from "[Current Status]":
> - [transition.name] → [transition.to.name]
> - [transition.name] → [transition.to.name]
> - ...
>
> Which transition would you like to use?

Let the user pick from the available options, then retry with the correct status name.

**If transition requires resolution field:**

Some transitions (e.g., "Done") require a resolution. If ACLI reports a missing required field:

> This transition requires a resolution. Common options:
> - Done
> - Won't Do
> - Duplicate
> - Cannot Reproduce
>
> Which resolution? (or type a custom one)

Use curl as fallback to execute transitions with additional fields:
```bash
source ~/.jira-config
curl -s -X POST \
  -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
  -H "Content-Type: application/json" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>/transitions" \
  -d '{"transition": {"id": "TRANSITION_ID"}, "fields": {"resolution": {"name": "Done"}}}'
```

**General error handling:**
- 400 → "Transition failed. [Parse error message]. The ticket may require fields that weren't provided."
- 403 → "No permission to transition this ticket. Check your Jira project role."
- 404 → "Ticket not found. It may have been deleted or moved."
- 409 → "Conflict — the ticket's status may have been changed by someone else. Let me re-fetch."

**On 409 conflict:** Re-fetch the issue, check current status, and retry.

## Step 4: Verify Transition

After successful transition, confirm the status changed:

```bash
acli jira workitem view <TICKET-ID> --json --fields "status"
```

Read `fields.status.name` and confirm:

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
| Hardcoding transition IDs | ACLI handles discovery — pass status name, not ID |
| Not checking current status first | Avoid unnecessary API calls and confusing errors |
| Ignoring required fields | If transition fails, check if resolution or other fields are needed |
| Not verifying after transition | Re-fetch to confirm the status actually changed |
| Assuming transition names are universal | Names vary per project — let ACLI match or fall back to curl |
| Transitioning without user consent | Always confirm, even if the command requested it |
