# Handoff — 2026-07-13

## What's Done

**CaseLifecycleEvent enrichment + composed GoalExpression (#571, #548) — landed on main as 8a5d25af.** CaseLifecycleEvent gains caseDefinitionName, namespace, contextSnapshot (JsonNode) — consumers discriminate by case type without repository round-trips. GoalExpression redesigned as sealed recursive tree (AllOf, AnyOf, Single) — supports nested anyOf(allOf(...), ...) composition. Handler simplified from 30-line instanceof chain to single satisfiedGoalName() call. YAML parser recursive with parse-time validation. Design review: 5 rounds, 16 issues, all resolved ($16.26). Three follow-up issues created by the design review: #714 (milestone SLA TODO), #715 (schema-level GoalExpression), #716 (cross-repo consumer updates).

## Immediate Next Step

Pick next work from What's Next. Pre-existing ledger test failure (CommitmentContext constructor mismatch in TrustGatedAttestationPolicyTest) still blocks full `mvn install` — fix before next ledger work. Pre-existing checkstyle error in runtime module — investigate and fix.

## Cross-Module

- casehub-work: engine-adapter commit for #710 landed on branch issue-287-retrofit-work-spis-namedstrategy (not main)
- casehub-neocortex: FeatureValue.of()/toFeatureMap() on branch issue-137-approx-dtw-typed-cbr (not main)

## What's Left

- #648 — OutcomeRecorder.addAttestation — deferred to dedicated ledger session · S · Med (cross-repo)
- #646 — per-case CONTEXT_CHANGED serialization · M · Med
- #702 — event/handler ExecutorRef migration · M · Low
- #714 — milestone SLA TODO (MilestoneActivatedEventHandler:200) · XS · Low
- #715 — schema-level GoalExpression model update (jsonschema2pojo) · S · Low
- #716 — cross-repo consumer updates for #571/#548 (clinical, aml, life, devtown) · M · Low
- Pre-existing: ledger CommitmentContext test mismatch (qhorus-api constructor change) · XS · Low
- Pre-existing: runtime checkstyle error · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #695 | DAG-aware parallel execution driver | L | High | Unblocked by #694 |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
| #689 | WorkItems boundary — typed payload | M | Med | |
| #635 | Rename io.casehub.api → io.casehub.engine.api | L | Low | Cross-repo |

## References

- Design review workspace: ~/adr/casehub-engine/lifecycle-event-goal-composition-20260713-022602/
- Garden: GE-20260713-905e2e (ide_create_file VFS-only persistence gotcha)
