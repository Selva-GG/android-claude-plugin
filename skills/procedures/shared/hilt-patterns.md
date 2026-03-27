# Hilt ViewModel Injection Patterns

## Injection Order

Hilt injects in this order:
1. **Constructor** (`@Inject constructor(...)`) — runs first
2. **`init {}` block** — runs during construction
3. **Field injection** (`@Inject lateinit var`) — runs AFTER constructor
4. **Method injection** (`@Inject fun`) — runs AFTER field injection

**Key implication**: `init {}` cannot access field-injected services.

## BaseViewModel `lateinit` Fields

`BaseViewModel` has these field-injected services:
- `navigationService: IAppNavigationService`
- `dialogQueueService: IDialogQueueService`
- `customTabManager: ICustomTabManager`
- `productSelectionManager: IProductSelectionManager`

### NEVER access these in `init {}`

```kotlin
// WRONG — crashes with UninitializedPropertyAccessException
init {
    productSelectionManager.selectedProduct // CRASH
    navigationService.navigateBack()        // CRASH
    viewModelScope.launch {
        // Also crashes — Dispatchers.Main.immediate runs synchronously
        productSelectionManager.availableProducts.collect { ... }
    }
}
```

### Use `onDependenciesReady()` instead

`BaseViewModel` provides `onDependenciesReady()` — called via `@Inject` method injection AFTER all fields are initialized:

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
    private val myService: IMyService,  // constructor-injected — available in init
) : BaseIntentViewModel<MyState, MyIntent>(MyReducer()) {

    // Constructor-injected services are safe in init
    init {
        viewModelScope.launch {
            myService.data.collect { ... }  // OK
        }
    }

    // BaseViewModel fields are safe here
    override fun onDependenciesReady() {
        val product = productSelectionManager.selectedProduct.value  // OK
        navigationService.navigateTo(...)                             // OK
    }
}
```

### When to use which

| Need | Use |
|------|-----|
| Access constructor-injected service | `init {}` |
| Access `navigationService`, `dialogQueueService`, `productSelectionManager` | `onDependenciesReady()` |
| Access both | Split — constructor services in `init`, BaseViewModel services in `onDependenciesReady` |
| Don't need BaseViewModel services at startup | Just use `init {}` |

## `@AssistedInject` ViewModels

`@HiltViewModel` with `@AssistedInject` + `@AssistedFactory` — Hilt still performs member injection (field + method). `onDependenciesReady()` works correctly.

```kotlin
@HiltViewModel(assistedFactory = DetailViewModel.Factory::class)
class DetailViewModel @AssistedInject constructor(
    private val service: IMyService,
    @Assisted val itemId: String,
) : BaseIntentViewModel<...>(...) {

    @AssistedFactory
    interface Factory { fun create(itemId: String): DetailViewModel }

    override fun onDependenciesReady() {
        // productSelectionManager available here
    }
}
```

## Don't Duplicate Injection

If a service is already in `BaseViewModel` (e.g., `productSelectionManager`), don't also inject it via constructor. Use `onDependenciesReady()` to access it.

```kotlin
// WRONG — duplicate injection
class MyViewModel @Inject constructor(
    private val productSelectionMgr: IProductSelectionManager,  // duplicate!
) : BaseIntentViewModel<...>(...) {
    init { productSelectionMgr.selectedProduct }  // works but wasteful
}

// CORRECT — use onDependenciesReady
class MyViewModel @Inject constructor() : BaseIntentViewModel<...>(...) {
    override fun onDependenciesReady() {
        productSelectionManager.selectedProduct  // from BaseViewModel, no duplicate
    }
}
```

## `viewModelScope.launch` in `init` Behavior

`viewModelScope` uses `Dispatchers.Main.immediate`. When called from `init` on the main thread, the coroutine starts **synchronously** — it does NOT defer to the next frame. This means:

- Any `lateinit` access inside the launched coroutine also crashes
- `yield()` or `delay(0)` can defer execution, but this is fragile
- Prefer `onDependenciesReady()` for coroutines that need BaseViewModel services
