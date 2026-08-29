# Handover — issue-995-unified-judgment-target

**Branch:** `issue-995-unified-judgment-target`
**Covers:** #995, #999, #1000, #957
**Date:** 2026-08-29

## What happened

This session completed Batches 1-3 of the unified JudgmentTarget plan:

1. **#995 Task 1 (test migration) — DONE.** All test files migrated from `HumanTaskTarget` (as BindingTarget) to `JudgmentTarget` with `HumanRoutingConfig`. Planning module (571 tests), runtime dispatch (8 tests), CBR retain (21 tests) all pass.

2. **#995 Task 2 (DelegatingJudgmentScheduler) — DONE.** `DelegatingJudgmentScheduler` (`@DefaultBean`) bridges `JudgmentScheduleRequest` → `HumanTaskScheduleRequest` when `routingConfig instanceof HumanRoutingConfig`. `publishJudgmentSchedule()` expanded to handle human routing: candidate resolution, `HumanTaskRoutingStrategy`, bridge validation, title/scope expression resolution. `NoOpJudgmentScheduler` deleted. Dead code removed: `publishHumanTaskSchedule()`, `evaluateInputMapping()`, `resolveExpiresAtDeadline(HumanTaskTarget)`. `JudgmentScheduleRequest` gains 8 fields.

3. **#999 Task 3 (JudgmentEscalator SPI) — DONE.** SPI types: `JudgmentEscalator extends NamedStrategy`, `EscalationDecision` sealed (ReYield, RouteHigher, Fault), `EscalationContext` record. Built-in strategies: `FaultEscalator` (`@DefaultBean`, id="fault"), `ReYieldEscalator` (id="re-yield"). `CaseDefinition.maxEscalations` added. `EngineStrategyResolver` wired. `JudgmentEscalationHandler` resolves escalator, counts prior events, executes decision, writes outcome to EventLog metadata.

4. **Pre-existing fix: Confidence type change.** Adapted `CbrRetrievalService` and `AgentExperienceRecorder` to neocortex `Confidence` record (was `Double`). 15 test files updated.

## Key decisions

**D1: EscalationContext avoids JudgmentResponse dependency.** `EscalationContext` in `api/spi/judgment/` can't reference `JudgmentResponse` (in `common/spi/`) — `api` doesn't depend on `common`. Solution: decompose response fields (decision, evidence, callerId, callerType) into the context record.

**D2: Dispatch tests assert on JudgmentScheduleRequest, not HumanTaskScheduleRequest.** `RecordingJudgmentScheduler` (non-DefaultBean) displaces `DelegatingJudgmentScheduler` in tests. Tests verify the engine's unified dispatch, not the downstream delegation. `DelegatingJudgmentScheduler` is tested separately.

## Resume from

**Remaining on this branch (Tasks 4-5):**
- Task 4: `JudgmentNodeExecutor` + SWF `casehub:judgment` callable (#1000) — blocking executor for DAG threads, CompletableFuture per pending judgment, handler wiring
- Task 5: React module integration tests (#957) — two @QuarkusTest classes

**Handler execution paths not yet wired:**
- ReYield: PlanItem `tryMarkReDispatching()` (DELEGATED → DISPATCHING), re-publish judgment request with feedback — needs `JudgmentScheduler` + `BlackboardRegistry` injection
- RouteHigher: same PlanItem transition, elevated trust threshold
- Fault: mark PlanItem FAULTED, write `_diagnostics`

**Cross-repo work slot** needed after engine changes land — 8 repos in one IntelliJ workspace for semantic refactoring.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-995-unified-judgment-target/2026-08-29-unified-judgment-target-design.md` |
| Decisions | `specs/issue-995-unified-judgment-target/decisions.md` |
| Plan | `plans/2026-08-29-unified-judgment-target.md` |
| .plan | `.plan` (queue position 0/1, active #995) |
