# Brainstorm: Rework Jira Setup Skill to Follow Official Docs

**Date:** 2026-03-16
**Status:** Draft

## What We're Building

Rewrite `jira-setup.md` to follow the official Atlassian ACLI documentation exactly, and make the agent fully hands-on — it does the setup itself interactively instead of telling the user to do things manually.

## Why This Change

The current skill has several deviations from official docs:

| Issue | Current (Wrong) | Official (Correct) |
|-------|-----------------|---------------------|
| Brew tap | `brew tap atlassian/acli` | `brew tap atlassian/homebrew-acli` |
| Auth syntax | `--token "$JIRA_TOKEN"` (as flag value) | `echo "$TOKEN" \| acli jira auth login --token` (stdin pipe) |
| Auth status | Not checked | Should use `acli jira auth status` before re-prompting |
| Auth methods | Token only | Both `--web` (OAuth browser) and `--token` (stdin) |
| Binary fallback | Not supported | curl download for Intel/Apple Silicon if brew unavailable |

## Key Decisions

### 1. Keep `~/.jira-config` — needed for raw curl skills
- `jira-estimating.md`, `jira-worklogging.md`, `jira-transitioning.md` all use `source ~/.jira-config` + curl with Basic auth
- ACLI's internal store doesn't expose raw tokens for curl usage
- **Decision:** Keep `~/.jira-config` for curl-based skills, but also run `acli jira auth login` so acli commands work natively

### 2. Let user choose auth method
- Present two options: **Browser OAuth** (`--web`) or **Token** (stdin pipe)
- **Browser OAuth (recommended):**
  - Agent runs `acli jira auth login --web` → browser opens → user clicks approve → done
  - Agent parses site + email from `acli jira auth status` output (it returns both!)
  - Agent only asks user for **API token** (1 question) → writes `~/.jira-config`
  - **Total user effort: 1 browser click + 1 token paste**
- **Token flow (fallback for headless/CI):**
  - Agent asks for site, email, token (3 questions)
  - `echo "$TOKEN" | acli jira auth login --site "$SITE" --email "$EMAIL" --token`
  - Writes `~/.jira-config` with same values
  - **Total user effort: 3 credential inputs**
- **Recommendation:** Default to Browser OAuth — smoothest UX

### 3. Auto-setup from any Jira skill
- Every Jira skill currently checks `which acli && test -f ~/.jira-config`
- **New behavior:** If either check fails, auto-run the full setup flow inline, then continue with the original task
- User should never hit a "not authenticated" dead end

### 4. Agent does the work, not the user
- Agent runs all install commands itself (brew tap, brew install, or curl binary download)
- Agent only asks the user for credentials (site, email, token) or auth method choice
- Agent verifies every step before moving on
- If something fails, agent diagnoses and retries or offers alternatives — doesn't just say "do it yourself"

## Proposed Flow

```
1. Check `acli jira auth status`
   ├─ Authenticated AND ~/.jira-config exists → DONE (skip everything)
   └─ Not authenticated or acli missing or config missing ↓

2. Install ACLI (if `which acli` fails)
   ├─ brew available → `brew tap atlassian/homebrew-acli && brew install acli`
   ├─ no brew + Apple Silicon → curl -LO arm64 binary → chmod +x → sudo mv /usr/local/bin/
   └─ no brew + Intel → curl -LO amd64 binary → chmod +x → sudo mv /usr/local/bin/
   → Verify: `acli --version` (MUST pass before continuing)

3. Ask user: "Browser OAuth (recommended) or API Token?"
   ├─ Browser OAuth (recommended):
   │   → `acli jira auth login --web` (browser opens, user clicks approve)
   │   → Parse site + email from `acli jira auth status`
   │   → Ask user for API token ONLY (1 question — explain: "needed for worklog/transition APIs")
   │   → Write ~/.jira-config using parsed site/email + user's token
   └─ Token (fallback):
       → Ask for site, email, token (3 questions)
       → `echo "$TOKEN" | acli jira auth login --site "$SITE" --email "$EMAIL" --token`
       → If auth fails: diagnose error, re-ask for the failing credential, retry (up to 3x)
       → Write ~/.jira-config with same values

4. Write ~/.jira-config → chmod 600 → verify by sourcing

5. Verify auth: `acli jira auth status` (MUST show authenticated)

6. Verify connection: agent picks a ticket automatically
   → Try extracting from current branch name (e.g., MA-3433)
   → If no branch ticket, use a known project key with `acli jira workitem list --project MA --limit 1`
   → If that fails, ask user for a ticket ID as last resort
```

## Smoothness Principles

1. **Agent does ALL commands** — user never copies/pastes shell commands
2. **User only provides credentials** — site, email, token (3 questions max)
3. **Failures retry inline** — wrong token? re-ask immediately, don't bail
4. **Verification is automatic** — agent picks a test ticket, user doesn't type one
5. **Auto-triggers from any skill** — user never sees "not authenticated" errors

## What Changes

| File | Change |
|------|--------|
| `skills/jira-setup.md` | Full rewrite — correct brew tap, stdin auth, auth status check, binary fallback, user choice of auth method |
| `skills/jira-fetching.md` | Replace prereq check with `acli jira auth status`, auto-setup if fails |
| `skills/jira-transitioning.md` | Same prereq update |
| `skills/jira-estimating.md` | Same prereq update |
| `skills/jira-worklogging.md` | Same prereq update |
| `commands/start-task.md` | Update prereq check |
| `commands/finish-task.md` | Update prereq check |

## Open Questions

None — all resolved during brainstorming.
