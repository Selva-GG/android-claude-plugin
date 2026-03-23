---
name: refresh-cache
description: Scan the MeApp Android codebase and regenerate Layer 2 cache files (.claude/cache/)
---

# Refresh Cache

Regenerate the Layer 2 pre-analysed cache files by scanning the MeApp Android codebase.

**REQUIRED:** Use the `android:refresh-cache` skill.

This will scan the codebase and generate 7 cache files at `<project-root>/.claude/cache/`:
- `component-catalog.md` — shared composables from features/common/
- `service-map.md` — all services with interface → implementation → DI mapping
- `repository-map.md` — all repositories with full chain (interface → impl → API → DAO → entity)
- `route-map.md` — all navigation routes with parameters
- `di-graph.md` — all DI modules with bindings
- `model-map.md` — all domain models and enums
- `feature-index.md` — all feature directories with structure

After completion, display the summary with entry counts.
