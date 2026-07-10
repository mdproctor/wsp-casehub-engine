# HANDOFF — engine#700 orchestration unification

**Branch:** `issue-700-unify-orchestration-model`
**Issue:** casehubio/engine#700
**Status:** Phase 1 production code ~80% complete, uncommitted

## What Was Done

Designed and began implementing the shared orchestration type foundation for engine#700 — unifying engine's blackboard model with blocks' agentic patterns. Full gap analysis of engine#101 (superseded), engine#694-698 (fold into #700). Cross-repo blocker analysis written to `docs/cross-repo-blockers.md`.

**New shared types created (engine-api, uncommitted):**
- `OutcomeKind` — shared outcome taxonomy (SUCCESS/DECLINED/FAILED/EXPIRED/ESCALATED) + tests
- `ExecutorRef` / `SimpleExecutorRef` — shared executor identity interface + tests
- `RoutingResult` / `Assignment` — unified routing result replacing `AgentAssignment`

**Production code updated (uncommitted):**
- `PlanItem` — gained `description` field, threaded through `PlanItemRecord`, `PlanItemSaveRequest`, all persistence stores, `PlanItemRestorer`
- `AgentRoutingStrategy.select()` — return type changed from `Uni<AgentAssignment>` to `Uni<RoutingResult>`
- All 6 `AgentRoutingStrategy` implementations migrated
- All production callers migrated: `CaseContextChangedEventHandler`, `DefaultWorkOrchestrator`, `TrustWeightedImplementationRoutingStrategy`, `TrustCandidateClassifier`
- Blocks `RoutingSupport.TrustFilterOutcome.Decided` — now carries `RoutingResult`

**Production compiles clean.** 0 production errors, 26 test errors (mechanical `AgentAssignment` → `RoutingResult` migration).

## Immediate Next Step

Migrate remaining test files from `AgentAssignment` to `RoutingResult` (~8 test files, mechanical). Then commit, continue with Phase 1E (FailurePolicy) and Phase 2 (async BlackboardPlanConfigurer).

## What's Left (this branch)

- Test migration for RoutingResult · S · Low
- Delete `AgentAssignment.java` · XS · Low
- Phase 1E: Unified FailurePolicy · M · Med
- Phase 1F: Remove duplicated types · S · Low
- Phase 2: Async BlackboardPlanConfigurer · S · Med
- Phase 5: Issue management · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 3: Blocks-side alignment | L | Med | Separate blocks branch |
| — | Phase 4: Peer architecture | XL | High | Depends on Phase 3 |

## References

- Plan: `~/.claude/plans/proud-dancing-hartmanis.md`
- Cross-repo blockers: `docs/cross-repo-blockers.md`
