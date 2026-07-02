# Handoff — 2026-07-02

## What's Done

**This session closed 2 issues:**

- **#609** (branch `issue-609-agent-subsumption-matching`): AgentCandidateFactory subsumption matching via VocabularyRegistry — CDI conversion, two-tier matching, NoOpVocabularyRegistry @DefaultBean
- **#528** (closed as already resolved): WorkerFunction.Flow extraction — confirmed fully resolved by #567

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
