---
title: "The Replay the Engine Ignored"
date: 2026-09-25
author: mdp
tags: [dlq, replay, idempotency, recovery, persistence, resilience]
entry_type: note
subtype: diary
projects: casehub-engine
---

The dead letter queue has a replay mechanism. Worker fails, retries exhaust, entry lands in the DLQ, admin reviews it, triggers replay. `DeadLetterReplayService` loads the case, finds the original scheduled event in the log, resolves the worker and capability from the definition, and dispatches a fresh `WorkerScheduleEvent`. The unit tests covered every guard rail — unknown ID, already replayed, already discarded, no event log, faulted case, success path. All green.

Nobody had tested whether the engine actually executes the replayed worker.

## The silent skip

The integration test told the story. Worker fails, case faults, DLQ entry appears — all correct. Admin unfaults the case, flips the worker to succeed, triggers replay. `DeadLetterReplayService` dispatches the event, marks the entry as REPLAYED. But the worker never runs. No error, no warning. The `WorkerScheduleEventHandler` logs "WorkerScheduleEvent processed" and moves on.

The cause was in `decideAction()`. The handler checks for existing `WORKER_SCHEDULED` events with the same input hash. The original scheduled event — from before the failure — is still in the event log. Same case, same worker, same input, same hash. The handler concludes the work was already scheduled and skips it.

The idempotency guard is doing exactly what it was designed to do. The replay mechanism just didn't tell it this was intentional re-execution.

## Two fields, one line each

The fix was `ExecutionMode.REINVOKED` on the schedule event. The handler already had the bypass:

```java
boolean isReinvoked =
    eventLog.getMetadata().has("executionMode")
        && "REINVOKED".equals(eventLog.getMetadata().get("executionMode").asText());
ScheduleAction action =
    isReinvoked ? ScheduleAction.createNew() : decideAction(existing, inputDataHash);
```

The replay service was constructing the event with the three-arg constructor — `new WorkerScheduleEvent(caseInstance, worker, capability)` — which sets `executionMode` to null. The full constructor with `ExecutionMode.REINVOKED` and a new `ExecutionOrigin.REPLAY` was the entire fix.

What makes this interesting isn't the fix — it's the gap. The unit test mocked `EventDispatcher`, so the dispatched event never reached the handler. The handler's idempotency logic never ran against a replayed event. The only way to find this was an integration test that starts a real case, lets it fault, recovers it, replays it, and checks whether the worker actually ran.

## The unfault problem

Writing the integration test surfaced a second gap. `WorkerRetriesExhaustedEventHandler` always transitions the case to FAULTED. FAULTED is terminal. `DeadLetterReplayService` rejects terminal cases. Every DLQ entry's case is FAULTED by the time anyone could replay it.

The first version of the test called `instance.setState(CaseStatus.RUNNING)` directly — a hack that works in-memory but has no production equivalent. So we built one: `CaseRecoveryService.unfault(UUID caseId)`. It validates the case is FAULTED, sets state to RUNNING synchronously, re-registers the completion tracker (the original `CompletableFuture` was already completed exceptionally), and dispatches `CaseStatusChanged` through the event bus so the handler persists the transition and re-evaluates bindings.

The binding re-evaluation is the subtle part. After unfaulting, `CaseContextChangedEvent` fires, bindings are re-evaluated, and the same worker would normally be re-dispatched. But the idempotency guard — the same one that was silently killing replays — now correctly blocks the duplicate. The admin's explicit DLQ replay, with `REINVOKED`, is the only path that gets through.

The idempotency guard went from being the bug to being the feature.

## Closing the epic

This was the last issue in the persistence coherence epic — eighteen issues across multiple sessions, from SPI naming conventions through contract tests through recovery strategy extraction to this DLQ integration test. The branch touched 151 files. The persistence layer has a consistent shape now: Repository for entity lifecycle, Store for value persistence, CrossTenant prefix for system-level access, contract tests that any implementation must pass.

The one thing I'd flag for the future: `SnapshotRecoveryStrategy.recover()` throws `IllegalStateException` when no snapshot exists. The design spec says it should fall back to event-log replay for cases created before the snapshot column existed. That's a migration-path concern — not urgent while there are no deployed instances, but it'll need addressing before the first one.
