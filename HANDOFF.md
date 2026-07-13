# Handoff — 2026-07-13

## What's Done

**S/XS cleanup batch (#712) — landed on main as 8495680d.** 7 engine issues closed, 2 cross-repo follow-ups completed. Key changes: tenancyId on SubCaseGroupLifecycleEvent (#709), outputSchema→outputProjection rename (#677), bindingName threading through submit() (#676), TrustPhase enum + evidentialCheckPhases (#711), CBR experiences flow to WorkerContext (#707). Test suite failure #669 resolved by tenant threading. Types/labels #653 confirmed already done. Work-adapter tenancyId #710 committed to casehub-work. FeatureValue.of()/toFeatureMap() added to neocortex memory-api (branch issue-137-approx-dtw-typed-cbr).

## Immediate Next Step

All pushed. Pick next work from What's Next. Pre-existing ledger test failure (CommitmentContext constructor mismatch in TrustGatedAttestationPolicyTest) blocks full `mvn install` — fix before next ledger work.

## Cross-Module

- casehub-work: engine-adapter commit for #710 landed on branch issue-287-retrofit-work-spis-namedstrategy (not main)
- casehub-ops: TrustRoutingPolicy constructor update committed to ops main
- casehub-neocortex: FeatureValue.of()/toFeatureMap() on branch issue-137-approx-dtw-typed-cbr (not main)

## What's Left

- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low
- Pre-existing: ledger CommitmentContext test mismatch (qhorus-api constructor change) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |

## References

- Garden: GE-20260713-905e2e (ide_create_file VFS-only persistence gotcha)
