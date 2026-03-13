---
title: GitHub CLI (gh) Feasibility & Setup Skill
date: 2026-03-14
status: complete
---

# GitHub CLI (gh) Feasibility & Setup Skill

## What We're Building

Add a `gh-setup` internal skill and prerequisite guards to all files using `gh` CLI. Ensure `gh` commands across the plugin are feasible and handle failure modes properly. Also add PR update support via `gh pr edit` for existing PRs.

## Current gh Usage (4 files, 3 distinct commands)

| File | Command | Purpose |
|------|---------|---------|
| `skills/creating-pr.md` | `gh pr create` | Creates the pull request |
| `skills/jira-fetching.md` | `gh pr list --head "<TICKET-ID>*"` | Check linked PRs for a ticket |
| `commands/start-task.md` | `gh pr list --head "$(git branch --show-current)"` | Check if PR already linked |
| `commands/finish-task.md` | `gh pr list --head "$(git branch --show-current)"` | Check if PR already linked |

## Full gh CLI Feasibility

| Operation | Command | Feasible | Notes |
|-----------|---------|----------|-------|
| Create PR | `gh pr create` | Yes | Core feature, supports --title, --body, --base, --draft, --reviewer, --label |
| Update PR title/body | `gh pr edit --title/--body` | Yes | Works on existing PRs by number or branch |
| List PRs | `gh pr list` | Yes | Supports --head, --json, --jq filters |
| View PR details | `gh pr view` | Yes | Returns full PR info as JSON |
| Search PRs | `gh pr list --search` | Yes | GitHub search syntax for partial matching |
| Merge PR | `gh pr merge` | Yes | Supports --squash, --merge, --rebase |
| Close PR | `gh pr close` | Yes | |
| Add reviewers | `gh pr edit --add-reviewer` | Yes | Post-creation reviewer management |
| Comment on PR | `gh pr comment` | Yes | Add review comments |
| Check PR status | `gh pr status` | Yes | Shows PR checks and review status |

## Feasibility Analysis

### `gh pr create` (creating-pr.md)
- **Feasible:** Yes — core `gh` functionality, well-tested
- **Failure modes:** Not installed, not authenticated, no remote, rate limit, PR already exists
- **Current error handling:** Already has auth failure + "already exists" handling in creating-pr.md

### `gh pr edit` (NEW — creating-pr.md)
- **Feasible:** Yes — update existing PR title, body, reviewers, labels
- **Use case:** When a PR already exists for the branch, offer to update it instead of erroring
- **Command:** `gh pr edit --title "NEW" --body "NEW"` (uses current branch by default)

### `gh pr list --head BRANCH` (start-task.md, finish-task.md)
- **Feasible:** Yes — simple query, returns JSON
- **Failure modes:** Not installed, not authenticated, no matching PRs (returns empty `[]`)
- **Note:** `--jq` flag is built into `gh` (no external `jq` dependency)

### `gh pr list --search` (jira-fetching.md — REPLACING glob)
- **Feasible:** Yes — uses GitHub search syntax
- **Fix for glob bug:** `gh pr list --search "<TICKET-ID> in:title"` finds PRs by ticket ID in title

### `gh pr list --head "<TICKET-ID>*"` (jira-fetching.md — CURRENT, BROKEN)
- **BUG:** `gh pr list --head` does **exact branch name matching**, NOT glob. The `*` wildcard won't expand.
- **Fix:** Replace with `gh pr list --search "<TICKET-ID> in:title"` (decided during brainstorm)

## Key Decisions

1. **Scope:** gh-setup handles installation + authentication only (no repo/remote/scope verification)
2. **Invocability:** Internal only (`user-invocable: false`) — triggered by prerequisite guards
3. **Prerequisite enforcement:** Hard prerequisite on ALL files using `gh` — block and redirect to gh-setup if missing
4. **Git setup:** Not in scope — assume git is already installed and configured
5. **Bug fix:** The `--head "<TICKET-ID>*"` glob in jira-fetching.md → replace with `--search "<TICKET-ID> in:title"`
6. **PR update support:** Add `gh pr edit` to creating-pr.md — when PR already exists, offer to update title/body instead of just showing the existing URL

## gh-setup Skill Design

**Steps:**
1. Check `which gh` — if found, skip to Step 4
2. Check `which brew` — if found, run `brew install gh`
3. If no brew, show manual install link: https://cli.github.com/
4. Check `gh auth status` — if authenticated, done
5. Run `gh auth login` (interactive browser flow)
6. Verify: `gh auth status` shows authenticated
7. Report success/failure

**Guard clause for other skills:**
```
### Prerequisites
Check if GitHub CLI is available and authenticated:
\`\`\`bash
which gh && gh auth status
\`\`\`
If either check fails: **REQUIRED:** Use the `android:gh-setup` skill.
```

## PR Update Flow (creating-pr.md enhancement)

When `gh pr create` fails with "already exists":

Current behavior:
> "A PR already exists for this branch: [URL]. Open it instead?"

New behavior:
> "A PR already exists for this branch: [URL]
>
> Options:
> - View existing PR
> - Update PR title and description (regenerate from current commits/Jira context)
> - Skip — keep existing PR as-is"

If user chooses update:
```bash
gh pr edit --title "<TICKET-ID>: <new summary>" --body "$(cat <<'EOF'
[regenerated PR body from current commits and Jira context]
EOF
)"
```

## Files to Modify

| # | File | Action |
|---|------|--------|
| 1 | `skills/gh-setup.md` | **CREATE** — internal setup skill |
| 2 | `skills/creating-pr.md` | Add gh prerequisite guard + PR update flow |
| 3 | `skills/jira-fetching.md` | Add gh prerequisite guard + fix glob bug → `--search` |
| 4 | `commands/start-task.md` | Add gh prerequisite guard (alongside existing jira-setup guard) |
| 5 | `commands/finish-task.md` | Add gh prerequisite guard |

## Failure Modes & Handling

| Failure | Detection | Response |
|---------|-----------|----------|
| gh not installed | `which gh` fails | Redirect to gh-setup |
| gh not authenticated | `gh auth status` non-zero | Redirect to gh-setup |
| No git remote | `gh` errors with "no repository" | "No GitHub remote found. Run `git remote add origin <URL>`." |
| Rate limit | HTTP 403 with rate limit message | "GitHub API rate limit reached. Wait a few minutes." |
| PR already exists | `gh pr create` returns existing URL | Offer: view / update / skip |
| Empty PR list | `gh pr list` returns `[]` | Normal — means no PRs found, proceed |
| PR edit fails | `gh pr edit` non-zero exit | "Failed to update PR. [error message]" |

## Open Questions

None — all resolved during discussion.
