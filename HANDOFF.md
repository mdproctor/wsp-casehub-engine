# Handoff — 2026-07-01

## What's Done

**#620, #621, #622 — Hybrid orchestration follow-ups (branch closed)**

Three follow-up fixes from the #490 epic, all on one branch:

- **signalId retry threading (#620):** `signalAndAwait()` no longer hangs when workers throw and retries exhaust. signalId threads through `WorkerRetryContext`, Quartz JobDataMap, `WorkerRetriesExhaustedEvent`, and handler `recordCompletion()`. Guard quarantine path also covered. Fixed `WorkerRetriesExhaustedEvent` field ordering (tenancyId at position 2 per protocol).

- **SequentialPlanningStrategy perf (#621):** `select()` replaced O(n) `getAllPlanItems()` + stream-to-map with O(1) `findPlanItemByBindingName()` per binding. New default method on `CasePlanModel`. `activeByBinding` renamed to `latestByBinding`.

- **Bulk signal audit (#622):** Event log payload now includes full updates map; metadata includes `updatedKeys` array. `SignalReceivedEventHandler` converted to constructor injection.

## Cross-Module

**Consumer repos still need capabilityNames migration** (from prior session):
- casehub-life#47, casehub-aml#85, casehub-devtown#117, casehub-desiredstate#50, casehubio/parent#328

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #582 | Generalize GoalBasedCompletion beyond success/failure | M | Med | Follow-on from #581 |
| #592 | External-backend recovery gap | M | Med | Pre-existing gap |
| — | WorkerRuntime.spawnCase()/awaitCase() full implementation | M | Med | Event bus listener for CASE_STATUS_CHANGED |
| — | WorkflowPlanningStrategy (Tier 3) | L | High | SW-backed durable orchestration — future |
| — | SequentialPlanningStrategy integration test fix | S | Med | Bindings fire concurrently in @QuarkusTest — unit tests pass |
