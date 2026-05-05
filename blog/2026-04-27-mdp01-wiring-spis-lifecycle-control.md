---
layout: post
title: "Wiring the SPIs and adding lifecycle control"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [quarkus, cdi, spi, lifecycle]
---

The Worker Provisioner SPIs — `WorkerStatusListener`, `WorkerContextProvider`, `CaseChannelProvider` — had been defined with contract tests and no-op defaults. Nothing called them.

Wiring them in was mechanical in places but the call sites matter. `onWorkerStarted` goes in `WorkerExecutionJobListener.jobToBeExecuted()` — right when Quartz is about to fire the job, before execution begins. `onWorkerCompleted` in `WorkflowExecutionCompletedHandler` after output is applied to the context. `onWorkerStalled` in `WorkerRetriesExhaustedEventHandler`. `buildContext()` is called in `WorkerScheduleEventHandler` before scheduling — implementations can prepare worker context before the worker touches the job. `openChannel` fires on case start; `listChannels` plus `closeChannel` fire when the case reaches a terminal state.

For testing, we used `@Alternative @Priority(1)` static inner classes in `@QuarkusTest` — recording implementations that track invocation arguments, reset in `@BeforeEach`. No Mockito, no test profiles. In CDI 4.0, `@Priority(1)` activates the alternative globally across the entire test suite — the same behaviour that causes `AmbiguousResolutionException` when you do it accidentally becomes an asset when you do it deliberately with recording state that doesn't change observable behaviour.

`cancelCase`, `suspendCase`, and `resumeCase` are now on `CaseHub` and `CaseHubRuntime`. State validation lives in `CaseHubReactor` — terminal states throw `IllegalStateException`, wrong-source transitions throw `IllegalStateException`, unknown caseIds throw `IllegalArgumentException`. Publishing to `CASE_STATUS_CHANGED` handles the rest; the existing handler takes care of persistence, scheduler cancellation, and channel cleanup.

Resume had one wrinkle. When a case transitions `SUSPENDED → RUNNING`, the engine doesn't automatically know to re-evaluate which workers are eligible — it waits for a context change event. We added a `CONTEXT_CHANGED` publish in `CaseStatusChangedHandler` for that specific transition:

```java
if (newState == CaseStatus.RUNNING) {
    eventBus.publish(
        EventBusAddresses.CONTEXT_CHANGED,
        new CaseContextChangedEvent(caseInstance, caseInstance.getCaseContext().asJsonNode()));
}
```

One line. Without it, resume would be silent — the case moves to RUNNING but nothing fires until the next signal arrives. With it, any binding whose condition is already satisfied in the current context fires immediately.
