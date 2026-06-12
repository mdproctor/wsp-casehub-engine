# Handoff — 2026-06-12

**Head commit (engine main):** 10cec8fe — fix(actor-state): update stale exclude-types
**CI:** green — run #27425001279 (all modules pass, published to GitHub Packages)

## What Changed This Session

### engine#480 — CaseDefinition key collision (CLOSED)

Root-caused the "cross-test Quartz contamination" to a CaseDefinition registry key collision. `SimpleCaseHubBean` (Java) and `YamlSimpleCaseHubBean` (YAML) shared key `(test, Document Processing Test, 1.0.0)` with incompatible JQ expressions. Non-deterministic CDI bean ordering caused flaky failures across 12 tests.

Fix: unique keys for each bean + fail-fast duplicate detection in `registerKnownDefinitions()` + `CaseKeyUniquenessTest` regression guard. Garden entry `GE-20260612-279b44` submitted.

### engine#473 — CI timeout fix (CLOSED)

Flipped default Maven profile from `persistence-hibernate` to `persistence-memory`. CI no longer needs Docker/Testcontainers. Blocked on #480 — unblocked after the key collision fix landed.

### engine#481 — actor-state test failure (CLOSED)

Stale class names in `quarkus.arc.exclude-types` — `casehub-work` renamed `ClaimDeadlineJob` → `ClaimDeadlineTimerJob` and removed `ExpiryCleanupJob`. Added `ExcludeTypesValidationTest` to catch future stale entries.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 465 | Validate panel event model for Drools re-fire | XS | Low | Design gate for #446 |
| 446 | WorkingMemoryBridge — typed Drools facts | M | Med | Unblocked |

## Key References

- Blog: `blog/2026-06-12-mdp01-the-registry-that-ate-the-scheduler.md`
- Garden: `GE-20260612-279b44` (CDI bean ordering + registry collision gotcha)
