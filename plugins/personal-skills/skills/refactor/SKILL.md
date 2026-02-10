---
name: refactor
description: "Autonomous test-first refactoring for Go and Java codebases with self-healing loop, rollback safety, and single-commit output. Use when the user asks to refactor a target (file, package, class, method, or symbol) in a Go or Java repo. Triggers on: refactor requests, code cleanup, maintainability improvements, dead code removal, deduplication, or extracting abstractions. Accepts a TARGET argument specifying what to refactor."
---

# Autonomous Refactor Suite

Refactor `<TARGET>` using a test-first, self-healing loop. Operate autonomously and make reasonable decisions without asking questions unless hitting a hard blocker.

`<TARGET>` is provided as a prompt argument: a file/directory path, Go package name, Java package/class/method, exported/public symbol, or command/handler identifier. If not provided or unresolvable, STOP and ask.

## Hard Blockers (STOP and Report)

STOP immediately if any of these are true:
- `<TARGET>` cannot be resolved to any file or symbol.
- Missing credentials or environment required to run tests.
- Test/build command cannot be determined.
- Baseline tests or lint are already failing (red base).
- Working tree is dirty and user has not acknowledged it.
- Repo primary language is not Go or Java.
- Iteration budget (10 cycles) exhausted without reaching green.

## Definitions

- **External behavior / contract**: public API surface (exported symbols, CLI flags, HTTP routes, protobuf/JSON fields with names+tags+field numbers, SQL schema/migrations, config keys+defaults), error contract (types/codes, user-facing messages), retry/backoff/timeouts/context-cancellation semantics, concurrency guarantees (thread-safety, locking order, goroutine/thread lifecycle), observability contract (log field names/keys, metric names/labels consumed downstream).
- **Baseline**: current HEAD/working tree state before any changes.
- **Last green state**: last commit or working tree state where full suite and linters pass locally.
- **Objectively wrong test**: asserts implementation details rather than behavior, has data race in test code itself, depends on non-deterministic ordering without accounting for it, or contains demonstrably incorrect expected value. When in doubt, the test is NOT wrong.

## Target Resolution Priority

1. Explicit path provided by user.
2. Exported/public symbol match.
3. Entrypoint/usage (imports, call sites, routes, CLI wiring).
4. Highest-churn core files in the inferred area.

Do NOT rename exported/public identifiers, config keys, wire fields, routes, or CLI flags unless explicitly allowed.

## Phase 0: Pre-flight Checks

1. **Language Check**: Confirm Go or Java. If neither, STOP.
2. **Target Resolution**: Verify `<TARGET>` exists per priority list. If not, STOP.
3. **Working Tree Check**: `git status --porcelain`.
   - Clean: proceed.
   - Dirty: warn user, `git stash`, note stash in final output.
4. **Monorepo Detection**: Check for multiple `go.mod`, Maven multi-module (`<modules>` in parent pom), or Gradle multi-project (`settings.gradle` with `include`). If monorepo, scope commands to the module containing `<TARGET>`.
5. **Tooling Availability**:
   - Verify test runner (`go` / `mvn` or `./mvnw` / `gradle` or `./gradlew`).
   - Go: check `golangci-lint`; fall back to `go vet ./...` if missing.
   - Java: detect repo-standard formatter/linter (e.g., `spotlessApply`, `checkstyle`) from build config. Do not introduce new tooling.
   - No test runner determinable: STOP.

## Phase 1: Discovery (Read-only)

1. **Scope**: List primary files/classes/functions for `<TARGET>` and call graph boundaries.
2. **Side-effect Inventory**: Enumerate behaviors to preserve:
   - I/O boundaries (filesystem, network, DB, subprocess)
   - Concurrency (locks, goroutines/threads, shared state)
   - Retries/backoff/timeouts/cancellation
   - Error mapping and user-facing messages
   - Caching/memoization and invalidation rules
   - Logging/metrics patterns (keys/labels)
3. **Implicit Dependencies**: Search for non-obvious couplings:
   - String-based references in configs/README/docs
   - Reflection/DI wiring (Spring `@Qualifier`, `@ConditionalOn*`)
   - Codegen inputs/outputs (proto/openapi, mockgen, annotation processors)
   - Build tags/profiles/feature flags/env vars
4. **Gap Analysis**: Identify existing tests and coverage gaps for `<TARGET>` behaviors.

## Phase 2: Test Plan (Before Refactor)

Based on gap analysis, decide if additional tests are needed. If existing tests adequately cover `<TARGET>`: skip to Phase 3. Otherwise add targeted unit tests FIRST (deterministic; no real network/DB).

**Test guidelines when adding**:
- Cover: empty/invalid config; missing resources/keys; 403/404 handling; timeouts/retries/cancellation (mocks/fakes); concurrency/caching correctness; error contract mapping.
- Prefer table-driven tests (Go). Prefer fakes over mocks when simpler.
- Avoid sleep-based timing. Do NOT introduce flaky tests.

## Phase 3: Tooling Reference

### Go

| Task | Command |
|---|---|
| Full suite | `go test -count=1 -race ./...` (or module-scoped) |
| Target-scoped | `go test -count=1 -race ./path/to/pkg` |
| Compile | `go build ./...` |
| Format | `gofmt -w .` |
| Lint (primary) | `golangci-lint run` |
| Lint (fallback) | `go vet ./...` |

### Java

| Task | Maven | Gradle |
|---|---|---|
| Compile | `./mvnw -q compile` | `./gradlew compileJava compileTestJava -q` |
| Full suite | `./mvnw -q test` | `./gradlew test -q` |
| Target-scoped | `./mvnw -q test -pl <module> -Dtest=<TestClass>` | `./gradlew :<module>:test --tests "<pattern>" -q` |
| Format/Lint | Repo-standard tasks (e.g., `spotlessApply`, `checkstyle`) | Same |

**Build Tool Selection**: `mvnw`/`pom.xml` → Maven. `gradlew`/`build.gradle*` → Gradle. Both → prefer CI config choice.

**Java concurrency**: Run target-scoped tests up to 3 times for thread-safety confidence.

## Phase 4: Autonomous Loop (Max 10 Cycles)

### Step A: Baseline Validation (Once)

1. Compile check.
2. Full test suite.
3. Lint.
4. If ANY fail, STOP and report. Do not refactor on red base.

### Step B: Iterative Cycle

1. **Choose one atomic refactor step** (single-purpose: simplify error handling, extract seam/interface, remove duplication/dead code, inline unnecessary abstraction). Prefer refactors that increase testability without changing behavior. Update only internal/local names unless explicitly allowed.
2. **Refactor Ledger**: State `Modifying [X] to [Y], preserving [Z]` referencing specific contract points from side-effect inventory.
3. **Apply** the minimal change.
4. **Fast Verification**: Target-scoped tests first. Fix failures before full gate.
5. **Format**: `gofmt -w .` (Go) or repo-standard formatter (Java).
6. **Full Gate**: Lint + full test suite.
7. **Gate Result**:
   - All pass: record as new last green state, proceed.
   - Failure: enter Self-Healing (Step C).

### Step C: Self-Healing & Rollback

1. **Diagnose** from test/lint output. Fix the **implementation**, not the tests.
2. **Re-run** failing command after each fix.
3. **Flaky handling**: Re-run failing test up to 2 times to confirm deterministic before acting.
4. **Back-off**: If step cannot be fixed within 2 attempts:
   - `git diff --name-only` to identify changed files.
   - `git checkout -- <files>` to revert to last green state.
   - Log the failed approach.
   - Retry with smaller/different approach (counts as new cycle).
5. **Budget**: At 10 cycles without stable green: `git checkout -- .` to revert ALL to baseline, STOP, report.

## Phase 5: Commit (Single Commit)

### Pre-commit Final Check

ALL must pass before committing:
- Full test suite.
- Lint clean (Go: `golangci-lint run` or `go vet`; Java: repo-standard).
- Contract checklist from side-effect inventory satisfied.

### Commit Message Format

```
refactor(<TARGET>): <concise summary>

What:
- <what was simplified, bullet list>

Why:
- <why it improves maintainability>

Preserved contract:
- <key items from side-effect inventory verified>

Verification:
- <commands run + pass status>
```

If stash was created in pre-flight, remind user to `git stash pop`.

## Final Output

### Summary
- Key improvements (bullets).

### Execution Log
- All commands run + results (test counts, pass/fail, `-race` for Go, lint status, compile status for Java).
- Cycles used out of budget.
- Failed approaches that were reverted (with brief reason).

### Git Artifacts
- Single commit hash + full commit message.

### Behavior Verification Notes
- Contract points explicitly tested/validated.
- For Java concurrent code: number of repeated test runs and results.

### Residual Risks
- Technical debt or follow-ups not addressed (with rationale).
- Tooling degradations (e.g., golangci-lint unavailable).
- Stash reminder if applicable.