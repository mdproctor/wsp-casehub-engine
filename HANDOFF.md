# Handoff — 2026-06-12

**Branch:** issue-473-fix-ci-timeouts
**Head commit (engine):** 8562036d — fix(runtime): flip default Maven profile to persistence-memory for CI
**Head commit (workspace):** (workspace main — scaffold committed on branch)

## What Changed This Session

### engine#471 — domainContentBytes() overrides (MERGED via PR #474)

Added `domainContentBytes()` to `CaseLedgerEntry` (4 fields) and `WorkerDecisionEntry` (5 fields) for Merkle content integrity. Required by ledger#128's build-time guard. Three CI iterations:
1. Compilation failure — ledger SNAPSHOT on GitHub Packages didn't have `domainContentBytes()` yet (ledger#128 not merged). Waited for ledger#128 to land.
2. `WorkerDecisionEntry` missing override — same guard, second entity. Fixed.
3. `HumanTaskScheduleHandlerTest` — `WorkItemTemplate.tenancyId` null (NOT NULL constraint) + `WorkItem.tenancyId` null in template mode. Fixed test helper + production handler.

### engine#473 — fix CI timeouts (IN PROGRESS, not pushed)

Two commits on `issue-473-fix-ci-timeouts`:

**Commit 1 — ledger guard cherry-pick:** `domainContentBytes()` overrides (same as #471, needed on this branch since it predates the #471 merge).

**Commit 2 — profile flip:**
- `persistence-memory` is now `activeByDefault` in `runtime/pom.xml`; `persistence-hibernate` is opt-in (`-P persistence-hibernate,!persistence-memory`)
- `quarkus-jdbc-h2` promoted to main test deps
- `quarkus.datasource.db-kind=postgresql` added explicitly to `application.properties` (was in `persistence-hibernate` module's main resources — polluted consumer classpath)
- `application-test.properties` deleted; `QuarkusConfigManagerTest` fixtures moved to base `application.properties` (the `memory` profile doesn't load `application-test.properties` because the active profile is `memory`, not `test`)

### engine#480 — cross-test Quartz contamination (FILED, not started)

Discovered during #473 investigation. When the full runtime suite runs under `persistence-memory`, 5 tests fail with `ConditionTimeoutException` — cases start but workers never execute. Root cause: `CaseFaultedStateTest` starts 6 cases with `AlwaysFailingCaseHubBean` (retries, failures, event bus saturation). Stale Quartz jobs and event bus messages bleed into subsequent test classes. Hidden in `persistence-hibernate` by Testcontainers startup latency (~8-12s drain window).

**Production-correct fix direction:** terminal cases (FAULTED, COMPLETED, CANCELLED) should cancel all their Quartz jobs in `CaseStatusChangedHandler`. This fixes the contamination as a side-effect of correct behavior.

## Immediate Next Step

**Start engine#480.** The #473 branch is committed but NOT pushed — it needs #480 resolved first so the full local suite passes before pushing. Follow fix-ci discipline: reproduce isolated, root-cause, fix, verify full suite, then push.

Investigation done so far:
- `SimpleCaseHubBeanTest` alone: 1.7s, passes
- After `CaseFaultedStateTest`: 10.2s timeout, no "Agent selected" for `document-processor` in logs
- The context-change event for `.working.status == "processing"` never fires after CaseFaultedStateTest runs
- Fix direction: cancel Quartz jobs on terminal case state, or clear scheduler between test classes

## Branch State

| Branch | Repo | Status |
|--------|------|--------|
| `issue-473-fix-ci-timeouts` | engine | 2 unpushed commits, blocked on #480 |
| `issue-473-fix-ci-timeouts` | workspace | scaffold only |
| `issue-471-domain-content-bytes` | engine | MERGED (PR #474) |

## What's Left

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| 480 | Cross-test Quartz contamination | S | Med | Filed, not started — blocks #473 push |
| 473 | Fix CI timeouts | S | Low | Committed, not pushed — blocked by #480 |
| 465 | Validate panel event model for Drools re-fire | XS | Low | Not started |
| 446 | WorkingMemoryBridge — typed Drools facts | M | Med | Unblocked |

## Key References

- PR #474: merged (engine#471)
- Issue #480: cross-test Quartz contamination
- Spec: `specs/issue-473-fix-ci-timeouts/2026-06-11-ci-timeout-fix-design.md` (reviewed, findings addressed in implementation)
