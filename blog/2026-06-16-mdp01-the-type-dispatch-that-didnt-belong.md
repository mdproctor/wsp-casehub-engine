---
layout: post
title: "The Type Dispatch That Didn't Belong"
date: 2026-06-16
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [architecture, spi, quartz, worker-execution]
---

## The Type Dispatch That Didn't Belong

The worker execution path in CaseHub's Quartz scheduler had a structural problem: `QuartzWorkerExecutionJob` knew too much. It resolved worker functions by type — sync, agent, flow — applied output schemas, handled timeouts, managed failure events, and published completion. The Quartz adapter was also the execution engine.

That conflation meant two things couldn't happen: replacing Quartz with another scheduler without duplicating all the execution logic, and testing the execution path without a Quartz context. Both matter more than they used to now that the worker model is growing.

The fix was a clean vertical cut. `WorkerExecutor` is a new SPI in `common/internal/executor/` — it takes a `WorkerFunction` (the sealed type we introduced earlier on this branch), input data, a worker context, timeout, output schema, and lineage metadata. Returns `Uni<WorkerResult>`. The sealed switch over `Sync`, `AgentExec`, and `Flow` happens exactly once, inside `DefaultWorkerExecutor` in the runtime module. Sync and agent functions run on Quarkus-managed virtual threads with timeout enforcement; flow workers delegate to `WorkflowExecutor` with their own internal timeout model.

`QuartzWorkerExecutionJob` is now a thin adapter: resolve the EventLog, load the case instance and definition, build the worker context, call `workerExecutor.execute()`, subscribe fire-and-forget. Success publishes `WORKER_EXECUTION_FINISHED` with PlannedAction enrichment. Failure routes to `QuartzRetryService`.

That retry service is the other extraction. The listener previously owned retry logic for both the synchronous path (`jobWasExecuted` catching `JobExecutionException`) and the asynchronous path (consuming `WORKFLOW_EXECUTION_FAILED` events over the Vert.x event bus). Two codepaths doing the same thing — persist the failure event log, resolve the retry policy, count prior failures, decide retry or exhaust. `QuartzRetryService` collapses both into a single `Uni<Void> handleFailure()` chain that uses `RetryPolicies.evaluate()` for the decision.

With both paths converging on the retry service, `WORKFLOW_EXECUTION_FAILED` has no consumers left. The event bus address and the `WorkflowExecutionFailed` record are deleted. One fewer async hop in the failure path.

`WorkerExecutionConfig` moved from `scheduler-quartz` to `common/internal/executor/` — the default timeout is a shared concern, not a scheduler concern. `RetryPolicies` and `RetryDecision` (the sealed retry/exhaust type) were already there from earlier on this branch.

The Quartz listener simplifies to `jobToBeExecuted()` only — fire lifecycle events and persist the start event log. `jobWasExecuted()` is now a no-op because the fire-and-forget job never throws; errors are handled asynchronously by the retry service.

What makes this interesting architecturally: the sealed `WorkerFunction` hierarchy (from earlier in this branch) and the `WorkerExecutor` SPI together mean adding a new scheduler is a matter of writing a thin adapter that calls `execute()`. No type dispatch, no output schema evaluation, no retry logic to duplicate. The scheduler just resolves context and subscribes.
