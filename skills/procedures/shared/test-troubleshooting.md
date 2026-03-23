# Test Troubleshooting

> Error-to-fix lookup table for Android/Kotlin unit tests. Referenced by `/android:write-tests` during failure diagnosis.

## MockK Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `no answer found for: Mock.method()` | Missing stub for a method call | Add `coEvery { mock.method() } returns value` or use `mockk(relaxed = true)` |
| `Verification failed: call was not made` | Method wasn't called, or wrong argument matcher | Check argument in `match { }`, or verify the logic path actually calls it |
| `ClassCastException` on mock return | Relaxed mock returning wrong default for generic type | Add explicit `coEvery { }` stub with correct return type |
| `Missing mocked calls inside every { }` | Stub block is empty or uses wrong syntax | Ensure `every { }` block contains exactly one mock call |
| `MockK could not find MockK` on `@MockK` field | `MockKAnnotations.init(this)` not called | Add `MockKAnnotations.init(this)` in `@BeforeEach` |
| `io.mockk.MockKException: no answer found` on `toString` | Mock used in string interpolation | Use `mockk(relaxed = true)` or stub `toString()` |
| `class redefinition failed` with `mockkStatic` | JaCoCo agent conflicts with MockK bytecode manipulation | Wrap the static method behind an interface; mock the interface instead |
| `spyk` + suspend function returns wrong value | Known MockK issue with `spyk` and coroutines | Use `mockk()` with explicit stubs instead of `spyk()` |
| `mockkObject` leaks between tests | Singleton mock state persists | Add `unmockkObject(MySingleton)` in `@AfterEach` |
| `every` used for suspend function | Non-suspend mock API for suspend function | Use `coEvery` instead of `every` for suspend functions |
| `verify` used for suspend function | Non-suspend verify for suspend function | Use `coVerify` instead of `verify` for suspend functions |
| `answers` used for suspend function | Non-suspend answers for suspend function | Use `coAnswers` instead of `answers` |
| `Missing calls inside verify { }` block | Verify block contains no mock interactions | Ensure the verify block calls a mock method, not a real method |

## Coroutine Testing Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Module with the Main dispatcher had failed to initialize` | No `MainDispatcherRule` | Add `@JvmField @RegisterExtension val mainDispatcherRule = MainDispatcherRule()` |
| Test hangs indefinitely (>60s) | Infinite coroutine in init: `while(true) + delay()` or `state.collect` | Use `StandardTestDispatcher` + `advanceTimeBy(200)` + `@AfterEach onCleared()`. Never use `runTest` or `advanceUntilIdle()` |
| `runTest` times out after 60s | Uncompleted coroutine in `viewModelScope` | Either: (a) cancel scope in test, or (b) use `StandardTestDispatcher` without `runTest` |
| Assertions run before coroutine completes | Missing `advanceUntilIdle()` | Add `advanceUntilIdle()` after dispatching intent, before assertions |
| `IllegalStateException: This job has not completed yet` | `runTest` detected leaked coroutines | Cancel all running jobs before `runTest` exits, or use `StandardTestDispatcher` pattern |
| Multiple `TestCoroutineScheduler` desync | Different dispatchers using different schedulers | Ensure ALL dispatchers in the test share the SAME scheduler. Pass the `MainDispatcherRule`'s dispatcher to injected dispatchers |
| `advanceUntilIdle()` hangs forever | `while(!destroyed) + delay()` loop | Use `advanceTimeBy(200) + runCurrent()` instead. The loop creates infinite scheduled tasks |
| `runTest` timeout reduced from 60s | Need faster failure | Use `runTest(timeout = 10.seconds) { }` |
| `Dispatchers.setMain already set` | Multiple rules setting Main dispatcher | Ensure only ONE `MainDispatcherRule` per test class |

## Turbine Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `No value produced in 3s` | Flow didn't emit, or emission was conflated | Check stub returns `flowOf(value)` not `emptyFlow()`. For `StateFlow`, initial value counts as first emission |
| `Unconsumed events` at end of test | Items emitted but not consumed | Add `cancelAndIgnoreRemainingEvents()` at end of `test { }` block |
| `Turbine can only collect flows within a TurbineContext` | Using `.test {}` outside `turbineScope` | Since Turbine 1.0.0+, wrap in `turbineScope { }` for multiple flows |
| Nested `.test {}` blocks don't coordinate | Testing multiple flows with nested blocks | Use `turbineScope { }` + `flow.testIn(this)` instead of nesting |
| `StateFlow` conflates intermediate emissions | StateFlow skips rapid emissions | Use `UnconfinedTestDispatcher` or emit from `MutableStateFlow` with delays |
| `stateIn` flow never activates | `SharingStarted.WhileSubscribed` or `Lazily` needs active collector | Use `backgroundScope.launch { flow.collect {} }` to keep collector active |

## JaCoCo Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Coverage shows 0% despite passing tests | JaCoCo report task ran before tests | Run `./gradlew test` THEN `./gradlew jacocoTestReport` |
| Branch coverage shows uncoverable branches | Kotlin compiler generates null checks for non-null types | See `test-coverage.md` false positives table — exclude from target |
| `when` exhaustive else branch uncovered | Sealed class with all cases covered; `else` is dead code | Exclude from target, document as unreachable |
| Coroutine state machine branches | JaCoCo sees bytecode branches from `suspend` compilation | Upgrade to JaCoCo 0.8.14+ which fixes most Kotlin false positives |
| `data class` `copy()`/`toString()` uncovered | Generated methods that are never called in tests | Exclude from target — testing generated code has no value |

## Assertion Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Expected non-null value` / `IllegalStateException` | `requireNotNull` or `!!` hit null in production code | Ensure mock returns non-null where required |
| `fail("Expected exception") was reached` | Exception not thrown in error test | Check that `coEvery` has `throws` not `returns` |
| `Type mismatch: Required X, found Y` | Mock stub returns wrong type | Fix the `returns` value type to match method signature |
| `Unresolved reference` | Missing import or typo | Add import. Common: `assertThat` needs `com.google.common.truth.Truth.assertThat` |
| Proto builder returns null in verify | Builder chain unreliable with relaxed mocks | Verify method completes without throwing; don't `coVerify` DataStore call args |
