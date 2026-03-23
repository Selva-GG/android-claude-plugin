# Storage

## Overview

| Storage | Use For | Location |
|---------|---------|----------|
| Room DB | Structured relational data (accounts, entries, devices) | `data/storage/db/` |
| Protobuf DataStore | User preferences, tokens, small config | `data/storage/datastore/` |
| Proto definitions | DataStore schema | `app/src/main/proto/` |

## Room Database

### AppDatabase

```kotlin
@Database(
    entities = [
        AccountEntity::class, DeviceEntity::class, BodyScaleEntity::class,
        EntryEntity::class, BodyScaleEntryEntity::class, ...
    ],
    views = [ActiveEntryEntity::class],
    version = 1,
    exportSchema = false,
)
@TypeConverters(DateConverter::class, JsonConverter::class, WeightUnitConverter::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun accountDao(): AccountDao
    abstract fun deviceDao(): DeviceDao
    abstract fun entryDao(): EntryDao
    abstract fun logDao(): LogDao
}
```

### Entity Pattern

Entities in `data/storage/db/entity/`:

```kotlin
@Entity(tableName = "accounts")
data class AccountEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "email") val email: String,
    @ColumnInfo(name = "display_name") val displayName: String?,
    @ColumnInfo(name = "weight_unit") val weightUnit: WeightUnit,
    @ColumnInfo(name = "created_at") val createdAt: Date?,
)
```

Rules:
- `data class` with `@Entity` annotation
- Explicit `tableName`
- `@PrimaryKey` on ID field
- `@ColumnInfo(name = ...)` for column mapping
- Use `@ForeignKey` for relationships
- Prefer `val` (immutable) fields
- Meaningful names, no abbreviations
- KDoc documentation required

### DAO Pattern

DAOs in `data/storage/db/dao/`:

```kotlin
@Dao
interface AccountDao {

    @Query("SELECT * FROM accounts WHERE id = :id")
    suspend fun getById(id: String): AccountEntity?

    @Query("SELECT * FROM accounts")
    fun getAll(): Flow<List<AccountEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(account: AccountEntity)

    @Update
    suspend fun update(account: AccountEntity)

    @Delete
    suspend fun delete(account: AccountEntity)

    @Query("DELETE FROM accounts")
    suspend fun deleteAll()
}
```

Rules:
- `@Dao` interface
- Use `suspend` for one-shot operations (insert, update, delete, single query)
- Use `Flow<T>` for observable queries (lists that update reactively)
- Never access database on main thread
- KDoc on each method

### Type Converters

```kotlin
class DateConverter {
    @TypeConverter fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }
    @TypeConverter fun dateToTimestamp(date: Date?): Long? = date?.time
}
```

Register in `@TypeConverters` on `AppDatabase`.

### Adding a New Entity

1. Create `data/storage/db/entity/FooEntity.kt`
2. Create `data/storage/db/dao/FooDao.kt`
3. Add entity to `@Database(entities = [...])` in `AppDatabase`
4. Add `abstract fun fooDao(): FooDao` to `AppDatabase`
5. Provide DAO in `core/di/DatabaseModule.kt`:
   ```kotlin
   @Provides fun provideFooDao(db: AppDatabase): FooDao = db.fooDao()
   ```

## Protobuf DataStore

### Base Class

```kotlin
abstract class BaseProtoDataStore<T : MessageLite>(
    protected val dataStore: DataStore<T>
) {
    val dataFlow: Flow<T> get() = dataStore.data
    suspend fun getData(): T = dataStore.data.first()
    suspend fun updateData(transform: suspend (T) -> T): T = dataStore.updateData(transform)
    protected abstract fun getDefaultInstance(): T
    open suspend fun clearData() { ... }
}
```

### Existing DataStores

| DataStore | Proto File | Stores |
|-----------|-----------|--------|
| `UserDataStore` | `user_profile.proto` | Theme, accounts, tokens |
| `FcmDataStore` | `fcm_token.proto` | Firebase messaging tokens |
| `HealthConnectDataStore` | `health_connect.proto` | Integration data, sync status |
| `GoalAlertDataStore` | `goal_alert.proto` | Per-account goal alert state |
| `BluetoothPreferencesDataStore` | `bluetooth_preferences.proto` | BLE device MAC filtering |

### Proto Definition

Proto files in `app/src/main/proto/`:

```protobuf
syntax = "proto3";

option java_package = "com.dmdbrands.gurus.weight.data.storage.datastore";
option java_multiple_files = true;

message FooPreferences {
    string selected_id = 1;
    bool is_enabled = 2;
    int64 last_sync_timestamp = 3;
}
```

After adding/modifying a `.proto` file, rebuild: `./gradlew assembleDebug`.

### Reading & Writing

```kotlin
// Reading (reactive)
userDataStore.dataFlow.collect { prefs ->
    val theme = prefs.themeMode
}

// Reading (one-shot)
val prefs = userDataStore.getData()

// Writing (atomic)
userDataStore.updateData { current ->
    current.toBuilder()
        .setThemeMode(ThemeMode.DARK)
        .build()
}
```

Rules:
- Always use `updateData { }` for writes (atomic)
- Expose `Flow` for reactive reads
- Use `suspend fun getData()` for one-shot reads
- Never modify proto objects directly — use `.toBuilder().set*().build()`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Database operation on main thread | Use `suspend` or `Flow` |
| Missing entity in `@Database` | Add to entities array in AppDatabase |
| Missing DAO provider | Add to DatabaseModule |
| Direct proto mutation | Use `.toBuilder().set*().build()` |
| Non-atomic DataStore write | Always use `updateData { }` transform |
