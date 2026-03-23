---
name: finish-task
description: Complete task — run 4 parallel validation agents, fix issues, create PR, log work, transition to In Review (Steps 5-6 standalone)
---

# Finish Task

You are completing work on a Jira ticket. This runs Steps 5-6 (validate + finalize) of the pipeline. Use this when you started work manually without `/android:start-task`.

**HARD RULE — No MCP Tools:** Never use `mcp__claude_ai_Atlassian__*` or any MCP Atlassian tools. All Jira operations MUST use ACLI (`acli`), curl + REST API, or `gh` CLI. If ACLI is not authenticated, auto-run the `android:jira-setup` skill inline — do NOT fall back to MCP.

## Prerequisites

Check GitHub CLI:
```bash
which gh && gh auth status
```
If either check fails: **REQUIRED:** Use the `android:gh-setup` skill.

## Step 1: Identify Jira Ticket

Extract ticket ID from current branch:
```bash
git branch --show-current
```
Parse: `([A-Z]+-[0-9]+)` — match the first occurrence.

**If unable to extract:**
> Could not detect Jira ticket from branch name `[branch]`. What is the ticket ID?

## Step 2: Load Procedures

**REQUIRED:** Use the `android:android-procedures` skill to load project conventions.

These provide:
- Project-specific review checklist
- Build/test/lint commands
- Conventions to verify against

## Step 3: Self-Review Checklist

Before running automated validation, prompt a self-review using the loaded procedures. Use the `shared/review-checklist.md` from procedures:

> **Pre-verification checklist:**
> [Project-specific items from review-checklist.md]
> - [ ] Removed debug code, TODOs, and commented-out code?
> - [ ] No secrets or credentials in the diff?
>
> Proceed with validation? (yes / let me fix something first)

If user wants to fix something, wait for them to return.

## Step 4: Validate (parallel agents)

Run the validation flow — same logic as `/android:validate`:

1. Spawn 4 agents in parallel: test-runner, lint-analyzer, code-reviewer, spec-checker
2. Merge findings by severity (critical → warning → info)
3. Fix critical issues, then warnings
4. Re-validate until all pass (or 3+ failures → show all issues, let user decide)

See `commands/validate.md` for the full validation flow.

**Do not proceed until validation passes.**

## Step 5: Check Branch Freshness

```bash
git fetch origin
```

### Behind base branch?
```bash
git log HEAD..origin/<base-branch> --oneline | wc -l
```

If behind:
> Your branch is [N commits] behind `[base-branch]`.
> - Rebase onto `[base-branch]` (recommended)
> - Merge `[base-branch]` into your branch
> - Skip

If user chooses rebase/merge:
1. Execute the rebase/merge
2. **Re-run Step 4 validation** — the merge may introduce failures
3. Only proceed once validation passes again

### Stale branch?
```bash
git log -1 --format="%ci" HEAD
```

If last commit is >7 days old:
> Last commit on this branch was [N days] ago. Consider rebasing before PR.

## Step 6: Show Diff Summary

```bash
git diff <base-branch>..HEAD --stat
git log <base-branch>..HEAD --oneline
```

> **Changes to include in PR:**
> - **Commits:** [N]
> - **Files changed:** [N] (+insertions, -deletions)
>
> [File list from --stat]
>
> Proceed to create PR? (yes / no — let me review)

## Step 7: Create Pull Request

Ask:
> Create a Pull Request? (yes / no)

If yes:

**Coverage data handoff:** If `android:write-tests` was run in this session, pass coverage tables to `android:creating-pr`.

**REQUIRED:** Use the `android:creating-pr` skill.

If no: skip to Step 8.

## Step 8: Log Work Time

**REQUIRED:** Use the `android:jira-worklogging` skill.

## Step 9: Transition to In Review

**If PR was created:** Auto-transition using `android:jira-transitioning` skill. Target: **In Review**.

**If no PR:**
> No PR was created. Still move ticket to "In Review"? (yes / no — keep as In Progress)

## Step 10: Final Summary

> **Task Complete: [TICKET-ID] — [Summary]**
>
> | Action | Result |
> |--------|--------|
> | Verification | All passed (Tests, Lint, Review, Spec) |
> | PR | [URL] or Skipped |
> | Work Logged | [time] or Skipped |
> | Status | [In Review / In Progress / unchanged] |
>
> **PR URL:** [clickable URL]

## Error Recovery

**If any step fails:**
- Present the error clearly with context
- Offer to retry or skip the step
- Track which steps were completed

**If user wants to abort:**
> Task completion aborted. Current state:
> - Verification: [passed/failed/not run]
> - PR: [created at URL / not created]
> - Work logged: [Xh / not logged]
> - Jira status: [current status]
>
> You can resume with `/android:finish-task` later.

**If verification keeps failing (3+ times):**
Show ALL remaining findings from all 4 agents. Let user decide:
- Fix specific issues
- Skip remaining warnings and proceed
- Abort and come back later

**Never auto-create a draft PR.** Always let the user decide.
