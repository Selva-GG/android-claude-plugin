---
title: "refactor: Improve write-tests, start-task, finish-task skills from ViewModel testing lessons"
type: refactor
status: active
date: 2026-03-16
origin: docs/brainstorms/2026-03-16-skill-improvements-from-viewmodel-testing-brainstorm.md
---

# Improve Android Plugin Skills from ViewModel Testing Lessons

## Context

After writing tests for 10 ViewModels (MA-3433–MA-3442), multiple skill gaps were identified. The skills hardcode JUnit 4, can't handle infinite coroutine loops, don't support subtask workflows, and have overly strict constant extraction. This plan addresses all 17 improvements from the brainstorm.

## Files to Modify

| # | File | Changes |
|---|------|---------|
| 1 | `commands/write-tests.md` | A1–A7 (7 improvements) |
| 2 | `commands/start-task.md` | B1–B3 (3 improvements) |
| 3 | `commands/finish-task.md` | C1, C3, C4 (3 improvements) |
| 4 | `skills/jira-worklogging.md` | C2, C3 (2 improvements) |
| 5 | `skills/creating-pr.md` | C1, C5 (2 improvements) |
| 6 | `skills/procedures/testing.md` | D1 (1 improvement) |

New file:
| 7 | `skills/procedures/test-context-cache.md` | D2 (context persistence) |

---

## Phase 1: `commands/write-tests.md`

### A1: Auto-detect JUnit version (Step 0.2)

Replace the hardcoded JUnit 4 assumption in the test framework detection table.

**Current (Step 0.2):**
```
| Test framework | `testImplementation` — JUnit 4 (`junit:junit`) or JUnit 5 (`org.junit.jupiter`) |
```

**Replace with:**
```markdown
| Test framework | `testImplementation` — scan for: |
|                | • `org.junit.jupiter` → **JUnit 6** (default) |
|                | • `junit:junit` → **JUnit 4** (legacy) |
|                | • If both present → use JUnit 6 for new tests |

### 0.2b JUnit Version Mapping

After detecting the JUnit version, use this mapping for ALL generated test code:

| Concept | JUnit 4 | JUnit 6 |
|---------|---------|---------|
| Test annotation | `org.junit.Test` | `org.junit.jupiter.api.Test` |
| Setup | `@Before` | `@BeforeEach` |
| Teardown | `@After` | `@AfterEach` |
| Rule | `@get:Rule` | `@JvmField @RegisterExtension` |
| Exception test | try/catch + `fail()` | `assertThrows<Exception> { }` |
| Imports prefix | `org.junit.` | `org.junit.jupiter.api.` |
| MainDispatcherRule base | `TestWatcher` | `BeforeEachCallback, AfterEachCallback` |

**Store detected version in session context as `junitVersion: 4 | 6`.**
```

Also update Step 0.3 "Learn Existing Test Patterns" — change:
```
- **Setup pattern**: `@Before fun setUp()` with `MockKAnnotations.init(this)` vs manual init
```
to:
```
- **Setup pattern**: `@BeforeEach` (JUnit 6) or `@Before` (JUnit 4) with `MockKAnnotations.init(this)` vs manual init
```

### A2: Init Block Analysis (new Step 2.5)

Insert after current Step 2.4 (Known Pitfalls Detection):

```markdown
### 2.5 Init Block Coroutine Analysis

Scan the class init block (and parent class init) for infinite coroutine patterns:

| Pattern | Detection | Risk |
|---------|-----------|------|
| `while (!isDestroyed)` + `delay()` | Grep for `while\s*\(` + `delay\(` in same method | `advanceUntilIdle()` hangs forever |
| `state.collect {` | Grep for `state\.collect` in init | `runTest` cleanup hangs (60s timeout per test) |
| `.collect {` on hot Flow | Any `.collect` on a Flow that never completes | `runTest` cleanup hangs |
| Polling with `delay()` | `delay()` inside a loop | `advanceUntilIdle()` chases infinite iterations |

**If ANY infinite pattern detected:**

Flag: "⚠ Infinite coroutine detected in init block"

**Test-side remediation (apply automatically):**
1. Use `StandardTestDispatcher()` (NOT `UnconfinedTestDispatcher`)
2. Never use `runTest` — tests must be plain functions
3. Never use `advanceUntilIdle()` — use `advanceTimeBy(200)` + `runCurrent()`
4. Add `@AfterEach` that calls `onCleared()` via reflection:
   ```kotlin
   @AfterEach
   fun tearDown() {
       val method = viewModel::class.java.getDeclaredMethod("onCleared")
       method.isAccessible = true
       method.invoke(viewModel)
   }
   ```
5. Add helper:
   ```kotlin
   private fun advanceScheduler() {
       testDispatcher.scheduler.advanceTimeBy(200)
       testDispatcher.scheduler.runCurrent()
   }
   ```

**ViewModel-side suggestions (present to user, don't auto-apply):**
- Replace `state.collect { }` with `.map { relevantFields }.distinctUntilChanged().collect { }`
- Flag `while(true) + delay()` loops as candidates for Flow-based alternatives

**If NO infinite patterns detected:** Use standard `UnconfinedTestDispatcher` + `runTest` pattern.
```

### A3: @AssistedInject Detection (Step 2.1)

Add to the Dependency Map section:

```markdown
### 2.1b @AssistedInject Detection

If the class constructor has `@AssistedInject`:
1. Identify `@Assisted` parameters — these are runtime values, not injected dependencies
2. In test setup, pass them as regular constructor arguments:
   ```kotlin
   viewModel = FooViewModel(
       assistedParam = "test-value",    // @Assisted — use test constant
       service = mockService,            // @Inject — use mock
   )
   ```
3. Do NOT try to mock assisted parameters
4. Add assisted param values to `companion object` constants
```

### A4: Dynamic Coverage Targets (Step 7)

Add to Step 7.1 before running JaCoCo:

```markdown
### 7.0 Determine Coverage Target

Scan the source file imports and line count:

**Hardware ViewModel indicators** (any match = hardware):
- Imports containing: `GGDeviceService`, `WifiScaleService`, `GGPermissionService`, `BluetoothAdapter`, `WifiManager`, `BLEDiscoveryManager`, `WiFiConfigManager`
- Class body > 800 lines
- Extends a class with "Scale" or "Ble" or "Wifi" in the name

**Targets:**
| Type | Instructions | Branches | Lines |
|------|-------------|----------|-------|
| Standard class | 95% | 90% | 95% |
| Hardware ViewModel | 50% | 30% | 50% |

Display the selected target before running JaCoCo.
```

### A5: Auto-detect Test Infrastructure (Step 0.3)

Add after the existing pattern detection:

```markdown
### 0.3b Detect Test Infrastructure

Search the test source tree for project-specific test utilities:

```bash
find . -path "*/src/test/*" -name "*Extensions*.kt" -o -name "*Fixtures*.kt" -o -name "*TestHelper*.kt"
```

For each found, read and note:
- **ViewModel extension functions** (e.g., `initTestDependencies()`) — use in all ViewModel tests
- **Test fixture objects** (e.g., `TestFixtures.activeAccount`) — use instead of complex mocks
- **Custom test rules/extensions** (e.g., `MainDispatcherRule`) — use instead of creating new ones
- **Base test classes** — extend if project convention

Store in session context for reuse across test classes.
```

### A6: Relax Constant Extraction (Step 5)

Replace the current "BLOCKING GATE" language:

**Current title:** `## Step 5: Extract Constants — BLOCKING GATE`
**New title:** `## Step 5: Extract Constants`

Replace the strict rules with:

```markdown
### 5.1 Smart Extraction Rules

Extract to `companion object` when:
- Value is used in **2+ tests**
- Numeric test data (except `0`, `1`, `-1`, `true`, `false`)
- String values used as test identifiers (IDs, names, emails)

**Allow inline when:**
- String inside `match { }` lambda (e.g., `match<Toast> { it.title == "Success!" }`)
- Single-use value in exactly 1 test with clear meaning
- Error messages that are assertion targets
- Boolean/null literals

Remove all "MANDATORY", "NON-NEGOTIABLE", "BLOCKING GATE" language.
```

### A7: Batch Mode (Step 0.1 + new Step 8)

Update Step 0.1:

```markdown
### 0.1 Parse Arguments

Extract from the argument:
- **ClassNames**: one or more classes to test (e.g., `LoginViewModel SignupViewModel`)
- **--refresh**: if present, force rescan even if cached

If multiple classes provided:
- Run Step 0 once (cache context)
- Loop Steps 1–7 for each class
- After all classes: display Step 8 batch summary
```

Add new Step 8:

```markdown
## Step 8: Batch Summary (multi-class only)

If multiple classes were tested, display:

| Class | Tests | Instructions | Branches | Lines |
|-------|-------|-------------|----------|-------|
| LoginViewModel | 32 | 95.8% | 100% | 100% |
| SignupViewModel | 29 | 96.5% | 92% | 98% |
| **Total** | **61** | | | |
```

---

## Phase 2: `commands/start-task.md`

### B1: Subtask Awareness (after Step 2)

Insert after Step 2:

```markdown
## Step 2b: Check for Subtasks

After fetching the ticket, check if it has subtasks:

```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>?fields=subtasks" \
  | python3 -c "import json,sys; subtasks=json.load(sys.stdin).get('fields',{}).get('subtasks',[]); [print(f'{s[\"key\"]}: {s[\"fields\"][\"summary\"]}') for s in subtasks]"
```

**If subtasks found:**
> This ticket has N subtasks:
> 1. TICKET-1: Summary
> 2. TICKET-2: Summary
> ...
>
> Work through them one by one? (yes / no — just work on parent)

If yes:
- Create parent branch first (if not exists)
- For each subtask: create branch from parent, implement, PR to parent, worklog, transition
- Track progress across subtasks
```

### B2: Smart Base Branch (Step 6)

Add to Step 6 branch creation logic:

```markdown
### Smart Base Branch Detection

Before creating the branch:

1. Check if ticket is a subtask (has `parent` field in Jira response)
2. If subtask:
   ```bash
   git branch -r --list "*<PARENT-TICKET-ID>*"
   ```
3. If parent branch found:
   > This is a subtask of <PARENT-ID>. Use `<parent-branch>` as base? (yes / use main)
4. Default: use parent branch for subtasks
```

### B3: Skip Estimate for Subtasks (Step 4)

Replace unconditional estimate call:

```markdown
## Step 4: Review Time Estimate

**If ticket type is "Subtask":**
- If no estimate exists: skip this step silently
- If estimate exists: display but don't prompt to change

**Otherwise:** Use the `android:jira-estimating` skill as before.
```

---

## Phase 3: `commands/finish-task.md`

### C1: Auto-include Coverage Data (Step 6)

Add to Step 6 before calling `creating-pr`:

```markdown
**Coverage data handoff:**
If `android:write-tests` was run in this session, the coverage tables are in session context. Pass them to `android:creating-pr` so they're auto-included in the PR body under a "## Coverage" section.
```

### C3: Inline Worklog Args

Update the ARGUMENTS section at the top:

```markdown
**Arguments format:** `[TICKET-ID] [worklog TIME [DATE]]`

Examples:
- `/android:finish-task` — interactive
- `/android:finish-task MA-3433 worklog 10m` — log 10m today
- `/android:finish-task MA-3433 worklog 10m 2026-03-16` — log 10m on specific date

Parse args before Step 1. If worklog time and date provided, skip the worklog prompt in Step 7.
```

### C4: Auto-transition (Step 8)

Replace:
```markdown
**Only if a PR was created in Step 6.**
Ask: "Move ticket to In Review?"
```

With:
```markdown
**If a PR was created in Step 6:** Auto-transition to "In Review" without asking.
**If no PR was created:** Ask: "No PR was created. Still move ticket to In Review?"
```

---

## Phase 4: `skills/jira-worklogging.md`

### C2: REST API Primary

Replace ACLI worklog fetch with curl:

```markdown
## Step 1: Show Existing Worklogs

```bash
source ~/.jira-config && curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  "https://$JIRA_SITE/rest/api/3/issue/<TICKET-ID>/worklog" \
  | python3 -c "..."
```

**Do not use ACLI for worklog operations.** ACLI's worklog subcommand has unreliable flags. Use curl + REST API v3 for all worklog reads and writes.
```

### C3: Inline Args Parsing

Add at the top of the skill:

```markdown
## Step 0: Parse Arguments

If arguments include time and date:
- Parse time: `10m`, `1h`, `30m`, `2h30m`
- Parse date: `YYYY-MM-DD` or `today` (default)
- If both provided: skip interactive prompt, go directly to Step 3 (submit)
- If only time provided: use today's date, skip time prompt
- If neither: run interactive flow as before
```

---

## Phase 5: `skills/creating-pr.md`

### C5: Auth Scope Fallback

Add to error handling section:

```markdown
### gh auth scope errors

If `gh pr create` or `gh pr edit` fails with "missing required scopes":
1. **First fallback:** Use `gh api` directly:
   ```bash
   gh api repos/OWNER/REPO/pulls --method POST --field title="..." --field body="..."
   ```
2. **Last resort:** Suggest `gh auth refresh -h github.com -s read:project`
```

### C1: Coverage Table Integration

Clarify the existing placeholder:

```markdown
### Coverage Data from write-tests

If coverage tables exist in session context (from `android:write-tests`):
- Insert each table under a `## JaCoCo Coverage` heading in the PR body
- Format: one markdown table per tested class
- Include notes about unreachable code if present
- Place after "## Summary" and before "## Test plan"
```

---

## Phase 6: `skills/procedures/testing.md`

### D1: JUnit 6 Default

Update all code examples to JUnit 6 annotations. Add a "JUnit 4 Legacy" collapsible section.

**Key changes:**
- `@Before` → `@BeforeEach`
- `@After` → `@AfterEach`
- `@get:Rule` → `@JvmField @RegisterExtension`
- `org.junit.Test` → `org.junit.jupiter.api.Test`
- `MainDispatcherRule : TestWatcher()` → `MainDispatcherRule : BeforeEachCallback, AfterEachCallback`
- Remove `com.dmdbrands.gurus.weight` package references — use `[package]` placeholder
- Add note: "Detect JUnit version from build.gradle.kts. These examples use JUnit 6 (default). See JUnit 4 Legacy section below for older projects."

---

## Phase 7: Context Persistence (new file)

### D2: `skills/procedures/test-context-cache.md`

```markdown
## Test Context Caching

After Step 0.6 (Cache Results), persist to `.claude/test-context.json`:

{
  "framework": "junit6",
  "mockLibrary": "mockk",
  "assertionLibrary": "truth",
  "coroutines": true,
  "turbine": true,
  "mockStyle": "relaxed",
  "namingConvention": "backtick",
  "jacocoConfigured": true,
  "jacocoReportPath": "app/build/reports/jacoco/...",
  "testInfrastructure": {
    "initTestDependencies": true,
    "testFixtures": true,
    "mainDispatcherRule": true
  },
  "lastUpdated": "2026-03-16T10:00:00Z",
  "buildGradleModTime": "2026-03-16T09:00:00Z"
}

On subsequent runs (without `--refresh`):
1. Read `.claude/test-context.json`
2. Compare `buildGradleModTime` with actual file mtime
3. If stale: re-scan (Step 0.2–0.6)
4. If fresh: use cached context, skip to Step 1
```

---

## Verification

After implementing each phase:
1. Read the modified skill file end-to-end to verify coherence
2. Check that no MeApp-specific patterns remain (search for `dmdbrands`, `gurus`, `weight`)
3. Verify JUnit 6 is the default in all examples
4. Run a mental walkthrough: "If I invoke `/android:write-tests FooViewModel`, does it detect JUnit version, handle infinite loops, use correct annotations?"

## Sources

- **Origin brainstorm:** [docs/brainstorms/2026-03-16-skill-improvements-from-viewmodel-testing-brainstorm.md](docs/brainstorms/2026-03-16-skill-improvements-from-viewmodel-testing-brainstorm.md)
- Session: MA-3433–MA-3442 (10 ViewModel test PRs)
- Key decisions: auto-detect JUnit, 95%/50% dynamic coverage, StandardTestDispatcher for infinite loops
