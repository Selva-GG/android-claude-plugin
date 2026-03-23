# Brainstorm: Orchestrator + Agent Team Plugin Restructure

**Date:** 2026-03-23
**Status:** Decided
**Scope:** Full restructure of android-claude-plugin from linear sequential workflow to orchestrator + agent team architecture

---

## What We're Building

A 6-step development pipeline that uses an **orchestrator** (main agent) for context-dependent work and **agent teams** (parallel sub-agents) for independent tasks.

### The 6 Steps

| Step | Name | Who | Parallel? |
|------|------|-----|-----------|
| 1 | Gather Context | Agent team | Yes — Jira + Figma + codebase scan |
| 2 | Analyse & Plan | Orchestrator | No — needs all context to plan |
| 3 | Setup | Orchestrator | No — sequential (estimate, transition, branch) |
| 4 | Implement | Orchestrator | No — layer-by-layer, context continuity |
| 5 | Validate | Agent team | Yes — 4 validators in parallel |
| 6 | Finalize | Agent team | Yes — PR + worklog + Jira transition |

### Core Principle

- **Orchestrator** = the developer brain. Does planning, implementation, fixing.
- **Agents** = the hands. Do parallel grunt work (research, validation, finalization).
- Agents don't talk to each other. They report to the orchestrator.

---

## Why This Approach

**Problems with current linear flow:**
1. **Speed** — Everything runs sequentially. Jira fetch waits, then Figma, then codebase scan. Validation runs build → test → lint one by one.
2. **Quality** — No structured analysis step. Implementation starts without a plan. Validation is just build/test/lint — no code review or spec check.
3. **Structure** — start-task is a monolithic 16-step command. No separation between gathering, planning, implementing, and validating.

**What this restructure solves:**
- Steps 1, 5, 6 run agents in parallel → faster
- Step 2 (analyse-plan) forces structured planning before coding
- Step 4 (implement-layers) enforces bottom-up layer discipline
- Step 5 adds code-reviewer and spec-checker beyond just build/test/lint

---

## Key Decisions

### 1. Commands: Keep both, share steps via skill references
- `start-task` = full lifecycle (Steps 1-6)
- `finish-task` = standalone for Steps 5-6 (when user started work manually)
- Sharing mechanism: both commands reference the same `analyse-plan`, `implement-layers`, and validation agent definitions. No duplicated instructions — commands are thin orchestration wrappers that call shared skills/agents.

### 2. Implementation is sequential, by the orchestrator
- Orchestrator implements all layers itself — needs context continuity
- Order: Domain → Data (Repository + Storage) → Service → ViewModel → UI
- Auto-detects affected layers from the plan, skips unused ones
- Orchestrator writes tests inline alongside each layer's code (basic happy + error paths)
- Full JaCoCo coverage analysis runs later in Step 5 via the test-runner agent

### 3. Validation agents share working directory (no worktrees)
- All 4 validation agents are read-only — they report findings, never modify code
- Shared directory is faster. Worktrees only if conflicts arise (adaptive).
- Orchestrator does all fixing after collecting results.

### 4. Procedures reorganized by layer + shared
- Current flat 12 files → grouped into layer sub-folders
- New structure:

```
procedures/
├── SKILL.md                        (main index — updated)
├── domain/                         (new — minimal, pure Kotlin rules)
├── data/
│   ├── storage.md                  (moved from root)
│   ├── networking-api.md           (moved from root)
│   └── services-repositories.md    (moved from root)
├── core/
│   └── dependency-injection.md     (moved from root)
├── ui/
│   ├── mvi-pattern.md              (moved from root)
│   ├── compose-ui.md               (moved from root)
│   └── navigation-dialogs.md       (moved from root)
└── shared/
    ├── testing.md                  (moved from root)
    ├── test-patterns.md            (moved from root)
    ├── test-troubleshooting.md     (moved from root)
    ├── test-coverage.md            (moved from root)
    └── review-checklist.md         (moved from root)
```

### 5. Code-reviewer: critical rules inline + reference files for depth
- 17 hard rules (!! banned, I prefix, @PreviewTheme, etc.) embedded inline in agent definition
- For deeper MVI/Compose/testing checks, agent loads relevant procedure files at runtime
- Best of both: fast for common checks, thorough for deep checks

### 6. Spec-checker: Jira always, Figma optional
- Always checks Jira acceptance criteria against implementation
- Only checks Figma if design references were captured in Step 1 gather phase
- Adaptive — lightweight for backend tasks, thorough for UI tasks

### 7. 3-Layer Knowledge System (hybrid approach)

Instead of writing 30+ static procedure files for every codebase area, use a hybrid:

**Layer 1: Static Procedures (hand-written, rarely change)**
- Rules that are DECISIONS, not facts
- The existing 12 procedure files, reorganized by layer
- Examples: "!! is banned", "use MeAppTheme tokens", "I prefix on interfaces"
- These are guardrails — the orchestrator loads them as rules

**Layer 2: Pre-Analysed Cache (auto-generated, refreshed periodically)**
- A scan of the entire codebase, run once (or on-demand with `/android:refresh-cache`)
- Produces structured .md files mapping the codebase
- Location: project-local at `<project>/.claude/cache/` (gitignored)
- Cache files:

```
.claude/cache/
├── component-catalog.md    ← All 99 shared composables with signatures
├── service-map.md          ← 25 services: interface → implementation → DI module
├── repository-map.md       ← 15 repos: interface → impl → API → DAO → entity
├── route-map.md            ← 40+ routes with parameters
├── di-graph.md             ← 16 DI modules: what provides what
├── model-map.md            ← Domain models: API vs storage vs feature
├── feature-index.md        ← 37 features: files, structure, VMs, screens
└── last-scan-timestamp     ← When cache was last refreshed
```

- These are FACTS derived from code — not opinions or rules
- Auto-generated = always accurate, zero manual maintenance
- Refreshed via `/android:refresh-cache` or auto-suggested when >7 days old

**Cache file schema example** (component-catalog.md):
```markdown
# Component Catalog
Generated: 2026-03-23T14:30:00

## Buttons
| Component | File | Props | Usage |
|-----------|------|-------|-------|
| AppButton | features/common/components/AppButton.kt | text, onClick, enabled, style | Primary action button |
| AppFabButton | features/common/components/AppFabButton.kt | icon, onClick | Floating action |

## Inputs
| Component | File | Props | Usage |
|-----------|------|-------|-------|
| AppInput | features/common/components/AppInput.kt | value, onValueChange, label, imeAction | Standard text input |
...
```

**Cache file schema example** (service-map.md):
```markdown
# Service Map
Generated: 2026-03-23T14:30:00

| Service | Interface | Implementation | DI Module | Layer |
|---------|-----------|---------------|-----------|-------|
| Account | IAccountService | AccountService | ServiceModule | core/service/ |
| Entry | IEntryService | EntryService | ServiceModule | data/services/ |
...
```

**Layer 3: Runtime Analysis (per-task, fresh every time)**
- The analyse-plan skill reads Layer 1 + Layer 2 + Jira ticket
- Scans SPECIFIC files to determine what needs changing for THIS task
- Produces the implementation plan with exact files to create/modify

**How they combine at runtime:**
```
Layer 1 (static)  → "How should code look?" (rules)
Layer 2 (cached)  → "What already exists?" (facts about codebase)
Layer 3 (runtime) → "What needs changing for this task?" (task-specific)
```

**Dynamic loading — nothing is loaded blindly:**
- The analyse-plan skill reads the Jira ticket first
- Based on keywords and context, it selects ONLY relevant cache files and procedures
- Example: "add new repository" → loads repository-map.md + service-map.md + procedures/data/*.md
- Example: "fix dashboard UI" → loads component-catalog.md + feature-index.md + procedures/ui/*.md
- Example: "new feature end-to-end" → loads ALL cache files (needs full picture)
- This keeps context small and focused per task, not bloated with irrelevant info

### 8. Deduplication check before creating anything (critical rule)

The analyse-plan skill must CHECK for similar existing code before proposing new files. But finding something similar does NOT mean reusing it — name similarity is not domain similarity.

**The flow:**
1. **FIND** — search cache + codebase for similar interfaces, models, services, enums
2. **UNDERSTAND** — is it the same domain concept, or just a name collision?
3. **RECOMMEND** — explain in simple words what was found, why reuse is right or wrong
4. **ASK** — present findings to user with clear options, let user confirm or override

**Hard rule:** Never silently decide to reuse or extend. Always present findings and reasoning to the user. The user knows the domain better than the orchestrator.

**What the user sees:**
```
I found some existing code that looks related:

1. DashboardType enum has BABY_SCALE
   → But it controls which dashboard tab to show, not which product is selected.
   → I recommend creating a NEW ProductType enum.

2. DeviceService (23KB) has deviceListFlow
   → It has the data we need, but it's already too large to extend.
   → I recommend creating a NEW ProductSelectionManager that reads from DeviceService.

Do you agree? Or should I reuse any of these?
```

### 9. Cache format and location
- Format: Markdown (.md) — human-readable, Claude reads naturally
- Location: `<project-root>/.claude/cache/` — per-project, gitignored
- New skill: `/android:refresh-cache` — regenerates all Layer 2 cache files
- Auto-stale detection: if `last-scan-timestamp` > 7 days, suggest refresh

---

## New Files to Create

### Agents (4 new)

| Agent | Purpose | Runs in |
|-------|---------|---------|
| `agents/test-runner.md` | Run `./gradlew test`, report pass/fail with details | Step 5 |
| `agents/lint-analyzer.md` | Run `./gradlew detekt`, report violations | Step 5 |
| `agents/code-reviewer.md` | Check MVI, null safety, theming, naming conventions | Step 5 |
| `agents/spec-checker.md` | Compare implementation vs Jira requirements + Figma | Step 5 |

### Skills (3 new)

| Skill | Purpose | Used in |
|-------|---------|---------|
| `skills/analyse-plan/SKILL.md` | Read all gathered context, identify affected layers/files, produce implementation plan | Step 2 |
| `skills/implement-layers/SKILL.md` | Guide orchestrator through Domain → Data → Service → ViewModel → UI with tests at each step | Step 4 |
| `skills/refresh-cache/SKILL.md` | Scan entire codebase, generate Layer 2 cache files (.claude/cache/*.md) | On-demand / auto-stale |

### Commands (2 rewritten + 3 new)

| Command | Change |
|---------|--------|
| `commands/start-task.md` | Rewrite to 6-step orchestrator flow with agent spawning |
| `commands/finish-task.md` | Rewrite to share Steps 5-6 from start-task, standalone entry point |
| `commands/validate.md` | NEW — Step 5 standalone (4 parallel validation agents) |
| `commands/analyse.md` | NEW — Step 2 standalone (analyse + plan only) |
| `commands/refresh-cache.md` | NEW — Trigger Layer 2 cache regeneration |

### Procedures (restructured, not rewritten)

Move existing files into layer sub-folders. Update SKILL.md index. No content changes needed — just reorganization.

---

## What Stays Unchanged

| File | Why |
|------|-----|
| `skills/jira-setup/` | Setup infrastructure — still needed |
| `skills/gh-setup/` | Setup infrastructure — still needed |
| `skills/jira-fetching/` | Used by Step 1 gather agents |
| `skills/jira-estimating/` | Used by Step 3 setup |
| `skills/jira-transitioning/` | Used by Steps 3 and 6 |
| `skills/jira-worklogging/` | Used by Step 6 finalize |
| `skills/creating-pr/` | Used by Step 6 finalize |
| `skills/verifying-build/` | Kept as reference. test-runner and lint-analyzer agents call this skill internally for build/test/lint commands. Not used directly by commands anymore. |
| `skills/design-reference/` | Used by Step 1 gather agents |
| `commands/write-tests.md` | Standalone — independent of the pipeline |

---

## Resolved Questions

1. **Should finish-task be absorbed?** → No. Keep both commands, share validation/finalize steps.
2. **Should procedures be reorganized?** → Yes. By layer (domain/, data/, core/, ui/) + shared/.
3. **Should validation agents use worktrees?** → No. Shared directory, read-only. Adaptive — worktree as fallback.
4. **How should code-reviewer get rules?** → Both: critical rules inline + reference procedure files for depth.
5. **Should spec-checker include Figma?** → Jira always, Figma only if designs were captured in Step 1.
6. **Should implementation use agent teams?** → No. Orchestrator does it sequentially — needs context continuity across layers.
7. **Can within-layer work be parallelized?** → Yes in theory (independent domains), but only when task spans multiple domains. Not the default.
8. **Fix priority when agents conflict?** → Severity-based. Each agent tags findings as critical/warning/info. Fix criticals first across all agents, then warnings.
9. **How does analyse-plan detect affected layers?** → Both keyword matching from Jira + codebase scan, then present detected layers to user for confirmation.
10. **When are tests written?** → Orchestrator writes tests inline during implementation. No separate /write-tests invocation per layer. Full JaCoCo analysis during validation step.
11. **How do subtasks work?** → Shared gather/plan for parent ticket. Each subtask only does Steps 3-6 (branch, implement, validate, PR).
12. **What happens after 3+ validation failures?** → Show ALL remaining findings across all 4 agents. Let user pick which to fix, skip, or accept. No auto-draft PR.
13. **Should individual steps be exposed as standalone commands?** → Yes. Add `/android:validate` (Step 5) and `/android:analyse` (Step 2) as standalone commands.

---

## Summary of Changes

```
CURRENT                              NEW
─────────────────────────────       ─────────────────────────────
commands/                            commands/
  start-task.md (16 steps)     →      start-task.md (6-step orchestrator)
  finish-task.md (9 steps)     →      finish-task.md (shared steps 5-6)
  write-tests.md               →      write-tests.md (unchanged)
                                      validate.md (NEW — step 5 standalone)
                                      analyse.md (NEW — step 2 standalone)
                                      refresh-cache.md (NEW — Layer 2 cache regen)

skills/                              skills/
  (10 existing)                →      (10 existing — unchanged)
                                      analyse-plan/SKILL.md (NEW)
                                      implement-layers/SKILL.md (NEW)
                                      refresh-cache/SKILL.md (NEW)

  procedures/                        procedures/
    SKILL.md                   →      SKILL.md (updated index)
    12 flat files              →      domain/ data/ core/ ui/ shared/

                                     agents/ (NEW DIRECTORY)
                                       test-runner.md (NEW)
                                       lint-analyzer.md (NEW)
                                       code-reviewer.md (NEW)
                                       spec-checker.md (NEW)

                                     .claude/cache/ (AUTO-GENERATED, gitignored)
                                       component-catalog.md
                                       service-map.md
                                       repository-map.md
                                       route-map.md
                                       di-graph.md
                                       model-map.md
                                       feature-index.md
                                       last-scan-timestamp
```

**Total: 4 new agents, 3 new skills, 5 commands (2 rewritten + 3 new), 12 restructured procedure files, 7 cache files (auto-generated).**
