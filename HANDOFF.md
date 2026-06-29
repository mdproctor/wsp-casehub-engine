# Handoff — 2026-06-29

## What's Done

**#578 + #579: migrate work-adapter from casehub-work to casehub-work-api — CLOSED**

Production dependency changed from `casehub-work` (full runtime) to `casehub-work-api` (SPI only). Observer migrated from `WorkItemLifecycleEvent` (runtime) to `WorkLifecycleEvent`/`WorkItemEvent` (api types). All `source()` → `WorkItem` casts eliminated. Template mode rewritten to use `WorkItemCreator.create()` with `templateId`. Adversarial design review completed (7 findings addressed). One casehub-work-api change: relaxed `WorkItemCreateRequest.build()` title validation when `templateId` is set. Garden entry: GE-20260629-59082a (InMemoryWorkItemStore tenancyId filtering gotcha).

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #461 | Composite WorkerExecutionManager for multi-worker co-deployment | M | Med | Not blocked, not urgent |
