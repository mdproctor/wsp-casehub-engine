# Handoff — 2026-07-17

## What's Done

- Typed context for WorkItem boundary (#689) — implemented, reviewed, squashed, PRed.
- CbrConfig temporalDecayHalfLifeDays (#733) — implemented, reviewed, squashed, PRed.
- Both landed on fork main; PR #744 updated to cover both.

## Immediate Next Step

Merge PR #744 after CI passes. Then open companion work repo PR for the typed WorkItem boundary (#689 work repo changes on branch `issue-689-workitem-typed-context`).

## Cross-Module

- Work repo: branch `issue-689-workitem-typed-context` has companion changes for #689 (pushed to fork). Needs a PR to casehubio/work.
- IoT: can now set `temporalDecayHalfLifeDays` in the `cbr:` YAML block (iot#64).

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
