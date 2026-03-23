# Services & Repositories

## Layer Separation

```
ViewModel → Service (business logic) → Repository (data access) → API / DAO / DataStore
```

| Layer | Responsibility | Location |
|-------|---------------|----------|
| Repository | Data fetching, caching, model mapping. NO business logic. | `data/repository/` |
| Service | Business logic, orchestration, combines multiple repos. | `core/service/` or `data/services/` |
| ViewModel | UI logic, side effects, state management. | `features/<feature>/` |

## Repository Pattern

### Interface (Domain)

```kotlin
// domain/repository/IFooRepository.kt
interface IFooRepository {
    suspend fun getFoo(id: String): Foo
    suspend fun saveFoo(foo: Foo): Foo
    suspend fun deleteFoo(id: String)
    fun observeFoos(): Flow<List<Foo>>
    suspend fun getUnsyncedFoos(): List<Foo>
}
```

Rules:
- Lives in `domain/repository/` — pure Kotlin, no Android dependencies
- `I` prefix naming: `IFooRepository`
- `suspend` for one-shot operations
- `Flow<T>` for reactive/observable data
- Returns domain models, not API or entity types
- KDoc on each method

### Implementation (Data)

```kotlin
// data/repository/FooRepository.kt
@Singleton
class FooRepository @Inject constructor(
    private val fooAPI: IFooAPI,
    private val fooDao: FooDao,
) : IFooRepository {

    override suspend fun getFoo(id: String): Foo {
        val response = fooAPI.getFoo(id)
        val entity = response.toEntity()
        fooDao.insert(entity)  // Cache locally
        return entity.toDomain()
    }

    override fun observeFoos(): Flow<List<Foo>> {
        return fooDao.getAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

Rules:
- `@Singleton` + `@Inject constructor`
- Implements the `I`-prefixed interface
- Handles: API calls, DAO operations, model mapping
- **NO business logic** — no calculations, no conditional flows, no orchestration
- Maps between 3 model types: API response ↔ Entity ↔ Domain model

### Registration

```kotlin
// core/di/RepositoryModule.kt
@Provides @Singleton
fun provideFooRepository(api: IFooAPI, dao: FooDao): IFooRepository =
    FooRepository(api, dao)
```

## Service Pattern

### Interface (Domain)

```kotlin
// domain/services/IFooService.kt
interface IFooService {
    suspend fun processAndSaveFoo(input: FooInput): Result<Foo>
    val activeFoo: StateFlow<Foo?>
    fun calculateFooScore(foo: Foo, account: Account): Int
}
```

Rules:
- Lives in `domain/services/` or `domain/interfaces/`
- `I` prefix naming: `IFooService`
- Contains business logic method signatures
- Can expose `StateFlow` for reactive state

### Implementation

```kotlin
// core/service/FooService.kt
@Singleton
class FooService @Inject constructor(
    private val fooRepository: IFooRepository,
    private val accountRepository: IAccountRepository,
) : IFooService {

    private val _activeFoo = MutableStateFlow<Foo?>(null)
    override val activeFoo: StateFlow<Foo?> = _activeFoo.asStateFlow()

    override suspend fun processAndSaveFoo(input: FooInput): Result<Foo> {
        // Business logic here
        val account = accountRepository.getActiveAccount() ?: return Result.failure(...)
        val processed = input.toFoo(weightUnit = account.weightUnit)
        return try {
            val saved = fooRepository.saveFoo(processed)
            _activeFoo.value = saved
            Result.success(saved)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override fun calculateFooScore(foo: Foo, account: Account): Int {
        // Pure business logic — calculations, rules, transformations
        return when {
            foo.value > account.target -> 100
            foo.value > account.target * 0.8 -> 80
            else -> 50
        }
    }
}
```

Rules:
- Business logic, orchestration, combining multiple repositories
- Can hold reactive state via `StateFlow`
- Inject repositories (interfaces), never DAOs or APIs directly
- Handle errors and return `Result<T>` or throw

### Registration

```kotlin
// core/di/ServiceModule.kt
@Provides @Singleton
fun provideFooService(
    fooRepo: IFooRepository,
    accountRepo: IAccountRepository,
): IFooService = FooService(fooRepo, accountRepo)
```

## Model Mapping

Three model layers with extension functions for conversion:

```
API Model (FooResponse)  ←→  Entity (FooEntity)  ←→  Domain Model (Foo)
        ↑ from API                ↑ in DB                  ↑ used everywhere
```

```kotlin
// Mapping extensions (typically in repository or model file)
fun FooResponse.toEntity() = FooEntity(id = id, name = name, weight = weight)
fun FooEntity.toDomain() = Foo(id = id, name = name, weight = weight)
fun Foo.toRequest() = CreateFooRequest(name = name, weight = weight)
```

## Weight Unit Convention

Weights are stored as **tenths** (e.g., 152.1 lbs = 1521). Always convert for display:

```kotlin
val displayWeight = storedWeight / 10.0  // 1521 → 152.1
```

Conversion between LB/KG happens at the service layer using `IUnitProcessable`.

## Coroutine Scope Management

**NEVER** create manual `CoroutineScope()` in services or repositories. Inject the shared application scope instead.

```kotlin
// core/di/ApplicationScope.kt
@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class ApplicationScope

// core/di/CoroutineScopeModule.kt
@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopeModule {
    @Provides @Singleton @ApplicationScope
    fun provideApplicationScope(): CoroutineScope =
        CoroutineScope(SupervisorJob() + Dispatchers.IO)
}
```

Usage in services:

```kotlin
@Singleton
class FooService @Inject constructor(
    @ApplicationScope private val scope: CoroutineScope,
    private val fooRepository: IFooRepository,
) : IFooService {

    override fun startSync() {
        scope.launch {  // ✅ Shared supervised scope
            fooRepository.syncAll()
        }
    }
}
```

Why: Manual `CoroutineScope()` instances cause memory leaks, orphan coroutines, and race conditions (e.g., AccountService cancel-and-recreate on account switch).

## Date Formatting

Use **`DateTimeFormatter`** (thread-safe), never `SimpleDateFormat`:

```kotlin
// ✅ Thread-safe — safe in OkHttp's concurrent interceptor threads
private val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")

// ❌ NOT thread-safe — crashes under concurrent access
private val formatter = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", Locale.US)
```

This is critical in `AuthTokenInterceptor` and `TokenAuthenticator` which run on OkHttp's thread pool.

## Null Safety in Repositories/Services

```kotlin
// ✅ Early return
val account = accountRepository.getActive() ?: return Result.failure(NoAccountException())

// ✅ Safe access in Flow
fooDao.getById(id).map { entity ->
    entity?.toDomain()
}

// ❌ NEVER
val account = accountRepository.getActive()!!
```

## Existing Key Repositories

| Interface | Implementation | Data Sources |
|-----------|---------------|--------------|
| `IAccountRepository` | `AccountRepository` | AuthAPI, UserAPI, AccountDao, UserDataStore |
| `IEntryRepository` | `EntryRepository` | EntryAPI, EntryDao |
| `IGoalRepository` | `GoalRepository` | GoalAPI, GoalSettingsEntity |
| `IDeviceRepository` | `DeviceRepository` | DeviceAPI, DeviceDao |
| `IIntegrationRepository` | `IntegrationRepository` | IntegrationAPI, IntegrationsSettingsEntity |

## Existing Key Services

| Interface | Implementation | Purpose |
|-----------|---------------|---------|
| `IAccountService` | `AccountService` | Account CRUD, active account flow, multi-account |
| `IEntryService` | `EntryService` | Weight entries, latest entry flow |
| `IGoalService` | `GoalService` | Goal management, progress calculation |
| `IDeviceService` | `DeviceService` | Scale pairing, device management |
| `IDashboardService` | `DashboardService` | Dashboard metrics aggregation |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Business logic in repository | Move to service — repos only fetch/cache/map |
| Injecting DAO in ViewModel | Inject service → service uses repo → repo uses DAO |
| Injecting API in service | Inject repository — services don't touch APIs directly |
| Missing `@Singleton` | All repos and services must be singleton-scoped |
| Returning entity from repository | Map to domain model before returning |
| Missing model mapping | Each layer has its own model type — map between them |
| Manual `CoroutineScope()` in service | Inject `@ApplicationScope` instead |
| `SimpleDateFormat` in concurrent code | Use `DateTimeFormatter` (thread-safe) |
