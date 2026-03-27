---
name: ticket-splitting
description: Split large features into ordered tickets with dependency graph, branch chain, and phase planning
user-invocable: true
---

# Ticket Splitting & Dependency Planning

Split a large feature into manageable tickets with clear dependencies, branch ordering, and phase planning. Prevents the "we mixed 3 tasks into 1 PR" problem.

## When to Use

- A single ticket covers multiple layers (data + UI + service)
- Figma designs show multiple screens for one feature
- Implementation would touch >10 files
- Multiple product types need the same feature (weight + BP + baby)

## Step 1: Identify Atomic Units

Break the feature into atomic units by asking:

1. **What are the distinct screens/views?** (Each = potential UI ticket)
2. **What shared components are needed?** (Each = potential component ticket)
3. **What data layer changes are needed?** (Entities, DAOs, repos, services)
4. **What infrastructure changes are needed?** (Navigation, DI, models)

Use AskUserQuestion:
> "I've identified [N] atomic units for this feature. Review and adjust?"

## Step 2: Classify by Layer

Group atomic units into layers:

| Layer | Examples |
|-------|---------|
| **Data** | Room entities, DAOs, repositories, migrations |
| **Infrastructure** | Navigation routes, DI modules, shared models |
| **Components** | Reusable UI components (input fields, list items) |
| **Screens** | Feature-specific screens (totals view, detail view) |
| **Integration** | Service layer, ViewModel wiring, product switching |

## Step 3: Build Dependency Graph

For each unit, identify what it depends on:

```
Data layer tickets (no UI dependencies)
  └→ Infrastructure tickets (need data models)
       └→ Component tickets (need infra)
            └→ Screen tickets (need components)
                 └→ Integration tickets (need all of the above)
```

**Rules:**
- Data tickets NEVER depend on UI tickets
- Component tickets before screen tickets
- Shared components before product-specific screens
- Data layer for all products before any product-specific UI

## Step 4: Phase Assignment

Group into phases (tickets within a phase can run in parallel):

```
Phase 1: [tickets with no dependencies — can start immediately]
Phase 2: [tickets that depend on Phase 1]
Phase 3: [tickets that depend on Phase 2]
...
```

Use AskUserQuestion:
> "Proposed phases:
> Phase 1: [list]
> Phase 2: [list]
> Phase 3: [list]
> Proceed with this ordering?"

## Step 5: Branch Chain

Determine the branch chain based on dependencies:

```
base-branch
  ├→ ticket-A (Phase 1, independent)
  ├→ ticket-B (Phase 1, independent)
  └→ ticket-C (Phase 1, independent)
       └→ ticket-D (Phase 2, depends on C)
            ├→ ticket-E (Phase 3, depends on D)
            └→ ticket-F (Phase 3, depends on D)
```

**Rules:**
- Each ticket gets its own branch
- Branch from the ticket it depends on (not always from main)
- If a ticket depends on multiple tickets, branch from the latest in the chain
- After rebasing, update PR base branches

## Step 6: Create/Update Jira Tickets

For each identified ticket:

1. **Check if a ticket already exists** — search Jira for similar titles
2. **If exists**: update title/description to match the refined scope
3. **If not**: create new ticket with clear AC

Use AskUserQuestion:
> "Found existing ticket [KEY]: [title]. Reuse and update, or create new?"

**Ticket description template:**
```
[What this ticket does — 1 sentence]

Depends on: [ticket keys]
Base branch: [branch name]

Acceptance Criteria:
- [specific, testable criteria]
```

## Step 7: Output Summary

```
Feature Split: [Feature Name]

| Phase | Ticket | Scope | Depends On | Base Branch |
|-------|--------|-------|------------|-------------|
| 1 | MA-XXXX | ... | None | main |
| 1 | MA-YYYY | ... | None | main |
| 2 | MA-ZZZZ | ... | MA-XXXX | MA-XXXX-branch |
| 3 | MA-WWWW | ... | MA-ZZZZ | MA-ZZZZ-branch |

Branch chain:
  main → MA-XXXX → MA-ZZZZ → MA-WWWW
  main → MA-YYYY (parallel)
```

## Common Patterns

### Multi-Product History (Weight + BP + Baby)
```
Phase 1: Data entities (per product, parallel)
Phase 1: Shared components (HistoryRowLayout)
Phase 2: Totals views (per product, parallel)
Phase 3: Detail views (per product, parallel)
Phase 4: Integration (switcher, service layer)
```

### Multi-Product Entry (Weight + BP + Baby)
```
Phase 1: Input components (AppTextArea, short input)
Phase 2: Entry type switching (ActiveEntryForm sealed class)
Phase 3: Product-specific screens (per product, parallel)
```

### New Feature End-to-End
```
Phase 1: Domain models + enums
Phase 2: Data layer (entities, DAO, repo)
Phase 3: Service + DI
Phase 4: ViewModel + Reducer
Phase 5: UI screens
Phase 6: Integration tests
```
