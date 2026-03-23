---
name: implement-layers
description: Guide the orchestrator through sequential layer-by-layer implementation (Domain → Data → Core → Service → ViewModel → UI) with inline tests per layer
user-invocable: false
---

# Implement Layers

Guide the orchestrator through implementing the plan produced by `analyse-plan`. Layers are implemented sequentially because each depends on the previous one's contracts.

## Input

Receives the confirmed implementation plan containing:
- Affected layers (which are active, which are skipped)
- Files to create/modify per layer
- Procedures to load per layer
- Key design decisions

## Execution Order

```
Domain → Data → Core → Service → ViewModel → UI
```

**Skip any layer marked as not affected in the plan.** Only implement active layers.

## Layer 1: Domain (pure Kotlin)

**Purpose:** Define interfaces, models, enums — the contracts that other layers depend on.

**Procedures to load:** None (pure Kotlin, no Android dependencies)

**What to create/modify:**
- `domain/enums/` — new enums
- `domain/model/` — new data classes (in appropriate sub-dir: `api/`, `storage/`, `common/`, `feature/`)
- `domain/repository/` — new repository interfaces (`I` prefix required)
- `domain/services/` — new service interfaces (`I` prefix required)
- `domain/interfaces/` — new shared interfaces

**Rules:**
- No Android framework imports
- No library dependencies (except Kotlin stdlib)
- All interfaces use `I` prefix
- Data classes are immutable
- All repository/service methods are `suspend` (except Flow-returning properties)

**Tests:** Usually none needed (interfaces + data classes only). Test if there's logic in enums or model methods.

**Completion:** Orchestrator now knows all interface signatures and model shapes for subsequent layers.

---

## Layer 2: Data (Room + Retrofit + DataStore)

**Purpose:** Implement domain interfaces — connect to APIs, databases, and local storage.

**Procedures to load:**
- `procedures/data/storage.md` (if touching Room/DataStore)
- `procedures/data/networking-api.md` (if touching Retrofit/API)
- `procedures/data/services-repositories.md` (for repository patterns)

**What to create/modify:**
- `data/storage/db/entity/` — Room entities (`@Entity`)
- `data/storage/db/dao/` — Room DAOs (`@Dao`)
- `data/storage/db/AppDatabase.kt` — add new entities and DAOs
- `data/storage/db/converter/` — type converters if needed
- `data/storage/datastore/` — Proto DataStore classes (follow `BaseDataStore` pattern)
- `data/api/` — Retrofit API interfaces
- `data/repository/` — Repository implementations
- `data/services/` — Service implementations (data-heavy ones)

**Rules:**
- Entities use `@Entity(tableName = "snake_case")`
- DAOs use `@Insert(onConflict = REPLACE)`, `suspend` for write ops, `Flow` for observe ops
- Repositories implement domain interfaces
- All API methods are `suspend`
- No `!!` operators — use `?: return`, `?.let`, `requireNotNull()`

**Tests — write immediately after this layer:**
- Repository tests: mock API/DAO/DataStore, test CRUD + error paths + Flow delegation
- Use `@MockK(relaxUnitFun = true)` for DAOs, `@MockK` for APIs
- Follow patterns in `procedures/shared/testing.md`

---

## Layer 3: Core (DI + Infrastructure)

**Purpose:** Wire new interfaces to implementations via Hilt. Update infrastructure if needed.

**Procedures to load:**
- `procedures/core/dependency-injection.md`

**What to create/modify:**
- `core/di/RepositoryModule.kt` — `@Binds` for new repositories
- `core/di/ServiceModule.kt` — `@Binds` for new services
- `core/di/DataStoreModule.kt` — `@Provides` for new DataStores
- `core/di/DatabaseModule.kt` — `@Provides` for new DAOs
- `core/di/APIModule.kt` — `@Provides` for new API interfaces
- `core/navigation/AppRoute.kt` — new routes if needed
- `core/network/` — interceptors/config if needed

**Rules:**
- All bindings go in `SingletonComponent`
- Use `@Binds` for interface → implementation (abstract fun)
- Use `@Provides` for concrete instances (fun with body)
- Follow existing module patterns exactly

**Tests:** Usually none (DI is verified by build compilation and integration tests).

---

## Layer 4: Service (business logic)

**Purpose:** Implement service interfaces with business logic, orchestration, and state management.

**Procedures to load:**
- `procedures/data/services-repositories.md`

**What to create/modify:**
- `core/service/` — new service implementations
- `data/services/` — new data-heavy service implementations

**Rules:**
- Services inject repository/other service INTERFACES, never implementations
- Use `@ApplicationScope` for coroutine scopes, never manual `CoroutineScope()`
- Log with `AppLog` using a `TAG` constant in companion object
- Keep methods ≤ 60 lines, classes ≤ 600 lines (detekt enforces)

**Tests — write immediately after this layer:**
- Mock all injected repositories and services
- Test business logic, orchestration, error handling, fallback behavior
- Test Flow combinations and transformations
- Follow patterns in `procedures/shared/testing.md`

---

## Layer 5: ViewModel (MVI state management)

**Purpose:** Connect services to UI via the MVI pattern (State + Intent + Reducer + ViewModel).

**Procedures to load:**
- `procedures/ui/mvi-pattern.md`

**What to create/modify:**
- `features/<feature>/viewmodel/<Feature>ViewModel.kt` — extends `BaseIntentViewModel`
- `features/<feature>/viewmodel/<Feature>Reducer.kt` — pure reducer + State + Intent
- `features/<feature>/strings/<Feature>Strings.kt` — static text constants

**Rules:**
- ViewModel extends `BaseIntentViewModel<State, Intent>` (or `BaseViewModel` if no reducer)
- Always call `super.handleIntent(intent)` FIRST in `handleIntent()`
- State is an immutable `data class` with `@Stable` annotation and `ImmutableList` for lists
- Reducer is a pure function: `(State, Intent) → State?` — no side effects, no coroutines
- Side effects (API calls, navigation) go in ViewModel's `handleIntent()`, not in Reducer
- Never inject `NavigationService` or `DialogQueueService` — they're in `BaseViewModel`
- Use `@AssistedInject` + `@AssistedFactory` for ViewModels with runtime parameters

**Tests — write immediately after this layer:**
- Reducer tests: pure function testing, no mocks needed, test every intent + state preservation
- ViewModel tests: mock services, test side effects (navigation, dialog, API calls)
- Use `.value` for final state assertions, Turbine only for intermediate emissions
- Never re-test reducer logic in ViewModel tests

---

## Layer 6: UI (Jetpack Compose)

**Purpose:** Build the screens and components that the user sees.

**Procedures to load:**
- `procedures/ui/compose-ui.md`
- `procedures/ui/navigation-dialogs.md` (if new routes)

**What to create/modify:**
- `features/<feature>/<Feature>Screen.kt` — main composable
- `features/<feature>/components/` — sub-composables
- `features/<feature>/strings/<Feature>Strings.kt` — if not created in Layer 5

**Rules:**
- All previews use `@PreviewTheme` + wrap in `MeAppTheme { }`
- Never hardcode colors/typography/spacing — use `MeAppTheme.colorScheme.*`, `.typography.*`, `.spacing.*`
- One composable per file (only `@Preview` shares a file)
- Check `features/common/` (component-catalog) for existing composables before creating new ones
- Use `collectAsStateWithLifecycle()`, never `collectAsState()`
- `LazyColumn`/`LazyRow` must have `key` per item
- Static text in `strings/` objects, never hardcoded in composables

**Tests:** UI tests are not written inline. They are covered by the validation step (test-runner agent).

---

## Completion Signal

When all active layers are implemented:

> **Implementation complete.**
>
> | Layer | Status | Files | Tests |
> |-------|--------|-------|-------|
> | Domain | [done/skipped] | N created, M modified | N tests |
> | Data | [done/skipped] | N created, M modified | N tests |
> | Core | [done/skipped] | N created, M modified | — |
> | Service | [done/skipped] | N created, M modified | N tests |
> | ViewModel | [done/skipped] | N created, M modified | N tests |
> | UI | [done/skipped] | N created, M modified | — |
>
> Ready for validation. Proceeding to Step 5.

The orchestrator then moves to the validation step (Step 5).
