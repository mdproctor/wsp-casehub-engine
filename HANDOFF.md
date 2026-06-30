# Handoff — 2026-06-30

## What's Done

**#591: Worker.capabilityNames + YamlCaseHub.augment() — CLOSED**

Worker record changed from `List<Capability> capabilities` to `Set<String> capabilityNames` in casehub-worker-api. Workers declare support by name; the engine resolves authoritative `Capability` instances from `CaseDefinition.getCapabilities()`. `Set<String>` enables O(1) contains() and rejects duplicates. ~60 engine test files migrated.

`YamlCaseHub.getDefinition()` is `final`. Subclasses override `augment(CaseDefinition)` — called inside the DCL between YAML loading and caching. Replaces three inconsistent consumer patterns (life's DCL duplication, aml's `@PostConstruct` delegation, devtown's race-prone mutation).

`DeadLetterReplayService` resolves capability from EventLog metadata instead of `worker.capabilities().stream().findFirst()`.

Design-reviewed (6 rounds, 18 issues, $20.10). casehub-worker-api SNAPSHOT published with `Worker.capabilityNames` + `WorkerFunction.None`.

**#509: Binding.inputSchemaOverride — already done, closed**

Was fully implemented in a previous session (commit `373b4d75`). Closed during this session.

## Cross-Module

**Consumer repos need migration** (all filed, all blocked by engine SNAPSHOT):
- casehub-life#47 — 8 CaseHubs → `augment()` + `capabilityNames()`
- casehub-aml#85 — 2 CaseHubs
- casehub-devtown#117 — 2 CaseHubs (fixes race condition)
- casehub-desiredstate#50 — `CaseTransitionExecutor` Worker builder call
- casehubio/parent#328 — PLATFORM.md + casehub-engine.md doc sync

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap documented in design review |
| #593 | RecoveryStatus health check integration | S | Low | Wire to @Liveness or @Readiness |
| #594 | QuartzWEM line 91 multi-JVM TODO cleanup | S | Low | Pre-existing design debt |
