---
name: analyse
description: Analyse a Jira ticket and produce an implementation plan (Step 2 standalone)
argument-hint: "[optional: TICKET-ID]"
---

# Analyse

Analyse a Jira ticket and produce an implementation plan without starting the full task lifecycle. This runs Steps 1-2 of the pipeline only.

**HARD RULE — No MCP Tools:** Never use `mcp__claude_ai_Atlassian__*` or any MCP Atlassian tools. All Jira operations MUST use ACLI (`acli`), curl + REST API, or `gh` CLI.

## Step 1: Get Ticket ID

If an argument was provided (e.g., `/android:analyse MA-3523`), use it as the ticket ID.

Otherwise ask:
> What Jira ticket do you want to analyse? (e.g., MA-3523)

**Validate format:** Ticket ID should match pattern `[A-Z]+-[0-9]+`.

## Step 2: Verify CLI Setup

Verify ACLI is authenticated:
```bash
acli jira auth status 2>&1 && test -f ~/.jira-config && source ~/.jira-config && test -n "$JIRA_TOKEN"
```
If any check fails: auto-run the `android:jira-setup` skill inline.

## Step 3: Gather Context (parallel)

Run these in parallel:

**Agent A:** Use the `android:jira-fetching` skill to fetch ticket details.

**Agent B:** If Figma links found, use the `android:design-reference` skill.

**Agent C:** Read `.claude/cache/` files relevant to the Jira ticket keywords. If cache doesn't exist or is stale, suggest `/android:refresh-cache`.

## Step 4: Analyse & Plan

**REQUIRED:** Use the `android:analyse-plan` skill with all gathered context.

This will:
- Load relevant cache files and procedures dynamically
- Run a runtime codebase scan
- Perform deduplication check (find similar existing code, explain, ask user)
- Produce an implementation plan with affected layers and files
- Ask user for confirmation

## Step 5: Present Result

After the plan is confirmed:

> **Analysis Complete: [TICKET-ID] — [Summary]**
>
> | Layer | Action | Files |
> |-------|--------|-------|
> | Domain | [CREATE/MODIFY/SKIP] | [count] |
> | Data | [CREATE/MODIFY/SKIP] | [count] |
> | Core | [CREATE/MODIFY/SKIP] | [count] |
> | Service | [CREATE/MODIFY/SKIP] | [count] |
> | ViewModel | [CREATE/MODIFY/SKIP] | [count] |
> | UI | [CREATE/MODIFY/SKIP] | [count] |
>
> To implement: run `/android:start-task [TICKET-ID]`
> The start-task command will use this analysis as input.
