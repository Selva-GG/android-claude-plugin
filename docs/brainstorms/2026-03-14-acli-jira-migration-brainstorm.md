# Brainstorm: Migrate Jira Skills from MCP to ACLI

**Date:** 2026-03-14

## What We're Building

Refactor all 5 Jira-related skills in the android-claude-plugin to use the **Atlassian CLI (ACLI)** instead of MCP Atlassian tools (`mcp__claude_ai_Atlassian__*`). This removes the dependency on Claude AI's MCP Atlassian integration, giving direct CLI control over Jira operations.

Additionally, create a new **`jira-setup`** skill that handles ACLI installation and authentication on first run.

## Why This Approach

- **MCP dependency risk**: If MCP server is down, token expires, or Claude's Atlassian OAuth breaks, all Jira operations fail
- **CLI is more reliable**: ACLI runs locally, credentials stored locally, no intermediary
- **Better debugging**: Can test commands manually in terminal
- **Portable**: Works outside of Claude Code too (CI, scripts)
- **Official**: ACLI is Atlassian's own CLI, actively maintained

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **CLI tool** | ACLI (Official Atlassian CLI) | Official, brew installable, actively maintained |
| **Migration strategy** | Full replacement — remove all MCP calls | Simpler, one dependency, no hybrid complexity |
| **Setup** | Auto-install via brew + guided auth. Separate `jira-setup` skill | Clean separation, reusable, handles first-run gracefully |
| **Credentials** | `~/.jira-config` file | Shared between ACLI and curl. We control the format. Stable |
| **Worklog** | curl + Jira REST API v3 | ACLI doesn't support worklogs. curl uses same credentials from .jira-config |
| **PR link checking** | `gh` (GitHub CLI) | Already in our stack. No Jira needed for this |
| **Time estimates** | ACLI `--from-json` if supported, curl fallback | Use ACLI when possible, curl for anything ACLI can't do |
| **File naming** | Keep same skill file names | No breaking changes for start-task/finish-task references |
| **Scope** | All 5 Jira skills + new jira-setup skill | Complete migration, no partial MCP dependency left |

## Tool Mapping

| Operation | Primary Tool | Command | Fallback |
|-----------|-------------|---------|----------|
| View issue | ACLI | `acli jira workitem view KEY --json --fields "*all"` | curl |
| Transition status | ACLI | `acli jira workitem transition --key KEY --status "In Progress"` | curl |
| Edit fields | ACLI | `acli jira workitem edit --key KEY --summary "..."` | curl |
| Edit estimate/points | ACLI | `acli jira workitem edit --key KEY --from-json tmp.json` | curl |
| Add comment | ACLI | `acli jira workitem comment create --key KEY --body "text"` | curl |
| Assign issue | ACLI | `acli jira workitem assign --key KEY --assignee @me` | curl |
| Add worklog | curl | `curl -X POST .../rest/api/3/issue/KEY/worklog` | — |
| Check PR links | gh | `gh pr list` / `gh api` | — |
| Auth setup | ACLI | `acli jira auth login --site X --email Y --token` | — |

## ACLI Limitations (and workarounds)

| Missing Feature | Workaround |
|----------------|------------|
| No worklog command | curl + REST API v3 (POST `/rest/api/3/issue/{key}/worklog`) |
| No remote issue links | Use `gh` CLI for PR-related checks |
| No direct `--estimate` flag on edit | Use `--from-json` with temp JSON file, or curl fallback |
| No `getTransitionsForJiraIssue` equivalent | Transition command handles this internally; if needed, use curl |

## Credentials Architecture

### `~/.jira-config` file format:
```bash
JIRA_SITE="yourcompany.atlassian.net"
JIRA_EMAIL="you@email.com"
JIRA_TOKEN="your-api-token"
```

### How it's used:
- **ACLI**: Authenticated via `acli jira auth login` (uses its own config)
- **curl**: `source ~/.jira-config` then use `$JIRA_EMAIL:$JIRA_TOKEN` for basic auth
- **Both** set up during `jira-setup` skill

### curl worklog example:
```bash
source ~/.jira-config
curl -s -X POST \
  -H "Authorization: Basic $(echo -n "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
  -H "Content-Type: application/json" \
  "https://$JIRA_SITE/rest/api/3/issue/MA-3418/worklog" \
  -d '{
    "timeSpentSeconds": 600,
    "comment": {
      "version": 1,
      "type": "doc",
      "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Implemented unit tests for IntegrationRepository"}]}]
    }
  }'
```

## `jira-setup` Skill Design

### Triggers:
- Called automatically by any Jira skill if ACLI not found or `.jira-config` missing
- Can also be called manually: `/android:jira-setup`

### Steps:
1. Check if `acli` is installed (`which acli`)
2. If not: `brew tap atlassian/acli && brew install acli`
3. If brew not available: show manual install instructions
4. Ask user for:
   - Jira site URL (e.g., `yourcompany.atlassian.net`)
   - Email address
   - API token (guide them to create one at https://id.atlassian.com/manage-profile/security/api-tokens)
5. Save to `~/.jira-config`
6. Run `echo "$TOKEN" | acli jira auth login --site "$SITE" --email "$EMAIL" --token`
7. Verify: `acli jira workitem view SOME-KEY --json` (use a known ticket to test)
8. Report success or failure

## Skills to Refactor

### 1. `jira-fetching.md`
**Current**: `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` + `mcp__claude_ai_Atlassian__getJiraIssue`
**New**: `acli jira workitem view KEY --json --fields "*all"`
- Parse JSON output for all fields
- No cloudId needed — ACLI handles auth internally
- ADF description parsing stays the same (JSON structure)
- Remote links → use `gh` for PR checks, skip Jira remote links

### 2. `jira-transitioning.md`
**Current**: `mcp__claude_ai_Atlassian__getTransitionsForJiraIssue` + `mcp__claude_ai_Atlassian__transitionJiraIssue`
**New**: `acli jira workitem transition --key KEY --status "In Progress"`
- ACLI handles transition discovery internally
- Still verify after transition with `acli jira workitem view KEY --json`

### 3. `jira-estimating.md`
**Current**: `mcp__claude_ai_Atlassian__editJiraIssue` with `timeoriginalestimate` and `customfield_10016`
**New**: `acli jira workitem edit --key KEY --from-json tmp.json` (with estimate/points in JSON)
- Fallback: curl PUT to REST API if ACLI can't handle custom fields

### 4. `jira-worklogging.md`
**Current**: `mcp__claude_ai_Atlassian__addWorklogToJiraIssue`
**New**: `curl -X POST .../rest/api/3/issue/KEY/worklog` with ADF comment
- Source `~/.jira-config` for auth
- Comment must be in ADF format (v3 requirement)
- timeSpentSeconds instead of human-readable (convert in skill)

### 5. `creating-pr.md`
**Current**: `mcp__claude_ai_Atlassian__addCommentToJiraIssue` + `mcp__claude_ai_Atlassian__getJiraIssueRemoteIssueLinks`
**New**:
- Comment: `acli jira workitem comment create --key KEY --body "PR: URL"`
- PR link check: `gh pr list --head BRANCH` or `gh api repos/ORG/REPO/pulls?head=BRANCH`

## Files Changed

| File | Action |
|------|--------|
| `skills/jira-fetching.md` | Refactor: MCP → ACLI |
| `skills/jira-transitioning.md` | Refactor: MCP → ACLI |
| `skills/jira-estimating.md` | Refactor: MCP → ACLI + curl fallback |
| `skills/jira-worklogging.md` | Refactor: MCP → curl REST API |
| `skills/creating-pr.md` | Refactor: MCP → ACLI + gh |
| `skills/jira-setup.md` | **NEW**: ACLI install + auth setup |
| `commands/start-task.md` | Minor: add jira-setup check at start |
| `commands/finish-task.md` | No changes (calls same skill names) |

## Open Questions

None — all resolved during brainstorming.

## References

- [ACLI Official Docs](https://developer.atlassian.com/cloud/acli/guides/introduction/)
- [ACLI Jira Workitem Commands](https://developer.atlassian.com/cloud/acli/reference/commands/jira-workitem/)
- [ACLI Install macOS](https://developer.atlassian.com/cloud/acli/guides/install-macos/)
- [ACLI Auth Login](https://developer.atlassian.com/cloud/acli/reference/commands/jira-auth-login/)
- [ACLI Homebrew Tap](https://github.com/atlassian/homebrew-acli)
- [Jira REST API v3 Worklogs](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-worklogs/)
- [ADF Comment Format for v3](https://community.developer.atlassian.com/t/worklog-comment-format-for-api-v3/31199)
