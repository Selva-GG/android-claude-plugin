---
name: finish-task
description: Complete task — self-review checklist, verify build/tests/lint, check branch freshness, create PR, log work, link to Jira, transition to In Review
---

# Finish Task

You are completing work on a Jira ticket. Follow each step in order. Every external action requires user confirmation. Do not skip steps.

### Prerequisites

Check if GitHub CLI is available and authenticated:
```bash
which gh && gh auth status
```
If either check fails: **REQUIRED:** Use the `android:gh-setup` skill.

## Step 1: Identify Jira Ticket

Extract the Jira ticket ID from the current branch name:
```bash
git branch --show-current
```

Parse the ticket ID pattern (e.g., `MA-3353` from `MA-3353-add-staging-build-type`).
Regex: `([A-Z]+-[0-9]+)` — match the first occurrence.

**If unable to extract:**
> Could not detect Jira ticket from branch name `[branch]`. What is the ticket ID?

## Step 2: Discover Project Procedures

Walk up from the current working directory to the project root. At each level, check for procedure files:

```
Check: <current-dir>/.claude/skills/procedures.md
Check: <current-dir>/.claude/skills/procedures-*.md    (e.g., procedures-android.md, procedures-ios.md)
Check: <parent-dir>/.claude/skills/procedures.md
Check: <parent-dir>/.claude/skills/procedures-*.md
... (continue up to filesystem root or home directory)
```

**If exactly 1 file found:** Load it automatically.

**If multiple files found:** **ALWAYS ask the user. Never auto-select. Never recommend. Never assume based on working directory.** Present all options equally and wait:

> **Multiple procedure files found:**
>
> | # | Path | Type |
> |---|------|------|
> | 1 | `Android/.claude/skills/procedures.md` | Android |
> | 2 | `iOS/.claude/skills/procedures.md` | iOS |
>
> Which procedures should I follow? (number, or multiple like "1,2")

**If found:** The loaded procedures will be used for:
- Project-specific review checklist (Step 2b)
- Build/test/lint commands (Step 3 — supplements verifying-build skill)
- Conventions to verify against

**If NOT found:** Use generic checklist below.

## Step 2b: Self-Review Checklist

Before running automated verification, prompt a self-review.

**If procedures.md was loaded** — use the review checklist from procedures.md. It should define project-specific checks (e.g., "All interfaces use `I` prefix", "No `!!` operators", "Compose previews use @PreviewTheme").

Present the project-specific checklist items PLUS these universal checks:

> **Pre-verification checklist:**
> [Project-specific items from procedures.md review section]
> - [ ] Removed debug code, TODOs, and commented-out code?
> - [ ] No secrets or credentials in the diff?
>
> Proceed with verification? (yes / let me fix something first)

**If no procedures.md** — use the generic checklist:

> **Pre-verification checklist** — have you:
> - [ ] Handled edge cases and error scenarios?
> - [ ] Added or updated tests for your changes?
> - [ ] Removed debug code, TODOs, and commented-out code?
> - [ ] Checked that no secrets or credentials are in the diff?
> - [ ] Followed the project's architecture and naming conventions?
>
> Proceed with verification? (yes / let me fix something first)

If user wants to fix something, wait for them to return.

## Step 3: Verify Build + Tests + Lint

**REQUIRED:** Use the `android:verifying-build` skill.

This will:
- Discover build commands from CLAUDE.md or auto-detect project type
- Run build, tests, and lint sequentially
- Report results with durations
- Loop until all pass

**Do not proceed until all checks pass.**

## Step 4: Check Branch Freshness

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
2. **Re-run verification** (Step 3) — the merge may introduce failures
3. Only proceed once verification passes again

### Stale branch?
```bash
git log -1 --format="%ci" HEAD
```

If last commit is more than 7 days old:
> ⚠ Last commit on this branch was [N days] ago. The codebase may have changed significantly.
> Consider rebasing before creating a PR.

## Step 5: Show Diff Summary

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

## Step 6: Create Pull Request

Ask the user:
> Create a Pull Request? (yes / no)

If yes:
**REQUIRED:** Use the `android:creating-pr` skill.

This will:
- Check for uncommitted changes
- Determine and confirm base branch
- Check branch freshness (may be redundant with Step 4 — skip if already done)
- Push branch if needed
- Confirm PR title, reviewers, labels, draft status
- Create PR with Jira-prefixed title, summary, and test plan
- Return the PR URL

If no: skip to Step 7.

## Step 7: Log Work Time

**REQUIRED:** Use the `android:jira-worklogging` skill.

This will:
- Show existing worklogs
- Ask if user wants to log time
- Accept time input with optional description
- Submit worklog to Jira
- Show updated totals

## Step 8: Transition to In Review

**Only if a PR was created in Step 6.**

**REQUIRED:** Use the `android:jira-transitioning` skill.

Target status: **In Review**

If no PR was created, ask:
> No PR was created. Still move ticket to "In Review"? (yes / no — keep as In Progress)

## Step 9: Final Summary

Present the completion summary with all actions taken:

> **Task Complete: [TICKET-ID] — [Summary]**
>
> | Action | Result |
> |--------|--------|
> | Verification | All passed (Build, Tests, Lint) |
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

**If user wants to abort mid-flow:**
> Task completion aborted. Current state:
> - Verification: [passed/failed/not run]
> - PR: [created at URL / not created]
> - Work logged: [Xh / not logged]
> - Jira status: [current status]
>
> You can resume with `/android:finish-task` later.

**If verification keeps failing:**
After 3 failed verification loops:
> Verification has failed [N times]. Options:
> - Continue fixing
> - Skip verification and create PR as draft (mark as WIP)
> - Abort and come back later

Only allow draft PR on repeated failures — never skip verification for a ready-for-review PR.
