---
title: "refactor: Orchestrator + Agent Team Plugin Restructure"
type: refactor
status: active
date: 2026-03-23
origin: docs/brainstorms/2026-03-23-orchestrator-agent-team-restructure-brainstorm.md
---

# Orchestrator + Agent Team Plugin Restructure

## Overview

Restructure the android-claude-plugin from a linear, sequential workflow into an orchestrator + agent team architecture. The orchestrator (main agent) handles context-dependent work (planning, implementation, fixing). Agent teams handle parallel independent tasks (research, validation, finalization).

This introduces a 3-layer knowledge system: static procedures (hand-written rules), pre-analysed cache (auto-generated codebase maps), and runtime analysis (per-task scanning). Together, these eliminate the need for 30+ static procedure files while keeping the orchestrator informed.

(see brainstorm: docs/brainstorms/2026-03-23-orchestrator-agent-team-restructure-brainstorm.md)

## Problem Statement

1. **Speed** — All steps run sequentially. Jira fetch, Figma check, codebase scan, build, test, lint — one after another.
2. **Quality** — No structured analysis step before implementation. No code review or spec check in validation. Only build/test/lint.
3. **Structure** — `start-task.md` is a monolithic 16-step command with no separation between gathering, planning, implementing, and validating.
4. **Knowledge gaps** — 12 procedure files cover only ~20% of the codebase. 80% (shared composables, services, DI, features) is undocumented. Writing 30+ static files is not scalable.

## Proposed Solution

A 6-step pipeline with parallel agent teams at steps 1, 5, and 6:

| Step | Name | Who | Parallel? |
|------|------|-----|-----------|
| 1 | Gather Context | Agent team | Yes — Jira + Figma + cache reader |
| 2 | Analyse & Plan | Orchestrator | No — needs all context |
| 3 | Setup | Orchestrator | No — estimate, transition, branch |
| 4 | Implement | Orchestrator | No — layer-by-layer, context continuity |
| 5 | Validate | Agent team | Yes — 4 validators in parallel |
| 6 | Finalize | Agent team | Yes — PR + worklog + Jira transition |

Plus a 3-layer knowledge system:
- **Layer 1 (static)**: Existing 12 procedure files reorganized by layer — rules/decisions
- **Layer 2 (cached)**: 7 auto-generated .md files mapping the codebase — facts
- **Layer 3 (runtime)**: Per-task analysis combining Jira + cache + code scanning

## Acceptance Criteria

### Functional Requirements

- [ ] `/android:start-task TICKET-ID` runs the full 6-step pipeline with parallel agents at steps 1, 5, 6
- [ ] `/android:finish-task` runs steps 5-6 standalone (shares validation/finalize logic with start-task)
- [ ] `/android:validate` runs step 5 standalone (4 parallel validation agents)
- [ ] `/android:analyse TICKET-ID` runs step 2 standalone (analyse + plan only)
- [ ] `/android:refresh-cache` scans the MeApp codebase and generates 7 Layer 2 cache files at `.claude/cache/`
- [ ] Step 2 performs deduplication check: finds similar existing code, explains domain relevance, asks user before reuse/create decisions
- [ ] Step 2 dynamically loads only relevant cache files and procedures based on Jira ticket content
- [ ] Step 4 implements layers sequentially (Domain → Data → Core → Service → ViewModel → UI), auto-skipping unaffected layers
- [ ] Step 4 writes tests inline per layer (basic happy + error paths)
- [ ] Step 5 runs test-runner, lint-analyzer, code-reviewer, spec-checker in parallel (shared directory, read-only)
- [ ] Step 5 merges findings by severity (critical → warning → info), orchestrator fixes issues
- [ ] Step 5 after 3+ failures: shows all remaining issues, lets user decide what to fix/skip/accept
- [ ] Subtasks: shared gather/plan for parent, each subtask runs steps 3-6 independently
- [ ] All existing Jira/GitHub/PR/worklog skills continue to work unchanged

### Non-Functional Requirements

- [ ] No MCP tools used — all operations via ACLI, curl, gh CLI
- [ ] Cache files are gitignored (project-local at `.claude/cache/`)
- [ ] Auto-stale detection: suggest refresh when cache >7 days old
- [ ] Agent definitions follow compound-engineering format (name, description, model, color frontmatter)

## Implementation Phases

### Phase 1: Restructure Procedures (move files only, no content changes)

Create layer sub-folders and move existing procedure files. Update the SKILL.md index.

**Files to create:**
- `skills/procedures/domain/` (empty directory — domain rules are minimal)
- `skills/procedures/data/` (directory)
- `skills/procedures/core/` (directory)
- `skills/procedures/ui/` (directory)
- `skills/procedures/shared/` (directory)

**Files to move:**

| From | To |
|------|-----|
| `skills/procedures/storage.md` | `skills/procedures/data/storage.md` |
| `skills/procedures/networking-api.md` | `skills/procedures/data/networking-api.md` |
| `skills/procedures/services-repositories.md` | `skills/procedures/data/services-repositories.md` |
| `skills/procedures/dependency-injection.md` | `skills/procedures/core/dependency-injection.md` |
| `skills/procedures/mvi-pattern.md` | `skills/procedures/ui/mvi-pattern.md` |
| `skills/procedures/compose-ui.md` | `skills/procedures/ui/compose-ui.md` |
| `skills/procedures/navigation-dialogs.md` | `skills/procedures/ui/navigation-dialogs.md` |
| `skills/procedures/testing.md` | `skills/procedures/shared/testing.md` |
| `skills/procedures/test-patterns.md` | `skills/procedures/shared/test-patterns.md` |
| `skills/procedures/test-troubleshooting.md` | `skills/procedures/shared/test-troubleshooting.md` |
| `skills/procedures/test-coverage.md` | `skills/procedures/shared/test-coverage.md` |
| `skills/procedures/review-checklist.md` | `skills/procedures/shared/review-checklist.md` |

**Files to update:**
- `skills/procedures/SKILL.md` — update the sub-skill trigger table with new paths (`data/storage.md` instead of `storage.md`)

**Verification:** All relative links in SKILL.md point to valid files after move.

---

### Phase 2: Create Layer 2 Cache System (refresh-cache skill + command)

**Files to create:**

`skills/refresh-cache/SKILL.md`:
```
---
name: refresh-cache
description: Scan the MeApp Android codebase and generate Layer 2 cache files at .claude/cache/
user-invocable: false
---
```

Content: Instructions to scan the codebase at `app/src/main/java/com/dmdbrands/gurus/weight/` and generate 7 cache files:

| Cache File | What it maps | Source directories to scan |
|-----------|-------------|--------------------------|
| `component-catalog.md` | 99+ shared composables with signatures, props, usage | `features/common/` |
| `service-map.md` | 25+ services: interface → implementation → DI module → layer | `domain/services/`, `data/services/`, `core/service/` |
| `repository-map.md` | 15+ repos: interface → implementation → API → DAO → entity | `domain/repository/`, `data/repository/`, `data/api/`, `data/storage/db/` |
| `route-map.md` | 40+ routes with parameters and nesting | `core/navigation/AppRoute.kt` |
| `di-graph.md` | 16 DI modules: what each provides/binds | `core/di/` |
| `model-map.md` | Domain models organized by API/storage/feature/common | `domain/model/`, `domain/enums/` |
| `feature-index.md` | 37 features: directory structure, VMs, screens, components | `features/` |

Each cache file includes a `Generated: YYYY-MM-DDTHH:MM:SS` header.
A `last-scan-timestamp` file records when the cache was last refreshed.

Output location: `<project-root>/.claude/cache/` (must be gitignored).

Schema example for `service-map.md`:
```markdown
# Service Map
Generated: 2026-03-23T14:30:00

| Service | Interface | Implementation | DI Module | Layer |
|---------|-----------|---------------|-----------|-------|
| Account | IAccountService | AccountService | ServiceModule | core/service/ |
| Entry | IEntryService | EntryService | ServiceModule | data/services/ |
| Dashboard | IDashboardService | DashboardService | ServiceModule | core/service/ |
...
```

`commands/refresh-cache.md`:
```
---
name: refresh-cache
description: Regenerate Layer 2 cache files by scanning the MeApp codebase
---
```

Content: Thin wrapper that calls the `android:refresh-cache` skill.

---

### Phase 3: Create Analyse & Plan Skill

`skills/analyse-plan/SKILL.md`:
```
---
name: analyse-plan
description: Read gathered context (Jira + Figma + cache), identify affected layers and files, perform deduplication check, produce implementation plan with user confirmation
user-invocable: false
---
```

**Sections in the skill:**

1. **Inputs** — Receives: Jira ticket data, Figma screenshots (optional), cache data (selected files)
2. **Load relevant procedures** — Based on detected layers, load procedures from `procedures/data/`, `procedures/ui/`, etc.
3. **Load relevant cache** — Based on Jira keywords, load specific cache files (not all 7)
4. **Runtime scan** — grep/read specific code files to understand current state
5. **Deduplication check** (critical):
   - FIND similar existing interfaces, models, services, enums
   - UNDERSTAND: same domain concept or name collision?
   - RECOMMEND: explain in simple words what was found and why reuse is right or wrong
   - ASK: present findings with options, let user confirm or override
   - NEVER silently decide to reuse based on name similarity alone
6. **Produce plan** — List affected layers, files to create/modify, procedures to load per layer, key design decisions
7. **User confirmation** — Show plan, ask user to confirm or adjust

`commands/analyse.md`:
```
---
name: analyse
description: Analyse a Jira ticket and produce an implementation plan (Step 2 standalone)
argument-hint: "[optional: TICKET-ID]"
---
```

Content: Gets ticket ID, runs jira-fetching + cache-reader, then calls `android:analyse-plan` skill.

---

### Phase 4: Create Implement Layers Skill

`skills/implement-layers/SKILL.md`:
```
---
name: implement-layers
description: Guide the orchestrator through sequential layer-by-layer implementation (Domain → Data → Core → Service → ViewModel → UI) with inline tests
user-invocable: false
---
```

**Sections:**

1. **Input** — Receives the implementation plan from analyse-plan (affected layers, files to create/modify, procedures per layer)
2. **Layer execution order** — Domain → Data (Repository + Storage) → Core (DI) → Service → ViewModel → UI
3. **Per-layer process**:
   - Load relevant procedures for this layer
   - Create/modify files as specified in plan
   - Write inline tests (basic happy path + error path) immediately after each layer
   - Mark layer complete, carry context forward
4. **Skip logic** — If plan says layer is not affected, skip it
5. **Completion signal** — When all layers done, signal "implementation complete" to the orchestrator

---

### Phase 5: Create 4 Validation Agents

Create `agents/` directory at plugin root.

#### `agents/test-runner.md`

```markdown
---
name: test-runner
description: "Run full test suite and JaCoCo coverage analysis. Reports test results and coverage metrics."
model: inherit
color: green
---
```

Body:
1. Run `./gradlew test` — report pass/fail with failure details
2. Run `./gradlew jacocoTestReport` — parse XML for changed classes
3. Report per-class coverage (instructions, branches, lines, methods)
4. Tag findings: CRITICAL (test failure), WARNING (coverage below target), INFO (coverage stats)

#### `agents/lint-analyzer.md`

```markdown
---
name: lint-analyzer
description: "Run detekt static analysis and report violations with file, line, and rule details."
model: inherit
color: yellow
---
```

Body:
1. Run `./gradlew detekt` — parse output
2. Report each violation: file, line, rule, severity
3. Tag findings: CRITICAL (`UnsafeCallOnNullableType` — !! operator), WARNING (method too long, other detekt rules), INFO (style suggestions)

#### `agents/code-reviewer.md`

```markdown
---
name: code-reviewer
description: "Review implementation against MeApp conventions: MVI pattern, null safety, theming, naming, and architectural rules."
model: inherit
color: blue
---
```

Body:
1. **Inline critical rules** (17 hard rules from procedures/SKILL.md):
   - `!!` is banned
   - No hardcoded colors/typography/spacing
   - All interfaces use `I` prefix
   - All API methods are `suspend`
   - Logging uses `AppLog` only
   - Previews use `@PreviewTheme` + `MeAppTheme { }`
   - `super.handleIntent(intent)` always called first
   - Never inject NavigationService/DialogQueueService directly
   - State classes use `@Stable` + `ImmutableList`
   - `collectAsStateWithLifecycle()` not `collectAsState()`
   - `LazyColumn`/`LazyRow` must have `key`
   - No manual `CoroutineScope()`, no `GlobalScope`, no `runBlocking`
   - No `SimpleDateFormat`
   - Methods ≤ 60 lines, classes ≤ 600 lines
   - One composable per file
   - Static text in `strings/` objects
   - No direct push to `dev` or `main`

2. **Deep checks** — load relevant procedure files:
   - If ViewModel changed → load `procedures/ui/mvi-pattern.md`
   - If Compose changed → load `procedures/ui/compose-ui.md`
   - If DI changed → load `procedures/core/dependency-injection.md`
   - If Repository/Service changed → load `procedures/data/services-repositories.md`

3. Tag findings: CRITICAL / WARNING / INFO

#### `agents/spec-checker.md`

```markdown
---
name: spec-checker
description: "Compare implementation against Jira acceptance criteria and optionally Figma designs."
model: inherit
color: cyan
---
```

Body:
1. Receive: Jira ticket data (title, description, acceptance criteria), implementation file list, Figma screenshots (optional)
2. For each acceptance criterion: check if implementation addresses it
3. If Figma designs were captured in Step 1: compare implementation screens against design
4. Report: which criteria are met (✓), which are missing (✗), which are partially met (~)
5. Tag findings: CRITICAL (missing criterion), WARNING (partially met), INFO (all met)

---

### Phase 6: Create Validate Command

`commands/validate.md`:
```
---
name: validate
description: Run 4 parallel validation agents (test-runner, lint-analyzer, code-reviewer, spec-checker), merge findings, fix issues
---
```

Body:
1. Spawn 4 agents in parallel (shared working directory, read-only)
2. Collect all results
3. Merge findings by severity: critical first, then warning, then info
4. Present merged findings to user
5. Orchestrator fixes issues
6. Re-run validation (loop until all pass or 3+ failures)
7. After 3+ failures: show ALL remaining findings, let user decide what to fix/skip/accept

---

### Phase 7: Rewrite Commands (start-task + finish-task)

#### `commands/start-task.md` — Rewrite

Replace the current 16-step monolith with a 6-step orchestrator flow:

**Step 1: Gather** — Spawn 3 agents in parallel:
- Agent A: `android:jira-fetching` skill (fetch ticket, parse ADF, detect subtasks)
- Agent B: `android:design-reference` skill (if Figma links found)
- Agent C: Read `.claude/cache/` files relevant to Jira keywords

Check cache staleness: if `last-scan-timestamp` > 7 days, suggest `/android:refresh-cache` before proceeding.

**Step 2: Analyse & Plan** — Call `android:analyse-plan` skill with gathered context.

**Step 3: Setup** — Sequential:
- `android:jira-estimating` (if not subtask)
- `android:jira-transitioning` → In Progress
- Create git branch (detect base from parent ticket or dev/main)

**Step 4: Implement** — Call `android:implement-layers` skill with the plan.
- Orchestrator follows the plan layer by layer
- Loads procedures dynamically per layer
- Writes tests inline

**Step 5: Validate** — Call the validation flow (same logic as `commands/validate.md`):
- Spawn 4 agents in parallel
- Merge findings, fix, re-validate
- Loop or escalate after 3+ failures

**Step 6: Finalize** — Spawn 3 tasks in parallel:
- `android:creating-pr` skill
- `android:jira-worklogging` skill
- `android:jira-transitioning` → In Review

**Subtask handling:**
- If parent ticket has subtasks: gather/plan once for parent
- Each subtask runs steps 3-6 independently (own branch, own PR)

**Keep:** All existing hard rules (no MCP, no push without instruction, Jira ticket ID prefix, etc.)

#### `commands/finish-task.md` — Rewrite

Thin wrapper that runs Steps 5-6 from start-task. For when user started work manually without `/android:start-task`.

1. Extract ticket ID from branch name
2. Verify CLI setup (ACLI + gh)
3. Run Step 5 (validate) — same as start-task
4. Run Step 6 (finalize) — same as start-task

References the same validation and finalize logic. No duplication.

---

### Phase 8: Update Plugin Configuration

`plugin.json` — No changes needed. Claude Code auto-discovers agents from `agents/` directory.

Add `.claude/cache/` to project `.gitignore` (at the MeApp repo level, not the plugin repo).

---

## Alternative Approaches Considered

1. **Write 30+ static procedure files** — Rejected. Too much manual maintenance, content drifts from codebase reality. Layer 2 cache solves this with auto-generation. (see brainstorm: Decision #7)

2. **Agent teams during implementation** — Rejected. Implementation layers depend on each other (Service needs Repository interface). Parallel agents would break contracts. Orchestrator needs context continuity. (see brainstorm: Decision #2)

3. **Worktrees for validation agents** — Rejected. All 4 agents are read-only. Shared directory is faster. Worktrees add overhead for no benefit. (see brainstorm: Decision #3)

4. **Absorb finish-task into start-task** — Rejected. Users need a standalone entry point for validation/finalize when they started work manually. Both commands share the same logic via skills. (see brainstorm: Decision #1)

## System-Wide Impact

### Interaction Graph

```
/android:start-task
  ├── Step 1: jira-fetching, design-reference, cache-reader (parallel)
  ├── Step 2: analyse-plan (calls procedures, cache, runtime scan)
  ├── Step 3: jira-estimating, jira-transitioning, git (sequential)
  ├── Step 4: implement-layers (calls procedures per layer)
  ├── Step 5: test-runner, lint-analyzer, code-reviewer, spec-checker (parallel)
  └── Step 6: creating-pr, jira-worklogging, jira-transitioning (parallel)

/android:finish-task
  ├── Step 5: same validation agents
  └── Step 6: same finalize agents

/android:validate → Step 5 only
/android:analyse → Steps 1-2 only
/android:refresh-cache → regenerates .claude/cache/
```

### Error Propagation

- Step 1 agent failure → orchestrator reports which agent failed, offers retry
- Step 2 plan rejected by user → loops back to ask what to adjust
- Step 4 build failure mid-implementation → stop, show error, let user fix
- Step 5 validation failure → orchestrator fixes, re-runs (max 3 loops)
- Step 6 PR creation failure → offer retry or skip

### State Lifecycle

- Cache files: generated by refresh-cache, read-only during tasks, stale after 7 days
- Implementation plan: exists in orchestrator memory during step 4, not persisted
- Validation findings: collected from 4 agents, merged by orchestrator, discarded after fixes

## Dependencies & Prerequisites

- ACLI installed and authenticated (`~/.jira-config`)
- GitHub CLI installed and authenticated
- MeApp Android project accessible at the working directory
- `.claude/cache/` directory writable (will be created by refresh-cache)

## Risk Analysis

| Risk | Mitigation |
|------|-----------|
| Cache becomes stale, leads to wrong analysis | Auto-stale detection (7 days). Suggest refresh before critical tasks. |
| Dedup check makes wrong recommendation | Always asks user. Never silently decides. User overrides. |
| 4 parallel agents overwhelm system resources | Agents are lightweight (read-only). No worktrees. Shared directory. |
| Procedure files break after move to sub-folders | Phase 1 only moves files + updates index. No content changes. Verify links. |
| start-task rewrite introduces regressions | Phase 7 is last. All underlying skills/agents are tested individually first. |

## Files Changed Summary

### New Files (14)

| File | Phase | Purpose |
|------|-------|---------|
| `agents/test-runner.md` | 5 | Validation agent — tests + JaCoCo |
| `agents/lint-analyzer.md` | 5 | Validation agent — detekt |
| `agents/code-reviewer.md` | 5 | Validation agent — conventions |
| `agents/spec-checker.md` | 5 | Validation agent — Jira/Figma check |
| `skills/analyse-plan/SKILL.md` | 3 | Step 2: analyse + plan |
| `skills/implement-layers/SKILL.md` | 4 | Step 4: layer-by-layer guide |
| `skills/refresh-cache/SKILL.md` | 2 | Layer 2 cache generator |
| `commands/validate.md` | 6 | Step 5 standalone |
| `commands/analyse.md` | 3 | Step 2 standalone |
| `commands/refresh-cache.md` | 2 | Cache regen trigger |
| `skills/procedures/domain/` | 1 | Directory (empty initially) |
| `skills/procedures/data/` | 1 | Directory for data procedures |
| `skills/procedures/core/` | 1 | Directory for core procedures |
| `skills/procedures/ui/` | 1 | Directory for UI procedures |

### Moved Files (12)

| From | To | Phase |
|------|-----|-------|
| `procedures/storage.md` | `procedures/data/storage.md` | 1 |
| `procedures/networking-api.md` | `procedures/data/networking-api.md` | 1 |
| `procedures/services-repositories.md` | `procedures/data/services-repositories.md` | 1 |
| `procedures/dependency-injection.md` | `procedures/core/dependency-injection.md` | 1 |
| `procedures/mvi-pattern.md` | `procedures/ui/mvi-pattern.md` | 1 |
| `procedures/compose-ui.md` | `procedures/ui/compose-ui.md` | 1 |
| `procedures/navigation-dialogs.md` | `procedures/ui/navigation-dialogs.md` | 1 |
| `procedures/testing.md` | `procedures/shared/testing.md` | 1 |
| `procedures/test-patterns.md` | `procedures/shared/test-patterns.md` | 1 |
| `procedures/test-troubleshooting.md` | `procedures/shared/test-troubleshooting.md` | 1 |
| `procedures/test-coverage.md` | `procedures/shared/test-coverage.md` | 1 |
| `procedures/review-checklist.md` | `procedures/shared/review-checklist.md` | 1 |

### Rewritten Files (3)

| File | Phase | Change |
|------|-------|--------|
| `commands/start-task.md` | 7 | Full rewrite — 16 steps → 6-step orchestrator |
| `commands/finish-task.md` | 7 | Rewrite — shares steps 5-6 via skill references |
| `skills/procedures/SKILL.md` | 1 | Update index table with new sub-folder paths |

### Auto-Generated Files (7, gitignored)

| File | Generated by | Content |
|------|-------------|---------|
| `.claude/cache/component-catalog.md` | refresh-cache | 99+ shared composables |
| `.claude/cache/service-map.md` | refresh-cache | 25+ services mapped |
| `.claude/cache/repository-map.md` | refresh-cache | 15+ repos mapped |
| `.claude/cache/route-map.md` | refresh-cache | 40+ routes mapped |
| `.claude/cache/di-graph.md` | refresh-cache | 16 DI modules mapped |
| `.claude/cache/model-map.md` | refresh-cache | Domain models mapped |
| `.claude/cache/feature-index.md` | refresh-cache | 37 features indexed |

### Unchanged Files (13)

All existing skills (jira-setup, gh-setup, jira-fetching, jira-estimating, jira-transitioning, jira-worklogging, creating-pr, design-reference, verifying-build) and write-tests command remain unchanged.

## Execution Order

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7 → Phase 8
  │          │          │          │          │          │          │          │
  │          │          │          │          │          │          │          └─ .gitignore
  │          │          │          │          │          │          └─ start-task + finish-task
  │          │          │          │          │          └─ validate command
  │          │          │          │          └─ 4 agents
  │          │          │          └─ implement-layers skill
  │          │          └─ analyse-plan skill + analyse command
  │          └─ refresh-cache skill + command
  └─ Move 12 procedure files + update index
```

Each phase is independently testable. If phase N breaks, phases 1..N-1 still work.

## Sources & References

### Origin

- **Brainstorm document:** [docs/brainstorms/2026-03-23-orchestrator-agent-team-restructure-brainstorm.md](docs/brainstorms/2026-03-23-orchestrator-agent-team-restructure-brainstorm.md) — Key decisions: 6-step pipeline, 3-layer knowledge system, dedup check before creating, severity-based fix priority, dynamic cache/procedure loading

### Internal References

- Agent format reference: compound-engineering `agents/workflow/lint.md` (frontmatter: name, description, model, color)
- Agent format reference: superpowers `agents/code-reviewer.md` (description with inline examples)
- Existing plan: `docs/plans/2026-03-16-001-refactor-skills-from-viewmodel-testing-lessons-plan.md` — patterns for auto-detection, dynamic targets, context caching
- Existing plan: `docs/plans/2026-03-14-001-refactor-jira-skills-mcp-to-acli-plan.md` — ACLI/curl patterns, setup skill prerequisites
