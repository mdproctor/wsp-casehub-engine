---
title: "When Workers Won't Shut Up: Intermediate Output in Long-Running Scopes"
date: 2026-08-02
author: Mark Proctor
tags: [casehub, engine, lifecycle-scopes, architecture]
---

Most workflow engines treat worker execution as a simple request-response: dispatch a task, get a result, apply it, move on. That model works until you need a worker that stays alive for the duration of a compound or an entire case — monitoring a data feed, maintaining a conversation, accumulating state across multiple context changes. These workers produce output *while they're still running*, and the engine needs somewhere to put it.

CaseHub's lifecycle scopes introduce three execution modes: TRANSIENT (the classic fire-and-forget), REINVOKED (re-invoked on each trigger with accumulated state), and PERSISTENT (a long-running virtual thread with a mailbox). The first mode had full output application wiring from day one. The other two were silently discarding their output.

## The silent discard

The existing code in `QuartzWorkerExecutionJob.onSuccess()` had a guard for non-TRANSIENT workers returning `Success` (as opposed to `Completed`, which signals "I'm done, complete my PlanItem"). The guard published to an event bus address that nothing consumed, carrying only a case ID and binding name — no actual output data. The worker's intermediate results vanished.

The fix wasn't as simple as routing through the existing completion handler. `WorkflowExecutionCompletedHandler` does too much for interim output: it records episodic "COMPLETED" status, fires `workerStatusListener.onWorkerCompleted()`, records routing outcomes, and triggers signal settlement. All of that assumes the worker is *finished*. For a scoped worker returning intermediate results, you want the output applied and downstream bindings re-evaluated — nothing more.

## Two paths, one applier

We built a dedicated `ScopedWorkerOutputHandler` that does exactly three things: apply output to the case context with conflict resolution, write an EventLog entry, and publish `CONTEXT_CHANGED`. No episodic updates, no lifecycle events, no settlement tracking.

The conflict resolution piece was interesting. When two workers — or two invocations of the same REINVOKED worker — produce output concurrently, per-key conflict resolution prevents lost updates. We extracted a `ContextOutputApplier` service that serializes output application per case via a `ReentrantLock`:

```java
public JsonNode apply(CaseInstance instance, Map<String, Object> output, String bindingName) {
    ReentrantLock lock = locks.computeIfAbsent(instance.getUuid(), k -> new ReentrantLock());
    lock.lock();
    try {
        JsonNode contextBefore = instance.getCaseContext().snapshot().asJsonNode();
        Binding binding = findBindingByName(instance, bindingName);
        String strategy = binding != null ? binding.getConflictResolverStrategy() : null;
        // ... per-key application with ConflictResolver.resolve() ...
        JsonNode contextAfter = instance.getCaseContext().asJsonNode();
        return contextDiffStrategy.compute(contextBefore, contextAfter);
    } finally {
        lock.unlock();
    }
}
```

`ReentrantLock` is virtual-thread-friendly — it unmounts the carrier thread on contention rather than pinning (JEP 444). The lock map is evicted when the case reaches a terminal state, following the same pattern as `CaseEvaluationSerializer`.

We deliberately chose `ReentrantLock` over `CaseEvaluationSerializer` here. The serializer uses a coalescing model — a single pending evaluator per case, overwritten on each new submission. That's correct for stateless re-evaluation of the latest context, but it would silently drop concurrent scoped outputs. Each output carries unique data that must be applied.

## The guard that almost missed

The original dispatch guard used `!(workerResult.outcome() instanceof WorkerOutcome.Completed)` — meaning any non-Completed outcome would be treated as interim output. That includes `Declined`, `Failed`, and `Expired`, which need to reach `WorkflowExecutionCompletedHandler.handleSemanticFailure()` for proper outcome policy handling, rerouting, and PlanItem lifecycle management.

The fix: `workerResult.outcome() instanceof WorkerOutcome.Success`. Only genuine interim success output takes the scoped path. Everything else falls through to the normal completion handler.

## Why this matters beyond CaseHub

The pattern is general. Any system with long-running processes that produce intermediate results needs to solve the same problem: how do you apply output from an ongoing process without triggering the completion machinery? The answer is a lighter-weight output application path — same conflict resolution, same downstream re-evaluation, but none of the lifecycle side-effects that assume the process is finished.

The `ContextOutputApplier` extraction also sets up a future consolidation. `WorkflowExecutionCompletedHandler` currently duplicates the same output application logic — binding lookup, strategy resolution, per-key `ConflictResolver` calls. A follow-on refactors it to delegate to the shared applier, eliminating the duplication and ensuring both paths serialize through the same per-case lock.
