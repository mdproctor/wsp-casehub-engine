# Handoff — 2026-06-30

## What's Done

**#585: WorkItemLifecycleAdapter observer migration — CLOSED**

Observer was already migrated to `WorkItemEvent` in 386bb144. This session fixed stale Javadoc and CLAUDE.md references.

**#593: Recovery health check — CLOSED**

`WorkerRecoveryCoordinator` (runtime, `@Priority(22)`) extracts recovery initiation from `QuartzWorkerExecutionManager`. `@Liveness` health check at `/q/health/live` with configurable timeout (`casehub.engine.recovery.timeout`, default 60s). `quarkus-smallrye-health` added to runtime — first SmallRye Health infrastructure in the engine. Design-reviewed (3 rounds, 11 issues, $9.70).

**#594: Stale TODO removal — CLOSED**

Multi-JVM fan-out TODO removed from `QuartzWorkerExecutionManager`. Architecture uses RAM store and routes via `CompositeWorkerExecutionManager`.

## Cross-Module

**Consumer repos still need capabilityNames migration** (from previous session, all filed):
- casehub-life#47 — 8 CaseHubs → `augment()` + `capabilityNames()`
- casehub-aml#85 — 2 CaseHubs
- casehub-devtown#117 — 2 CaseHubs (fixes race condition)
- casehub-desiredstate#50 — `CaseTransitionExecutor` Worker builder call
- casehubio/parent#328 — PLATFORM.md + casehub-engine.md doc sync

**casehub-work#278 unblocked** — engine no longer references `WorkLifecycleEvent`

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap documented in design review |
| — | HumanTaskRecoveryService health check | S | Low | Follow-up from #593 design review — different failure profile |
| — | Integration tests for WorkerRecoveryCoordinator | S | Low | Follow-up from #593 final review — unit tests sufficient for now |
