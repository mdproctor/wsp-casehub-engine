# Handover — issue-995-unified-judgment-target

**Branch:** `issue-995-unified-judgment-target`
**Covers:** #995, #999, #1000, #957
**Date:** 2026-08-30

## What happened

All 5 batches of the unified JudgmentTarget plan are complete:

1. **#995 Task 1 (unified types) — DONE.** JudgmentTarget expanded with RoutingConfig, HumanTaskTarget stripped from BindingTarget, YAML parsing updated, all switch/instanceof sites migrated.

2. **#995 Task 2 (handler unification) — DONE.** DelegatingJudgmentScheduler bridges JudgmentScheduleRequest → HumanTaskScheduleRequest. publishJudgmentSchedule() handles human routing (candidates, routing strategy, bridge validation).

3. **#999 Task 3 (escalator SPI) — DONE.** JudgmentEscalator NamedStrategy with FaultEscalator (@DefaultBean) and ReYieldEscalator. JudgmentEscalationHandler resolves escalator and executes decisions.

4. **#1000 Task 4 (DAG integration) — DONE this session.**
   - `JudgmentNodeResult` sealed type (Completed, ReYielded, Faulted) in common/spi
   - `JudgmentNodeExecutor` (@ApplicationScoped, common) — blocking await with per-cycle timeout, BlockingQueue per pending judgment
   - `JudgmentTimeoutException` and `JudgmentFaultException`
   - `PlanItem.tryMarkReDispatching()` — CAS DELEGATED → DISPATCHING
   - Handler execution paths wired: JudgmentEscalationHandler publishes JUDGMENT_RE_DISPATCH / JUDGMENT_FAULT events and enqueues to JudgmentNodeExecutor
   - JudgmentCompletedHandler enqueues Completed/Faulted to JudgmentNodeExecutor (outside instanceof JudgmentTarget for SWF support)
   - `JudgmentPlanItemHandler` (planning) consumes events: tryMarkReDispatching + JudgmentScheduler.schedule() for re-yield, markFaulted() for fault
   - `CasehubJudgment` (flow) — SWF callable for `casehub:judgment`, delegates to JudgmentNodeExecutor

5. **#957 Task 5 (react integration tests) — DONE this session.**
   - `ReActExecutionIntegrationTest` — full end-to-end: case with react worker, StubChatModel (tool-use then final answer), case reaches COMPLETED
   - `ReActAuditTrailTest` — verifies REACT_CYCLE EventLog entries (cycleIndex, toolCalls) and WORKER_EXECUTION_COMPLETED metadata (reactCycleCount)
   - Tests hosted in planning module (integration test hub) with casehub-engine-react as test dependency
   - Fixed react module test config: MemoryPlanItemStore → InMemoryPlanItemStore, added missing exclusions

## Key decisions

**D1: JudgmentNodeExecutor in common, not runtime.** The flow module (which has CasehubJudgment callable) depends on common, not runtime. Placing the executor in common allows both runtime handlers and flow callable to access it.

**D2: Handler execution via event bus.** JudgmentEscalationHandler (runtime) publishes JUDGMENT_RE_DISPATCH / JUDGMENT_FAULT events consumed by JudgmentPlanItemHandler (planning). This follows the existing cross-module communication pattern — runtime publishes events, planning manages PlanItems.

**D3: React tests in planning module.** The react module's @QuarkusTest infrastructure has a pre-existing issue (MemoryPlanItemStore typo causing silent PlanItem loss). Tests hosted in the planning module where the integration test infrastructure is proven.

## Ready for work-end

All batches complete. The branch is ready for review, squash, and close. Cross-repo migration (work slot for 8 repos) is tracked separately and deferred until after this branch lands.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-995-unified-judgment-target/2026-08-29-unified-judgment-target-design.md` |
| Decisions | `specs/issue-995-unified-judgment-target/decisions.md` |
| Plan | `plans/2026-08-29-unified-judgment-target.md` |
| .plan | `.plan` (all 5 batches done) |
