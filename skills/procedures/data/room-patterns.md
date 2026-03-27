# Room Query Patterns

## PopulatedActiveEntry Queries

### MUST use `SELECT *` (not `SELECT e.*`)

Room's `@Transaction` + `@Relation` fetches related entities in a **second query by ID**. The main query only needs to return parent rows — Room handles JOINs automatically.

```kotlin
// CORRECT — Room fetches relations by ID
@Transaction
@Query("SELECT * FROM entry_view WHERE accountId = :accountId")
fun getEntries(accountId: String): Flow<List<PopulatedActiveEntry>>

// WRONG — Room can't resolve @Relation fields from aliased columns
@Transaction
@Query("SELECT e.* FROM entry_view e INNER JOIN bpm_entry bp ON e.id = bp.id ...")
fun getEntries(accountId: String): Flow<List<PopulatedActiveEntry>>
```

**Why**: `SELECT e.*` only returns `entry_view` columns. `@Relation` annotations on `PopulatedActiveEntry` (bpmEntry, scaleEntry, babyEntry) need to resolve by ID — Room does this automatically with `@Transaction`. INNER JOIN is redundant and breaks the relation resolution.

### Filtering by related table — use subquery, not JOIN

When you need to filter by a child table column but return `PopulatedActiveEntry`:

```kotlin
// CORRECT — subquery filters, Room fetches relations
@Transaction
@Query("""
    SELECT * FROM entry_view
    WHERE accountId = :accountId
      AND id IN (SELECT id FROM baby_entry WHERE babyProfileId = :babyProfileId)
    ORDER BY datetime(entryTimestamp) DESC
""")
fun getBabyDayDetail(accountId: String, babyProfileId: String): Flow<List<PopulatedActiveEntry>>

// WRONG — JOIN breaks @Relation resolution
@Transaction
@Query("""
    SELECT e.* FROM entry_view e
    INNER JOIN baby_entry be ON e.id = be.id
    WHERE be.babyProfileId = :babyProfileId
""")
fun getBabyDayDetail(...): Flow<List<PopulatedActiveEntry>>
```

### Aggregated queries CAN use JOIN and aliases

Queries that return **non-Relation types** (e.g., `HistoryMonth`, `BpHistoryMonth`, `BabyDailySummaryResult`) can use JOINs and aliased columns freely — Room maps by column name to data class fields.

```kotlin
// CORRECT — aggregated result, no @Relation involved
@Query("""
    SELECT AVG(bp.systolic) AS avgSystolic, ...
    FROM entry_view e
    INNER JOIN bpm_entry bp ON e.id = bp.id
    WHERE e.accountId = :accountId
    GROUP BY ...
""")
fun getBpmMonthlyHistory(accountId: String): Flow<List<BpHistoryMonth>>
```

## Format Consistency

Ensure the format output by grouped/monthly queries matches the format expected by detail queries.

Example: If monthly history outputs `"Mar 2025"` (via CASE statement), the detail query must accept `"Mar 2025"` as the `:month` parameter — not `"2025-03"`.

## Separate DAO for Reads

When the existing DAO is large (>500 lines), create a separate read-only DAO:
- `EntryDao` — CRUD + sync operations (existing, don't touch)
- `HistoryDao` — history read queries (new, clean)

Register the new DAO in `AppDatabase` and `DatabaseModule`.
