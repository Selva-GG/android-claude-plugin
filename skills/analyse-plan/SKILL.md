---
name: analyse-plan
description: Read gathered context (Jira + Figma + cache), identify affected layers and files, perform deduplication check, produce implementation plan with user confirmation
user-invocable: false
---

# Analyse & Plan

Read all gathered context from Step 1 (Jira ticket, Figma designs, cache data) and produce a structured implementation plan. This skill is the brain of Step 2 in the orchestrator pipeline.

## Inputs

This skill receives context from Step 1 gather agents:
- **Jira ticket data** — title, description, acceptance criteria, type, parent, subtasks
- **Figma screenshots** — optional, only if design references were found
- **Cache data** — selected Layer 2 cache files relevant to the Jira ticket

## Step 1: Load Relevant Cache Files

Based on Jira ticket keywords, load ONLY the cache files that are relevant. Do not load all 7 files.

| Jira keywords | Cache files to load |
|---------------|-------------------|
| repository, repo, data, database, storage, entity, DAO | `repository-map.md`, `model-map.md` |
| service, manager, business logic, handler | `service-map.md`, `di-graph.md` |
| screen, UI, component, button, input, dialog, compose | `component-catalog.md`, `feature-index.md` |
| navigation, route, screen flow, deep link | `route-map.md`, `feature-index.md` |
| model, enum, type, data class | `model-map.md` |
| DI, inject, module, binding, provide | `di-graph.md` |
| new feature (end-to-end) | ALL cache files |

If cache files don't exist or `last-scan-timestamp` is >7 days old:
> Cache is stale (last refreshed N days ago). Run `/android:refresh-cache` first?
> - Yes — run refresh, then continue
> - No — proceed with runtime scan only (slower but works)

## Step 2: Load Relevant Procedures

Based on which layers will be affected (determined in Step 3), load ONLY relevant procedure files:

| Layer affected | Procedures to load |
|---------------|-------------------|
| Domain | (none — pure Kotlin, minimal rules) |
| Data | `procedures/data/storage.md`, `procedures/data/networking-api.md`, `procedures/data/services-repositories.md` |
| Core | `procedures/core/dependency-injection.md` |
| Service | `procedures/data/services-repositories.md` |
| ViewModel | `procedures/ui/mvi-pattern.md` |
| UI | `procedures/ui/compose-ui.md`, `procedures/ui/navigation-dialogs.md` |
| Tests | `procedures/shared/testing.md` (always loaded) |

## Step 3: Runtime Scan (Layer 3)

Scan the actual codebase to understand the current state. This fills gaps that the cache doesn't cover.

### 3a: Identify affected areas

Based on Jira ticket content, search for:
```bash
# Search for related code
grep -r "keyword1\|keyword2" --include="*.kt" -l <src-root>

# Check if models/interfaces already exist
grep -r "class.*ModelName\|interface.*IServiceName" --include="*.kt" -l <src-root>
```

### 3b: Determine affected layers

For each finding, classify which layer it belongs to:

| File location | Layer |
|--------------|-------|
| `domain/model/`, `domain/enums/` | Domain |
| `domain/repository/`, `domain/services/` | Domain (interfaces) |
| `data/repository/`, `data/api/`, `data/storage/`, `data/services/` | Data |
| `core/di/`, `core/network/`, `core/config/`, `core/initialization/` | Core |
| `core/service/` | Service |
| `features/*/viewmodel/`, `features/*Reducer.kt`, `features/*ViewModel.kt` | ViewModel |
| `features/*Screen.kt`, `features/*/components/`, `features/*/screen/` | UI |

### 3c: Read specific files

For each affected file, read it to understand:
- Current implementation
- Patterns used
- Dependencies
- What needs to change

## Step 4: Deduplication Check (CRITICAL)

Before proposing any new files, check if similar code already exists.

**Process: FIND → UNDERSTAND → RECOMMEND → ASK**

### 4a: FIND similar things

For each new interface/model/service/enum proposed:
```bash
# Search for similar names
grep -r "class.*SimilarName\|interface.*ISimilarName\|enum.*SimilarName" --include="*.kt" -l <src-root>

# Search for similar concepts
grep -r "keyword" --include="*.kt" -l <src-root>
```

### 4b: UNDERSTAND domain relevance

For each match found, read the file and determine:
- Is it the **same domain concept**? (same purpose, same responsibility)
- Or just a **name collision**? (similar name, different purpose)

Ask yourself:
- "If I reuse this, would it make the code MORE or LESS clear?"
- "Would the existing code's purpose be diluted by adding my new use case?"
- "Is the existing code in the right layer for my needs?"

### 4c: RECOMMEND in simple words

Present each finding to the user:

```
I found some existing code that looks related:

1. [ExistingClass] in [file path]
   → It does [what it does in simple terms].
   → Your task needs [what you need].
   → These are [same concept / different concept] because [reason].
   → I recommend: [REUSE / EXTEND / CREATE NEW]

Do you agree? Or should I [reuse/create new] instead?
```

### 4d: ASK for confirmation

**NEVER silently decide to reuse or extend.** Always present findings and let the user confirm.

Options to present:
- Agree with all recommendations
- Override specific recommendations
- "Let me explain my preference"

## Step 5: Produce Implementation Plan

After dedup check is confirmed, produce the structured plan:

```
IMPLEMENTATION PLAN: [TICKET-ID]

Affected layers:
  [✓/✗] Domain  — [CREATE/MODIFY/SKIP] [file list]
  [✓/✗] Data    — [CREATE/MODIFY/SKIP] [file list]
  [✓/✗] Core    — [CREATE/MODIFY/SKIP] [file list]
  [✓/✗] Service — [CREATE/MODIFY/SKIP] [file list]
  [✓/✗] ViewModel — [CREATE/MODIFY/SKIP] [file list]
  [✓/✗] UI      — [CREATE/MODIFY/SKIP] [file list]

Files to create: [list with full paths]
Files to modify: [list with full paths and what changes]

Procedures to load per layer:
  Domain: (none)
  Data: [list]
  Core: [list]
  ...

Key design decisions:
  - [decision 1 with reasoning]
  - [decision 2 with reasoning]

Does this plan look right?
  [ ] Yes, proceed
  [ ] Adjust something
```

## Step 6: User Confirmation

Wait for user to confirm or adjust the plan. If adjustments requested:
1. Incorporate changes
2. Re-present the updated plan
3. Ask for confirmation again

Only proceed to implementation after explicit user approval.

## Output

The confirmed plan is passed to the orchestrator (or to the `implement-layers` skill) as the implementation guide.
