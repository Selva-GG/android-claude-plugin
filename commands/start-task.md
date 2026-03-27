---
name: start-task
description: Full task lifecycle — 6-step orchestrator pipeline with parallel agent teams for gathering, validation, and finalization
argument-hint: "[optional: TICKET-ID]"
---

# Start Task

You are beginning work on a Jira ticket. Follow the 6-step orchestrator pipeline. Every state change requires user confirmation. Do not skip steps.

**HARD RULE — User Prompts:** For ALL questions that require user input (confirmations, choices, yes/no), use the `AskUserQuestion` tool. Do NOT embed questions as inline text in responses — they get lost in output. Fallback to inline text ONLY if `AskUserQuestion` is unavailable.

**HARD RULE — No MCP Tools:** Never use `mcp__claude_ai_Atlassian__*` or any MCP Atlassian tools. All Jira operations MUST use ACLI (`acli`), curl + REST API, or `gh` CLI. If ACLI is not authenticated, auto-run the `android:jira-setup` skill inline — do NOT fall back to MCP.

---

## Step 0: Get Ticket ID

If an argument was provided (e.g., `/android:start-task MA-3523`), use it as the ticket ID.

Otherwise ask:
> What Jira ticket are you working on? (e.g., MA-3523)

**Validate format:** Ticket ID should match pattern `[A-Z]+-[0-9]+`. If it doesn't match, ask again.

## Step 0.5: Verify CLI Setup

Verify ACLI is authenticated:
```bash
acli jira auth status 2>&1 && test -f ~/.jira-config && source ~/.jira-config && test -n "$JIRA_TOKEN"
```
If any check fails: **REQUIRED:** Auto-run the `android:jira-setup` skill inline.

Check GitHub CLI:
```bash
which gh && gh auth status
```
If either check fails: **REQUIRED:** Use the `android:gh-setup` skill.

---

## Step 1: Gather Context (parallel agents)

Run these tasks in parallel:

### Agent A: Fetch Jira Ticket

**REQUIRED:** Use the `android:jira-fetching` skill.

This will:
- Retrieve all task details (summary, description, status, estimate, comments, links)
- Parse ADF description into readable markdown
- Validate whether requirements are sufficient
- Detect Figma links and image attachments
- Detect subtasks

**Do not proceed until requirements are confirmed as clear.**

### Agent B: Fetch Designs (if applicable)

If Agent A detected Figma links or design references:
**REQUIRED:** Use the `android:design-reference` skill to open and capture designs.

If no designs found: skip.

### Agent C: Read Cache

Check if `.claude/cache/` exists and is not stale:
```bash
if [ -f ".claude/cache/last-scan-timestamp" ]; then
  cache_date=$(cat .claude/cache/last-scan-timestamp)
  echo "Cache last refreshed: $cache_date"
else
  echo "No cache found"
fi
```

If cache doesn't exist or is >7 days old:
**Auto-run** the `android:refresh-cache` skill inline. Do not ask — just run it. If it fails, proceed without cache and use runtime scan only.

If cache exists, read ONLY the cache files relevant to the Jira ticket keywords. Do not load all 7 files — match keywords from the ticket to determine which cache files are useful:
- "repository", "data", "storage" → `repository-map.md`, `model-map.md`
- "service", "manager", "logic" → `service-map.md`, `di-graph.md`
- "screen", "UI", "component" → `component-catalog.md`, `feature-index.md`
- "navigation", "route" → `route-map.md`
- "new feature" (broad scope) → load all cache files

### Check for Subtasks

After fetching the ticket, check if it has subtasks:
```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>?fields=subtasks" \
  | python3 -c "import json,sys; subtasks=json.load(sys.stdin).get('fields',{}).get('subtasks',[]); [print(f\"{s['key']}: {s['fields']['summary']} ({s['fields']['status']['name']})\") for s in subtasks]"
```

**If subtasks found:**
> This ticket has N subtasks:
> 1. TICKET-1: Summary (Status)
> 2. TICKET-2: Summary (Status)
>
> Gather and plan ONCE for the parent, then implement each subtask separately? (yes / no — just work on parent)

If yes: gather/plan covers all subtasks. Each subtask runs Steps 3-6 independently (own branch, own PR).

---

## Step 2: Analyse & Plan

**REQUIRED:** Use the `android:analyse-plan` skill with all gathered context.

This will:
- Load relevant cache files and procedures dynamically
- Run a runtime codebase scan
- Perform deduplication check (FIND → UNDERSTAND → RECOMMEND → ASK)
- Produce an implementation plan with affected layers and files
- Ask user for confirmation

**Do not proceed until the plan is confirmed by the user.**

---

## Step 3: Setup

### 3a: Check Assignment

Check the assignee field from the Jira ticket data (fetched in Step 1):

**If unassigned:**
> This ticket is unassigned. Assign it to yourself? (yes/no)

If yes:
```bash
acli jira workitem assign --key <TICKET-ID> --assignee @me
```

**If assigned to someone else:**
> This ticket is assigned to [Name]. Continue anyway? (yes/no)

**If assigned to the user:** Proceed silently.

### 3b: Review Time Estimate

**If ticket type is "Subtask":**
- If no estimate exists: skip silently
- If estimate exists: display but don't prompt to change

**Otherwise:** Use the `android:jira-estimating` skill.

### 3c: Transition to In Progress

**REQUIRED:** Use the `android:jira-transitioning` skill.

Target status: **In Progress**

If already In Progress, inform the user and skip.

### 3d: Prepare Working Branch

Check current git state:
```bash
git status
git branch --show-current
```

**If uncommitted changes exist:**
> You have uncommitted changes on `[current-branch]`. What should we do?
> - Stash changes and switch
> - Commit changes first
> - Stay on current branch

**Check if branch exists for this ticket:**
```bash
git branch --list "*<TICKET-ID>*"
git branch -r --list "*<TICKET-ID>*"
```

If branch exists: offer to switch to it.

**Create new branch:**
- Format: `<TICKET-ID>-<kebab-case-summary>` (truncated to ~60 chars)
- **Base branch detection (in order):**
  1. If user specified a base branch in arguments: use that
  2. If subtask with parent ticket: search `git branch -r --list "*<PARENT-TICKET-ID>*"` for parent branch
  3. Search for feature branches: `git branch -r --list "*phase*"`, `*feature*`
  4. Otherwise: use latest `dev` (or `main` if no `dev`)

```bash
git fetch origin <base-branch>
git checkout -b <branch-name> origin/<base-branch>
```

### Step 3e: Verify Build Prerequisites

Before implementation, verify the project builds:
```bash
./gradlew compileDebugKotlin 2>&1 | tail -5
```

**If build fails:**
- Check for missing `google-services.json` → ask user to provide it
- Check for GitHub Packages auth (401 errors) → check `local.properties` for `gpr.user`/`gpr.key`
- Check for other dependency issues → show error, ask user to fix

**Do not proceed to implementation until the build compiles.**

---

## Step 4: Implement

**REQUIRED:** Use the `android:implement-layers` skill with the confirmed plan from Step 2.

The orchestrator follows the plan layer by layer:
- Domain → Data → Core → Service → ViewModel → UI
- Skips layers not affected
- Loads relevant procedures per layer
- Writes tests inline per layer

**During implementation:** Follow the conventions and patterns defined in `android:android-procedures`. If it defines how to structure files, name classes, write tests, or handle specific patterns — follow those rules.

When implementation is complete (user confirms "done", "finish", "ready for PR", etc.), proceed to Step 5.

---

## Step 5: Validate

Run the validation flow. This is the same logic as `/android:validate`:

1. Spawn 4 agents in parallel: test-runner, lint-analyzer, code-reviewer, spec-checker
2. Merge findings by severity (critical → warning → info)
3. Fix critical issues, then warnings
4. Re-validate until all pass (or 3+ failures → ask user)

See `commands/validate.md` for the full validation flow.

**Do not proceed until validation passes.**

---

## Step 6: Finalize

### 6a: Check Branch Freshness

```bash
git fetch origin
git log HEAD..origin/<base-branch> --oneline | wc -l
```

If behind:
> Your branch is [N commits] behind `[base-branch]`.
> - Rebase onto `[base-branch]`
> - Merge `[base-branch]` into your branch
> - Skip

If user chooses rebase/merge: execute, then **re-run Step 5 validation**.

### 6b: Show Diff Summary

```bash
git diff <base-branch>..HEAD --stat
git log <base-branch>..HEAD --oneline
```

> **Changes to include in PR:**
> - Commits: [N]
> - Files changed: [N] (+insertions, -deletions)
>
> Proceed to create PR? (yes / no)

### 6c: Create Pull Request

Ask:
> Create a Pull Request? (yes / no)

If yes:
**REQUIRED:** Use the `android:creating-pr` skill.

If `android:write-tests` was run in this session, pass coverage tables to the PR body.

### 6d: Log Work Time

**REQUIRED:** Use the `android:jira-worklogging` skill.

### 6e: Transition to In Review

**If PR was created:** Auto-transition to "In Review" using the `android:jira-transitioning` skill.

**If no PR:** Ask if user still wants to transition.

### 6f: Final Summary

> **Task Complete: [TICKET-ID] — [Summary]**
>
> | Action | Result |
> |--------|--------|
> | Branch | [branch-name] |
> | Verification | All passed (Tests, Lint, Review, Spec) |
> | PR | [URL] or Skipped |
> | Work Logged | [time] or Skipped |
> | Status | [In Review / In Progress] |
>
> **PR URL:** [clickable URL]

---

## Error Recovery

**If any step fails:**
- Present the error clearly with context
- Offer to retry or skip the step
- Track which steps were completed

**If user wants to abort:**
> Task aborted. Current state:
> - Jira status: [status — was it changed?]
> - Branch: [was a branch created?]
> - Estimate: [was it updated?]
> - Verification: [passed/failed/not run]
> - PR: [created at URL / not created]
> - Work logged: [Xh / not logged]
>
> You can resume with `/android:finish-task` later.
