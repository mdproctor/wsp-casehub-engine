# Handoff — 2026-07-17

## What's Done

Typed context for WorkItem boundary (#689) — implemented, reviewed, squashed, PRed.

- Engine: HumanTaskTarget gains payloadType/resolutionType. Bridge validation on input (initialise) and output (deserialise). BridgeResolver.resolveByTypeNameStrict(). HumanTaskScheduleEvent carries type names. Design spec with 3-round adversarial review. CLAUDE.md updated.
- Work repo: WorkItemRef, WorkItemCreateRequest, WorkItem entity gain type name fields. Flyway V10. HumanTaskScheduleHandler passes type names through. PlanItemCompletionApplier validates resolution before PlanItem completion, writes workItemValidationFailed signal on failure.
- Engine PR: casehubio/engine/pull/744
- Follow-up issues filed: engine#740 (linked data reference protocol), engine#742 (ActionGate resolutionTypeName)

## Immediate Next Step

Merge engine PR #744 after CI passes. Then open companion work repo PR for the typed WorkItem boundary.

## Cross-Module

- Work repo: branch `issue-689-workitem-typed-context` has companion changes (pushed to fork). Needs a PR to casehubio/work.
- AML, clinical, devtown: can now declare payloadType/resolutionType on HumanTask bindings for compile-time safety.

## What's Left

- Engine PR #744 needs merge · XS · Low
- Work repo companion PR needs creation and merge · XS · Low
- engine#742: ActionGate resolutionTypeName threading · S · Low
- engine#740: linked data reference protocol (platform-wide) · L · High

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Rewrite unified execution model spec (resolve contradictions) | M | Med | From prior session |
| — | Phase 0: sequentialMerge() on DagPlan | S | Low | Blocks ExecutionPlan retirement |
| #732 | Wire CaseContextStoreFactory through recovery path | M | Med | Blocks durable factories |
| #740 | Linked data reference protocol | L | High | Platform-wide, ContextBridge arc |
| #600 | HTN — hierarchical task decomposition | L | High | Under #595 epic |
