---
name: creating-pr
description: Use when creating a pull request — pushing branch, formatting PR title and body, determining base branch, adding reviewers, including test plan, linking to Jira, and handling draft PRs
user-invocable: false
---

# Creating a Pull Request

## Overview

Create a well-formatted GitHub pull request with Jira context, proper title format, test plan, and optional reviewers/labels. Works with any project type.

### Prerequisites
Check if GitHub CLI is available and authenticated:
```bash
which gh && gh auth status
```
If either check fails: **REQUIRED:** Use the `android:gh-setup` skill.

## Step 1: Pre-flight Checks

### Uncommitted changes
```bash
git status
```
If uncommitted changes exist:
> There are uncommitted changes:
> - [modified files list]
>
> Commit them before creating the PR? (yes / no — I'll commit manually)

If user says yes, create a commit following git rules:
- Prefix with Jira ticket ID: `<TICKET-ID>: [description]`
- Never commit without explicit user instruction
- Never amend — always create new commits
- Never include `Co-Authored-By` trailers in commit messages

### Branch status
```bash
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
```
If no upstream tracking branch → will need to push in Step 5.

### Commits exist
```bash
git log <base-branch>..HEAD --oneline
```
If no commits:
> No commits between your branch and [base-branch]. Nothing to create a PR for.
Stop here.

## Step 2: Determine Base Branch

**Auto-detect:**
```bash
git remote show origin | grep "HEAD branch"
```

This returns the default branch (usually `main`, `dev`, or `develop`).

**Verify with user:**
> Target base branch: `[detected branch]`
> Is this correct? (yes / use [other branch])

**If detection fails** (detached HEAD, no remote):
> Could not detect the base branch. What should this PR target?

## Step 3: Check Branch Freshness

```bash
git fetch origin <base-branch>
git log HEAD..origin/<base-branch> --oneline
```

If the base branch has commits not in the current branch:
> Your branch is [N commits] behind `[base-branch]`.
>
> Options:
> - Rebase onto latest `[base-branch]` (recommended)
> - Merge `[base-branch]` into your branch
> - Create PR as-is (may have conflicts)
>
> What would you prefer?

If user chooses rebase/merge, execute and re-verify if build was already passing.

## Step 4: Gather PR Context

**Commit history:**
```bash
git log <base-branch>..HEAD --oneline
```

**Diff summary:**
```bash
git diff <base-branch>..HEAD --stat
```

**Jira context** (from the fetched issue, if available):
- Ticket ID and summary
- Description / acceptance criteria
- Subtasks completed

**Changed file analysis (go deep — read actual diffs):**

Run `git diff <base-branch>..HEAD` to read the actual changes, then:
- For each changed file, identify WHAT changed (new class, modified logic, deleted code)
- Group by concern: "Security", "UI", "Data layer", "Tests", "Config", etc.
- Produce 3-5 descriptive bullet points that explain WHY each change was made, not just what file was touched
- Flag sensitive files changed (build configs, CI, migrations, network/auth code)

Example — instead of:
> - Modified NetworkModule.kt

Write:
> - Add `CertificatePinner` to OkHttpClient for `api.weightgurus.com` to prevent MITM attacks (OWASP M3)

## Step 5: Push Branch

If not already pushed:
```bash
git push -u origin <current-branch>
```

If already pushed but local is ahead:
```bash
git push
```

**Error handling:**
- Auth failure → "Git push failed. Check your GitHub authentication (`gh auth status`)."
- Protected branch → "Cannot push to this branch. It may be protected."
- Rejected (non-fast-forward) → "Push rejected — remote has changes. Pull first?"
- Rejected (pre-push hook) → "Pre-push hook failed. [Show hook output]"

## Step 6: Create the PR

### Confirm title and options

> **Create Pull Request:**
> - **Title:** `<TICKET-ID>: <summary>`
> - **Base:** `<base-branch>` ← `<current-branch>`
> - **Commits:** [N commits]
> - **Changes:** [N files changed, +X, -Y]
>
> Options:
> - Edit title? (enter new title or press enter to keep)
> - Draft PR? (yes/no, default: no)
> - Add reviewers? (enter GitHub usernames or press enter to skip)
> - Add labels? (enter labels or press enter to skip)

### Build the PR body

Generate from commits, Jira context, and diff analysis:

```bash
gh pr create \
  --title "<TICKET-ID>: <summary>" \
  --base "<base-branch>" \
  [--draft] \
  [--reviewer "user1,user2"] \
  [--label "label1,label2"] \
  --body "$(cat <<'EOF'
## Summary
- [Bullet point 1: what changed and why — derived from commits]
- [Bullet point 2]
- [Bullet point 3]

## Jira
**[TICKET-ID]:** [Summary]
[If acceptance criteria exist: brief AC summary]

## Test Plan
- [ ] Build passes
- [ ] Unit tests pass
- [ ] Lint/static analysis passes
- [ ] [Specific test scenario 1 from acceptance criteria]
- [ ] [Specific test scenario 2]
- [ ] [Edge case scenario if applicable]

## Changes
[Auto-generated from diff --stat: which modules/files changed]

[IF coverage tables exist in session context from android:write-tests:]
## Coverage
[Insert each stored JaCoCo coverage table here — one per class tested.
Loop through all classes that were tested in this session.
Each table should be in the format from write-tests Step 7.6:
## JaCoCo Coverage: `ClassName`
| Metric | Covered | Total | Coverage |
...
Include notes about unreachable code if present.]
[END IF]

## Screenshots
<!-- Add before/after screenshots for UI changes. Remove this section if N/A -->

---
Generated with Claude Code
EOF
)"
```

**Test plan notes:**
- Do NOT hardcode project-specific commands (e.g., `./gradlew test`) — keep generic
- Pull specific test scenarios from the Jira acceptance criteria
- Include edge cases relevant to the changes
- For UI changes, always include the Screenshots section
- For API changes, include request/response validation scenarios

**Error handling:**
- "already exists" → see **Existing PR Flow** below
- "missing required scopes" → fall back to `gh api` for PR creation/updates:
  ```bash
  gh api repos/OWNER/REPO/pulls --method POST --field title="..." --field head="..." --field base="..." --field body="..."
  ```
  Only suggest `gh auth refresh -h github.com -s read:project` as last resort if `gh api` also fails.
- Auth failure → "GitHub CLI not authenticated. Run `gh auth login`."
- Repo not found → "Could not find the remote repository. Check `git remote -v`."
- Rate limit → "GitHub API rate limit reached. Wait a few minutes and retry."

### Existing PR Flow

If `gh pr create` fails because a PR already exists for this branch:

```bash
gh pr view --json url,title,body,state --jq '{url: .url, title: .title, state: .state}'
```

Present options:
> A PR already exists for this branch: [URL]
> **Title:** [current title]
> **Status:** [Open / Draft]
>
> Options:
> - **Update** — regenerate title and description from current commits and Jira context
> - **View** — open the existing PR (skip creation)
> - **Skip** — keep existing PR as-is

**If user chooses Update:**

Regenerate the PR body using the same logic from Step 4 (gather context from commits, Jira, diff analysis), then:

```bash
gh pr edit \
  --title "<TICKET-ID>: <new summary>" \
  --body "$(cat <<'EOF'
[regenerated PR body — same format as creation]
EOF
)"
```

On success:
> PR updated: [URL]
> - Title: [old] → [new]

Continue to Step 7 (pass PR URL to worklog).

## Step 7: Pass PR URL to Worklog

Do NOT add the PR URL as a Jira comment. Instead, store the PR URL in session context so the `android:jira-worklogging` skill can automatically include it in the worklog description.

> PR URL to include in worklog: `[PR URL]`

The worklog skill will append `PR: <PR URL>` to the auto-generated description. This keeps all work tracking in the worklog rather than scattering it across comments.

## Step 8: Present Result

**Formatting rule:** Always render the PR URL as a clickable markdown link — never output a bare URL. Use the format: `[PR #N — TICKET-ID: Summary](https://github.com/...)`.

> **PR Created:**
> - **URL:** [PR #N — TICKET-ID: Summary](url)
> - **Title:** [TICKET-ID]: [summary]
> - **Base:** [base-branch] ← [current-branch]
> - **Commits:** [N commits]
> - **Status:** [Ready for review / Draft]
> [If reviewers: **Reviewers:** user1, user2]
> [If labels: **Labels:** label1, label2]
> [If linked to Jira: **Jira:** Linked to [TICKET-ID]]

## PR Title Format

**Always:** `<TICKET-ID>: <Short imperative description>`

**Rules:**
- Imperative mood ("Add", "Fix", "Update" — not "Added", "Fixed", "Updated")
- Under 70 characters for the title
- Ticket ID prefix is mandatory
- Description should summarize the WHY, not list files changed

**Good examples:**
- `MA-3353: Add staging build type to prevent debug hitting production`
- `MA-3322: Fix null pointer in account repository`
- `MA-3312: Add unit tests for local integration repository`
- `MA-4001: Update minimum SDK version to 26`

**Bad examples:**
- `Fix bug` (no ticket ID, vague)
- `MA-3353: Updated LoginScreen.kt, LoginViewModel.kt, LoginReducer.kt` (lists files, past tense)
- `MA-3353: Added staging build type and also fixed the notification module and updated dependencies` (too long, multiple concerns)

## PR Body Guidelines

- **Summary:** 2-5 bullet points, focus on WHY not WHAT
- **Jira:** Always include ticket ID and summary
- **Test plan:** Concrete checklist from acceptance criteria — not vague "test it works"
- **Changes:** Auto-generated file/module summary
- **Coverage:** Include JaCoCo coverage tables from session context if `android:write-tests` was run — one table per tested class
- **Screenshots:** Required for UI changes, explicitly mark N/A otherwise
- **Breaking changes:** Call out explicitly if any

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing Jira ticket in title | Always prefix with ticket ID |
| Pushing to wrong base branch | Always detect and confirm base branch |
| Creating PR with failing tests | Run verification first — never skip |
| Force-pushing after PR creation | Never force-push — create new commits |
| Vague test plan | Include specific scenarios from acceptance criteria |
| Not checking if branch is behind base | Always fetch and check before creating PR |
| Hardcoding build commands in test plan | Keep test plan generic, reference CLAUDE.md |
| Adding PR URL as a Jira comment | Pass PR URL to worklog instead — keeps tracking in one place |
| Vague PR summary bullets | Read actual git diff, not just --stat — explain WHY each change was made |
| Outputting bare PR URL as plain text | Always use markdown link: `[PR #N — TICKET: Summary](url)` |
| Adding Co-Authored-By to commits | Never include Co-Authored-By trailers — plain commit messages only |
