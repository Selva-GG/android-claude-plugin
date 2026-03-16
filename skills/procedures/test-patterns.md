# Test Patterns

> Reusable test patterns for Android/Kotlin unit tests. Referenced by `/android:write-tests` during test generation.

## Dialog Callback Testing

When a method enqueues a dialog (Confirm, Alert), test the callback behavior:

```kotlin
@Test
fun `delete shows confirm dialog and deletes on confirm`() {
    val dialogSlot = slot<DialogModel.Confirm>()

    viewModel.handleIntent(Intent.Delete(itemId))
    advanceScheduler()

    verify { dialogQueueService.enqueue(capture(dialogSlot)) }

    // Invoke the captured callback
    dialogSlot.captured.onConfirm?.invoke()
    advanceScheduler()

    coVerify { service.delete(itemId) }
}

@Test
fun `delete confirm dialog dismiss does not delete`() {
    val dialogSlot = slot<DialogModel.Confirm>()

    viewModel.handleIntent(Intent.Delete(itemId))
    advanceScheduler()

    verify { dialogQueueService.enqueue(capture(dialogSlot)) }

    // Invoke cancel — should NOT trigger deletion
    dialogSlot.captured.onCancel?.invoke()
    advanceScheduler()

    coVerify(exactly = 0) { service.delete(any()) }
}
```

**Key:** Always test both confirm AND cancel/dismiss paths.

---

## Reducer-ViewModel Test Separation

**Rule:** Never re-test reducer logic in ViewModel tests.

- **Reducer tests** → pure state transitions (intent in → state out)
- **ViewModel tests** → side effects only (API calls, navigation, dialogs, toasts)

If an intent is handled by both reducer AND ViewModel:
- Reducer test: verify state fields changed
- ViewModel test: verify `coVerify { service.doThing() }` — do NOT re-assert state

```kotlin
// GOOD: ViewModel test focuses on side effect
@Test
fun `Submit calls service and navigates`() {
    viewModel.handleIntent(Intent.Submit)
    advanceScheduler()
    coVerify { service.save(any()) }
    coVerify { navigationService.navigateBack() }
}

// BAD: ViewModel test re-testing reducer behavior
@Test
fun `Submit sets isLoading true`() {  // This belongs in ReducerTest!
    viewModel.handleIntent(Intent.Submit)
    assertThat(viewModel.state.value.isLoading).isTrue()
}
```

---

## SUT Re-creation Pattern

When testing classes that collect Flows in their `init` block, stubs must be set BEFORE construction. Use a factory method:

```kotlin
private fun createViewModel(
    items: Flow<List<Item>> = flowOf(emptyList()),
    account: Flow<Account?> = flowOf(testAccount),
): FooViewModel {
    every { itemService.itemsFlow } returns items
    every { accountService.activeAccountFlow } returns account
    return FooViewModel(itemService, accountService).initTestDependencies()
}

@Test
fun `shows items from flow`() {
    val vm = createViewModel(items = flowOf(listOf(testItem)))
    advanceScheduler()
    assertThat(vm.state.value.items).hasSize(1)
}

@Test
fun `handles null account`() {
    val vm = createViewModel(account = flowOf(null))
    advanceScheduler()
    assertThat(vm.state.value.account).isNull()
}
```

**When to use:** Any class where `init {}` calls `viewModelScope.launch { flow.collect { } }`. You cannot re-stub after construction — the init already captured the old stub.

---

## State Preservation Verification

When testing a reducer intent, verify that UNRELATED fields are NOT modified:

```kotlin
@Test
fun `SetLoading only changes isLoading`() {
    val initial = state.copy(
        items = listOf(testItem),
        error = "previous error",
        selectedTab = Tab.HISTORY,
    )

    val result = reducer.reduce(initial, Intent.SetLoading(true))

    // Target field changed
    assertThat(result?.isLoading).isTrue()
    // Unrelated fields preserved
    assertThat(result?.items).isEqualTo(initial.items)
    assertThat(result?.error).isEqualTo(initial.error)
    assertThat(result?.selectedTab).isEqualTo(initial.selectedTab)
}
```

**When to use:** Every reducer test. Prevents bugs where `state.copy()` accidentally drops fields.

---

## Network Guard Clause Patterns

Services often have two network check patterns:

### Hard throw (`requireNetworkAvailable()`)
Throws if offline — test that exception propagates:

```kotlin
@Test
fun `sync throws when offline`() {
    stubNetworkUnavailable()
    assertThrows<NetworkException> { service.syncData() }
    coVerify(exactly = 0) { api.upload(any()) }  // API never called
}
```

### Soft check (`isNetworkAvailable()`)
Returns early without throwing — test that method is a no-op:

```kotlin
@Test
fun `sync skips silently when offline`() {
    stubNetworkUnavailable()
    service.syncData()  // No exception
    coVerify(exactly = 0) { api.upload(any()) }
}
```

---

## `.value` vs Turbine Guidance

**Rule:** Use `state.value` for final-state assertions. Use Turbine only for intermediate emissions.

```kotlin
// GOOD: Final state — use .value
@Test
fun `Submit updates state on success`() {
    viewModel.handleIntent(Intent.Submit)
    advanceScheduler()
    assertThat(viewModel.state.value.isLoading).isFalse()
    assertThat(viewModel.state.value.error).isNull()
}

// GOOD: Intermediate states — use Turbine
@Test
fun `Submit shows loading then clears`() {
    viewModel.state.test {
        skipItems(1) // initial state
        viewModel.handleIntent(Intent.Submit)
        assertThat(awaitItem().isLoading).isTrue()   // intermediate
        assertThat(awaitItem().isLoading).isFalse()  // final
        cancelAndIgnoreRemainingEvents()
    }
}
```

**Why:** `StateFlow` conflates emissions. `.value` always gives the latest. Turbine captures every emission including intermediate ones that `.value` would miss.

---

## Insert-or-Update Branching (Repository)

When a repository method does insert-or-update, test both paths:

```kotlin
@Test
fun `save inserts new record when not exists`() = runTest {
    coEvery { dao.findById(ID) } returns null
    repository.save(testItem)
    coVerify { dao.insert(any()) }
    coVerify(exactly = 0) { dao.update(any()) }
}

@Test
fun `save updates existing record when exists`() = runTest {
    coEvery { dao.findById(ID) } returns existingEntity
    repository.save(testItem)
    coVerify { dao.update(any()) }
    coVerify(exactly = 0) { dao.insert(any()) }
}
```

---

## Request Object Verification (slot + capture)

When verifying HOW a dependency was called (not just THAT it was called):

```kotlin
@Test
fun `save constructs correct API request`() = runTest {
    val requestSlot = slot<SaveRequest>()
    coEvery { api.save(capture(requestSlot)) } returns mockResponse

    service.save(name = "Test", weight = 75.5)

    val captured = requestSlot.captured
    assertThat(captured.name).isEqualTo("Test")
    assertThat(captured.weight).isEqualTo(75.5)
    assertThat(captured.timestamp).isNotNull()
}
```

**When to use:** Methods that build request objects from multiple parameters. Verifies the mapping is correct.

---

## Catch-and-Swallow vs Catch-and-Rethrow

Two exception handling patterns require different test strategies:

### Catch-and-swallow (logs but doesn't throw)
```kotlin
@Test
fun `sync logs error but does not throw on API failure`() = runTest {
    coEvery { api.sync() } throws RuntimeException("fail")
    service.syncData()  // Should NOT throw
    // Optionally verify error was logged or state was updated
}
```

### Catch-and-rethrow (wraps or propagates)
```kotlin
@Test
fun `fetch rethrows API exception`() = runTest {
    coEvery { api.fetch() } throws HttpException(mockResponse(500))
    assertThrows<HttpException> { service.fetchData() }
}
```

**Key:** Read the source to determine which pattern is used. Test accordingly.
