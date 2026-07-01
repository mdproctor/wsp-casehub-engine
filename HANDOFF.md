# Handoff — 2026-07-01

## What's Done

**This session closed 6 issues across 4 branches:**

- **#620, #621, #622** (branch `issue-620-signal-retry-perf-audit`): signalId retry threading, SequentialPlanningStrategy O(1) perf, bulk signal audit keys
- **#625** (branch `issue-625-trust-impl-routing`): TrustWeightedImplementationRoutingStrategy with configurable fallback binding
- **#549** (branch `issue-549-expires-at-expression`): expiresAtExpression — absolute deadline for humanTask WorkItems via ExpressionEngine.extractString()
- **#624** (branch `issue-624-groupstatus-isterminal`): GroupStatus.isTerminal()/isActive() + WorkItemLifecycleAdapter refactor
- **#627** (direct on main): neural-text → neocortex artifact/import rename in casehub-engine
- **#628** (hortora/engine): neural-text → neocortex rename in hortora garden engine

**Filed:** #629 — WorkerRecoveryHealthCheck should be @Readiness not @Liveness (flagged by trebreel — scaffold overlap)

## Cross-Module

**Consumer repos still need capabilityNames migration:**
- casehub-life#47, casehub-aml#85, casehub-devtown#117, casehub-desiredstate#50, casehubio/parent#328

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #629 | WorkerRecoveryHealthCheck @Liveness → @Readiness | XS | Low | Flagged by trebreel |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap |
| — | WorkerRuntime.spawnCase()/awaitCase() full implementation | M | Med | Event bus listener for CASE_STATUS_CHANGED |
| — | SequentialPlanningStrategy integration test fix | S | Med | Bindings fire concurrently in @QuarkusTest |
