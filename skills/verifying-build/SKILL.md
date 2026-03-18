---
name: verifying-build
description: Use when verifying that code changes compile, tests pass, and lint checks succeed — before creating a PR, after fixing a bug, or at any point to validate code health. Works with any project type by reading build commands from CLAUDE.md
user-invocable: false
---

# Verifying Build, Tests, and Lint

## Overview

Run the full verification suite (build + unit tests + static analysis) for the current project. All configured checks must pass before proceeding. Supports any project type.

## Step 1: Discover Build Commands

**Priority order for finding commands:**

### 1. CLAUDE.md (highest priority — source of truth)

Read the project's `CLAUDE.md` file (check current directory, then parent directories). Look for build/test/lint commands in sections like:
- "Build Commands", "Build & Run Commands"
- "Testing", "Test Commands"
- "Linting", "Static Analysis"
- Code blocks with `bash` or `shell` language tags

Extract the three commands:
- **Build command** (compile/assemble)
- **Test command** (unit tests)
- **Lint command** (static analysis / code quality)

### 2. Auto-detection (fallback)

If CLAUDE.md doesn't define commands, detect from project markers:

| Marker Files | Project Type | Build | Test | Lint |
|-------------|-------------|-------|------|------|
| `gradlew`, `build.gradle.kts` | Android/Gradle | `./gradlew assembleDebug` | `./gradlew test` | `./gradlew detekt` |
| `*.xcodeproj` | iOS/Xcode | `xcodebuild build ...` | `xcodebuild test ...` | (none) |
| `package.json` + `node_modules` | Node.js | `npm run build` | `npm test` | `npm run lint` |
| `Cargo.toml` | Rust | `cargo build` | `cargo test` | `cargo clippy` |
| `go.mod` | Go | `go build ./...` | `go test ./...` | `golangci-lint run` |
| `Makefile` | Make-based | `make build` | `make test` | `make lint` |
| `pyproject.toml` / `setup.py` | Python | (skip) | `pytest` | `ruff check .` or `flake8` |

### 3. Ask user (last resort)

If neither CLAUDE.md nor auto-detection works:

> Could not detect your project's build system. Please provide:
> - **Build command:** (e.g., `make build`)
> - **Test command:** (e.g., `make test`)
> - **Lint command:** (e.g., `make lint`, or "none")

### Multiple projects in workspace (monorepo)

If multiple project types are detected (e.g., Android + iOS in same repo):
> Multiple project types detected:
> - Android (gradlew found)
> - iOS (xcodeproj found)
>
> Which should I verify? (android / ios / both)

## Step 2: Run Verification

Run each command **sequentially**. Present results as each completes.

### For each command (build, test, lint):

**Before running:**
> Running [step name]... `[command]`

**Track duration:**
Record start time, end time, and display duration.

**Timeout handling:**
- Set a reasonable timeout per command (default: 10 minutes for build, 10 minutes for tests, 5 minutes for lint)
- If a command times out:
  > [Step] timed out after [X] minutes.
  > The command may be hanging. Want to: (retry / skip / increase timeout)

### Interpreting Results

**Build:**
- Exit code 0 → "Build: PASS ([duration])"
- Non-zero → parse error output:
  - Compilation errors: extract file, line, message
  - Missing dependencies: identify which dependency
  - Configuration errors: extract config issue
  - Present: "Build: FAIL — [N errors]" + specific errors

**Tests:**
- Exit code 0 → "Tests: PASS ([N tests], [duration])"
- Non-zero → parse output for:
  - Number of tests run, passed, failed, skipped
  - For each failure: test name, assertion message, expected vs actual
  - Present: "Tests: FAIL — [N of M failed]" + failure details

**Lint:**
- Exit code 0 → "Lint: PASS ([duration])"
- Non-zero → parse output for:
  - Number of violations/warnings/errors
  - For each: file, line, rule, message, severity
  - Present: "Lint: FAIL — [N violations]" + details
- **If no lint command configured:**
  > Lint: SKIPPED (no linter configured for this project)

## Step 3: Report Results

**All passed:**
> **Verification Complete** ([total duration]):
> - Build: PASS ([duration])
> - Tests: PASS ([N tests], [duration])
> - Lint: PASS ([duration])
>
> All checks passed. Ready to proceed.

**Any failed:**
> **Verification Failed:**
> - Build: [PASS/FAIL] ([duration])
> - Tests: [PASS ✓ / FAIL ✗ — N failures] ([duration])
> - Lint: [PASS ✓ / FAIL ✗ — N violations] ([duration])
>
> **Failures:**
> [Detailed error output for each failed step]
>
> Fix these issues before proceeding.

## Step 4: Fix and Re-verify Loop

On failure:
1. Present the specific errors clearly (file, line, message)
2. Help the user fix the issues if asked
3. **After ANY fix, re-run the FULL verification suite** — not just the previously failing step
   - A fix for tests could break the build
   - A fix for lint could introduce test failures
4. **Do not proceed to next steps (PR, etc.) until ALL pass**

**Loop until clean. No exceptions.**

## Step 5: Handle Edge Cases

**No tests found:**
> Tests: SKIPPED (no tests found — `[command]` returned 0 tests)
> ⚠ Consider adding tests before creating a PR.

Proceed but warn the user.

**Flaky tests (pass on retry):**
If a test fails and user says "retry":
- Re-run tests
- If they pass: "Tests: PASS (passed on retry — may be flaky)"
- Flag the flaky test names for the PR description

**Partial verification requested:**
If user explicitly asks to skip a step (e.g., "skip lint"):
> ⚠ Skipping lint at your request. This may cause CI failures.
Allow it but warn. Never skip build or tests without explicit user request.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Running only the failing step after a fix | Always re-run ALL steps — fixes can have side effects |
| Hardcoding project-specific commands | Read from CLAUDE.md first, then auto-detect |
| Proceeding with warnings as "good enough" | Warnings may be errors in CI — treat them seriously |
| No timeout on build commands | Set timeouts to prevent hanging |
| Skipping verification entirely | Never skip. This is the last line of defense before PR |
| Running in wrong directory | Ensure you're in the project root (where build files are) |
| Ignoring lint because "it's just style" | Linters catch real bugs (null safety, unused vars, etc.) |
