# Testing

> For the full automated test-writing workflow (with JaCoCo coverage analysis), use `/android:write-tests`. This file serves as a quick reference for testing patterns.

## Setup

| Library | Purpose |
|---------|---------|
| JUnit 6 (`org.junit.jupiter`) | Test framework (default) |
| JUnit 4 (`junit:junit`) | Legacy test framework (detect from build.gradle) |
| MockK | Mocking (Kotlin-native) |
| Turbine | Flow testing |
| kotlinx-coroutines-test | `runTest`, `TestDispatcher` |

Test location: `[module]/src/test/java/[package]/`

> **JUnit version:** Detect from `build.gradle.kts` — if `org.junit.jupiter` is present, use JUnit 6 annotations. If `junit:junit`, use JUnit 4. Default to JUnit 6 for new projects.

## Naming Convention

Use backtick syntax with descriptive names:

```kotlin
@Test
fun `load items updates state with fetched items`() { ... }

@Test
fun `submit with invalid form shows error toast`() { ... }

@Test
fun `SetQuery intent updates query in state`() { ... }
```

Pattern: `<action> <expected outcome>`

## Reducer Tests (Pure Functions)

Easiest to test — no mocks needed, just input/output:

```kotlin
class GoalReducerTest {
    private val reducer = GoalReducer()
    private val initialState = GoalState(
        form = FormGroup(GoalFormControls.create()),
    )

    @Test
    fun `Submit sets isLoading true and clears error`() {
        val result = reducer.reduce(initialState, GoalIntent.Submit)

        assertThat(result).isNotNull()
        assertThat(result?.isLoading).isTrue()
        assertThat(result?.error).isNull()
    }

    @Test
    fun `SetError sets error message and stops loading`() {
        val loadingState = initialState.copy(isLoading = true)
        val result = reducer.reduce(loadingState, GoalIntent.Error("Failed"))

        assertThat(result).isNotNull()
        assertThat(result?.isLoading).isFalse()
        assertThat(result?.error).isEqualTo("Failed")
    }

    @Test
    fun `unhandled intent returns null`() {
        val result = reducer.reduce(initialState, GoalIntent.OnBack)
        // OnBack is handled in ViewModel, not reducer
        // Reducer may return null or state.copy()
    }
}
```

Test every intent → expected state transition. Pure input/output — no setup needed.

## ViewModel Tests

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class GoalViewModelTest {
    private lateinit var viewModel: GoalViewModel
    private val goalService: IGoalService = mockk(relaxed = true)
    private val accountService: IAccountService = mockk(relaxed = true)
    private val entryService: IEntryService = mockk(relaxed = true)
    private val dialogUtility: IDialogUtility = mockk(relaxed = true)

    @JvmField
    @RegisterExtension
    val mainDispatcherRule = MainDispatcherRule()

    @BeforeEach
    fun setup() {
        // Stub flows before ViewModel init (init block collects them)
        every { accountService.activeAccountFlow } returns flowOf(testAccount)
        every { entryService.latestEntry } returns flowOf(null)

        viewModel = GoalViewModel(
            dialogUtility = dialogUtility,
            accountService = accountService,
            goalService = goalService,
            entryService = entryService,
        )
    }

    @Test
    fun `Submit calls goalService and navigates back on success`() = runTest {
        coEvery { goalService.updateGoal(any(), any(), any(), any()) } returns mockk()

        viewModel.handleIntent(GoalIntent.Submit)
        advanceUntilIdle()

        coVerify { goalService.updateGoal(any(), any(), any(), any()) }
    }

    @Test
    fun `Submit shows error on failure`() = runTest {
        coEvery { goalService.updateGoal(any(), any(), any(), any()) } throws Exception("API error")

        viewModel.handleIntent(GoalIntent.Submit)
        advanceUntilIdle()

        assertNotNull(viewModel.state.value.error)
    }
}
```

### MainDispatcherRule

Required for ViewModel tests that use `viewModelScope`.

**JUnit 6 (default) — Extension API:**
```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : BeforeEachCallback, AfterEachCallback {
    override fun beforeEach(context: ExtensionContext?) {
        Dispatchers.setMain(dispatcher)
    }
    override fun afterEach(context: ExtensionContext?) {
        Dispatchers.resetMain()
    }
}

// Usage:
@JvmField @RegisterExtension
val mainDispatcherRule = MainDispatcherRule()
```

<details>
<summary>JUnit 4 Legacy — TestWatcher</summary>

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    private val testDispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

// Usage:
@get:Rule val mainDispatcherRule = MainDispatcherRule()
```
</details>

### Infinite Init Coroutine Pattern

If a ViewModel has `while(!isDestroyed) + delay()` or `state.collect` in its init block, use `StandardTestDispatcher` instead of `UnconfinedTestDispatcher`:

```kotlin
private val testDispatcher = StandardTestDispatcher()

@JvmField @RegisterExtension
val mainDispatcherRule = MainDispatcherRule(testDispatcher)

// Never use runTest — it hangs waiting for infinite coroutines
// Never use advanceUntilIdle() — it chases infinite delay() loops forever
// Instead:
private fun advanceScheduler() {
    testDispatcher.scheduler.advanceTimeBy(200)
    testDispatcher.scheduler.runCurrent()
}

@AfterEach
fun tearDown() {
    // Cancel viewModelScope to stop infinite loops
    val method = viewModel::class.java.getDeclaredMethod("onCleared")
    method.isAccessible = true
    method.invoke(viewModel)
}
```

## Repository Tests

```kotlin
class FooRepositoryTest {
    private val fooAPI: IFooAPI = mockk()
    private val fooDao: FooDao = mockk(relaxed = true)
    private lateinit var repository: FooRepository

    @BeforeEach
    fun setup() {
        repository = FooRepository(fooAPI, fooDao)
    }

    @Test
    fun `getFoo fetches from API and caches in DAO`() = runTest {
        val apiResponse = FooResponse(id = "1", name = "Test")
        coEvery { fooAPI.getFoo("1") } returns apiResponse

        val result = repository.getFoo("1")

        assertThat(result.id).isEqualTo("1")
        coVerify { fooDao.insert(any()) }  // Verify caching
    }

    @Test
    fun `getFoo throws on API failure`() = runTest {
        coEvery { fooAPI.getFoo("1") } throws HttpException(mockk(relaxed = true))

        assertThrows<HttpException> {
            repository.getFoo("1")
        }
    }
}
```

## Service Tests

```kotlin
class FooServiceTest {
    private val fooRepository: IFooRepository = mockk()
    private val accountRepository: IAccountRepository = mockk()
    private lateinit var service: FooService

    @BeforeEach
    fun setup() {
        service = FooService(fooRepository, accountRepository)
    }

    @Test
    fun `processAndSaveFoo converts weight unit and saves`() = runTest {
        val account = testAccount.copy(weightUnit = WeightUnit.KG)
        coEvery { accountRepository.getActiveAccount() } returns account
        coEvery { fooRepository.saveFoo(any()) } answers { firstArg() }

        val result = service.processAndSaveFoo(FooInput(weight = 70.0))

        assertTrue(result.isSuccess)
        coVerify { fooRepository.saveFoo(match { it.weight != 70.0 }) } // Converted
    }
}
```

## MockK Patterns

```kotlin
// Relaxed mock — returns defaults for unstubbed calls
private val service: IFooService = mockk(relaxed = true)

// Strict mock — fails on unstubbed calls
private val service: IFooService = mockk()

// Stub suspend function
coEvery { service.getFoo("1") } returns testFoo

// Stub regular function
every { service.isEnabled } returns true

// Stub Flow
every { service.fooFlow } returns flowOf(testFoo)

// Verify suspend call
coVerify { service.saveFoo(any()) }

// Verify call count
coVerify(exactly = 1) { service.saveFoo(any()) }

// Verify no interaction
coVerify(exactly = 0) { service.deleteFoo(any()) }

// Capture argument
val slot = slot<Foo>()
coEvery { service.saveFoo(capture(slot)) } returns testFoo
// then: assertEquals("expected", slot.captured.name)

// Match specific argument
coVerify { service.saveFoo(match { it.name == "Test" }) }

// Answer with transformation
coEvery { service.saveFoo(any()) } answers { firstArg() }
```

## Flow Testing with Turbine

```kotlin
@Test
fun `observeFoos emits updated list`() = runTest {
    val flow = MutableSharedFlow<List<Foo>>()
    every { fooDao.getAll() } returns flow

    repository.observeFoos().test {
        flow.emit(listOf(testFoo))
        assertEquals(listOf(testFoo), awaitItem())

        flow.emit(emptyList())
        assertEquals(emptyList(), awaitItem())

        cancelAndIgnoreRemainingEvents()
    }
}
```

## What to Test

| Component | What to Test | Priority |
|-----------|-------------|----------|
| Reducer | Every intent → state transition | High (easy, pure) |
| ViewModel | Side effects: API calls, navigation, dialogs | High |
| Repository | API → cache → domain mapping, error handling | Medium |
| Service | Business logic, calculations, orchestration | Medium |
| Compose UI | Skip unless explicitly requested | Low |

## What NOT to Test

- Hilt modules (integration tests)
- Simple data classes with no logic
- Compose UI (no UI tests unless asked)
- Room generated code
- Third-party library behavior

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `MainDispatcherRule` | ViewModel tests crash without it |
| Forgetting `advanceUntilIdle()` | Coroutines don't complete — assertions run too early |
| Using `advanceUntilIdle()` with infinite init loops | Hangs forever — use `advanceTimeBy(200)` + `runCurrent()` instead |
| Using `runTest` with infinite init coroutines | `runTest` cleanup waits 60s per test — use plain functions instead |
| Testing UI composables | Only test ViewModel/Reducer — skip UI unless asked |
| Using `mockk()` when `mockk(relaxed = true)` needed | Relaxed avoids stubbing every method on complex mocks |
| Not stubbing flows before ViewModel init | `init` block collects flows — they must be stubbed in `@BeforeEach` |
| Using JUnit 4 annotations with JUnit 6 | `@Before` → `@BeforeEach`, `@get:Rule` → `@JvmField @RegisterExtension` |
