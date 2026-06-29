# Handoff — 2026-06-29

## What's Done

**#581: close epic #84 — Milestone, Goal, Stage CMMN alignment — CLOSED**

Removed `Goal.terminal` (goals are always terminal — non-terminal checkpoints are Milestones). Added `Milestone.parentStageId` for programmatic stage containment. Upgraded `CasePlanModel` from `Boolean` to `MilestoneLifecycleStatus` with atomic transition guards. Consolidated milestone evaluation to single path (`MilestoneLifecycleManager`), removing the duplicate `CaseContextChangedEventHandler.milestones()` path. Added registration-time WARNING for unreferenced goals. Created follow-on #582 (generalize GoalBasedCompletion). Epic #84 is now closeable.

## What's Left

- Consumer repo import migration (aml#69, devtown#96) · S · Low each

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| aml#69 | Propagate worker-api imports to aml | S | Low | Mechanical import swap |
| devtown#96 | Propagate worker-api imports to devtown | S | Low | Mechanical import swap |
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #578 | Migrate work-adapter from casehub-work to casehub-work-api | M | Med | casehub-work#275 SPI extraction |
| #579 | Migrate WorkItemLifecycleAdapter from source() to workItem() | S | Low | casehub-work#278 typed events |
