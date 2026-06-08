# Handoff — 2026-06-08

**Head commit (engine):** eaa84398 — docs: promote design specs for registry lifecycle and hybrid execution
**Head commit (workspace):** 649210b — feat: promote blog from issue-413-sx-scale-batch
**Both repos on:** main
**PR:** https://github.com/casehubio/engine/pull/451 (open — actor-state tests + design specs)

## What Changed This Session

**issue-274-registry-hydration-recovery closed.** Plans archived, EPIC-CLOSED.md stamped.

**S/XS batch (issue-413-sx-scale-batch) closed.** PR #451 open.
- #413 (XS): actor-state test gaps — partial-write contributor contract + deleted-channel race. Package-private test constructor added to QhorusActorStateContributor.
- #404 (S): BlackboardRegistry lifecycle analysis. Key finding: at WAITING state, WorkItemLifecycleAdapter routes via callerRef (not completionIndex). Phase 1 stateless-on-rest eviction is safe today — zero persistence changes.
- #200 (S): Hybrid execution design. FlowWorker gap already closed by casehub-engine-flow. Designed Worker(Plan.of(...)) + Plan.fromContext() for plan-based execution. Filed #448/#449.
- #187 (S): Closed as superseded — WorkerRegistry never materialised, WorkerProvisioner SPI is the right abstraction.

## Immediate Next Step

Run `/work engine#289` — ExpressionEvaluatorFactory SPI. S · Low · first in the Drools chain.

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- engine#433: persist `pendingActionGate` in `CaseInstanceEntity` (restart resilience) · M · Med
- engine#434: integration test for classifier-throws fail-safe · S · Low
- PR #451: awaiting review/merge

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

- PR: https://github.com/casehubio/engine/pull/451
- Epic: https://github.com/casehubio/engine/issues/445 (Full Drools Integration)
- Registry lifecycle spec: `docs/specs/issue-413-sx-scale-batch/2026-06-08-blackboard-registry-lifecycle-design.md`
- Hybrid execution spec: `docs/specs/issue-413-sx-scale-batch/2026-06-08-hybrid-execution-design.md`
