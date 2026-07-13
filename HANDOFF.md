# Handoff — 2026-07-13

## What's Done

CBR experience analyser + trust-weighted routing landed. Issue triage + cross-repo artifact migration session.

- Landed #728 (88b1e1ce) — ExperienceAnalyser, TrustRoutingPolicy.cbrWeight, TrustWeightedAgentStrategy CBR scoring. Closes devtown#133
- Closed #719, #720 (PlanItemStatus shim deleted — zero engine refs), #724 (EngineStrategyResolver catch-all + work#304 fixed Jandex)
- Confirmed #700 epic complete — shared type foundation landed. #702 verified as genuinely unfinished
- Migrated 4 consumers from old `casehub-engine-work-adapter` → `casehub-work-engine-adapter`: AML, Clinical (10 files — imports + hibernate packages), DevTown (7 files), SOC. All pushed
- Engine pushed (88b1e1ce). Old `casehub-engine-work-adapter` package deleted from GitHub Packages
- Parent POM already serves as the ecosystem BOM — no new module needed

**Pre-existing consumer build failures** (not caused by migration):
- AML: missing blocks import (`RoutingFeatureExtractor`)
- SOC: missing routing types (`CandidateSetStrategy`/`StaticSetStrategy`)
- Clinical: CDI Clock ambiguity (`ClinicalClockProducer` vs `qhorus ClockProducer`)

## Immediate Next Step

Pick next work from What's Next. Build is clean on engine main.

## Cross-Module

- casehub-blocks: TrustRoutingPolicy constructor (added Set<TrustPhase>) needs propagating to 3 test files · XS · Low

## What's Left

- #702 — event/handler ExecutorRef migration · M · Low
- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #716 — cross-repo consumer updates for #571/#548 · M · Low
- #723 — RoutingSignalAssembler unit tests · S · Low
- #725 — wire CaseContextStoreFactory through CaseHubRuntimeImpl · S · Low
- casehub-blocks: TrustRoutingPolicy 8th param propagation (3 test files) · XS · Low
- Pre-existing build failures in AML, SOC, Clinical (see above) · S–M · varies

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #725 | Wire CaseContextStoreFactory through CaseHubRuntimeImpl | S | Low | Follow-up from #419 |
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |
