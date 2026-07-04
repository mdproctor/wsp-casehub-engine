# Handoff — 2026-07-04

## What's Done

**#636 batch** — Six issues landed on one branch (`da237135` on main):
- **#636** — WorkerRuntime.spawnCase()/awaitCase() with CaseCompletionTracker, CaseTerminatedException, CaseDefinitionRegistry.findByName()
- **#637** — PlanItem AtomicReference + tryMarkRunning() CAS guard; merged filterAndIndexForDispatch
- **#641** — GateRequired CandidateSetStrategy evaluation moved upstream to WorkflowExecutionCompletedHandler
- **#642** — EngineStrategyResolver @DefaultBean detection via InjectableBean.isDefaultBean()
- **#629** — WorkerRecoveryHealthCheck @Liveness → @Readiness
- **#643** — PLATFORM.md routing strategy convention + garden protocol (cross-repo)

Design-reviewed (16 issues, 3 rounds). Follow-on issues filed: #646 (per-case CONTEXT_CHANGED serialization), #649 (PlanItem multi-source-state CAS loops).

## Cross-Module

**Consumer repos need routing strategy migration (#644):**
- Add `id()` to ActionRiskClassifier implementations: devtown, aml, clinical, life, soc, iot
- Add `id()` to TrustRoutingPolicyProvider implementations: devtown, aml, clinical, life, quarkmind, ops
- casehub-work#287 — retrofit WorkerSelectionStrategy, InstanceAssignmentStrategy, ClaimSlaPolicy

**Consumer repos still need capabilityNames migration (pre-existing):**
- casehub-life#47, casehub-aml#85, casehub-devtown#117, casehub-desiredstate#50, casehubio/parent#328

## What's Left

- #646 — per-case CONTEXT_CHANGED serialization (Option B) · M · Med
- #649 — PlanItem multi-source-state CAS loops · S · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | |
| #592 | External-backend recovery gap | M | Med | |
| blocks#30 | AI routing strategy implementations | M | Med | Unblocked by #634 |
