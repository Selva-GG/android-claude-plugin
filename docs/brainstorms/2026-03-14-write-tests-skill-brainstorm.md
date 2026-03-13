# Brainstorm: `write-tests` Skill

**Date:** 2026-03-14

## What We're Building

A user-invocable skill (`/android:write-tests ClassName`) that automates unit test writing for Android/Kotlin projects. The skill analyzes the target class, understands the project's testing setup, writes comprehensive tests, maximizes code coverage, and outputs a JaCoCo report — all without human intervention between steps.

**Scope:** Analysis → Write tests → Extract constants → Run tests → JaCoCo coverage maximization. Does NOT handle commit, PR, or worklog — those stay with existing skills (`finish-task`, `jira-worklogging`).

## Why This Approach

Writing unit tests across many classes follows a repeatable pattern: understand the class, understand its dependencies, write tests per method, maximize coverage. Codifying this into a skill ensures:

- Consistent test quality across all classes
- No missed edge cases or error paths
- Hardcoded strings extracted to constants from the start
- Coverage is maximized with smart iteration, not guesswork
- Known pitfalls (proto builders, safe-call bytecode artifacts) are handled automatically

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Genericity** | Generic Kotlin/Android with built-in knowledge of Repo/VM/Service/Reducer | Works across projects, but understands common Android architecture |
| **Analysis depth** | Full auto-analysis before writing | Maps dependencies, classifies methods, detects pitfalls — writes better tests |
| **JaCoCo timing** | Existing tests: JaCoCo first. New tests: JaCoCo at end. Always compulsory at end | Shows gaps upfront for existing files; always validates final coverage |
| **Approval gate** | Auto-proceed after showing analysis | Reduces friction; user sees the report but doesn't have to approve each time |
| **Constants** | Part of this skill | Extracts hardcoded strings to companion object as a cleanup step |
| **Invocation** | User-invocable: `/android:write-tests ClassName` | Can be called directly or composed into other workflows |
| **Project scan** | Once per session, cached, `--refresh` to rescan | Avoids redundant scans; refresh flag for when config changes |
| **Coverage loop** | Smart detection of unreachable + iterate until maximized | Doesn't chase impossible 100%; documents unreachable branches |

## Skill Flow (8 Steps)

### Step 0: Project Context Scan (once per session)
- Read `build.gradle.kts` → test dependencies
- Read 2-3 existing test files → learn patterns (mock style, assertions, naming, setup)
- Detect architecture, DI framework, base classes, interface naming
- Find JaCoCo config + report path
- Cache in session. `--refresh` to rescan

### Step 1: Locate & Classify
- Find source file, determine type: Repository / ViewModel / Service / Reducer / Other
- Each type has different testing needs (mocks, dispatchers, rules)

### Step 2: Full Auto-Analysis
- **Dependencies**: list constructor params, classify (API/DAO/DataStore/Service/Repo), decide mock strategy
- **Methods**: inventory all public/internal — suspend/Flow/command/query, map code paths
- **Dispatcher detection**: flag if class takes CoroutineDispatcher param
- **Known pitfalls**: proto builders, safe-call on non-nullable, var properties, assertThrows+suspend
- **Project conventions**: from cached context + CLAUDE.md

### Step 3: Check Existing Tests + Baseline JaCoCo
- If test file exists → run JaCoCo → show coverage table + identify gaps
- If no test file → skip

### Step 4: Write Tests
- Class structure: `@MockK`, `@Before`, `companion object` constants, correct imports
- **AAA pattern enforced** (Arrange-Act-Assert)
- **Test categories per method**: happy path, error path, null/empty inputs, edge cases, Flow tests (emission + completion + cancellation), side effect verification, state consistency after error
- **Dispatcher injection**: use TestDispatcher if class takes dispatcher param
- **Test isolation**: add `@After` with `clearMocks()` if shared mutable state detected
- Priority: public methods first → internal → skip private

### Step 5: Extract Constants
- Scan for remaining hardcoded literals → move to companion object
- Smart replacement: don't touch property access, don't self-reference

### Step 6: Run Tests + Failure Diagnosis Loop
- Run tests → if fail, diagnose using known error patterns (missing stub, wrong matcher, cast error, missing import, proto builder) → fix → re-run → repeat until green

### Step 7: Coverage Maximization Loop
1. Run JaCoCo → parse XML
2. Identify uncovered gaps
3. Smart filter: exclude unreachable (Kotlin bytecode artifacts, safe-call on non-nullable)
4. **Weak assertion audit**: flag tests that only assert `isNotNull()` when value assertions possible → strengthen
5. If coverable gaps remain → write more tests → run → JaCoCo → repeat
6. If only unreachable remain → document and stop
7. Output final coverage table + unreachable list + PR-ready markdown table

## Additional Features

### Common Pitfalls Database (baked into skill)
- Proto builder chains fail with relaxed mocks → verify no-throw instead of coVerify
- `assertThrows` doesn't work with suspend in JUnit 4 → use try/catch
- `var` properties don't smart-cast → need local val capture
- sed-style replacement bugs → smart replacement avoids self-referential constants and property access mutations

### Failure Diagnosis Intelligence
| Error Pattern | Diagnosis | Auto-fix |
|--------------|-----------|----------|
| `no answer found for` | Missing mock stub | Add coEvery/every |
| `verification failed` | Wrong argument matcher | Fix match {} or use any() |
| `ClassCastException` on mock | Relaxed mock returning wrong type | Add explicit stub |
| `Unresolved reference` | Missing import or wrong constant name | Add import or fix reference |
| `Type mismatch` | Wrong return type in stub | Fix returns value |
| Proto builder returns null | Builder chain unreliable with relaxed mock | Change to verify no-throw |

### Import Management
- Auto-add correct imports based on what's used: coEvery, coVerify, every, verify, flowOf, runTest, assertThat, flow, etc.

### PR-Ready Coverage Table
- Outputs a markdown table at the end, ready to paste into PR description

## Open Questions

None — all resolved during brainstorming.

## References

- [Best practices for Unit Testing Android Apps with MockK](https://dev.to/rchugunov/best-practices-for-unit-testing-android-apps-with-mockk-kotest-and-others-35j9)
- [Testing Kotlin coroutines on Android](https://developer.android.com/kotlin/coroutines/test)
- [Unit Test Generation: A Comprehensive Guide](https://zencoder.ai/blog/how-unit-test-generation-works)
- [7 Methods to Improve Unit Test Coverage](https://www.startearly.ai/post/7-methods-to-improve-unit-test-coverage)
- [NowInAndroid Testing Strategy](https://github.com/android/nowinandroid/wiki/Testing-strategy-and-how-to-test)
- [Unit Testing Best Practices Checklist](https://canro91.github.io/2021/07/05/UnitTestingBestPractices/)
