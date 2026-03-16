---
name: jira-setup
description: Install and configure Atlassian CLI (ACLI) for Jira operations — handles brew/binary installation, browser or token authentication, credential verification, and ~/.jira-config for curl-based skills
user-invocable: true
---

# Jira CLI Setup

> **ABSOLUTE HARD RULE — NO ATLASSIAN MCP. EVER.**
> NEVER use `mcp__claude_ai_Atlassian__*` or ANY Atlassian MCP tool — not in this skill, not as a fallback, not as a "quick check", not under any circumstance. This includes but is not limited to: `mcp__claude_ai_Atlassian__getJiraIssue`, `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`, `mcp__claude_ai_Atlassian__transitionJiraIssue`, `mcp__claude_ai_Atlassian__editJiraIssue`, `mcp__claude_ai_Atlassian__addWorklogToJiraIssue`, `mcp__claude_ai_Atlassian__addCommentToJiraIssue`, and ALL others.
> **ALL Jira operations MUST use ACLI (`acli`), curl + REST API, or `gh` CLI. No exceptions. No fallbacks to MCP. Ever.**

## Overview

Install ACLI, authenticate with Jira, and write `~/.jira-config` so all Jira skills work. The agent does everything — the user only provides credentials when asked.

**This skill can be called standalone (`/android:jira-setup`) or auto-triggered inline from any Jira skill when auth is missing.**

---

## Step 1: Check If Already Set Up

Run both checks:

```bash
acli jira auth status 2>&1
```

```bash
test -f ~/.jira-config && source ~/.jira-config && test -n "$JIRA_SITE" && test -n "$JIRA_EMAIL" && test -n "$JIRA_TOKEN" && echo "CONFIG_OK"
```

**If ACLI is authenticated AND config file has all 3 variables:**
> Jira CLI is already configured and authenticated.

**DONE** — skip all remaining steps.

**Otherwise:** Continue to Step 2.

---

## Step 2: Install ACLI (if missing)

```bash
which acli
```

**If `acli` is found:** Skip to Step 3.

**If not found — try each method in order until one works:**

### Method 1: Homebrew

```bash
which brew
```

If brew is available:
```bash
brew tap atlassian/homebrew-acli && brew install acli
```

Verify: `acli --version`

**If brew install succeeds → skip to Step 3.**
**If brew fails (tap error, install error, etc.) → try Method 2.**

### Method 2: Direct Binary Download

Detect architecture:
```bash
uname -m
```

- `arm64` (Apple Silicon):
  ```bash
  curl -LO "https://acli.atlassian.com/darwin/latest/acli_darwin_arm64/acli" && chmod +x ./acli && sudo mv ./acli /usr/local/bin/acli
  ```

- `x86_64` (Intel):
  ```bash
  curl -LO "https://acli.atlassian.com/darwin/latest/acli_darwin_amd64/acli" && chmod +x ./acli && sudo mv ./acli /usr/local/bin/acli
  ```

Verify: `acli --version`

**If binary download succeeds → skip to Step 3.**
**If fails (curl error, permission denied on sudo, etc.) → try Method 3.**

### Method 3: Binary to User-Local Path (no sudo)

If `sudo mv` failed, try a user-writable location instead:

```bash
mkdir -p ~/bin
```

Download again (same arch detection as Method 2), but move to `~/bin/`:
```bash
mv ./acli ~/bin/acli
```

Add to PATH if not already:
```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Verify: `acli --version`

**If succeeds → skip to Step 3.**
**If fails → try Method 4.**

### Method 4: Homebrew Install (if brew wasn't available, install brew first)

If brew was not available in Method 1:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After brew installs, retry:
```bash
brew tap atlassian/homebrew-acli && brew install acli
```

Verify: `acli --version`

**If succeeds → skip to Step 3.**
**If fails → Method 5 (last resort).**

### Method 5: Ask User (last resort only)

Only reach this if ALL of the above failed.

> I tried 4 different installation methods and all failed:
> - Homebrew: [error]
> - Direct download: [error]
> - User-local path: [error]
> - Homebrew install + retry: [error]
>
> Please install ACLI manually following: https://developer.atlassian.com/cloud/acli/guides/install-macos/
>
> Once installed, I'll handle everything else (auth, config, verification).

Wait for user to confirm ACLI is installed, then verify with `acli --version` and continue to Step 3.

---

## Step 3: Authenticate

Check if already authenticated:
```bash
acli jira auth status 2>&1
```

**If already authenticated:** Skip to Step 4 (just need to verify/write config file).

**If not authenticated — ask user to choose auth method:**

> How would you like to authenticate with Jira?
>
> 1. **Browser OAuth (recommended)** — opens your browser, you click approve. Easiest option.
> 2. **API Token** — you provide your site URL, email, and API token manually. Use this for headless/CI environments.

### Option A: Browser OAuth (recommended)

Run:
```bash
acli jira auth login --web
```

This opens the browser. Wait for the user to complete the OAuth flow.

After the browser flow completes, verify:
```bash
acli jira auth status 2>&1
```

**If auth status shows authenticated:** Parse the site and email from the output:
- `Site: <site>` → extract site value
- `Email: <email>` → extract email value

Now the agent needs the API token for `~/.jira-config` (used by curl-based skills). Open the token creation page automatically:

> I need an API token for worklog, transition, and estimate operations that use the Jira REST API directly.
>
> Create one (or copy an existing one) here: https://id.atlassian.com/manage-profile/security/api-tokens
>
> Paste the token below:

Wait for the user to provide the token.

**If auth failed or user closed browser:**
> Browser authentication didn't complete. Would you like to:
> 1. Try browser auth again
> 2. Switch to API token method

### Option B: API Token

Ask the user for credentials:

> To authenticate with Jira, I need:
>
> 1. **Jira site URL** — your Atlassian site (e.g., `yourcompany.atlassian.net`)
> 2. **Email** — the email on your Atlassian account
> 3. **API token** — create one at https://id.atlassian.com/manage-profile/security/api-tokens

Wait for user to provide all three values.

**Validate inputs:**
- Site URL: strip `https://` if provided. Must be a valid domain (e.g., `site.atlassian.net`).
- Email: must contain `@`
- Token: non-empty

**Authenticate:**
```bash
echo "<TOKEN>" | acli jira auth login --site "<SITE>" --email "<EMAIL>" --token
```

**If auth fails:**

| Error | Action |
|-------|--------|
| "unauthorized" or "invalid token" | Re-ask for token only: "Token was rejected. Please double-check and paste again." |
| "site not found" or "not found" | Re-ask for site only: "Site not reachable. Check the URL (e.g., yourcompany.atlassian.net)." |
| "invalid email" | Re-ask for email only |
| Network error | "Can't reach Jira. Check your internet connection." — offer retry |

**Retry up to 3 times per failing credential.** Re-ask only for the specific credential that failed, not all three.

---

## Step 4: Write ~/.jira-config

This file is required by curl-based Jira skills (worklogging, transitioning, estimating).

**Gather values:**
- If Browser OAuth was used: site and email were parsed from `acli jira auth status`, token was provided by user
- If Token auth was used: all three were provided by user

**Write the file:**
```bash
cat > ~/.jira-config << 'JIRAEOF'
JIRA_SITE="<site>"
JIRA_EMAIL="<email>"
JIRA_TOKEN="<token>"
JIRAEOF
chmod 600 ~/.jira-config
```

**Verify:**
```bash
source ~/.jira-config && echo "Site: $JIRA_SITE | Email: $JIRA_EMAIL | Token: [SET]"
```

---

## Step 5: Verify Connection

The agent picks a test ticket automatically — do NOT ask the user for one.

**Try extracting from current branch:**
```bash
git branch --show-current 2>/dev/null | grep -oE '[A-Z]+-[0-9]+'
```

**If a ticket ID was found:**
```bash
acli jira workitem view <TICKET-ID> --json --fields "summary" 2>&1
```

**If no branch ticket, try listing from known project:**
```bash
acli jira workitem list --project MA --limit 1 --json 2>&1
```

**If both fail — only then ask user:**
> Could not auto-detect a ticket for verification. Enter a ticket ID to test (e.g., MA-3353):

**On successful verification:**
> Jira CLI setup complete!
>
> | | |
> |---|---|
> | **ACLI** | Installed (`acli --version`) |
> | **Auth** | Authenticated ([auth type]) |
> | **Site** | [site URL] |
> | **Email** | [email] |
> | **Config** | `~/.jira-config` (chmod 600) |
> | **Verified** | [TICKET-ID] — [summary] |
>
> All Jira skills are ready to use.

**On verification failure:**
> Connection test failed. The credentials may be incorrect or you may lack project access.
>
> Options:
> - Re-enter credentials (restart auth)
> - Check permissions at https://[site]/jira/settings/system/project-permissions

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Wrong brew tap (`atlassian/acli`) | Correct tap is `atlassian/homebrew-acli` |
| Passing token as flag value (`--token "xyz"`) | Token must be piped via stdin: `echo "xyz" \| acli ... --token` |
| Including `https://` in site URL | Strip protocol — use `site.atlassian.net` only |
| Using password instead of API token | Must use API token from https://id.atlassian.com/manage-profile/security/api-tokens |
| Not setting file permissions | Always `chmod 600 ~/.jira-config` — contains API token |
| Asking user to run commands manually | Agent runs everything — user only provides credentials |
| Asking user for test ticket ID | Auto-detect from branch or project list first |
