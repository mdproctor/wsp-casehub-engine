# Handoff — 2026-07-14

## What's Done

CaseContextStoreFactory wiring (#725) + DAG parallel execution driver (#695) landed. Follow-up issue #732 filed for recovery path.

- Landed 6be93db6 — single squashed commit on main. Closes #725, #695
- #725: CaseHubRuntimeImpl resolves factory via StrategyResolver, generates UUID early, threads through reactor. YAML `context.storeFactory`. Durable factory guard until #732
- #695: `io.casehub.engine.plan` package in engine-common. DagPlan, DagNode, JoinType, NodeState, DagDriver (STREAMING + BARRIER), DagEventListener, DagResult. 42 tests. Pure java.util.concurrent
- 2 garden entries submitted: @DefaultBean CDI precedence (GE-20260714-e97b0a), diamond type inference (GE-20260714-aa950f)
- CLAUDE.md updated with both features

## Immediate Next Step

Pick next work from What's Next. Build is clean on engine main.

## Cross-Module

- casehub-blocks: TrustRoutingPolicy 8th param (Set\<TrustPhase\>) needs propagating to 3 test files · XS · Low

## What's Left

- #732 — recovery path factory wiring (filed this session, blocks durable factories) · M · Med
- #702 — event/handler ExecutorRef migration · M · Low
- #648 — OutcomeRecorder.addAttestation · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #716 — cross-repo consumer updates for #571/#548 · M · Low
- #723 — RoutingSignalAssembler unit tests · S · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low
- Pre-existing build failures in AML, SOC, Clinical · S–M · varies
- Pre-existing SNAPSHOT drift: QhorusMessageSignalBridge + CbrCaseRetainObserver test compilation errors · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #732 | Wire CaseContextStoreFactory through recovery path | M | Med | Filed this session, blocks durable factories |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | ContextBridge arc |
| #692 | Connector boundary — typed context via ContextBridge | M | Med | ContextBridge arc |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
