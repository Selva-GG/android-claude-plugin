---
name: data-layer-design
description: Guide architecture decisions for DAO, Repository, Service, and sealed type patterns when adding new data features
user-invocable: true
---

# Data Layer Architecture Design

Guide architecture decisions when adding new data features (new product types, new query patterns, new service APIs). Prevents the back-and-forth of "1 repo or 3? sealed or typed? where does conversion live?"

## When to Use

- Adding a new product type (BP, baby, glucose, etc.) to an existing feature
- Creating a new DAO/Repository/Service chain
- Deciding between sealed types vs typed functions
- Choosing where data transformations live

## Step 1: Understand the Data Shape

Before any architecture decisions, answer:

1. **Does the new data share the `entry` table?**
   - Check if the entity has a FK to `EntryEntity` (like `bpm_entry.id → entry.id`)
   - If yes → queries go in `HistoryDao` / existing entry pipeline
   - If no → separate DAO needed

2. **What return types are needed?**
   - Aggregated summaries (e.g., monthly averages) → custom Room result type (e.g., `BpHistoryMonth`)
   - Individual entries → `PopulatedActiveEntry` (Room handles `@Relation`)
   - Grouped data (e.g., weekly) → flat Room result + repo-level grouping

3. **Does the UI need product-switching?**
   - If yes → sealed types at service boundary
   - If no → typed functions are fine

## Step 2: DAO Decision

Use AskUserQuestion:
> "New queries needed. Add to existing DAO or create separate?"
> Options: "Add to existing [DaoName]" / "Create new [Feature]Dao"

**Guideline**: If existing DAO is >500 lines, create a new one. If <500 lines and queries are related, add to existing.

**Rules** (from `procedures/data/room-patterns.md`):
- `PopulatedActiveEntry` queries: `SELECT *`, no alias, no JOIN
- Aggregated queries: JOINs and aliases are fine
- Filter by child table: use subquery `id IN (SELECT ...)`

## Step 3: Repository Decision

Use AskUserQuestion:
> "Repository structure for [feature]?"
> Options: "Single repo with all functions" / "Separate repos per product type" / "Add to existing repo"

**Decision tree:**
- ≤ 6 functions → single repo
- 7-12 functions → consider splitting by concern (not by product)
- All functions serve one feature (e.g., history) → single repo
- Functions serve unrelated features → separate repos

**Rules:**
- Repo returns **typed** domain models (e.g., `List<BpmEntry>`, `List<HistoryMonth>`)
- Data transformations (unit conversion, grouping) happen in repo
- Use `ConversionTools` for unit math — don't inline calculations
- Sample data: `USE_SAMPLE_DATA` companion flag in repo, matching DAO return types

## Step 4: Service Decision

Use AskUserQuestion:
> "Service API style for [feature]?"
> Options: "Sealed types (2 functions via ProductSelection)" / "Typed functions per product" / "Add to existing service"

**Decision tree:**
- UI switches content by product type → sealed types + `ProductSelection` param
- UI shows one product at a time (no switching) → typed functions
- Feature is product-agnostic → no product param needed

**Sealed type pattern:**
```
interface IHistoryService {
    fun getGroupedHistory(product: ProductSelection): Flow<GroupedHistory>
    fun getDetail(product: ProductSelection, key: String): Flow<HistoryDetail>
}
```

**Rules:**
- Sealed types are a **service concern** — don't put them in repo
- Service uses internal `accountId` (set via `setAccountId()` from LoadingScreenViewModel)
- Prefer `ProductSelection` over `ProductType + babyProfileId` — sealed class carries all context

## Step 5: ViewModel Wiring

**Rules** (from `procedures/shared/hilt-patterns.md`):
- Access `BaseViewModel` services via `onDependenciesReady()`, not `init`
- Load all product data in parallel (one coroutine per product)
- Cancel previous jobs when product list changes (`collectLatest` + `Job` list)
- Don't duplicate-inject services from `BaseViewModel`

## Output

After decisions are made, produce a summary:

```
Data Layer Design: [Feature]

DAO: [existing/new] — [N queries]
Repository: [single/split] — [N functions]
Service: [sealed/typed] — [N functions]
Sealed types: [yes/no] — [list variants]
Sample data: [yes/no]

Flow: DAO → Repo (typed) → Service (sealed) → ViewModel (unwrap)
```
