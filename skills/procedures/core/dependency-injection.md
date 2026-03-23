# Dependency Injection (Hilt)

## Module Organization

All DI modules live in `core/di/`. Key modules:

| Module | Binds |
|--------|-------|
| `RepositoryModule` | 15+ repository interfaces → implementations |
| `ServiceModule` | 20+ service interfaces → implementations |
| `APIModule` | Retrofit API service creation |
| `NetworkModule` | HttpClient, OkHttpClient, interceptors |
| `DatabaseModule` | Room AppDatabase instance |
| `DataStoreModule` | Protobuf DataStore instances |
| `HealthConnectModule` | Health Connect client |
| `BLEServiceModule` | Bluetooth service |
| `NotificationModule` | Firebase notification handling |
| `UtilityModule` | Utility services |
| `LoggingModule` | Logging infrastructure |
| `InitializationModule` | App startup services |
| `SettingsManagerModule` | Settings management |

## Binding Pattern

### Repository/Service Binding

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {

    @Provides
    @Singleton
    fun provideAccountRepository(
        accountDao: AccountDao,
        userDataStore: UserDataStore,
        tokenManager: ITokenManager,
        authAPI: IAuthAPI,
        userAPI: IUserAPI,
    ): IAccountRepository = AccountRepository(
        accountDao, userDataStore, tokenManager, authAPI, userAPI
    )
}
```

Rules:
- All repositories and services are `@Singleton`
- Always bind to the `I`-prefixed interface type
- Install in `SingletonComponent::class`
- Use `@Provides` with explicit return type (the interface)

## ViewModel Injection

### Standard ViewModel

```kotlin
@HiltViewModel
class FooViewModel @Inject constructor(
    private val fooService: IFooService,
    private val accountService: IAccountService,
) : BaseIntentViewModel<FooState, FooIntent>(FooReducer()) {
    // ...
}
```

- `@HiltViewModel` + `@Inject constructor`
- Inject **interface types** only (e.g., `IFooService`, not `FooService`)
- No manual registration needed — Hilt discovers `@HiltViewModel` automatically

### ViewModel with Runtime Parameters (AssistedInject)

When a ViewModel needs a value from navigation (e.g., `scaleId`):

```kotlin
@HiltViewModel(assistedFactory = ScaleModeViewModel.Factory::class)
class ScaleModeViewModel @AssistedInject constructor(
    @Assisted val scaleId: String,
    private val deviceService: IDeviceService,
    private val scaleService: IScaleService,
) : BaseIntentViewModel<ScaleModeState, ScaleModeIntent>(ScaleModeReducer()) {

    @AssistedFactory
    interface Factory {
        fun create(scaleId: String): ScaleModeViewModel
    }

    override fun provideInitialState() = ScaleModeState()
}
```

Usage in Compose:
```kotlin
@Composable
fun ScaleModeScreen(scaleId: String) {
    val viewModel: ScaleModeViewModel = hiltViewModel(
        creationCallback = { factory: ScaleModeViewModel.Factory ->
            factory.create(scaleId)
        }
    )
    // ...
}
```

## Adding a New Dependency

### New Repository

1. Create interface in `domain/repository/IFooRepository.kt`
2. Create implementation in `data/repository/FooRepository.kt`
3. Add binding in `core/di/RepositoryModule.kt`:
   ```kotlin
   @Provides @Singleton
   fun provideFooRepository(api: IFooAPI, dao: FooDao): IFooRepository =
       FooRepository(api, dao)
   ```

### New Service

1. Create interface in `domain/services/IFooService.kt`
2. Create implementation in `core/service/FooService.kt` or `data/services/FooService.kt`
3. Add binding in `core/di/ServiceModule.kt`:
   ```kotlin
   @Provides @Singleton
   fun provideFooService(repo: IFooRepository): IFooService =
       FooService(repo)
   ```

### New API

1. Create interface in `data/api/IFooAPI.kt`
2. Add creation in `core/di/APIModule.kt`:
   ```kotlin
   @Provides @Singleton
   fun provideFooAPI(httpClient: HttpClient): IFooAPI =
       httpClient.createService(IFooAPI::class.java)
   ```

## Scope Rules

| Scope | When |
|-------|------|
| `@Singleton` | Repositories, services, API clients, database, DataStore |
| `@ViewModelScoped` | Rarely used — prefer constructor injection |
| No scope (unscoped) | Utility classes that are cheap to create |

## Annotation Processor

This project uses **KSP** (not KAPT) for Hilt annotation processing. Exception: `:wificonnect` module still uses KAPT.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Injecting `FooService` instead of `IFooService` | Always inject the interface |
| Missing `@Singleton` on provides | All repos/services need singleton scope |
| Forgetting to add binding for new repo/service | Build will fail with missing binding error |
| Using `@Binds` in object module | Use `@Provides` in `object` modules; `@Binds` requires `abstract class` |
| KAPT in new module | Use KSP — add `ksp(libs.hilt.compiler)` not `kapt` |
