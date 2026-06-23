# Handoff — 2026-06-23

## What's Done

**engine#561 — batch fix: all S/XS open issues**

Closed 7 issues in one branch (6 fixed, 1 closed as not-applicable):

- **#557** @PermitAll on ActorStateResource + ReactiveActorStateResource — unblocks devtown#90
- **#553** Unit test for expression engine create() type-mismatch IllegalStateException
- **#552** Null guard on evaluate(ExpressionEvaluator, JsonNode) for null asNode
- **#551** Cache Instance<ExpressionEngine> into immutable Map at @PostConstruct
- **#559** WorkerExecutionContext.set() in executeFlow() — unblocks aml#66
- **#547** deepCopyMap() in WritablePanelImpl constructor — unblocks devtown#15
- **#560** Closed as not-applicable — no CaseRetriever references in engine

4 commits, 756 tests green. Code review passed, squashed, pushed to main.

## Immediate Next Step

Pick up new work. All S/XS issues cleared.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #543 | Migrate Worker primitives to casehub-worker-api | L | High | Paused at stack depth 1 |
