---
name: jira-setup
description: Install and configure Atlassian CLI (ACLI) for Jira operations — handles brew installation, API token setup, authentication, and connection verification
user-invocable: true
---

# Jira CLI Setup

## Overview

Install and configure the Atlassian CLI (ACLI) and credentials file (`~/.jira-config`) so all Jira skills can operate without MCP dependencies.

## Step 1: Check ACLI Installation

```bash
which acli
```

**If found:** Skip to Step 4.

**If not found:** Continue to Step 2.

## Step 2: Check Homebrew

```bash
which brew
```

**If Homebrew is available:**

```bash
brew tap atlassian/acli && brew install acli
```

Wait for installation to complete. Verify:
```bash
acli --version
```

**If Homebrew is NOT available:**

> ACLI requires manual installation. Follow the instructions at:
> https://developer.atlassian.com/cloud/acli/guides/install-macos/
>
> After installing, re-run `/android:jira-setup` to continue configuration.

Stop here if manual install is needed.

## Step 3: Verify ACLI Installed

```bash
which acli
```

If still not found after install attempt:
> ACLI installation failed. Check the output above for errors.
> Try running `brew install acli` manually, or install from:
> https://developer.atlassian.com/cloud/acli/guides/install-macos/

Stop here on failure.

## Step 4: Check Existing Configuration

```bash
test -f ~/.jira-config && source ~/.jira-config && echo "JIRA_SITE=$JIRA_SITE" && echo "JIRA_EMAIL=$JIRA_EMAIL" && echo "JIRA_TOKEN=${JIRA_TOKEN:+[SET]}"
```

**If all 3 variables are set (non-empty):** Skip to Step 7.

**If file missing or variables incomplete:** Continue to Step 5.

## Step 5: Gather Credentials

Ask the user for each missing value:

> To configure Jira CLI, I need:
>
> 1. **Jira site URL** — your Atlassian site (e.g., `yourcompany.atlassian.net`)
> 2. **Email** — the email associated with your Atlassian account
> 3. **API token** — create one at https://id.atlassian.com/manage-profile/security/api-tokens
>
> Please provide these values.

**Validation:**
- Site URL: must not include `https://` prefix — strip it if provided. Must end in `.atlassian.net` or be a valid domain.
- Email: basic format check (contains `@`)
- Token: non-empty string

## Step 6: Write Configuration File

Write `~/.jira-config` with the gathered values:

```bash
cat > ~/.jira-config << 'JIRAEOF'
JIRA_SITE="<site>"
JIRA_EMAIL="<email>"
JIRA_TOKEN="<token>"
JIRAEOF
chmod 600 ~/.jira-config
```

**Important:** Set `chmod 600` to restrict access — the file contains an API token.

Verify the file was written:
```bash
source ~/.jira-config && echo "Config saved: $JIRA_SITE / $JIRA_EMAIL"
```

## Step 7: Authenticate ACLI

```bash
source ~/.jira-config
acli jira auth login --site "https://$JIRA_SITE" --email "$JIRA_EMAIL" --token "$JIRA_TOKEN"
```

**Error handling:**
- Auth failure → "Authentication failed. Check your site URL, email, and API token. You can re-run `/android:jira-setup` to reconfigure."
- Network error → "Could not reach Jira. Check your internet connection and site URL."
- Invalid site → "Site not found. Ensure the URL is correct (e.g., `yourcompany.atlassian.net`)."

## Step 8: Verify Connection

Ask the user for a test ticket ID:

> Enter a Jira ticket ID to verify the connection (e.g., MA-3353):

Then run:
```bash
acli jira workitem view <TICKET-ID> --json | head -20
```

**If successful:**
> Jira CLI setup complete! Connection verified with [TICKET-ID].
>
> **Configuration:**
> - ACLI: installed (`acli --version`)
> - Site: [site URL]
> - Email: [email]
> - Config: `~/.jira-config`
>
> All Jira skills are now ready to use.

**If verification fails:**
> Connection test failed. The ticket may not exist, or your credentials may be incorrect.
>
> Options:
> - Try a different ticket ID
> - Re-enter credentials (re-run `/android:jira-setup`)
> - Check permissions at https://[site]/jira/settings/system/project-permissions

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Including `https://` in site URL | Strip protocol — use `site.atlassian.net` only |
| Using password instead of API token | Must use API token from https://id.atlassian.com/manage-profile/security/api-tokens |
| Token with restricted permissions | Ensure token has read/write access to Jira projects |
| Not setting file permissions | Always `chmod 600 ~/.jira-config` — token is sensitive |
| Forgetting to source config | All skills use `source ~/.jira-config` before API calls |
