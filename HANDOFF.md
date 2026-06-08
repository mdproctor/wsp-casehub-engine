# Handoff — 2026-06-08

**Head commit (engine):** eaa84398 — docs: promote design specs for registry lifecycle and hybrid execution
**Head commit (workspace):** pending commit after this session
**Engine on:** issue-413-sx-scale-batch
**Workspace on:** issue-413-sx-scale-batch

## What Changed This Session

**issue-274-registry-hydration-recovery closed.** Plans archived, EPIC-CLOSED.md stamped. All work (code, spec, blog) was already merged/promoted from a prior session.

**#413 (XS) — actor-state test gaps closed.** Two tests added:
- `ActorStateAggregatorTest.partialWriteContributor_partialDataVisible_sourceExcluded` — documents that partial accumulator writes before a throw are NOT rolled back (contributor responsibility, not engine enforcement).
- `QhorusActorStateContributorTest.deletedChannel_caseIdNull_noException` — covers deleted-channel race: caseId null, no NPE. Added package-private test constructor to `QhorusActorStateContributor`.

**#404 (S) — BlackboardRegistry lifecycle analysis.** Full design doc written at `docs/specs/issue-413-sx-scale-batch/2026-06-08-blackboard-registry-lifecycle-design.md`. Key findings:
- At WAITING, all active PlanItems are DELEGATED. `completionIndex` is not needed for WorkItem re-entry (routes via callerRef, not completionIndex). Evicting at WAITING is safe today — zero persistence changes needed.
- Strategy B (stateless-on-rest) recommended in two phases: Phase 1 (evict at WAITING, safe now), Phase 2 (persist RUNNING PlanItems, needs `workerName` in PlanItemRecord).
- LRU without Phase 2 is dangerous: silent data loss if a RUNNING case is evicted.

**#200 (S) — True hybrid execution design.** FlowWorker dispatch gap (`call: casehub:dispatch`) is already closed by `casehub-engine-flow`. Rules-driven deferred to Drools epic #445. Plan-based execution design written: `Worker(Plan.of(...))` as new Worker function type + `Plan.fromContext(".executionPlan")` for dynamic LLM-generated plans. Filed #448 (Worker(Plan) impl) and #449 (YAML plan: binding type).

**#187 (S) — WorkerCandidateSource future consideration. Closed as superseded.** WorkerRegistry never materialised; `WorkerProvisioner` SPI + `CaseDefinitionRegistry` + `QuartzWorkerExecutionManager` are already the separate candidate sources the issue anticipated.

## Immediate Next Step

Run `/work engine#289` — ExpressionEvaluatorFactory SPI. S · Low · first in the Drools chain.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (restart resilience) · M · Med
- engine#434: integration test for classifier-throws fail-safe · S · Low
- Branch `issue-413-sx-scale-batch` open — needs work-end when ready

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#289 | ExpressionEvaluatorFactory SPI | S | Low | First in Drools chain (#445) |
| engine#80 + #81 | Typed CaseContext panels | M | High | Prerequisite for WorkingMemoryBridge |
| engine#446 | WorkingMemoryBridge | M | Med | Depends on #80/#81 |
| engine#5 | DroolsExpressionEvaluator | M | Med | Depends on #289 |
| engine#207 | RulesRouter + RULES_DECISION lineage | L | Med | Final Drools piece — depends on #446 |
| engine#383 | Oversight response loop | M | Med | Unblocked |
| engine#384 | PlanItem escalation state | M | Med | Unblocked |
| engine#442 | Universal routing architecture design | L | High | Design-first; affects engine#439 |
| engine#448 | Worker(Plan.of(...)) function type | M | Med | Plan-based execution Phase 1 |

## Key References

- Branch: `issue-413-sx-scale-batch` (engine + workspace)
- Epic: https://github.com/casehubio/engine/issues/445 (Full Drools Integration)
- Registry lifecycle spec: `docs/specs/issue-413-sx-scale-batch/2026-06-08-blackboard-registry-lifecycle-design.md`
- Hybrid execution spec: `docs/specs/issue-413-sx-scale-batch/2026-06-08-hybrid-execution-design.md`
- #448: Worker(Plan.of(...)) implementation (filed this session)
- #449: YAML plan: binding type (filed this session, blocked on #448)
