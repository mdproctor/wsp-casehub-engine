# Design Journal — issue-1141-cdi-event-wiring

## 2026-09-23

### #1141 — CDI event wiring

Injected `Event<T>` into 4 source beans and added `fireAsync()` calls at transition points:
- `ImprovementCircuitBreaker` — fires `CircuitBreakerStateChangedEvent` on state transitions in `evaluate()` and `manualReset()`. Tracks old vs new state; only fires when state actually changes.
- `ReadinessValidator` — fires `ComplianceLevelChangedEvent` when computed project level differs from cached level. Added `ConcurrentHashMap<UUID, ComplianceLevel>` cache; no event on first validation (no previous level to compare).
- `RegressionDetector` — fires `RegressionDetectedEvent` in `onMetricsDegraded()` with confidence score and category. Fires before the rollback/pause decision so the event captures the raw detection.
- `EvolutionTicker` — fires `TickEvaluatedEvent` for notable outcomes only (gate blocked or proposal generated). Quiet ticks (no consensus) don't fire — keeps the SSE stream meaningful.

Added `TestEvent<T>` (test helper) and `NoOpEvent<T>` (production no-op for convenience constructors). 11 new tests, 462 total pass.

### #1145 — YAML codegen entries

Added `GatePolicy`, `EscalationPolicy`, and `CategoryEscalationRules` to `yaml-record-mappings.yaml`. `GatePolicy` uses `Map<ImprovementStage, GateMode>` — the first enum-keyed map in the codegen.

Fixed a bug in `RecordEmitter.collectTypeImports()`: it only resolved imports for the last generic type parameter (the map value), not the key. Changed to iterate over all comma-separated parts in the generic signature. 17 codegen tests pass.
