# Handoff — 2026-06-29

## What's Done

**#590: CallableDispatchRegistry — extensible workflow call dispatch — CLOSED**

`CasehubCallableTaskBuilder` no longer hardcodes `casehub:dispatch`. A `CallableDispatchRegistry` CDI singleton maps call names to `CallableDispatcher` implementations. `CasehubDispatch` self-registers at `@PostConstruct`. First consumer: `casehub-desiredstate` for `desiredstate:dispatch`.

**#586, #587, #588, #589: composite WEM follow-ups — ALL CLOSED**

`WorkerFunction.None` in `casehub-worker-api` models external workers. `canExecute(WorkerFunction)` additive SPI on `WorkerExecutionManager` — Quartz overrides with positive handler delegation, composite delegates to backends, routing strategy checks both `supports()` and `canExecute()`. Non-blocking recovery with `RecoveryStatus`. Trigger methods documented as planned API. Design-reviewed (9 rounds, 19 issues, 11 verified, $23.21).

Cross-repo: `casehub-worker-api` SNAPSHOT published with `None` record. Workers repo — no changes needed. `parent#326` filed for PLATFORM.md + engine deep-dive sync.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap documented in design review |
| #593 | RecoveryStatus health check integration | S | Low | Wire to @Liveness or @Readiness |
| #594 | QuartzWEM line 91 multi-JVM TODO cleanup | S | Low | Pre-existing design debt |
| parent#326 | Sync PLATFORM.md + casehub-engine.md for None/canExecute | S | Low | Doc sync |
