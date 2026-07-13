# Handoff — 2026-07-13

## What's Done

Issue triage session. Closed #719, #720 (PlanItemStatus shim deleted — zero engine refs), #724 (EngineStrategyResolver catch-all + work#304 fixed Jandex). Confirmed #700 epic complete — shared type foundation landed. #702 (ExecutorRef event migration) verified as genuinely unfinished.

**⚠️ SNAPSHOT breakage:** Deleting PlanItemStatus broke AML — old `casehub-engine-work-adapter` SNAPSHOT still references it. Either restore the shim or publish a new work-adapter SNAPSHOT from the work repo. Address before pushing engine to origin.

## Immediate Next Step

Fix the PlanItemStatus SNAPSHOT breakage before pushing 415de3d1 to origin. Options: (1) revert the deletion and reopen #719/#720, (2) publish a new work-adapter SNAPSHOT from the work repo without the reference. Option 2 is the clean fix.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- **⚠️ PlanItemStatus SNAPSHOT breakage** — fix before pushing engine · XS · Low
- #702 — event/handler ExecutorRef migration · M · Low
- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #716 — cross-repo consumer updates for #571/#548 · M · Low
- #723 — RoutingSignalAssembler unit tests · S · Low
- #725 — wire CaseContextStoreFactory through CaseHubRuntimeImpl · S · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #725 | Wire CaseContextStoreFactory through CaseHubRuntimeImpl | S | Low | Follow-up from #419 |
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
