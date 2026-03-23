# MVI Pattern

## Core Classes

| Class | Location | Role |
|-------|----------|------|
| `IReducer<State, Intent>` | `domain/interfaces/IReducer.kt` | Interface — pure `reduce(state, intent) → State?` |
| `BaseIntentViewModel<State, Intent>` | `features/common/service/BaseIntentViewModel.kt` | Abstract ViewModel with `_state: MutableStateFlow`, `handleIntent()` |
| `BaseViewModel` | `features/common/viewmodel/BaseViewModel.kt` | Injects `navigationService`, `dialogQueueService`, `customTabManager` |

## Flow

```
User action → handleIntent(Intent) → super.handleIntent() → reducer.reduce() → new State → UI recomposes
                                   → ViewModel handles side effects (API, navigation, dialogs)
```

## State — @Stable Immutable Data Class

```kotlin
import androidx.compose.runtime.Stable
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

@Stable
data class FooState(
    val isLoading: Boolean = false,
    val items: ImmutableList<Item> = persistentListOf(),
    val error: String? = null,
    val account: Account? = null,
) : IReducer.State
```

Rules:
- **`@Stable` annotation** on ALL state data classes — tells Compose compiler that `equals()` is reliable for skip decisions
- Use **`ImmutableList<T>`** (not `List<T>`) for list fields — Compose treats `List<T>` as unstable regardless of `@Stable`
- Default list values use **`persistentListOf()`** (not `emptyList()`)
- Always a `data class` implementing `IReducer.State`
- All fields have defaults (so `provideInitialState()` is clean)
- Immutable — update via `.copy()`
- For forms, use `FormGroup<FooFormControls>` with `FormControl<T>` fields

**Why `@Stable` and not `@Immutable`?** `@Stable` says "equals() works for recomposition skipping" — always true for data classes. `@Immutable` requires deep immutability, which breaks for classes with mutable references.

## Intent — Sealed Interface

```kotlin
sealed interface FooIntent : IReducer.Intent {
    // UI-triggered actions
    data object Load : FooIntent
    data object Submit : FooIntent
    data object OnBack : FooIntent

    // Data-setting intents (handled by reducer)
    data class SetQuery(val query: String) : FooIntent
    data class SetItems(val items: List<Item>) : FooIntent  // accepts List, reducer converts to ImmutableList
    data class SetError(val message: String) : FooIntent
}
```

Rules:
- `sealed interface` implementing `IReducer.Intent`
- Use `data object` for no-arg intents, `data class` for parameterized
- Name pattern: `Load`, `Submit`, `OnBack` for actions; `Set*`, `Update*` for state mutations

## Reducer — Pure Function

```kotlin
class FooReducer : IReducer<FooState, FooIntent> {
    override fun reduce(state: FooState, intent: FooIntent): FooState? = when (intent) {
        is FooIntent.SetQuery -> state.copy(query = intent.query)
        is FooIntent.SetItems -> state.copy(items = intent.items.toPersistentList(), isLoading = false)
        is FooIntent.SetError -> state.copy(error = intent.message, isLoading = false)
        is FooIntent.Submit -> state.copy(isLoading = true, error = null)
        else -> null // intents handled only in ViewModel return null
    }
}
```

Rules:
- **PURE** — no coroutines, no API calls, no navigation, no side effects
- **List→ImmutableList conversions** (`toPersistentList()`) happen HERE in the Reducer, not in ViewModels or Screens
- Returns `null` for intents it doesn't handle (ViewModel handles those)
- Returns `state.copy()` (no changes) if intent is acknowledged but no state change needed
- One file: `<Feature>Reducer.kt` contains the Reducer class. State and Intent can live here too or in separate files.

## ViewModel — Side Effects

```kotlin
@HiltViewModel
class FooViewModel @Inject constructor(
    private val fooService: IFooService,
) : BaseIntentViewModel<FooState, FooIntent>(FooReducer()) {

    override fun provideInitialState() = FooState()

    override fun handleIntent(intent: FooIntent) {
        super.handleIntent(intent) // ALWAYS FIRST — routes to reducer
        when (intent) {
            FooIntent.Load -> load()
            FooIntent.Submit -> submit()
            FooIntent.OnBack -> navigationService.navigateBack()
            else -> {} // intents fully handled by reducer
        }
    }

    private fun load() {
        viewModelScope.launch {
            try {
                val items = fooService.getItems()
                handleIntent(FooIntent.SetItems(items))
            } catch (e: Exception) {
                handleIntent(FooIntent.SetError(e.message ?: "Failed to load"))
            }
        }
    }

    private fun submit() {
        dialogQueueService.showLoader(message = FooStrings.Saving)
        viewModelScope.launch {
            try {
                fooService.save(state.value.form.controls.toModel())
                dialogQueueService.dismissLoader()
                dialogQueueService.showToast(Toast(message = FooStrings.Saved))
                navigationService.navigateBack()
            } catch (e: Exception) {
                dialogQueueService.dismissLoader()
                handleIntent(FooIntent.SetError(e.message ?: "Failed to save"))
            }
        }
    }

    companion object {
        private const val TAG = "FooViewModel"
    }
}
```

Rules:
- `super.handleIntent(intent)` is **ALWAYS** the first line
- Side effects (API calls, navigation, dialogs) only in ViewModel, never in Reducer
- Use `viewModelScope.launch` for coroutines
- Update state by calling `handleIntent()` with data-setting intents (e.g., `SetItems`, `SetError`)
- Use `TAG` companion for `AppLog` logging
- `navigationService` and `dialogQueueService` are inherited from `BaseViewModel` — never inject them

## AssistedInject (Runtime Parameters)

When a ViewModel needs a runtime value (e.g., `scaleId` from navigation):

```kotlin
@HiltViewModel(assistedFactory = FooViewModel.Factory::class)
class FooViewModel @AssistedInject constructor(
    @Assisted val scaleId: String,
    private val deviceService: IDeviceService,
) : BaseIntentViewModel<FooState, FooIntent>(FooReducer()) {

    @AssistedFactory
    interface Factory {
        fun create(scaleId: String): FooViewModel
    }

    override fun provideInitialState() = FooState()
}
```

## Collecting State in Compose

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun FooScreen() {
    val viewModel: FooViewModel = hiltViewModel()
    val state by viewModel.state.collectAsStateWithLifecycle()

    BackHandler { viewModel.handleIntent(FooIntent.OnBack) }

    FooContent(state = state, onIntent = viewModel::handleIntent)
}

@Composable
private fun FooContent(state: FooState, onIntent: (FooIntent) -> Unit) {
    // Stateless composable — receives state, emits intents
}
```

Rules:
- **Always `collectAsStateWithLifecycle()`** — never plain `collectAsState()`
- Lifecycle-aware: stops collecting when below STARTED state, reducing unnecessary recompositions and memory pressure
- Import: `androidx.lifecycle.compose.collectAsStateWithLifecycle`
- Screen collects state + wires intents. Content is stateless and testable.

## Init Block Pattern

Load data on ViewModel creation:

```kotlin
init {
    viewModelScope.launch {
        accountService.activeAccountFlow.collect { account ->
            handleIntent(FooIntent.UpdateAccount(account))
        }
    }
}
```

Or one-shot:

```kotlin
init {
    handleIntent(FooIntent.Load)
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `super.handleIntent(intent)` | Reducer never runs — state never updates |
| Side effects in Reducer | Move to ViewModel's `handleIntent()` |
| Updating `_state.value` directly | Always go through `handleIntent()` → reducer |
| Using `!!` on state fields | Use `?: return` or `?.let {}` |
| Injecting navigationService | Already in BaseViewModel — just use it |
| `List<T>` in state fields | Use `ImmutableList<T>` with `persistentListOf()` default |
| Missing `@Stable` on state class | Add it — prevents unnecessary recompositions |
| `toPersistentList()` in ViewModel | Move conversion to Reducer |
| `collectAsState()` | Use `collectAsStateWithLifecycle()` instead |
