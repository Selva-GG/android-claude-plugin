---
name: start-task
description: Full task lifecycle — fetches Jira details, validates requirements, updates estimate, creates branch, transitions to In Progress, implements following project procedures, then finishes with self-review, build verification, PR creation, worklog, and transition to In Review
argument-hint: "[optional: TICKET-ID]"
---

# Start Task

You are beginning work on a Jira ticket. Follow each step in order. Every state change requires user confirmation. Do not skip steps.

## Step 1: Get Jira Ticket ID

If an argument was provided (e.g., `/android:start-task MA-3353`), use it as the ticket ID.

Otherwise ask the user:
> What Jira ticket are you working on? (e.g., MA-3353)

**Validate format:** Ticket ID should match pattern `[A-Z]+-[0-9]+` (e.g., MA-3353, PROJ-42). If it doesn't match, ask again.

## Step 2: Fetch and Validate Task

**REQUIRED:** Use the `android:jira-fetching` skill.

This will:
- Fetch the Atlassian cloud ID (and cache it)
- Retrieve all task details (summary, description, status, estimate, comments, links)
- Parse ADF description into readable markdown
- Present the full task summary to the user
- Validate whether requirements are sufficient for implementation
- Ask for clarification if description is incomplete
- Detect Figma links and image attachments (design references)
- If designs found, ask user if they want to view them

**Do not proceed until requirements are confirmed as clear.**

If design references were found and user wants to view them:
**REQUIRED:** Use the `android:design-reference` skill to open and capture the designs. This provides visual context for implementation.

## Step 3: Check Assignment

After fetching the task, check the assignee field:

**If unassigned:**
> This ticket is unassigned. Assign it to yourself? (yes/no)

If yes, call `mcp__claude_ai_Atlassian__editJiraIssue` to set the assignee. Use `mcp__claude_ai_Atlassian__lookupJiraAccountId` to find the user's account ID first.

**If assigned to someone else:**
> This ticket is assigned to [Name]. Continue anyway? (yes/no)

Warn but don't block — the user may be pairing or taking over.

**If assigned to the user:** Proceed silently.

## Step 4: Review Time Estimate

**REQUIRED:** Use the `android:jira-estimating` skill.

This will:
- Show current estimate and story points
- Compare estimate with any time already spent
- Ask user if the estimate needs updating
- Update via Jira API if requested

## Step 5: Transition to In Progress

**REQUIRED:** Use the `android:jira-transitioning` skill.

Target status: **In Progress**

If already In Progress, inform the user and skip.

## Step 6: Prepare Working Branch

### Check current git state
```bash
git status
git branch --show-current
```

**If there are uncommitted changes on the current branch:**
> You have uncommitted changes on `[current-branch]`. What should we do?
> - Stash changes and switch
> - Commit changes first
> - Stay on current branch (skip branch creation)

### Check if a branch already exists for this ticket
```bash
git branch --list "*<TICKET-ID>*"
git branch -r --list "*<TICKET-ID>*"
```

**If a branch exists:**
> A branch already exists for this ticket: `[branch-name]`
> - Switch to it
> - Create a new branch anyway
> - Stay on current branch

If switching, checkout the existing branch.

### Create a new branch

If no existing branch (or user wants a new one):

Generate branch name from ticket:
- Format: `<TICKET-ID>-<kebab-case-summary>`
- Lowercase the ticket ID portion of the summary
- Replace spaces and special chars with hyphens
- Truncate to ~60 characters total
- Example: `MA-3353-add-staging-build-type`

> Create branch `[generated-name]`? (yes / use different name / stay on current branch)

If yes:
```bash
git checkout -b <branch-name>
```

**Base branch:** Create from the latest main/dev branch:
```bash
git fetch origin <base-branch>
git checkout -b <branch-name> origin/<base-branch>
```

## Step 7: Load Project Procedures

**REQUIRED:** Use the `android:android-procedures` skill.

This loads the Android project conventions, architecture rules, hard rules, and sub-skill guides bundled with this plugin.

## Step 8: Hand Off to Implementation

Present the complete task context:

> **Task ready for implementation:**
>
> | | |
> |---|---|
> | **Ticket** | [TICKET-ID]: [Summary] |
> | **Type** | [Story / Bug / Task] |
> | **Priority** | [Priority] |
> | **Status** | In Progress |
> | **Estimate** | [Xh / X story points] |
> | **Branch** | `[branch-name]` |
> | **Procedures** | Android (from plugin) |
>
> **Key requirements:**
> - [Bullet 1 from acceptance criteria]
> - [Bullet 2]
> - [Bullet 3]
>
> What would you like to work on first?

**During implementation:** Follow the conventions and patterns defined in `android:android-procedures`. If it defines how to structure files, name classes, write tests, or handle specific patterns — follow those rules.

---

## Phase 2: Finishing the Task

After implementation is complete (user confirms they're done or says "finish", "done", "ready for PR", etc.), continue with the finishing flow automatically.

## Step 9: Self-Review Checklist

Before running automated verification, prompt a self-review using the checklist from `android:android-procedures` (the `review-checklist` sub-skill).

> **Pre-verification checklist:**
> - [ ] All interfaces use `I` prefix?
> - [ ] No `!!` operators anywhere in changed files?
> - [ ] Compose previews use `@PreviewTheme` + `MeAppTheme { }`?
> - [ ] No hardcoded colors, typography, or spacing?
> - [ ] `AppLog` used instead of `Log`/`Timber`?
> - [ ] `super.handleIntent(intent)` called first in ViewModel?
> - [ ] All API methods are `suspend`?
> - [ ] Removed debug code, TODOs, and commented-out code?
> - [ ] No secrets or credentials in the diff?
>
> Proceed with verification? (yes / let me fix something first)

If user wants to fix something, wait for them to return.

## Step 10: Verify Build + Tests + Lint

**REQUIRED:** Use the `android:verifying-build` skill.

This will:
- Discover build commands from CLAUDE.md or auto-detect project type
- Run build, tests, and lint sequentially
- Report results with durations
- Loop until all pass

**Do not proceed until all checks pass.**

## Step 11: Check Branch Freshness

```bash
git fetch origin
```

### Behind base branch?
```bash
git log HEAD..origin/<base-branch> --oneline | wc -l
```

If behind:
> Your branch is [N commits] behind `[base-branch]`.
>
> - Rebase onto `[base-branch]` (recommended — cleaner history)
> - Merge `[base-branch]` into your branch
> - Skip (may cause conflicts in PR)

If user chooses rebase/merge:
1. Execute the rebase/merge
2. **Re-run verification** (Step 10) — the merge may introduce failures
3. Only proceed once verification passes again

### Stale branch?
```bash
git log -1 --format="%ci" HEAD
```

If last commit is more than 7 days old:
> ⚠ Last commit on this branch was [N days] ago. The codebase may have changed significantly.
> Consider rebasing before creating a PR.

## Step 12: Show Diff Summary

Before PR creation, show what will be included:

```bash
git diff <base-branch>..HEAD --stat
git log <base-branch>..HEAD --oneline
```

> **Changes to include in PR:**
> - **Commits:** [N commits]
> - **Files changed:** [N files] (+[insertions], -[deletions])
>
> [File list from --stat]
>
> [If sensitive files changed:]
> ⚠ **Sensitive files modified:** [build.gradle, CI config, migration, etc.]
>
> Proceed to create PR? (yes / no — let me review)

## Step 13: Create Pull Request

Ask the user:
> Create a Pull Request? (yes / no)

If yes:
**REQUIRED:** Use the `android:creating-pr` skill.

This will:
- Check for uncommitted changes
- Determine and confirm base branch
- Push branch if needed
- Confirm PR title, reviewers, labels, draft status
- Create PR with Jira-prefixed title, summary, and test plan
- Return the PR URL

If no: skip to Step 15.

## Step 14: Link PR to Jira

**Only if a PR was created in Step 13.**

Offer to add the PR link to the Jira ticket:
> Link this PR to [TICKET-ID] in Jira? (yes/no)

If yes, add a comment to the Jira issue with the PR URL:
Call `mcp__claude_ai_Atlassian__addCommentToJiraIssue` with the PR URL.

Check first if a link already exists (GitHub-Jira integration may auto-link):
Call `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks` — if PR URL is already linked, skip.

## Step 15: Log Work Time

**REQUIRED:** Use the `android:jira-worklogging` skill.

This will:
- Show existing worklogs
- Ask if user wants to log time
- Accept time input with optional description
- Submit worklog to Jira
- Show updated totals

## Step 16: Transition to In Review

**Only if a PR was created in Step 13.**

**REQUIRED:** Use the `android:jira-transitioning` skill.

Target status: **In Review**

If no PR was created, ask:
> No PR was created. Still move ticket to "In Review"? (yes / no — keep as In Progress)

## Step 17: Final Summary

Present the completion summary with all actions taken:

> **Task Complete: [TICKET-ID] — [Summary]**
>
> | Action | Result |
> |--------|--------|
> | Verification | All passed (Build, Tests, Lint) |
> | PR | [URL] or Skipped |
> | Jira Link | Linked / Skipped / Already linked |
> | Work Logged | [time] or Skipped |
> | Status | [In Review / In Progress / unchanged] |
>
> **PR URL:** [clickable URL]

---

## Error Recovery

**If any step fails:**
- Present the error clearly with context
- Offer to retry or skip the step
- Track which steps were completed

**If user wants to abort mid-flow:**
> Task aborted. Current state:
> - Jira status: [status — was it changed?]
> - Branch: [was a branch created?]
> - Estimate: [was it updated?]
> - Verification: [passed/failed/not run]
> - PR: [created at URL / not created]
> - Work logged: [Xh / not logged]
>
> You can resume with `/android:finish-task` later.

**If verification keeps failing:**
After 3 failed verification loops:
> Verification has failed [N times]. Options:
> - Continue fixing
> - Skip verification and create PR as draft (mark as WIP)
> - Abort and come back later

Only allow draft PR on repeated failures — never skip verification for a ready-for-review PR.
