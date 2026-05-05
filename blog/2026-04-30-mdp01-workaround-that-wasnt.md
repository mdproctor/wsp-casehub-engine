---
layout: post
title: "The Workaround That Wasn't"
date: 2026-04-30
type: pivot
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [serverlessworkflow, quarkus-flow, upstream]
excerpt: "We submitted a PR to fix a null output bug in sdk-java. The maintainer closed it: it wasn't a bug."
---

When writing behaviour-documenting tests for `WorkflowExecutionListener` in
quarkus-flow, we kept hitting a strange failure. The Q2 test was checking whether
`onWorkflowCompleted` carries output — `completedOutput` was null after `await()`
returned, even though the hook appeared to have fired.

Claude diagnosed it as a bug in sdk-java. `instanceData().output()` routes through
`futureRef.get().join()`, and `futureRef` is assigned only after `startExecution()`
returns. For synchronous workflows the CompletableFuture chain completes inline before
that assignment, so the join returns null. Plausible. We submitted a PR (#1356).

Javi, the sdk-java maintainer, closed it with a correction: `instanceData().output()`
is intentionally a blocking join for callers outside the event chain — people who hold
a `WorkflowInstance` after `start()` completes. Inside a listener callback, you use
`event.output()`. It's a first-class field on `WorkflowCompletedEvent`, populated
directly from the task result before the event fires.

The irony: we had read `WorkflowCompletedEvent.java` early in the investigation. Claude
saw `event.output()` and documented it as "the workaround." It was never a workaround.
It was the API.

What made the wrong diagnosis stick was the NPE. Calling `.asMap()` on a null
`WorkflowModel` threw inside the listener — but `LifecycleEventsUtils.publishEvent()`
catches listener exceptions and attaches them as suppressed exceptions on the
CompletableFuture. The workflow completes normally. `completedOutput` was null not
because the hook didn't fire, but because the NPE prevented the `set()` call from
being reached. Silent failure masking as missing behaviour.

The fix was two lines:

```java
// Wrong — NPE, silently swallowed by the event publisher:
var outputModel = event.workflowContext().instanceData().output();

// Right — always populated before the event fires:
var outputModel = event.output();
```

We updated quarkus-flow PR #508, corrected the Q2 test and its javadoc, and closed
#1356 with an explanation. Javi also raised sdk-java #1357 — moving `output()` off
`WorkflowInstanceData` entirely so the confusing API surface is gone. We submitted
#1359 for that; he'd already done it himself.

Separately, #186 landed today: `WorkerScheduleEventHandler` now opens a worker-specific
channel and posts a Qhorus COMMAND after submitting a worker execution. The missing
piece that was bypassing the entire normative obligation lifecycle for every
CaseHub-orchestrated work unit.
