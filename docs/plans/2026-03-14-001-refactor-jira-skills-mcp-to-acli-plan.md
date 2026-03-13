---
title: "refactor: Migrate Jira skills from MCP to ACLI"
type: refactor
status: active
date: 2026-03-14
origin: docs/brainstorms/2026-03-14-acli-jira-migration-brainstorm.md
---

# Migrate Jira Skills from MCP to ACLI

## Overview

Replace all 9 Atlassian MCP tool calls (`mcp__claude_ai_Atlassian__*`) across 6 files in the android-claude-plugin with the official Atlassian CLI (ACLI), curl + Jira REST API v3, and GitHub CLI (`gh`). Create a new `jira-setup` skill for automated installation and authentication.

(see brainstorm: docs/brainstorms/2026-03-14-acli-jira-migration-brainstorm.md)

## Problem Statement / Motivation

All Jira operations in the plugin depend on Claude AI's MCP Atlassian integration. This creates:
- **Single point of failure**: MCP down or token expired → all Jira skills break
- **No debugging**: Can't test MCP calls manually in terminal
- **Not portable**: Only works inside Claude Code with MCP configured

ACLI is Atlassian's official CLI, runs locally, credentials stored locally, and can be tested independently.

## Proposed Solution

| Operation | Current (MCP) | New Tool | Command |
|-----------|--------------|----------|---------|
| View issue | `getJiraIssue` | ACLI | `acli jira workitem view KEY --json` |
| Get cloud ID | `getAccessibleAtlassianResources` | — | Eliminated (ACLI handles auth internally) |
| Transition | `getTransitionsForJiraIssue` + `transitionJiraIssue` | ACLI | `acli jira workitem transition --key KEY --status "..."` |
| Edit fields | `editJiraIssue` | ACLI | `acli jira workitem edit --key KEY --from-json tmp.json` |
| Assign | `lookupJiraAccountId` + `editJiraIssue` | ACLI | `acli jira workitem assign --key KEY --assignee @me` |
| Add comment | `addCommentToJiraIssue` | ACLI | `acli jira workitem comment create --key KEY --body "..."` |
| Add worklog | `addWorklogToJiraIssue` | curl | `curl -X POST .../rest/api/3/issue/KEY/worklog` |
| Get remote links | `getJiraIssueRemoteIssueLinks` | gh | `gh pr list --head BRANCH` |

Credentials shared via `~/.jira-config` (source-able bash file with `JIRA_SITE`, `JIRA_EMAIL`, `JIRA_TOKEN`).

## Acceptance Criteria

### Phase 1: Setup
- [x] `jira-setup.md` skill created in `skills/`
- [ ] Running `/android:jira-setup` installs ACLI via brew (if missing)
- [ ] Guides user through API token creation and saves to `~/.jira-config`
- [ ] Authenticates ACLI via `acli jira auth login`
- [ ] Verifies connection with a test `acli jira workitem view`

### Phase 2: Core Jira Skills
- [x] `jira-fetching.md` uses `acli jira workitem view --json` instead of MCP
- [x] `jira-fetching.md` uses `gh` for PR link checking instead of `getJiraIssueRemoteIssueLinks`
- [x] `jira-transitioning.md` uses `acli jira workitem transition` instead of MCP
- [x] `jira-transitioning.md` still verifies transition with `acli jira workitem view`
- [x] `jira-estimating.md` uses `acli jira workitem edit --from-json` for estimates/points
- [x] `jira-estimating.md` falls back to curl if ACLI can't handle custom fields
- [x] `jira-worklogging.md` uses curl + REST API v3 with ADF comment format
- [x] `jira-worklogging.md` sources credentials from `~/.jira-config`
- [x] `jira-worklogging.md` converts human-readable time to `timeSpentSeconds`

### Phase 3: Command Updates
- [x] `creating-pr.md` uses ACLI for Jira comments, `gh` for PR link checking
- [x] `start-task.md` line 46 uses `acli jira workitem assign` instead of `editJiraIssue` + `lookupJiraAccountId`
- [x] `start-task.md` adds jira-setup check at the beginning (before Step 2)
- [x] `finish-task.md` lines 174/177 use ACLI comment + `gh` PR check instead of MCP

### Phase 4: Verification
- [ ] Zero `mcp__claude_ai_Atlassian__` references remain in any file
- [ ] All skill frontmatter (name, description) remains unchanged
- [ ] `start-task` and `finish-task` still reference skills by same names
- [ ] Error handling patterns match existing convention (`Error → "message"` format)

## Implementation Phases

### Phase 1: Create `jira-setup.md` (NEW)

**File:** `skills/jira-setup.md`

```
---
name: jira-setup
description: Install and configure Atlassian CLI (ACLI) for Jira operations — handles brew installation, API token setup, authentication, and connection verification
user-invocable: true
---
```

**Steps:**
1. Check `which acli` — if found, skip to Step 4
2. Check `which brew` — if found, run `brew tap atlassian/acli && brew install acli`
3. If no brew, show manual install link: https://developer.atlassian.com/cloud/acli/guides/install-macos/
4. Check if `~/.jira-config` exists and has all 3 vars — if yes, skip to Step 7
5. Ask user for site URL, email, API token (with link to https://id.atlassian.com/manage-profile/security/api-tokens)
6. Write `~/.jira-config`:
   ```bash
   JIRA_SITE="site.atlassian.net"
   JIRA_EMAIL="user@email.com"
   JIRA_TOKEN="token"
   ```
7. Authenticate: `echo "$TOKEN" | acli jira auth login --site "$SITE" --email "$EMAIL" --token`
8. Verify: `acli jira workitem view KNOWN-KEY --json` (ask user for a test ticket ID)
9. Report success/failure

**Guard clause for other skills:**
All Jira skills should start with:
```
### Prerequisites
Check if ACLI is available and `~/.jira-config` exists:
\`\`\`bash
which acli && test -f ~/.jira-config
\`\`\`
If either check fails: **REQUIRED:** Use the `android:jira-setup` skill.
```

---

### Phase 2: Refactor Core Jira Skills (5 files)

#### 2a. `skills/jira-fetching.md`

**Changes:**
- Remove Step 1 (Get Cloud ID) entirely — no longer needed
- Replace Step 2 (`getJiraIssue`) with:
  ```bash
  acli jira workitem view KEY --json --fields "*all"
  ```
- Parse JSON output the same way (field paths stay the same since ACLI returns Jira's native JSON)
- Replace `getJiraIssueRemoteIssueLinks` (Step: Fetching Remote Links) with:
  ```bash
  gh pr list --head BRANCH --json url,title,state
  ```
- Remove all `cloudId` references
- Keep ADF parsing logic unchanged
- Keep error handling format, update error messages:
  - MCP 404 → `acli` returns non-zero exit + error text → same user message
  - MCP 403 → check exit code + stderr
  - MCP 401 → "Authentication failed. Run `/android:jira-setup` to reconfigure."

**Key field mapping (ACLI JSON vs MCP JSON):**
ACLI `--json` output matches Jira REST API format — field paths (`fields.summary`, `fields.status.name`, etc.) should be identical. Verify during testing.

#### 2b. `skills/jira-transitioning.md`

**Changes:**
- Remove Step 1 (Get Cloud ID)
- Remove Step 3 (`getTransitionsForJiraIssue`) — ACLI handles transition discovery internally
- Replace Step 7 (`transitionJiraIssue`) with:
  ```bash
  acli jira workitem transition --key KEY --status "TARGET_STATUS"
  ```
- Replace Step 8 verification (`getJiraIssue`) with:
  ```bash
  acli jira workitem view KEY --json --fields "status"
  ```
- Simplify steps: Check status → Transition → Verify (3 steps instead of 8)
- Keep fuzzy matching logic for status names
- Keep user confirmation before transitioning
- If ACLI transition fails with unrecognized status, show available transitions:
  ```bash
  acli jira workitem transition --key KEY --help
  ```
  Or fall back to curl: `GET /rest/api/3/issue/KEY/transitions`

#### 2c. `skills/jira-estimating.md`

**Changes:**
- Remove cloudId references
- Replace `editJiraIssue` for time estimate with:
  ```bash
  # Write temp JSON with estimate
  echo '{"fields": {"timeoriginalestimate": SECONDS}}' > /tmp/jira-edit.json
  acli jira workitem edit --key KEY --from-json /tmp/jira-edit.json
  rm /tmp/jira-edit.json
  ```
- Replace story points update similarly:
  ```bash
  echo '{"fields": {"customfield_10016": POINTS}}' > /tmp/jira-edit.json
  acli jira workitem edit --key KEY --from-json /tmp/jira-edit.json
  ```
- **Fallback**: If ACLI `--from-json` fails (400 error), use curl:
  ```bash
  source ~/.jira-config
  curl -s -X PUT \
    -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
    -H "Content-Type: application/json" \
    "https://$JIRA_SITE/rest/api/3/issue/KEY" \
    -d '{"fields": {"timeoriginalestimate": SECONDS}}'
  ```
- Keep time format parsing (human → seconds) unchanged
- Keep story points field fallback logic (customfield_10016 vs story_points)

#### 2d. `skills/jira-worklogging.md`

**Changes (most significant refactor):**
- Replace `addWorklogToJiraIssue` MCP call with curl:
  ```bash
  source ~/.jira-config
  curl -s -X POST \
    -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
    -H "Content-Type: application/json" \
    "https://$JIRA_SITE/rest/api/3/issue/KEY/worklog" \
    -d '{
      "timeSpentSeconds": SECONDS,
      "comment": {
        "version": 1,
        "type": "doc",
        "content": [{"type": "paragraph", "content": [{"type": "text", "text": "DESCRIPTION"}]}]
      }
    }'
  ```
- Add time conversion: human-readable → seconds
  - `2h` → 7200
  - `30m` → 1800
  - `4h 30m` → 16200
  - `1d` → 28800
- Comment must use ADF format (v3 requirement) — the JSON template above handles this
- Keep Step 1 (Show Existing Worklogs) — use `acli jira workitem view KEY --json` to get `fields.worklog.worklogs[]`
- Keep auto-generated description from git commits
- Keep date override logic
- Parse curl response for success (HTTP 201) vs error (4xx)
- Error handling:
  - HTTP 400 → "Failed to log work. Time tracking may be disabled."
  - HTTP 403 → "No permission. Run `/android:jira-setup` to check credentials."
  - HTTP 404 → "Ticket not found."

#### 2e. `skills/creating-pr.md`

**Changes:**
- Replace `addCommentToJiraIssue` (Step 7: Pass PR URL / Step in finish-task) with:
  ```bash
  acli jira workitem comment create --key KEY --body "PR: URL"
  ```
- Replace `getJiraIssueRemoteIssueLinks` with:
  ```bash
  gh pr list --head BRANCH --json url --jq '.[].url'
  ```
  If PR URL already appears → skip comment
- Remove cloudId references
- Keep PR body generation, title format, and all GitHub-specific logic unchanged

---

### Phase 3: Update Commands (2 files)

#### 3a. `commands/start-task.md`

**Changes:**
- **Add new Step 1.5** (after Step 1 Get Ticket ID, before Step 2 Fetch):
  ```
  ## Step 1.5: Verify Jira CLI Setup
  Check if ACLI is available:
  \`\`\`bash
  which acli && test -f ~/.jira-config
  \`\`\`
  If either check fails: **REQUIRED:** Use the `android:jira-setup` skill.
  ```
- **Step 3** (Check Assignment, line 46): Replace:
  - `mcp__claude_ai_Atlassian__lookupJiraAccountId` → not needed
  - `mcp__claude_ai_Atlassian__editJiraIssue` to set assignee → `acli jira workitem assign --key KEY --assignee @me`

#### 3b. `commands/finish-task.md`

**Changes:**
- **Step 7** (Link PR to Jira, lines 174/177): Replace:
  - `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks` → `gh pr list --head BRANCH --json url`
  - `mcp__claude_ai_Atlassian__addCommentToJiraIssue` → `acli jira workitem comment create --key KEY --body "PR: URL"`

---

### Phase 4: Verification

- [x] `grep -r "mcp__claude_ai_Atlassian" .` returns zero results
- [x] All 7 refactored files have correct frontmatter (name, description unchanged)
- [x] Run a manual dry-run through `/android:start-task` flow mentally (check all skill references resolve)
- [ ] Run `/android:jira-setup` to verify install + auth flow works

## Dependencies & Risks

| Risk | Mitigation |
|------|------------|
| ACLI JSON output format differs from MCP | Verify field paths during jira-fetching refactor. If different, add mapping layer |
| ACLI `--from-json` doesn't support `timeoriginalestimate` | curl fallback built into jira-estimating |
| User doesn't have Homebrew | jira-setup shows manual install instructions |
| ADF format for worklog comments is complex | Template is hardcoded in the skill — single paragraph text only |
| ACLI auth expires or needs refresh | jira-setup can be re-run anytime. Skills check auth and redirect |
| `acli jira workitem transition` doesn't list available transitions on failure | Fall back to curl `GET /rest/api/3/issue/KEY/transitions` |

## Files Changed Summary

| # | File | Action | Dependencies |
|---|------|--------|-------------|
| 1 | `skills/jira-setup.md` | **CREATE** | None (first) |
| 2 | `skills/jira-fetching.md` | REFACTOR | jira-setup |
| 3 | `skills/jira-transitioning.md` | REFACTOR | jira-setup |
| 4 | `skills/jira-estimating.md` | REFACTOR | jira-setup |
| 5 | `skills/jira-worklogging.md` | REFACTOR | jira-setup, `~/.jira-config` |
| 6 | `skills/creating-pr.md` | REFACTOR | jira-setup |
| 7 | `commands/start-task.md` | UPDATE (2 spots) | jira-setup, skills 2-4 |
| 8 | `commands/finish-task.md` | UPDATE (1 spot) | skills 5-6 |

**Recommended order:** 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8

## Sources & References

### Origin
- **Brainstorm document:** [docs/brainstorms/2026-03-14-acli-jira-migration-brainstorm.md](docs/brainstorms/2026-03-14-acli-jira-migration-brainstorm.md) — Key decisions: ACLI as primary tool, curl for worklogs, `~/.jira-config` for shared credentials, full MCP replacement

### ACLI Documentation
- [ACLI Introduction](https://developer.atlassian.com/cloud/acli/guides/introduction/)
- [ACLI Workitem Commands](https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem/)
- [ACLI Workitem View](https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-view/)
- [ACLI Workitem Edit](https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem-edit/)
- [ACLI Auth Login](https://developer.atlassian.com/cloud/acli/reference/commands/jira-auth-login/)
- [ACLI Homebrew Tap](https://github.com/atlassian/homebrew-acli)

### Jira REST API
- [REST API v3 Worklogs](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-worklogs/)
- [ADF Comment Format](https://community.developer.atlassian.com/t/worklog-comment-format-for-api-v3/31199)
