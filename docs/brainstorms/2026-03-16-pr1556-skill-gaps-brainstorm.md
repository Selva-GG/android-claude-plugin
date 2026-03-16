# Brainstorm: Gaps Found from PR #1556 Unit Test Skills

**Date:** 2026-03-16
**Source:** https://github.com/dmdbrands/meApp/pull/1556

## What We're Building

Add 14 missing test patterns and reference material to our `write-tests.md` skill, sourced from PR #1556's `unit-tests/` skill directory. All additions must be generic (no MeApp-specific references).

## Items to Add

### High Impact (7)

1. **Dialog callback testing** — `slot<DialogModel>()` + `capture()` + invoke `onConfirm` + `advanceUntilIdle()`
2. **Reducer-ViewModel test separation** — "Never re-test reducer logic in ViewModel tests"
3. **SUT re-creation pattern** — `createViewModel()` / `createService()` factory when flow stubs change
4. **State preservation verification** — assert unrelated fields NOT modified by intent
5. **Network guard clause patterns** — hard throw vs soft check testing strategies
6. **`.value` vs Turbine guidance** — `.value` for final-state, Turbine only for intermediate emissions
7. **Expanded troubleshooting** — add 20+ error-to-fix entries (currently 8)

### Medium Impact (7)

8. **Shared TestHelpers pattern** — centralized `httpException()`, network stubs
9. **Insert-or-update branching** — test both INSERT and UPDATE paths
10. **Request object verification** — `slot<>()` + `capture()` for argument inspection
11. **Catch-and-swallow vs catch-and-rethrow** — distinct test patterns
12. **`turbineScope` + `testIn(this)`** — multiple simultaneous flow testing
13. **JaCoCo 0.8.14 Kotlin false positives** — table of known false positives
14. **Method-level JaCoCo analysis** — per-method coverage, not just class-level

## Key Decision

All patterns must be written generically. Extract the concept, not the MeApp-specific implementation.

## Where to Add

| Items | Target Location in write-tests.md |
|-------|----------------------------------|
| 1, 5, 11 | Step 4.4 Type-Specific Patterns → new subsections |
| 2 | Step 4.4 ViewModel Tests → new rule |
| 3 | Step 4.4 ViewModel/Service Tests → new pattern |
| 4 | Step 4.3 → new category H |
| 6 | Step 4.4 ViewModel Tests → new guidance |
| 7 | Step 6.3 error table → expand |
| 8 | Step 0.3b → detect shared helpers |
| 9, 10 | Step 4.4 Repository Tests → expand |
| 12 | Step 4.3 E Flow Tests → expand |
| 13, 14 | Step 7.2 Smart Filter → expand |
