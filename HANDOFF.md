# Handoff — 2026-07-04

## What's Done

**#634** — Universal pluggable routing strategy architecture. NamedStrategy + StrategyResolver in platform-api, CandidateSetStrategy (replaces sealed ListEvaluator), CandidateMatchingStrategy (replaces hardcoded AgentCandidateFactory), 5 engine SPIs retrofitted. Design-reviewed (4 rounds, 18 issues). Implementation-reviewed (per-task + full-branch). Landed on main as `0001b5ac`.

Also filed during this session: #636 (WorkerRuntime.spawnCase/awaitCase), #637 (SequentialPlanningStrategy test fix), #641 (GateRequired eval timing), #642 (EngineStrategyResolver default determinism), #643 (PLATFORM.md + protocol docs), #644 (consumer repo migration).

## Cross-Module

**Consumer repos need routing strategy migration (#644):**
- Add `id()` to ActionRiskClassifier implementations: devtown, aml, clinical, life, soc, iot
- Add `id()` to TrustRoutingPolicyProvider implementations: devtown, aml, clinical, life, quarkmind, ops
- casehub-work#287 — retrofit WorkerSelectionStrategy, InstanceAssignmentStrategy, ClaimSlaPolicy

**Consumer repos still need capabilityNames migration (pre-existing):**
- casehub-life#47, casehub-aml#85, casehub-devtown#117, casehub-desiredstate#50, casehubio/parent#328

## What's Left

- #641 — move GateRequired evaluation upstream to WorkflowExecutionCompletedHandler · XS · Low
- #642 — EngineStrategyResolver default strategy determinism · S · Low
- #643 — PLATFORM.md + garden protocol documentation · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #629 | WorkerRecoveryHealthCheck @Liveness → @Readiness | XS | Low | Flagged by trebreel |
| #636 | WorkerRuntime.spawnCase()/awaitCase() full implementation | M | Med | Filed this session |
| #637 | SequentialPlanningStrategy integration test fix | S | Med | Filed this session |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap |
| blocks#30 | AI routing strategy implementations | M | Med | Unblocked by #634 |
