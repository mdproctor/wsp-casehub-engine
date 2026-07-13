# Handoff — 2026-07-13

## What's Done

**CaseContextStore SPI (#419) — landed on main as 274c6dd2.** Pluggable context storage below CaseContextImpl. Three new SPI types (CaseContextStore, CaseContextStoreFactory, MutableCaseContext). WritableLayerImpl delegates to store. All 8 instanceof CaseContextImpl eliminated. Store lifecycle on case terminal status. Example module with AuditingCaseContextStore + 21-test contract suite. Design spec adversarially reviewed (5 rounds, 18 issues verified). Follow-up: #725 (wire factory resolution through CaseHubRuntimeImpl).

## Immediate Next Step

Pick next work from What's Next. Build is clean — `mvn install -DskipTests` passes.

## Cross-Module

- casehub-blocks: TrustRoutingPolicy constructor (added Set<TrustPhase>) needs propagating to 3 test files (RoutingSupportTest, CbrAgentRoutingStrategyTest, LlmAgentRoutingStrategyTest)

## What's Left

- #725 — wire CaseContextStoreFactory resolution through CaseHubRuntimeImpl · S · Low
- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low
- #716 — cross-repo consumer updates for #571/#548 (clinical, aml, life, devtown) · M · Low
- #723 — RoutingSignalAssembler unit tests · S · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #725 | Wire CaseContextStoreFactory through CaseHubRuntimeImpl | S | Low | Follow-up from #419 |
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
