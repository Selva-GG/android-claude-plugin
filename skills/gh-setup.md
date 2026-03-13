---
name: gh-setup
description: Install and configure GitHub CLI (gh) for PR operations — handles brew installation, authentication via browser login, and connection verification
user-invocable: false
---

# GitHub CLI Setup

## Overview

Install and authenticate the GitHub CLI (`gh`) so all skills that interact with GitHub PRs can operate. This skill is triggered automatically by prerequisite guards in other skills.

## Step 1: Check gh Installation

```bash
which gh
```

**If found:** Skip to Step 4.

**If not found:** Continue to Step 2.

## Step 2: Install via Homebrew

```bash
which brew
```

**If Homebrew is available:**

```bash
brew install gh
```

Wait for installation to complete. Verify:
```bash
gh --version
```

**If Homebrew is NOT available:**

> GitHub CLI requires manual installation. Follow the instructions at:
> https://cli.github.com/
>
> After installing, retry the operation that brought you here.

Stop here if manual install is needed.

## Step 3: Verify gh Installed

```bash
which gh
```

If still not found after install attempt:
> GitHub CLI installation failed. Check the output above for errors.
> Try running `brew install gh` manually, or install from https://cli.github.com/

Stop here on failure.

## Step 4: Check Authentication

```bash
gh auth status
```

**If authenticated** (exit code 0, shows "Logged in to github.com"):
> GitHub CLI is already authenticated. Setup complete.

Done — skip remaining steps.

**If not authenticated** (exit code 1): Continue to Step 5.

## Step 5: Authenticate

Run the interactive login flow:

```bash
gh auth login
```

This launches a browser-based OAuth flow. The user will:
1. See a device code in the terminal
2. Open a browser to authorize the GitHub CLI
3. Return to the terminal once authorized

**If the user is in a headless environment** (no browser):

```bash
gh auth login --with-token
```

Ask the user to create a personal access token at https://github.com/settings/tokens with scopes: `repo`, `read:org`.

## Step 6: Verify Authentication

```bash
gh auth status
```

**If successful:**
> GitHub CLI setup complete!
>
> **Configuration:**
> - gh version: [version]
> - Authenticated as: [username]
> - Protocol: [HTTPS/SSH]
>
> All GitHub PR operations are now ready.

**If verification fails:**
> Authentication failed. Try running `gh auth login` manually.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skipping browser auth step | `gh auth login` requires completing the browser OAuth flow |
| Token without required scopes | Ensure token has `repo` scope at minimum |
| Corporate SSO not configured | Run `gh auth login` and select your organization when prompted |
| Using SSH but no SSH key | `gh auth login` will ask protocol — choose HTTPS if unsure |
