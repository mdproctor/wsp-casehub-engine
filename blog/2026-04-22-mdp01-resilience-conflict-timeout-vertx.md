---
layout: post
title: "casehub-resilience: conflict resolution, a timeout enforcer, and a Vert.x surprise"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [resilience, quarkus, vertx, eventbus]
---

The resilience module is open for review. The interesting parts weren't where I expected them.

I had three new things to build: `ConflictResolver` (a strategy for deciding which value wins when two workers write to the same `CaseContext` key), `CaseTimeoutEnforcer` (a scheduled scanner that faults RUNNING cases that have been running too long), and integration tests for both. The poison pill quarantine and dead letter queue were already there — treblereel had implemented them before I started.

Claude and I started with `ConflictResolver`. Three strategies: last writer wins (the default), first writer wins, and fail-on-conflict. The design constraint was that `Binding` lives in `api/` — a module with no dependency on `casehub-resilience`. Storing a `ConflictResolver` instance in `Binding` would create a cycle. The solution is to store a String strategy name instead, and resolve it in `WorkflowExecutionCompletedHandler` with an inline switch:

```java
private Object applyStrategy(String strategy, String key, Object existing, Object incoming) {
    if (strategy == null) return incoming;
    return switch (strategy) {
        case "FIRST_WRITER_WINS" -> existing;
        case "FAIL" -> throw new IllegalStateException(
            "Conflicting writes to key '" + key + "' — binding uses FAIL strategy");
        default -> {
            LOG.warnf("Unknown strategy '%s' for key '%s' — LAST_WRITER_WINS", strategy, key);
            yield incoming;
        }
    };
}
```

No cross-module import. The engine never touches `casehub-resilience` at the type level.

`CaseTimeoutEnforcer` had a false start. The natural approach was to consume `CASE_STARTED` events to record when a case began running. We added a `@ConsumeEvent(EventBusAddresses.CASE_STARTED)` to the enforcer — and the engine started dropping roughly half its case transitions silently, with no error logged.

The cause: `CaseStartedEventHandler` uses `eventBus.request()`, not `eventBus.publish()`. Point-to-point delivery. Two `@ConsumeEvent` beans on the same address created two competing consumers, and Vert.x distributed requests round-robin. No documentation mentions this. The fix was to drop the event listener entirely and record start times lazily in the scan loop — on first observation of a RUNNING case, store `Instant.now()`.

The integration with the persistence layer was wrong on the first pass too. The enforcer was calling `instance.setState(FAULTED)` directly and publishing `CASE_FAULTED` to the event bus. That bypasses the engine's normal fault path — which calls `updateStateAndAppendEvent()` to persist the state change, then publishes `CASE_STATUS_CHANGED` (not `CASE_FAULTED`) to trigger the downstream handler. The direct publish left no `EventLog` row and left the database inconsistent with the in-memory cache. We fixed it by matching the pattern in `WorkerRetriesExhaustedEventHandler`: persist first, then publish `CASE_STATUS_CHANGED`.

One other thing worth noting: `PropagationContext` — the budget-tracking value object — is still waiting in a separate PR. The enforcer currently uses elapsed wall-clock time instead of `propagation.isBudgetExhausted()`. It'll be a one-line upgrade when that PR merges.

The PR is open. Blackboard PRs #88 and #89 merged while we were working, so #90 is next in the stack.
