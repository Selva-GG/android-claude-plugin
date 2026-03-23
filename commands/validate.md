---
name: validate
description: Run 4 parallel validation agents (test-runner, lint-analyzer, code-reviewer, spec-checker), merge findings by severity, fix issues
---

# Validate

Run 4 validation agents in parallel to check the current implementation. This is Step 5 of the pipeline, usable standalone.

**HARD RULE — No MCP Tools:** Never use `mcp__claude_ai_Atlassian__*` or any MCP Atlassian tools.

## Step 1: Identify Context

Extract Jira ticket ID from the current branch:
```bash
git branch --show-current
```
Parse ticket ID pattern: `([A-Z]+-[0-9]+)` from the branch name.

If Jira context is needed (for spec-checker), fetch ticket data:
```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>?fields=summary,description" 2>/dev/null
```

If unable to fetch (no credentials, no ticket ID), run spec-checker without Jira data — it will report limited findings.

## Step 2: Spawn 4 Agents in Parallel

Launch all 4 validation agents simultaneously. They share the working directory and are read-only.

**Agent 1: test-runner**
- Prompt: "Run the full test suite and JaCoCo coverage analysis for this project. Report results."

**Agent 2: lint-analyzer**
- Prompt: "Run detekt static analysis on this project. Report all violations."

**Agent 3: code-reviewer**
- Prompt: "Review the changed files in the latest commits against MeApp conventions. Report violations."

**Agent 4: spec-checker**
- Prompt: "Compare the implementation against these Jira acceptance criteria: [criteria from Step 1]. Report which are met, missing, or partial."
- Only include Jira data if it was successfully fetched.
- Only include Figma screenshots if they were captured in Step 1 gather phase.

## Step 3: Merge Results

After all 4 agents return, merge their findings:

### 3a: Collect all findings

Combine findings from all agents into a single list.

### 3b: Sort by severity

1. **CRITICAL** — must fix before proceeding:
   - Test failures (test-runner)
   - `!!` operators (lint-analyzer)
   - `super.handleIntent` missing (code-reviewer)
   - Missing acceptance criteria (spec-checker)

2. **WARNING** — should fix:
   - Coverage below target (test-runner)
   - Long methods/classes (lint-analyzer)
   - Missing previews, hardcoded values (code-reviewer)
   - Partially met criteria (spec-checker)

3. **INFO** — optional:
   - Coverage stats (test-runner)
   - Style suggestions (lint-analyzer)
   - Improvement suggestions (code-reviewer)
   - All criteria met (spec-checker)

### 3c: Present merged results

```
## Validation Results

### Critical (must fix)
| # | Source | Issue |
|---|--------|-------|
| 1 | test-runner | 2 test failures in FooTest |
| 2 | lint-analyzer | !! operator at BarService.kt:45 |

### Warnings (should fix)
| # | Source | Issue |
|---|--------|-------|
| 3 | code-reviewer | Missing @PreviewTheme in BazScreen.kt |
| 4 | test-runner | FooService coverage at 78% (target: 90%) |

### Info
| # | Source | Issue |
|---|--------|-------|
| 5 | spec-checker | All 7 acceptance criteria met |
| 6 | test-runner | 42 tests passing |
```

## Step 4: Fix Loop

### 4a: Fix critical issues first

The orchestrator (main agent) fixes all CRITICAL issues:
- Test failures: read the failing test, understand the error, fix the code or test
- Banned operators: replace `!!` with safe alternatives
- Missing super calls: add them

### 4b: Fix warnings

After criticals are fixed, address warnings.

### 4c: Re-validate

After fixes, re-run validation (go back to Step 2).

### 4d: Loop limit

After **3 failed validation loops**, stop auto-fixing and present all remaining issues:

> Validation has failed 3 times. Remaining issues:
>
> | # | Severity | Source | Issue |
> |---|----------|--------|-------|
> | 1 | WARNING | ... | ... |
>
> What would you like to do?
> - Fix specific issues (tell me which ones)
> - Skip remaining warnings and proceed
> - Abort validation

**Let the user decide.** Do not create a draft PR automatically.

## Step 5: Validation Complete

When all checks pass (or user chooses to skip remaining warnings):

> **Validation passed.**
>
> | Agent | Result |
> |-------|--------|
> | test-runner | N tests passing, N% coverage |
> | lint-analyzer | 0 violations |
> | code-reviewer | 0 critical, 0 warnings |
> | spec-checker | N/N criteria met |
>
> Ready for finalization (PR, worklog, Jira transition).
