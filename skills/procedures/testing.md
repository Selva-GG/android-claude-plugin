# Testing

## Setup

| Library | Purpose |
|---------|---------|
| JUnit 4 | Test framework |
| MockK | Mocking (Kotlin-native) |
| Turbine | Flow testing |
| kotlinx-coroutines-test | `runTest`, `TestDispatcher` |

Test location: `app/src/test/java/com/dmdbrands/gurus/weight/`

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

        assertNotNull(result)
        assertTrue(result!!.isLoading)
        assertNull(result.error)
    }

    @Test
    fun `SetError sets error message and stops loading`() {
        val loadingState = initialState.copy(isLoading = true)
        val result = reducer.reduce(loadingState, GoalIntent.Error("Failed"))

        assertNotNull(result)
        assertFalse(result!!.isLoading)
        assertEquals("Failed", result.error)
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

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Before
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

Required for ViewModel tests that use `viewModelScope`:

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
```

## Repository Tests

```kotlin
class FooRepositoryTest {
    private val fooAPI: IFooAPI = mockk()
    private val fooDao: FooDao = mockk(relaxed = true)
    private lateinit var repository: FooRepository

    @Before
    fun setup() {
        repository = FooRepository(fooAPI, fooDao)
    }

    @Test
    fun `getFoo fetches from API and caches in DAO`() = runTest {
        val apiResponse = FooResponse(id = "1", name = "Test")
        coEvery { fooAPI.getFoo("1") } returns apiResponse

        val result = repository.getFoo("1")

        assertEquals("1", result.id)
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

    @Before
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
| Testing UI composables | Only test ViewModel/Reducer — skip UI unless asked |
| Using `mockk()` when `mockk(relaxed = true)` needed | Relaxed avoids stubbing every method on complex mocks |
| Not stubbing flows before ViewModel init | `init` block collects flows — they must be stubbed in `@Before` |
