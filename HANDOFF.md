# Handoff — 2026-07-13

## What's Done

**S/XS cleanup batch (#714, #715, #721, #722) — landed on main as 4c53ed55.** Immediate SLA violation firing when deadline already passed (#714). Recursive GoalExpression schema (#715). CommitmentContext, TrustRoutingPolicy, GoalExpression API drift fixes across 6 modules (#721, #722). Also landed: RoutingSignalProvider SPI (blocks#34, from parallel session).

## Immediate Next Step

Pick next work from What's Next. Build is clean — `mvn install -DskipTests` passes.

## Cross-Module

- casehub-work: engine-adapter commit for #710 landed on branch issue-287-retrofit-work-spis-namedstrategy (not main)
- casehub-neocortex: FeatureValue.of()/toFeatureMap() on branch issue-137-approx-dtw-typed-cbr (not main)
- casehub-blocks: TrustRoutingPolicy constructor (added Set<TrustPhase>) needs propagating to 3 test files (RoutingSupportTest, CbrAgentRoutingStrategyTest, LlmAgentRoutingStrategyTest)

## What's Left

- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low
- #716 — cross-repo consumer updates for #571/#548 (clinical, aml, life, devtown) · M · Low
- #723 — RoutingSignalAssembler unit tests · S · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
