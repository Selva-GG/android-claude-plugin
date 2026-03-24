---
name: refresh-cache
description: Scan the MeApp Android codebase and generate Layer 2 cache files at .claude/cache/ — maps components, services, repositories, routes, DI, models, and features
user-invocable: false
---

# Refresh Cache

Scan the MeApp Android codebase and generate 7 structured Markdown files that map the entire project. These cache files are used by the analyse-plan skill to understand what already exists before making implementation decisions.

**Output location:** `<project-root>/.claude/cache/` (gitignored, per-project)

**When to run:**
- First time setup (no cache exists)
- On-demand via `/android:refresh-cache`
- When analyse-plan detects cache is >7 days old

## Step 0: Detect Project Root

Find the Android project root by looking for `app/src/main/java/com/dmdbrands/gurus/weight/`:

```bash
# Walk up from CWD to find the project root
dir="$PWD"
while [ "$dir" != "/" ]; do
  if [ -d "$dir/app/src/main/java/com/dmdbrands/gurus/weight" ]; then
    echo "$dir"
    break
  fi
  dir=$(dirname "$dir")
done
```

Set `SRC_ROOT` to `<project-root>/app/src/main/java/com/dmdbrands/gurus/weight`

Create cache directory:
```bash
mkdir -p <project-root>/.claude/cache
```

## IMPORTANT: File Writing

Subagents may not have Write tool permissions. **Always use Bash with heredoc** to write cache files:
```bash
cat > "<project-root>/.claude/cache/filename.md" << 'ENDOFFILE'
content here
ENDOFFILE
```
Do NOT use the Write tool for cache files — it may be denied in subagent contexts.

## Step 1: Generate component-catalog.md

Scan `$SRC_ROOT/features/common/` for all Composable functions.

For each `.kt` file in `features/common/components/`, `features/common/service/`, `features/common/model/`, `features/common/enum/`, `features/common/strings/`, `features/common/helper/`:

1. Read the file
2. Extract: class/function name, parameters (props), file path
3. Categorize by type: Button, Input, Dialog, List, Card, Chart, Loader, Layout, Image, Other

Output format:
```markdown
# Component Catalog
Generated: YYYY-MM-DDTHH:MM:SS

## Buttons
| Component | File | Key Props | Description |
|-----------|------|-----------|-------------|
| AppButton | features/common/components/AppButton.kt | text, onClick, enabled, style | Primary action button |

## Inputs
| Component | File | Key Props | Description |
|-----------|------|-----------|-------------|
| AppInput | features/common/components/AppInput.kt | value, onValueChange, label, imeAction | Standard text input |

## Dialogs & Modals
...

## Lists & Reordering
...

## Cards & Containers
...

## Charts & Graphs
...

## Loaders & Progress
...

## Layout & Navigation
...

## Images & Media
...

## Helpers & Utilities
| Helper | File | Purpose |
|--------|------|---------|
| CommonHelper | features/common/helper/CommonHelper.kt | Shared utility functions |

## Models
| Model | File | Fields |
|-------|------|--------|
| DialogModel | features/common/model/DialogModel.kt | Confirm, Info, Custom variants |

## Enums
| Enum | File | Values |
|------|------|--------|
| AppSpacing | features/common/enum/AppSpacing.kt | ... |
```

## Step 2: Generate service-map.md

Scan `$SRC_ROOT/domain/services/`, `$SRC_ROOT/data/services/`, `$SRC_ROOT/core/service/` for all service interfaces and implementations.

For each service:
1. Find the interface (in `domain/services/` — starts with `I`)
2. Find the implementation (in `data/services/` or `core/service/`)
3. Find the DI binding (grep in `core/di/ServiceModule.kt`)
4. Note the layer (core vs data)

Output format:
```markdown
# Service Map
Generated: YYYY-MM-DDTHH:MM:SS

| Service | Interface | Implementation | DI Module | Layer | Size |
|---------|-----------|---------------|-----------|-------|------|
| Account | IAccountService | AccountService | ServiceModule | core/service/ | 36KB |
| Entry | IEntryService | EntryService | ServiceModule | data/services/ | 15KB |
| Dashboard | IDashboardService | DashboardService | ServiceModule | core/service/ | 13KB |
| Device | IDeviceService | DeviceService | ServiceModule | core/service/ | 23KB |
...

## Service Dependencies
| Service | Depends On |
|---------|-----------|
| AccountService | IAccountRepository, IAccountFlagRepository, ... |
| EntryService | IEntryRepository, IAccountService, ... |
...
```

## Step 3: Generate repository-map.md

Scan `$SRC_ROOT/domain/repository/`, `$SRC_ROOT/data/repository/`, `$SRC_ROOT/data/api/`, `$SRC_ROOT/data/storage/db/dao/`, `$SRC_ROOT/data/storage/db/entity/` for the full repository chain.

For each repository:
1. Find the interface (in `domain/repository/`)
2. Find the implementation (in `data/repository/`)
3. Find the API interface it calls (in `data/api/`)
4. Find the DAO it uses (in `data/storage/db/dao/`)
5. Find the Entity it persists (in `data/storage/db/entity/`)
6. Find the DI binding (grep in `core/di/RepositoryModule.kt`)

Output format:
```markdown
# Repository Map
Generated: YYYY-MM-DDTHH:MM:SS

| Repository | Interface | Implementation | API | DAO | Entity | DI Module |
|-----------|-----------|---------------|-----|-----|--------|-----------|
| Account | IAccountRepository | AccountRepository | IAuthAPI | AccountDao | AccountEntity | RepositoryModule |
| Entry | IEntryRepository | EntryRepository | IEntryAPI | EntryDao | EntryEntity | RepositoryModule |
...

## DataStore Repositories
| Repository | Interface | Implementation | DataStore | DI Module |
|-----------|-----------|---------------|-----------|-----------|
| UserSettings | IUserSettingsRepository | UserSettingsRepository | UserDataStore | RepositoryModule |
...
```

## Step 4: Generate route-map.md

Read `$SRC_ROOT/core/navigation/AppRoute.kt` and extract all sealed class routes.

For each route:
1. Name and nesting level (e.g., `AppRoute.ScaleDetails.ScaleMode`)
2. Parameters (e.g., `scaleId: String`)
3. Parent group

Output format:
```markdown
# Route Map
Generated: YYYY-MM-DDTHH:MM:SS

| Route | Full Path | Parameters | Group |
|-------|-----------|-----------|-------|
| Loading | AppRoute.Init.Loading | — | Init |
| Dashboard | AppRoute.Main.Dashboard | — | Main |
| Login | AppRoute.Auth.Login | — | Auth |
| ScaleMode | AppRoute.ScaleDetails.ScaleMode | scaleId: String | ScaleDetails |
...

## Route Groups
| Group | Routes | Description |
|-------|--------|-------------|
| Init | Loading | App initialization |
| Main | Dashboard, Entry, History, Settings, AppSync | Main tab navigation |
| Auth | Landing, Login, Signup, MultiAccountLanding | Authentication flow |
| ScaleDetails | ScaleMode, ScaleDisplayMetrics, ScaleUsers | Scale settings |
...
```

## Step 5: Generate di-graph.md

Scan `$SRC_ROOT/core/di/` for all Hilt modules.

For each module:
1. Module name and scope
2. What it `@Provides` or `@Binds`
3. Interface → Implementation mapping

Output format:
```markdown
# DI Graph
Generated: YYYY-MM-DDTHH:MM:SS

## RepositoryModule
Scope: SingletonComponent

| Binding | Interface | Implementation |
|---------|-----------|---------------|
| @Binds | IAccountRepository | AccountRepository |
| @Binds | IEntryRepository | EntryRepository |
...

## ServiceModule
Scope: SingletonComponent

| Binding | Interface | Implementation |
|---------|-----------|---------------|
| @Binds | IAccountService | AccountService |
...

## DataStoreModule
Scope: SingletonComponent

| Provides | Type | Notes |
|----------|------|-------|
| UserDataStore | DataStore | Proto-backed |
| BluetoothPreferencesDataStore | DataStore | Proto-backed |
...

## NetworkModule
...

## DatabaseModule
...
```

## Step 6: Generate model-map.md

Scan `$SRC_ROOT/domain/model/` and `$SRC_ROOT/domain/enums/` for all domain models and enums.

Categorize by sub-directory:
- `model/api/` — API request/response models (grouped by feature: auth, entry, device, etc.)
- `model/storage/` — Room-persisted models
- `model/common/` — Shared domain models
- `model/feature/` — Feature-specific models
- `model/feed/`, `model/goal/`, `model/integrations/`, `model/permission/` — Domain-specific
- `enums/` — Domain enums

Output format:
```markdown
# Model Map
Generated: YYYY-MM-DDTHH:MM:SS

## API Models
### auth/
| Model | File | Fields |
|-------|------|--------|
| LoginRequest | domain/model/api/auth/LoginRequest.kt | email, password |
...

### entry/
| Model | File | Fields |
|-------|------|--------|
| EntryResponse | domain/model/api/entry/EntryResponse.kt | id, weight, date, ... |
...

## Storage Models
| Model | File | Fields |
|-------|------|--------|
| AccountStorageModel | domain/model/storage/Account/... | id, email, ... |
...

## Common Models
| Model | File | Fields |
|-------|------|--------|
...

## Enums
| Enum | File | Values |
|------|------|--------|
| ProductType | domain/enums/ProductType.kt | MY_WEIGHT, BLOOD_PRESSURE, BABY |
| DashboardType | domain/enums/DashboardType.kt | MY_WEIGHT, BODY_SCALE, BLOOD_PRESSURE, BABY_SCALE |
| GoalType | domain/enums/GoalType.kt | ... |
...
```

## Step 7: Generate feature-index.md

Scan `$SRC_ROOT/features/` for all feature directories (excluding `common/` which is in component-catalog).

For each feature directory:
1. List all files and sub-directories
2. Identify: Screen, ViewModel, Reducer, components, strings, models, enums
3. Note which AppRoutes point to this feature

Output format:
```markdown
# Feature Index
Generated: YYYY-MM-DDTHH:MM:SS

| Feature | Directory | Screen | ViewModel | Reducer | Components | Strings | Routes |
|---------|-----------|--------|-----------|---------|------------|---------|--------|
| Dashboard | features/dashboard/ | DashboardScreen | DashboardViewModel | DashboardReducer | 5 | DashboardStrings | AppRoute.Main.Dashboard |
| Login | features/login/ | LoginScreen | LoginViewModel | LoginReducer | 2 | LoginStrings | AppRoute.Auth.Login |
...

## Feature Details

### dashboard
```
features/dashboard/
├── DashboardScreen.kt
├── viewmodel/
│   ├── DashboardViewModel.kt
│   └── DashboardReducer.kt
├── components/
│   ├── DashboardCard.kt
│   └── ...
├── strings/
│   └── DashboardStrings.kt
└── enum/
    └── DashboardTab.kt
```

### login
...
```

## Step 8: Write Timestamp

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ" > <project-root>/.claude/cache/last-scan-timestamp
```

## Step 9: Report Summary

Display:
```
Cache refreshed at <project-root>/.claude/cache/

| File | Entries |
|------|--------|
| component-catalog.md | N components |
| service-map.md | N services |
| repository-map.md | N repositories |
| route-map.md | N routes |
| di-graph.md | N modules |
| model-map.md | N models |
| feature-index.md | N features |

Cache valid for 7 days. Run /android:refresh-cache to regenerate.
```
