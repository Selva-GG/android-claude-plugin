# Brainstorm: Skill Improvements from ViewModel Testing Session

**Date:** 2026-03-16
**Context:** Lessons learned from writing tests for 10 ViewModels (MA-3433–MA-3442) across LoginViewModel, DashboardViewModel, EntryViewModel, SignupViewModel, SettingsViewModel, MyAccountsViewModel, HomeViewModel, WifiScaleSetupViewModel, BtWifiScaleSetupViewModel, ScaleDetailsViewModel.

## What We're Building

Improvements to three Android Claude plugin skills (`write-tests`, `start-task`, `finish-task`) to make them fully generic (not MeApp-specific) and address real-world issues encountered during bulk ViewModel testing.

## Key Decisions

1. **Fully generic** — no MeApp-specific patterns hardcoded. All project-specific patterns auto-detected from build.gradle.kts and existing test files.
2. **Default JUnit 6, fallback JUnit 4** — assume JUnit 6 unless build.gradle.kts has `junit:junit:4.x`.
3. **Infinite coroutine handling** — both test-side workaround AND ViewModel refactor suggestions.
4. **Dynamic coverage targets** — 95% for standard classes, 50% for hardware ViewModels (auto-detected).

---

## Improvements by Skill

### A. `write-tests` (commands/write-tests.md)

#### A1. JUnit 6 as default
- **Problem:** Hardcodes JUnit 4 (`@Before`, `@get:Rule`, `org.junit.Test`, try/catch for exceptions)
- **Fix:** In Step 0.2, detect JUnit version from build.gradle.kts:
  - `org.junit.jupiter` → JUnit 6: `@BeforeEach`, `@RegisterExtension`, `org.junit.jupiter.api.Test`, `assertThrows`
  - `junit:junit` → JUnit 4: `@Before`, `@get:Rule`, `org.junit.Test`, try/catch
- **Annotations mapping table:**

| JUnit 4 | JUnit 6 |
|---------|---------|
| `@Before` | `@BeforeEach` |
| `@After` | `@AfterEach` |
| `@get:Rule` | `@JvmField @RegisterExtension` |
| `org.junit.Test` | `org.junit.jupiter.api.Test` |
| `org.junit.rules.TestWatcher` | `BeforeEachCallback, AfterEachCallback` |

#### A2. Infinite coroutine loop detection
- **Problem:** ViewModels with `while(!destroyed) + delay()` or `state.collect {}` in init cause `runTest` to hang (3+ min) and `advanceUntilIdle()` to loop forever.
- **Fix:** Add Step 2.5 "Init Block Analysis":
  1. Scan init block for patterns: `while (`, `state.collect`, `.collect {`, `delay(`
  2. If infinite loop detected:
     - **Test side:** Use `StandardTestDispatcher` (not `UnconfinedTestDispatcher`), `advanceTimeBy(200)` + `runCurrent()` (never `advanceUntilIdle`), no `runTest`, `@AfterEach` with `onCleared()` via reflection
     - **ViewModel side:** Suggest replacing `state.collect` with `.map { }.distinctUntilChanged().collect`, flag `while(true)` loops as candidates for Flow-based alternatives
  3. Generate `advanceScheduler()` helper in test class

#### A3. @AssistedInject detection
- **Problem:** ViewModels with `@AssistedInject` need `@Assisted` params in constructor, not standard Hilt injection.
- **Fix:** In Step 2.1, detect `@AssistedInject` and generate constructor with assisted params as regular parameters:
```kotlin
viewModel = FooViewModel(
    assistedParam1 = "value",
    assistedParam2 = null,
    service1 = mockService1,
    ...
)
```

#### A4. Dynamic coverage targets
- **Problem:** 85% target unrealistic for 1000+ line ViewModels with BLE/WiFi hardware deps.
- **Fix:** In Step 7, detect hardware dependencies by scanning imports for patterns like:
  - `GGDeviceService`, `WifiScaleService`, `GGPermissionService`, `BluetoothAdapter`, `WifiManager`
  - Class size > 800 lines
- If detected: target 50% instructions, document uncoverable methods
- Otherwise: target 95% instructions

#### A5. Auto-detect test infrastructure
- **Problem:** Skill doesn't know about project-specific test helpers like `initTestDependencies()`, `TestFixtures`, custom base test classes.
- **Fix:** In Step 0.3, when scanning existing tests, also detect:
  - Extension functions on ViewModel (e.g., `initTestDependencies()`)
  - Test fixture objects (e.g., `TestFixtures.activeAccount`)
  - Custom test rules/extensions
  - Base test classes
- Use detected patterns in generated tests

#### A6. Relax constant extraction
- **Problem:** "Blocking gate" is too rigid. Some literals are clearer inline (error messages in `match {}` lambdas, single-use values in 1 test).
- **Fix:** Change from "blocking gate" to "best effort with exceptions":
  - Extract literals used in 2+ tests
  - Extract all numeric test data (except 0, 1, -1, true, false)
  - Allow inline strings in `match {}` lambdas and single-use assertions
  - Remove "BLOCKING GATE" language

#### A7. Batch mode
- **Problem:** Invoked 10 times separately for 10 ViewModels.
- **Fix:** Support `/android:write-tests ViewModelA ViewModelB ViewModelC`:
  - Run Step 0 once (cache context)
  - Loop Steps 1–7 per class
  - Final summary table with all classes, test counts, coverage

---

### B. `start-task` (commands/start-task.md)

#### B1. Subtask awareness
- **Problem:** Doesn't detect parent ticket with subtasks.
- **Fix:** After fetching ticket in Step 2:
  1. Check if ticket has subtasks (via REST API `subtasks` field)
  2. If yes, list them and ask: "This ticket has N subtasks. Work through them one by one?"
  3. If user says yes, auto-iterate: create branch per subtask, base on parent branch
  4. Track progress across subtasks

#### B2. Smart base branch for subtasks
- **Problem:** Defaults to `main`/`dev` but subtasks should target parent branch.
- **Fix:** In Step 6:
  1. If ticket is a subtask, check if parent ticket has a branch (`git branch -r --list "*PARENT-ID*"`)
  2. If found, suggest: "This is a subtask of PARENT-ID. Use `PARENT-BRANCH` as base?"
  3. Default to parent branch for subtasks

#### B3. Skip estimate for subtasks
- **Problem:** Estimate review unnecessary for small subtasks.
- **Fix:** In Step 4:
  - If ticket type is "Subtask" and no estimate exists, skip estimate step
  - If ticket type is "Subtask" and estimate exists, show but don't prompt to change

---

### C. `finish-task` (commands/finish-task.md)

#### C1. Auto-pick up coverage data from write-tests
- **Problem:** PR descriptions lack coverage tables unless manually added.
- **Fix:** In Step 6 (PR creation):
  1. Check if write-tests was run in this session (look for JaCoCo data in session context)
  2. If found, auto-include coverage table and test count in PR body
  3. Format: same markdown table generated by write-tests Step 7.6

#### C2. Worklog via REST API primary
- **Problem:** ACLI worklog commands had wrong flags.
- **Fix:** In jira-worklogging.md:
  - Use curl + REST API v3 as primary method (already partially done)
  - Remove ACLI worklog attempts entirely
  - Keep ACLI only for transitions and fetching

#### C3. Inline worklog args
- **Problem:** User had to specify worklog time separately.
- **Fix:** Accept args: `/android:finish-task worklog 10m 2026-03-16`
  - Parse time (`10m`, `1h`, `30m`) and date from args
  - Skip worklog prompt if both provided
  - Still ask if only partial args given

#### C4. Auto-transition after worklog
- **Problem:** Flow asks whether to transition after worklog.
- **Fix:** If PR was created in this flow, auto-transition to "In Review" without asking. Only ask if no PR was created.

#### C5. Handle gh auth scope issues
- **Problem:** `gh pr edit` needed `read:project` scope.
- **Fix:** In creating-pr.md:
  - Catch "missing required scopes" error
  - Fall back to `gh api` for PR updates (which doesn't need the scope)
  - Only suggest `gh auth refresh` as last resort

---

### D. Cross-cutting (procedures + testing sub-skill)

#### D1. Update testing.md sub-skill for JUnit 6
- **Problem:** References JUnit 4 patterns exclusively.
- **Fix:** Update `/skills/procedures/testing.md`:
  - Default examples use JUnit 6 annotations
  - Add "JUnit 4 compatibility" section for legacy projects
  - Update MainDispatcherRule example to Extension API

#### D2. Persist project context to file
- **Problem:** Context lost when conversation compacts.
- **Fix:** In write-tests Step 0.6:
  - Write cached context to `.claude/test-context.json` (framework, mock style, naming convention, JaCoCo config)
  - On subsequent runs, read from file if `--refresh` not passed
  - Auto-refresh if build.gradle.kts modification time is newer than cache

---

## Open Questions

None — all clarified during brainstorm dialogue.

## Next Steps

Run `/ce:plan` to create implementation plan with file-by-file changes.
