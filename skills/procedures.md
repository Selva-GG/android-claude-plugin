---
name: android-procedures
description: Implementation, review, and testing procedures for the MeApp Android project — Kotlin, Jetpack Compose, Hilt, MVI architecture
user-invocable: false
---

# Android Project Procedures

## Quick Reference

| | |
|---|---|
| **Package** | `com.dmdbrands.gurus.weight` |
| **Language** | Kotlin · JVM 11 · compileSdk 36 |
| **UI** | Jetpack Compose · Material 3 |
| **DI** | Hilt (KSP) |
| **Architecture** | MVI + Clean Architecture |
| **Navigation** | Navigation3 · sealed class routes |
| **Storage** | Room DB · Protobuf DataStore |
| **Network** | Retrofit 3 · OkHttp · Gson |
| **Testing** | JUnit 4 · MockK · Turbine |

## Build Commands

```bash
./gradlew assembleDebug        # Build
./gradlew test                 # Unit tests
./gradlew detekt               # Static analysis (!! banned)
```

## Architecture Layers

```
domain/          ← Pure Kotlin: interfaces, models, enums (no Android deps)
  ├── interfaces/    IReducer, IDialogQueueService
  ├── repository/    IAccountRepository, IEntryRepository, ...
  ├── services/      IAccountService, IGoalService, ...
  ├── model/         API models, storage models, feature models
  └── enums/         GoalType, MetricKey, ...

data/            ← Implements domain: APIs, repositories, storage
  ├── api/           Retrofit interfaces (IGoalAPI, IEntryAPI, ...)
  ├── repository/    Repository implementations
  ├── services/      Service implementations
  └── storage/
      ├── db/        Room DB: entities, DAOs, converters
      └── datastore/ Protobuf DataStore classes

core/            ← Infrastructure: DI, network, config
  ├── di/            15 Hilt modules (RepositoryModule, ServiceModule, ...)
  ├── navigation/    AppRoute sealed classes
  ├── network/       HttpClient, TokenManager, interceptors
  └── service/       Core service implementations

features/        ← UI: one directory per feature
  ├── common/        76+ shared composables, BaseViewModel, DialogQueueService
  └── <feature>/     Screen, ViewModel, Reducer, State, Intent, components/, strings/
```

## Sub-Skills (Detailed Guides)

Read a sub-skill when the task touches its **trigger files/packages**. Skip it if no trigger matches.

| Sub-Skill | Trigger — Read when task touches… |
|-----------|----------------------------------|
| [mvi-pattern](procedures/mvi-pattern.md) | `features/*/` — any ViewModel, Reducer, State, or Intent file; adding/modifying `handleIntent()` |
| [compose-ui](procedures/compose-ui.md) | `features/*/` — any `*Screen.kt`, `components/*.kt`; theming, previews, `@PreviewTheme`, `MeAppTheme.*` |
| [navigation-dialogs](procedures/navigation-dialogs.md) | `core/navigation/AppRoute.kt`; `navigationService.navigateTo()`; `dialogQueueService.*`; adding new routes |
| [dependency-injection](procedures/dependency-injection.md) | `core/di/*.kt`; adding `@Provides`/`@Binds`; new `@HiltViewModel`; `@AssistedInject` |
| [networking-api](procedures/networking-api.md) | `data/api/*.kt`; `core/network/`; interceptors; OkHttp/Retrofit; `SecureTokenStore`; new API endpoints |
| [storage](procedures/storage.md) | `data/storage/db/` (entities, DAOs); `data/storage/datastore/`; `*.proto` files; Room migrations |
| [services-repositories](procedures/services-repositories.md) | `data/repository/*.kt`; `core/service/*.kt`; `domain/repository/`; `domain/services/`; new service or repo |
| [testing](procedures/testing.md) | `src/test/` — writing or modifying any unit test; new `*Test.kt` file |
| [review-checklist](procedures/review-checklist.md) | **ALWAYS** — read before finishing any task (pre-PR checklist) |

**If your task only touches `core/network/` (e.g., certificate pinning, interceptor changes), you need `networking-api` but NOT `mvi-pattern` or `compose-ui`.** Match sub-skills to the files you'll actually change.

## Hard Rules (Enforced)

1. **`!!` is BANNED** — detekt enforces `UnsafeCallOnNullableType`. Use `?: return`, `?.let`, `requireNotNull()`.
2. **No hardcoded colors/typography/spacing** — always `MeTheme.colorScheme.*`, `.typography.*`, `.spacing.*`. Text colors: `textHeading`, `textBody`, `textSubheading`, `textError`. Backgrounds: `primaryBackground`, `secondaryBackground`. Actions: `primaryAction`, `inverseAction`, `errorAction`.
3. **All interfaces use `I` prefix** — `IAccountRepository`, `IEntryService`, `IDeviceService`.
4. **All API methods are `suspend`** — no blocking calls.
5. **Logging uses `AppLog` only** — never `Log` or `Timber` directly.
6. **Previews use `@PreviewTheme`** + wrap in `MeAppTheme { }`.
7. **One composable per file** — only `@Preview` functions share a file.
8. **Static text in `strings/` objects** — PascalCase `const val`, never hardcoded in composables.
9. **`super.handleIntent(intent)` always called first** in ViewModel's `handleIntent()`.
10. **Never inject NavigationService or DialogQueueService directly** — available via `BaseViewModel`.
11. **State classes use `@Stable` + `ImmutableList`** — `persistentListOf()` defaults, `toPersistentList()` in reducers.
12. **`collectAsStateWithLifecycle()`** — never plain `collectAsState()`.
13. **`LazyColumn`/`LazyRow` must have `key`** — stable unique key per item.
14. **No manual `CoroutineScope()`** — inject `@ApplicationScope` in services/repos.
15. **No `SimpleDateFormat`** — use `DateTimeFormatter` (thread-safe).
16. **No `runBlocking`, no `GlobalScope`** — detekt enforces.
17. **Methods ≤ 60 lines, classes ≤ 600 lines** — detekt enforces.

## New Feature File Structure

```
features/<feature-name>/
├── <Feature>Screen.kt          # Main composable
├── <Feature>ViewModel.kt       # ViewModel (side effects)
├── <Feature>Reducer.kt         # Pure reducer + State + Intent
├── components/                  # Sub-composables
│   └── <Component>.kt
└── strings/
    └── <Feature>Strings.kt     # Static text constants
```

## Modules

| Module | Purpose |
|--------|---------|
| `:app` | Main application — features, domain, data, core |
| `:notification` | Firebase push notifications |
| `:app:healthconnect` | Health Connect integration |
| `:app:wificonnect` | WiFi scale connectivity |
| `:app:appsync` | QR/barcode scan sync |
| `:bleWrapper` | Bluetooth Low Energy abstraction |
| `:iam` | In-App Messaging |
