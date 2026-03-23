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
| **Testing** | JUnit 6 (detect from build.gradle) · MockK · Turbine |

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

| Layer | Sub-Skill | Trigger — Read when task touches… |
|-------|-----------|----------------------------------|
| **UI** | [mvi-pattern](ui/mvi-pattern.md) | `features/*/` — any ViewModel, Reducer, State, or Intent file; adding/modifying `handleIntent()` |
| **UI** | [compose-ui](ui/compose-ui.md) | `features/*/` — any `*Screen.kt`, `components/*.kt`; theming, previews, `@PreviewTheme`, `MeAppTheme.*` |
| **UI** | [navigation-dialogs](ui/navigation-dialogs.md) | `core/navigation/AppRoute.kt`; `navigationService.navigateTo()`; `dialogQueueService.*`; adding new routes |
| **Core** | [dependency-injection](core/dependency-injection.md) | `core/di/*.kt`; adding `@Provides`/`@Binds`; new `@HiltViewModel`; `@AssistedInject` |
| **Data** | [networking-api](data/networking-api.md) | `data/api/*.kt`; `core/network/`; interceptors; OkHttp/Retrofit; `SecureTokenStore`; new API endpoints |
| **Data** | [storage](data/storage.md) | `data/storage/db/` (entities, DAOs); `data/storage/datastore/`; `*.proto` files; Room migrations |
| **Data** | [services-repositories](data/services-repositories.md) | `data/repository/*.kt`; `core/service/*.kt`; `domain/repository/`; `domain/services/`; new service or repo |
| **Shared** | [testing](shared/testing.md) | `src/test/` — writing or modifying any unit test; new `*Test.kt` file. Sub-files: [test-patterns](shared/test-patterns.md), [test-troubleshooting](shared/test-troubleshooting.md), [test-coverage](shared/test-coverage.md) |
| **Shared** | [review-checklist](shared/review-checklist.md) | **ALWAYS** — read before finishing any task (pre-PR checklist) |

**If your task only touches `core/network/` (e.g., certificate pinning, interceptor changes), you need `networking-api` but NOT `mvi-pattern` or `compose-ui`.** Match sub-skills to the files you'll actually change.

## Git Rules (Strictly Enforced)

- **NEVER push directly to `dev` or `main`** — all changes must go through a PR. If a `git push origin dev` or `git push origin main` is about to happen (outside of an explicit revert approved by the user), STOP and warn:
  > ⛔ Direct push to `[dev/main]` is not allowed. Create a PR from your feature branch instead.
- **NEVER commit without explicit user instruction.**
- **NEVER push without explicit user instruction.**
- **Always prefix commit messages with the Jira ticket ID** — e.g., `MA-3359: Add URL validation`.
- **Feature branches only** — create a branch per ticket; never work directly on `dev` or `main`.

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
