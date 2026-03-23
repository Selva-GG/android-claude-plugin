# Review Checklist

Complete this checklist before finishing any task. Every item must pass.

## Architecture

- [ ] MVI pattern followed: State is `@Stable data class`, Reducer is pure (no side effects), ViewModel calls `super.handleIntent()` first
- [ ] State classes annotated with `@Stable`
- [ ] List fields in state use `ImmutableList<T>` with `persistentListOf()` defaults
- [ ] List→ImmutableList conversions (`toPersistentList()`) in Reducer, not ViewModel
- [ ] Clean Architecture layers respected: domain has no Android deps, data implements domain interfaces, features use domain models
- [ ] New DI bindings added to appropriate module in `core/di/`
- [ ] Repository handles only data access — no business logic
- [ ] Service handles business logic — no direct API/DAO calls
- [ ] Services/repos use `@ApplicationScope` coroutine scope — no manual `CoroutineScope()`
- [ ] Navigation uses `navigationService` from BaseViewModel, not injected directly
- [ ] Dialogs/toasts use `dialogQueueService` from BaseViewModel, not injected directly

## Null Safety

- [ ] No `!!` operators anywhere (detekt will catch `UnsafeCallOnNullableType`)
- [ ] Safe alternatives used: `?: return`, `?: return@launch`, `?.let {}`, `requireNotNull()`
- [ ] `var` class properties captured to local `val` before null checks (smart-cast)

## Compose & UI

- [ ] State collected with `collectAsStateWithLifecycle()`, not `collectAsState()`
- [ ] All `LazyColumn`/`LazyRow`/`LazyVerticalGrid` have stable `key` parameters
- [ ] All colors use `MeAppTheme.colorScheme.*` — no hardcoded `Color(...)` values
- [ ] All typography uses `MeAppTheme.typography.*` — no hardcoded text styles
- [ ] All spacing uses `MeAppTheme.spacing.*` — no hardcoded `dp` values
- [ ] Compose previews use `@PreviewTheme` and wrap in `MeAppTheme { }`
- [ ] One composable per file (only previews share a file)
- [ ] Shared composables from `features/common/` used where applicable
- [ ] `AppInput` fields have `imeAction` and `onImeAction` with `FocusRequester`
- [ ] Checked `res/drawable/` for existing assets before adding new ones

## Naming & Conventions

- [ ] Interfaces use `I` prefix (`IFooRepository`, `IFooService`)
- [ ] Static text in PascalCase `Strings` objects in feature's `strings/` folder
- [ ] Logging uses `AppLog` with `TAG` companion constant — not `Log` or `Timber`
- [ ] API methods are `suspend` functions
- [ ] Feature files follow structure: Screen, ViewModel, Reducer, State, Intent, components/, strings/
- [ ] Boolean variables use `is`/`has`/`can` prefix: `isLoading`, `hasError`, `canDelete`
- [ ] Classes PascalCase, functions/variables camelCase, constants UPPER_SNAKE_CASE

## Code Quality (Detekt Enforced)

- [ ] No `!!` operators — `UnsafeCallOnNullableType`
- [ ] No `runBlocking` — `ForbiddenImport`
- [ ] No `GlobalScope` — `GlobalCoroutineUsage`
- [ ] No TODO/FIXME/STOPSHIP comments — `ForbiddenComment`
- [ ] Methods ≤ 60 lines — `LongMethod`
- [ ] Classes ≤ 600 lines — `LargeClass`
- [ ] Caught exceptions not silently swallowed — `SwallowedException`
- [ ] No `SimpleDateFormat` — use `DateTimeFormatter` (thread-safe)
- [ ] No debug code or commented-out code left behind
- [ ] No secrets or credentials in the diff
- [ ] No unused imports
- [ ] No duplicate methods — check for existing implementations first
- [ ] Error handling in all API/IO operations (try/catch or Result)
- [ ] Loaders dismissed in both success and error paths
- [ ] KDoc on public functions and complex logic

## Testing

- [ ] Reducer tests cover all intents → state transitions
- [ ] ViewModel tests cover side effects (API calls, navigation, dialogs)
- [ ] Tests use `MainDispatcherRule` for ViewModel tests
- [ ] MockK used for mocking (`coEvery`/`coVerify` for suspend functions)
- [ ] Tests use backtick naming: `` `action produces expected result` ``

## Build Verification

```bash
./gradlew assembleDebug    # Must pass
./gradlew test             # Must pass
./gradlew detekt           # Must pass
```

All three must pass before finishing.
