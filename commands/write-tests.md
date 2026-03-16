---
name: write-tests
description: Analyze a class, write comprehensive unit tests, extract constants, maximize code coverage with JaCoCo — supports Repository, ViewModel, Service, Reducer, and any Kotlin class
argument-hint: "ClassName [ClassName2 ...] [--refresh]"
---

# Write Unit Tests

You are writing unit tests for an Android/Kotlin class. Follow each step in order. Auto-proceed between steps — do not wait for user confirmation unless blocked.

## Step 0: Project Context Scan

**Run once per session.** If you already have project context cached from a previous `/android:write-tests` invocation in this session, skip to Step 1 — unless the user passed `--refresh`.

### 0.1 Parse Arguments

Extract from the argument:
- **ClassNames**: one or more classes to test (e.g., `LoginViewModel SignupViewModel`)
- **--refresh**: if present, force rescan even if cached

If multiple classes provided, run Step 0 once (cache context), then loop Steps 1–7 per class. After all classes, display Step 8 batch summary.

### 0.2 Scan Test Dependencies

Read the project's `build.gradle.kts` (app module) and identify:

| What to find | Where to look |
|-------------|---------------|
| Test framework | `testImplementation` — detect: `org.junit.jupiter` → **JUnit 6** (default); `junit:junit` → **JUnit 4** (legacy); if both → use JUnit 6 for new tests |
| Mock library | MockK (`io.mockk`), Mockito, or other |
| Coroutine testing | `kotlinx-coroutines-test` |
| Flow testing | Turbine (`app.cash.turbine`) |
| Assertion library | Truth (`com.google.common.truth`), AssertJ, or JUnit built-in |

### 0.2b JUnit Version Mapping

After detecting the JUnit version, use this mapping for ALL generated test code:

| Concept | JUnit 4 | JUnit 6 (default) |
|---------|---------|---------|
| Test annotation | `org.junit.Test` | `org.junit.jupiter.api.Test` |
| Setup | `@Before` | `@BeforeEach` |
| Teardown | `@After` | `@AfterEach` |
| Rule | `@get:Rule` | `@JvmField @RegisterExtension` |
| Exception test | try/catch + `fail()` | `assertThrows<Exception> { }` |
| Imports prefix | `org.junit.` | `org.junit.jupiter.api.` |
| MainDispatcherRule base | `TestWatcher` | `BeforeEachCallback, AfterEachCallback` |

Store detected version in session context as `junitVersion: 4 | 6`.

### 0.3 Learn Existing Test Patterns

Find 2–3 existing test files in `src/test/`:
```bash
find . -path "*/src/test/*" -name "*Test.kt" | head -3
```

Read each and note:
- **Mock declaration style**: `@MockK(relaxUnitFun = true)` vs `@MockK(relaxed = true)` vs inline `mockk()`
- **Setup pattern**: `@BeforeEach` (JUnit 6) or `@Before` (JUnit 4) with `MockKAnnotations.init(this)` vs manual init
- **Assertion style**: `assertThat(x).isEqualTo(y)` (Truth) vs `assertEquals(x, y)` (JUnit)
- **Naming convention**: backtick (`` fun `method does X`() ``) vs camelCase
- **Constants**: `companion object` with `private const val` vs inline strings
- **Coroutine tests**: `= runTest { }` pattern
- **Dispatcher rule**: Does `MainDispatcherRule` exist? Where?
- **`@After` teardown**: Is `clearMocks()` or `unmockkAll()` used?

### 0.3b Detect Test Infrastructure

Search the test source tree for project-specific test utilities:

```bash
find . -path "*/src/test/*" \( -name "*Extensions*.kt" -o -name "*Fixtures*.kt" -o -name "*TestHelper*.kt" -o -name "*Rule*.kt" \)
```

For each found, read and note:
- **ViewModel extension functions** (e.g., `initTestDependencies()`) — use in all ViewModel tests
- **Test fixture objects** (e.g., `TestFixtures.activeAccount`) — use instead of complex mocks
- **Custom test rules/extensions** (e.g., `MainDispatcherRule`) — reuse, don't recreate
- **Base test classes** — extend if project convention

Store in session context for reuse across test classes.

### 0.4 Detect Architecture & Infrastructure

- **Architecture**: MVI (has `IReducer`, `BaseIntentViewModel`), MVVM (has `ViewModel`), or other
- **DI framework**: Hilt (`@HiltViewModel`, `@Inject`), Koin, or manual
- **Base classes**: `BaseIntentViewModel`, `BaseViewModel`, custom base test classes
- **Interface naming**: `I` prefix (`IAccountRepository`) or not

### 0.5 Find JaCoCo Config

- Check `build.gradle.kts` for JaCoCo plugin and task name
- Identify XML report path (typically `app/build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml`)
- If no JaCoCo config found, note it — will skip JaCoCo steps

### 0.6 Cache Results

Store all findings in session context for reuse. Display a brief summary:

> **Project Context:**
> - Framework: [JUnit 4/6] + [MockK/Mockito] + [Truth/AssertJ]
> - Coroutines: [yes/no] | Turbine: [yes/no]
> - Architecture: [MVI/MVVM/other]
> - Mock style: [@MockK(relaxUnitFun = true) / relaxed / inline]
> - Naming: [backtick / camelCase]
> - JaCoCo: [configured / not found]

---

## Step 1: Locate & Classify

### 1.1 Find the Source File

Search for the target class:
```bash
find . -path "*/src/main/*" -name "<ClassName>.kt"
```

If multiple matches, pick the one matching the most likely package. If ambiguous, list options and ask the user.

If not found:
> Could not find `<ClassName>.kt` in the source tree. Please provide the full path.

### 1.2 Classify the Type

Read the file and determine the type:

| Type | Detection signals |
|------|------------------|
| **Repository** | In `data/repository/`; implements `IXxxRepository`; has API/DAO/DataStore dependencies |
| **ViewModel** | Extends `BaseIntentViewModel`, `BaseViewModel`, or `ViewModel`; has `@HiltViewModel` |
| **Reducer** | Implements `IReducer<State, Intent>`; pure function `reduce()` |
| **Service** | In `data/services/` or `core/service/`; implements `IXxxService` |
| **Other** | Doesn't match above — utility class, mapper, helper |

---

## Step 2: Full Auto-Analysis

Read the entire source file and produce the analysis.

### 2.1 Dependency Map

For each constructor parameter:

| Parameter | Type | Category | Mock Strategy |
|-----------|------|----------|---------------|
| `userApi` | `IUserAPI` | API | `@MockK` (strict — stub all calls) |
| `accountDao` | `AccountDao` | DAO | `@MockK(relaxUnitFun = true)` |
| `dataStore` | `UserDataStore` | DataStore | `@MockK(relaxUnitFun = true)` |
| `accountRepo` | `IAccountRepository` | Repository | `@MockK(relaxed = true)` |

### 2.1b @AssistedInject Detection

If the class constructor has `@AssistedInject`:
1. Identify `@Assisted` parameters — these are runtime values, not injected dependencies
2. In test setup, pass them as regular constructor arguments (use test constants)
3. Do NOT mock assisted parameters — they are plain values (String, Int, etc.)

```kotlin
// Example: ViewModel with @AssistedInject
viewModel = FooViewModel(
    sku = TEST_SKU,              // @Assisted — pass as plain value
    scaleInfo = null,            // @Assisted — nullable, pass null or test fixture
    deviceService = mockService, // @Inject — use mock
)
```

### 2.2 Method Inventory

For each public/internal method:

| Method | Return | Suspend | Flow | Type | Code Paths | Priority |
|--------|--------|---------|------|------|------------|----------|
| `getUser(id)` | `User?` | yes | no | Query | if null return, try/catch | High |
| `saveUser(user)` | `Unit` | yes | no | Command | try/catch | High |
| `userFlow` | `Flow<User>` | no | yes | Query (delegation) | single path | Medium |
| `calculateBmi(...)` | `Double` | no | no | Pure | if/when branches | High |

**Priority rules:**
- High: public methods with logic, branches, or side effects
- Medium: simple delegations, Flow properties
- Low: trivial getters, toString, companion object methods
- Skip: private methods (tested indirectly)

### 2.3 Dispatcher Detection

If the class takes a `CoroutineDispatcher` or `CoroutineScope` parameter, flag it:
> Dispatcher injection detected: `ioDispatcher: CoroutineDispatcher` — will use `TestDispatcher` in tests.

### 2.4 Known Pitfalls Detection

Scan the source for these patterns and flag each:

| Pattern | Risk | Mitigation |
|---------|------|------------|
| `toBuilder()?.setX()?.build()` (Proto) | Builder chain returns null with relaxed mocks | Verify method completes without throwing, don't `coVerify` DataStore call args |
| `?.id ?: ""` on non-nullable property | Unreachable branch in JaCoCo bytecode | Document as unreachable, exclude from coverage target |
| `var` class property used in null check | Won't smart-cast — compiler or runtime surprise | Test captures to local `val` |
| Multiple `when` branches | Each branch needs a test case | Map all branches |
| `try { } catch { }` blocks | Both success and failure paths need testing | Test both paths |

### 2.5 Init Block Coroutine Analysis

Scan the class init block (and parent class init) for infinite coroutine patterns:

| Pattern | Detection | Risk |
|---------|-----------|------|
| `while (!isDestroyed)` + `delay()` | Grep for `while\s*\(` + `delay\(` in same method | `advanceUntilIdle()` hangs forever — infinite scheduling loop |
| `state.collect {` | Grep for `state\.collect` in init | `runTest` cleanup hangs (60s timeout per test) |
| `.collect {` on hot Flow | Any `.collect` on a Flow that never completes | `runTest` cleanup hangs |
| Polling with `delay()` | `delay()` inside a loop | `advanceUntilIdle()` chases infinite iterations |

**If ANY infinite pattern detected:**

Flag: "Infinite coroutine detected in init block"

**Test-side remediation (apply automatically):**
1. Use `StandardTestDispatcher()` (NOT `UnconfinedTestDispatcher`)
2. Never use `runTest` — tests must be plain functions (no `= runTest { }`)
3. Never use `advanceUntilIdle()` — use `advanceTimeBy(200)` + `runCurrent()`
4. Add `@AfterEach` that calls `onCleared()` via reflection to cancel viewModelScope:
   ```kotlin
   @AfterEach
   fun tearDown() {
       val method = viewModel::class.java.getDeclaredMethod("onCleared")
       method.isAccessible = true
       method.invoke(viewModel)
   }
   ```
5. Add helper method:
   ```kotlin
   private fun advanceScheduler() {
       testDispatcher.scheduler.advanceTimeBy(200)
       testDispatcher.scheduler.runCurrent()
   }
   ```

**ViewModel-side suggestions (present to user, don't auto-apply):**
- Replace `state.collect { }` with `.map { relevantFields }.distinctUntilChanged().collect { }` — reduces unnecessary emissions
- Flag `while(true) + delay()` loops as candidates for Flow-based alternatives (e.g., `flow { while(true) { emit(Unit); delay(1500) } }.collect { }`)

**If NO infinite patterns detected:** Use standard `UnconfinedTestDispatcher` + `runTest` pattern as normal.

### 2.6 Display Analysis Report

Print the full analysis, then auto-proceed to Step 3.

---

## Step 3: Check Existing Tests + Baseline JaCoCo

### 3.1 Find Existing Test File

Look for `<ClassName>Test.kt` in the test source tree:
```bash
find . -path "*/src/test/*" -name "<ClassName>Test.kt"
```

### 3.2 If Test File Exists

1. Read the existing test file — note which methods are already tested
2. Run JaCoCo:
   ```bash
   ./gradlew jacocoTestReport
   ```
3. Parse the XML report for this specific class
4. Display baseline coverage:

   > **Baseline Coverage for `<ClassName>`:**
   > | Metric | Covered | Total | % |
   > |--------|---------|-------|---|
   > | Instructions | X | Y | Z% |
   > | Branches | X | Y | Z% |
   > | Lines | X | Y | Z% |
   > | Methods | X | Y | Z% |
   >
   > **Uncovered methods:** `methodA`, `methodB`
   > **Uncovered branches:** line 45 (if/else), line 78 (when)

5. Merge analysis from Step 2 with coverage data — focus new tests on uncovered areas

### 3.3 If No Test File Exists

Skip JaCoCo. All methods from Step 2 are uncovered — test everything.

---

## Step 4: Write Tests

### 4.1 Class Structure

Create or update the test file following the project's detected patterns (from Step 0):

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class <ClassName>Test {

    // Mocks — one per dependency from Step 2.1
    @MockK(relaxUnitFun = true) lateinit var dependency1: IDependency1
    @MockK(relaxed = true) lateinit var dependency2: IDependency2

    // System under test
    private lateinit var sut: ClassName

    companion object {
        private const val ACCOUNT_ID = "acc-123"
        private const val USER_NAME = "test-user"
        // All reusable test data as constants here
    }

    // Use @BeforeEach (JUnit 6) or @Before (JUnit 4) — based on detected version
    @BeforeEach
    fun setUp() {
        MockKAnnotations.init(this)
        sut = ClassName(dependency1, dependency2)
    }

    // @AfterEach only if shared mutable state detected (or infinite init coroutines — see Step 2.5)
    @AfterEach
    fun tearDown() {
        clearMocks(dependency1, dependency2)
    }
}
```

> **Tip:** Put reusable test data (IDs, names, emails) in `companion object` as `private const val` from the start. Single-use values in `match {}` lambdas or one-off assertions can stay inline.

### 4.2 Import Management

Auto-add imports based on what's needed:

| Usage | Import |
|-------|--------|
| Suspend mock stubs | `io.mockk.coEvery` |
| Suspend mock verification | `io.mockk.coVerify` |
| Regular mock stubs | `io.mockk.every` |
| Regular mock verification | `io.mockk.verify` |
| Flow creation | `kotlinx.coroutines.flow.flowOf`, `kotlinx.coroutines.flow.flow` |
| Coroutine test | `kotlinx.coroutines.test.runTest` |
| ExperimentalCoroutinesApi | `kotlinx.coroutines.ExperimentalCoroutinesApi` |
| Truth assertions | `com.google.common.truth.Truth.assertThat` |
| MockK annotations | `io.mockk.MockKAnnotations`, `io.mockk.impl.annotations.MockK` |
| Argument matching | `io.mockk.any`, `io.mockk.match` |
| Flow testing | `kotlinx.coroutines.flow.first`, `app.cash.turbine.test` (if Turbine available) |
| Slot capture | `io.mockk.slot`, `io.mockk.capture` |

Only add imports that are actually used.

### 4.3 Test Categories per Method

For **every method** identified in Step 2.2 (ordered by priority), write tests covering:

#### A. Happy Path
Normal successful execution with valid inputs.

```kotlin
// companion object {
//     private const val ACCOUNT_ID = "acc-123"
//     private const val NETWORK_ERROR = "Network error"
//     private const val DB_ERROR = "DB error"
//     private const val API_ERROR = "API error"
// }

@Test
fun `getUser returns user when API succeeds`() = runTest {
    coEvery { userApi.getUser(ACCOUNT_ID) } returns mockUser
    val result = sut.getUser(ACCOUNT_ID)
    assertThat(result).isEqualTo(mockUser)
}
```

#### B. Error Path
Exception handling, API failures, null responses.

**JUnit 6 (default):** Use `assertThrows`:
```kotlin
@Test
fun `getUser throws when API fails`() = runTest {
    coEvery { userApi.getUser(any()) } throws RuntimeException(NETWORK_ERROR)
    val exception = assertThrows<RuntimeException> { sut.getUser(ACCOUNT_ID) }
    assertThat(exception.message).isEqualTo(NETWORK_ERROR)
}
```

**JUnit 4 (legacy):** Use try/catch — `assertThrows` doesn't work reliably with `suspend` in JUnit 4:
```kotlin
@Test
fun `getUser throws when API fails`() = runTest {
    coEvery { userApi.getUser(any()) } throws RuntimeException(NETWORK_ERROR)
    try {
        sut.getUser(ACCOUNT_ID)
        fail("Expected exception")
    } catch (e: RuntimeException) {
        assertThat(e.message).isEqualTo(NETWORK_ERROR)
    }
}
```

#### C. Null and Empty Inputs
For each parameter that could be null, empty, or blank:

```kotlin
@Test
fun `getUsers returns empty list when API returns empty`() = runTest {
    coEvery { userApi.getUsers(any()) } returns emptyList()
    val result = sut.getUsers(ACCOUNT_ID)
    assertThat(result).isEmpty()
}
```

#### D. Edge Cases and Boundary Values
Boundary conditions, min/max values, zero-length inputs.

#### E. Flow Tests
For methods returning `Flow`:

```kotlin
@Test
fun `userFlow delegates to dataStore userFlow`() = runTest {
    every { dataStore.userFlow } returns flowOf(mockUser)
    val result = sut.userFlow.first()
    assertThat(result).isEqualTo(mockUser)
}

@Test
fun `userFlow handles error in upstream`() = runTest {
    every { dataStore.userFlow } returns flow { throw RuntimeException(DB_ERROR) }
    try {
        sut.userFlow.first()
        fail("Expected exception")
    } catch (e: RuntimeException) {
        assertThat(e.message).isEqualTo(DB_ERROR)
    }
}
```

#### E2. Multiple Flow Testing (turbineScope)
For classes with multiple flows that update together, use `turbineScope`:
```kotlin
@Test
fun `refresh updates both flows`() = runTest {
    turbineScope {
        val items = sut.itemsFlow.testIn(this)
        val status = sut.statusFlow.testIn(this)
        sut.refresh()
        assertThat(items.awaitItem()).hasSize(3)
        assertThat(status.awaitItem()).isEqualTo(Status.LOADED)
        items.cancelAndIgnoreRemainingEvents()
        status.cancelAndIgnoreRemainingEvents()
    }
}
```

#### F. Side Effect Verification
For command methods (methods that DO something):

```kotlin
@Test
fun `saveUser calls API and updates cache`() = runTest {
    coEvery { userApi.saveUser(any()) } returns mockResponse
    sut.saveUser(mockUser)
    coVerify { userApi.saveUser(match { it.id == ACCOUNT_ID }) }
    coVerify { userDao.insert(any()) }
}
```

#### G. State Consistency After Error
Verify state isn't left half-updated when an error occurs mid-operation:

```kotlin
@Test
fun `saveUser does not update cache when API fails`() = runTest {
    coEvery { userApi.saveUser(any()) } throws RuntimeException(API_ERROR)
    try {
        sut.saveUser(mockUser)
    } catch (_: RuntimeException) { }
    coVerify(exactly = 0) { userDao.insert(any()) }
}
```

#### H. State Preservation (Reducers)
For reducer tests, verify unrelated fields are NOT modified:
```kotlin
@Test
fun `SetLoading only changes isLoading`() {
    val initial = state.copy(items = listOf(testItem), error = ERROR_MSG)
    val result = reducer.reduce(initial, Intent.SetLoading(true))
    assertThat(result?.isLoading).isTrue()
    assertThat(result?.items).isEqualTo(initial.items)  // preserved
    assertThat(result?.error).isEqualTo(initial.error)  // preserved
}
```

> **Tip:** The examples above use constants for reusable values. Follow the same pattern for values used in 2+ tests.

### 4.4 Type-Specific Patterns

> **Full pattern reference:** See `procedures/test-patterns.md` for detailed examples with code.

#### Repository Tests
- Mock APIs (strict or relaxUnitFun), DAOs (relaxUnitFun), DataStores (relaxUnitFun)
- Test: API call → result mapping → cache/storage side effects
- Test: API failure → exception propagation or fallback
- Test: Flow delegation from DataStore
- Test: insert-or-update branching (both paths) — see `test-patterns.md`
- Test: request object construction via `slot<>()` + `capture()` — see `test-patterns.md`
- Test: catch-and-swallow vs catch-and-rethrow — see `test-patterns.md`

#### ViewModel Tests
- Requires `MainDispatcherRule` — use `@JvmField @RegisterExtension` (JUnit 6) or `@get:Rule` (JUnit 4)
- Mock services and repositories
- If infinite init coroutines detected (Step 2.5): use `StandardTestDispatcher` + `advanceScheduler()` helper, no `runTest`
- Otherwise: use `UnconfinedTestDispatcher` + `runTest` + `advanceUntilIdle()`
- Test: `handleIntent(Intent.X)` → assert `state.value`
- Test: navigation calls via `navigationService`
- Test: dialog/toast callbacks via `slot<DialogModel>()` + `capture()` — see `test-patterns.md`
- If project has `initTestDependencies()` (detected in Step 0.3b): call it after constructor
- **Use `.value` for final-state assertions, Turbine only for intermediate emissions** — see `test-patterns.md`
- **Never re-test reducer logic in ViewModel tests** — test side effects only. State changes belong in ReducerTest.
- **Use `createViewModel()` factory** when flow stubs must be set before construction — see `test-patterns.md`

#### Reducer Tests
- **No mocks needed** — pure function testing
- Test: `reducer.reduce(state, intent)` → assert returned state fields
- Test every intent in the `when` block
- Test: unhandled intent returns `null` or `state.copy()`
- Test: state transitions (e.g., loading → loaded → error)
- **Verify state preservation** — assert unrelated fields are NOT modified by each intent — see `test-patterns.md`
- Test: side-effect-only intents must return `null` from reducer

#### Service Tests
- Mock repositories and other services
- Test: business logic, calculations, data transformations
- Test: orchestration (calls A, then B, then C in order)
- Test: error handling and fallback behavior
- Test: network guard clauses — hard throw vs soft check — see `test-patterns.md`
- **Use `createService()` factory** when flow stubs must be set before construction — see `test-patterns.md`

### 4.5 Test Isolation

If the class under test has mutable state or if mocks are shared across tests that could interfere:
- Add `@AfterEach fun tearDown() { clearMocks(...) }` (JUnit 6) or `@After` (JUnit 4)
- Ensure each test sets up its own stubs — never rely on ordering

### 4.6 Dispatcher Injection

If the class takes a `CoroutineDispatcher` parameter:
```kotlin
private val testDispatcher = UnconfinedTestDispatcher()

@BeforeEach  // or @Before for JUnit 4
fun setUp() {
    MockKAnnotations.init(this)
    sut = ClassName(dependency1, testDispatcher)
}
```

---

## Step 5: Extract Constants

### 5.1 Smart Literal Scan

Scan test file for literals that should be extracted to `companion object`:

**Extract when:**
- Value is used in **2+ tests** — always extract to avoid duplication
- Numeric test data (except `0`, `1`, `-1`, `true`, `false`, array indices, `exactly = N`)
- String values used as test identifiers (IDs, names, emails, URLs)

**Allow inline when:**
- String inside `match { }` lambda (e.g., `match<Toast> { it.title == "Success!" }`)
- Single-use value in exactly 1 test with clear meaning
- Error messages that are assertion targets and used once
- Boolean/null literals

### 5.2 Smart Extraction Rules

Move flagged literals to `companion object` as `private const val`:

| Rule | Why |
|------|-----|
| Never replace inside the constant's own value assignment | Prevents self-referential: `val X = X` |
| Never replace property access in lambdas (`it.accountId`) | `it.ACCOUNT_ID` doesn't compile |
| Never replace partial matches inside longer strings | `"account-123"` should not affect `"my-account-1234"` |
| Use descriptive constant names | `ACCOUNT_ID` not `VAL_1` |

### 5.3 Verify After Extraction

After extraction, scan the file for:
- Any self-referential constants (`val X = X`)
- Any `it.CONSTANT_NAME` patterns (broken lambda property access)
- Any compilation errors introduced

### 5.4 Verification

Re-scan test methods. If any repeated literal (used 2+ times) or numeric test data exists inline → extract. Single-use inline strings in `match {}` or assertion targets are acceptable.

---

## Step 6: Run Tests + Failure Diagnosis

### 6.1 Run Tests

```bash
./gradlew :app:testDebugUnitTest --tests "com.package.<ClassName>Test"
```

### 6.2 If Tests Pass → Proceed to Step 7

### 6.3 If Tests Fail → Diagnose and Fix

Read the error output and match against known patterns. For the full 35+ entry lookup table, see `procedures/test-troubleshooting.md`. Common patterns:

| Error Pattern | Diagnosis | Fix |
|--------------|-----------|-----|
| `no answer found for: <MockClass>.<method>()` | Missing mock stub | Add `coEvery { mock.method() } returns ...` or `every { ... }` |
| `Verification failed: call ... was not made` | Wrong argument matcher or method not called | Check argument in `match { }`, or verify logic actually calls the method |
| `java.lang.ClassCastException` on mock return | Relaxed mock returning wrong default type | Add explicit `coEvery` stub with correct return type |
| `Unresolved reference: <name>` | Missing import or typo in constant | Add import or fix the reference |
| `Type mismatch: Required X, found Y` | Stub returns wrong type | Fix the `returns` value type |
| `Expected non-null value` / `IllegalStateException` | `requireNotNull` or `!!` hit null | Ensure mock returns non-null where required |
| Proto builder returns null in verify | Builder chain unreliable with relaxed mocks | Remove `coVerify` on DataStore, verify method completes without throwing |
| `fail("Expected exception") was reached` | Exception not thrown in error test | Check that `coEvery` has `throws` not `returns` |

**Fix → re-run → repeat until all tests pass.**

When fixing tests, extract any new reusable literals to `companion object` constants.

Maximum retry loops: 5. If still failing after 5 attempts:
> Tests still failing after 5 fix attempts. Remaining errors:
> [error details]
> Please review and fix manually, or provide guidance.

---

## Step 7: Coverage Maximization Loop

### 7.0 Determine Coverage Target

Before running JaCoCo, determine the appropriate coverage target based on class characteristics.

**Hardware ViewModel indicators** (any match = hardware class):
- Imports containing: `GGDeviceService`, `WifiScaleService`, `GGPermissionService`, `BluetoothAdapter`, `WifiManager`, `BLEDiscoveryManager`, `WiFiConfigManager`, or similar BLE/WiFi/hardware classes
- Class body > 800 lines
- Extends a class with "Scale", "Ble", or "Wifi" in the name

| Class Type | Instructions Target | Branches Target | Lines Target |
|------------|-------------------|-----------------|--------------|
| Standard | 95% | 90% | 95% |
| Hardware ViewModel | 50% | 30% | 50% |

Display the selected target before running JaCoCo.

### 7.1 Run JaCoCo

```bash
./gradlew jacocoTestReport
```

Parse the XML report for this specific class. Extract:
- Instructions: covered / total
- Branches: covered / total
- Lines: covered / total
- Methods: covered / total
- Per-method and per-line breakdown of what's uncovered

### 7.1b Method-Level Analysis

After class-level coverage, check per-method breakdown to find fully uncovered methods hidden by overall percentage. See `procedures/test-coverage.md` for the Python script.

### 7.2 Smart Filter — Exclude Unreachable

Analyze each uncovered item against the source code. See `procedures/test-coverage.md` for the full JaCoCo 0.8.14+ false positives table.

| Unreachable Pattern | How to Detect | Action |
|--------------------|--------------|--------|
| Safe-call on non-nullable (`?.id ?: ""` where type is `String`) | Check source: property type is non-nullable but code uses `?.` or `?: ""` | Exclude from target, document |
| Proto builder chain in try/catch | Method uses `toBuilder()?.setX()?.build()` inside try/catch | Exclude branches inside builder chain |
| Kotlin `when` exhaustive else | `else ->` on a sealed class with all cases covered | Exclude, document |
| Bytecode-generated null check on non-null return | JaCoCo shows branch on a non-nullable return statement | Exclude, document |

### 7.3 Weak Assertion Audit

Scan existing tests for weak assertions that don't actually verify behavior:

| Weak Assertion | Stronger Alternative |
|---------------|---------------------|
| `assertThat(result).isNotNull()` | `assertThat(result).isEqualTo(expectedValue)` |
| `assertThat(list).isNotEmpty()` | `assertThat(list).hasSize(expectedSize)` or `assertThat(list).containsExactly(...)` |
| `assertTrue(result is SomeType)` | `assertThat(result).isInstanceOf(SomeType::class.java)` + assert fields |
| No assertion after method call | Add `coVerify` for commands or `assertThat` for queries |

Strengthen weak assertions where possible.

When strengthening assertions, extract reusable expected values to `companion object` constants.

### 7.4 Coverage Gap Iteration

If there are **coverable** gaps remaining (after filtering unreachable):

1. Identify the specific uncovered methods/branches/lines
2. Write additional tests targeting those gaps
3. Re-run Step 5 (Extract Constants) on the entire test file — extract reusable literals
4. Run tests (Step 6 loop)
5. Re-run JaCoCo
6. Check again
7. Repeat until no more coverable gaps

> Every time new tests are added, re-run Step 5 constant extraction on the entire file.

**Maximum iterations:** 3. If still gaps after 3 rounds, document remaining gaps and stop.

### 7.5 Final Coverage Report

Display the final results:

> **Final Coverage for `<ClassName>`:**
> | Metric | Covered | Total | % |
> |--------|---------|-------|---|
> | Instructions | X | Y | Z% |
> | Branches | X | Y | Z% |
> | Lines | X | Y | Z% |
> | Methods | X | Y | Z% |
>
> [If baseline existed:]
> **Improvement:** Instructions +X%, Branches +X%, Lines +X%, Methods +X%
>
> [If unreachable items:]
> **Unreachable (excluded):**
> - N branches: [description] (lines X, Y)

### 7.6 PR-Ready Coverage Table

Output a markdown table ready to copy into a PR description:

```markdown
## JaCoCo Coverage: `<ClassName>`
| Metric | Covered | Total | Coverage |
|--------|---------|-------|----------|
| Instructions | X | Y | Z% |
| Branches | X | Y | Z% |
| Lines | X | Y | Z% |
| Methods | X/Y | Y | 100% |

[Notes about unreachable branches if any]
```

**Store in session context** for the `android:creating-pr` skill to pick up automatically. Accumulate coverage tables across multiple `/android:write-tests` invocations in the same session — each class gets its own table. The `creating-pr` skill will include all stored tables in the PR description under a "Coverage" section.

---

## Step 8: Batch Summary (multi-class only)

If multiple classes were tested in a single invocation, display a summary table:

> **Batch Test Summary:**
> | Class | Tests | Instructions | Branches | Lines |
> |-------|-------|-------------|----------|-------|
> | LoginViewModel | 32 | 95.8% | 100% | 100% |
> | SignupViewModel | 29 | 96.5% | 92% | 98% |
> | **Total** | **61** | | | |

---

## Done

The skill stops here. Output:

> **Tests complete for `<ClassName>`.**
> - Tests written: [N tests]
> - Tests passing: All green
> - Coverage: [summary]
>
> Ready for commit. Use `/android:finish-task` to commit, create PR, and log work.

---

## Error Recovery

**If the source file can't be found:**
> Could not locate `<ClassName>.kt`. Provide the full file path.

**If JaCoCo is not configured:**
> JaCoCo not found in build config. Skipping coverage analysis. Tests were written and pass.

**If build fails (not just tests):**
> Build failure detected — this is not a test issue. Fix the build first, then re-run `/android:write-tests <ClassName>`.

**If the class has no testable methods:**
> `<ClassName>` has no public/internal methods to test (only private or no logic). Nothing to test.
