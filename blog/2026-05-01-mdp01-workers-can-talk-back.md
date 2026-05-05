---
layout: post
title: "Workers can finally talk back"
date: 2026-05-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [casehub-engine, channels, quarkus]
---

The `CaseChannelProvider` SPI had `postToChannel` for a while. What it didn't
have was any mechanism for a worker to actually call it during execution. The
channel references existed in `WorkerContext`, but the field was always empty
and the context was never passed to the function that ran the worker. Workers
were handed a task, executed it, and had no way to report back mid-flight.

That gap is now closed.

## What the inventory said versus what the code said

The first thing I did was audit what was actually done versus what the issue
said was missing. The SPI methods were all there. The no-op defaults were there.
The contract tests were there. The integration test for `postToChannel` being
called when a worker was scheduled — there.

What wasn't there: any path from a running worker function to its case's channels.
`WorkerContext.channels` was a `List<CaseChannel>` that was always constructed as
empty, the `buildContext()` call in `WorkerScheduleEventHandler` discarded its
result, and `QuartzWorkerExecutionJob` — the class that actually invokes the
function — knew nothing about channels at all.

Three things needed to happen: `buildContext()` needed to know the case ID so it
could call `listChannels()`, the context needed to reach the function at execution
time, and workers needed a way to read it.

## The thread-local, and why it has to go inside the lambda

We added `UUID caseId` to `WorkerContextProvider.buildContext()`, wired
`EmptyWorkerContextProvider` to inject `CaseChannelProvider` and call
`listChannels(caseId)`, and created `WorkerExecutionContext` — a thin thread-local
holder that workers call to get their active context.

The integration test for it timed out. The worker's function ran but
`WorkerExecutionContext.current()` returned null.

Claude diagnosed the cause: `QuartzWorkerExecutionJob` runs the worker function
via `CompletableFuture.supplyAsync()`, which executes the lambda on a ForkJoinPool
thread. Setting the ThreadLocal on the Quartz thread before calling `supplyAsync`
does nothing — the pool thread has a completely separate ThreadLocal storage.

The fix is to set it as the first line *inside* the lambda:

```java
CompletableFuture.supplyAsync(() -> {
    WorkerExecutionContext.set(workerContext);
    try {
        return function.apply(inputData);
    } finally {
        WorkerExecutionContext.clear();
    }
});
```

Straightforward once you know why, but the symptom — a null context with no error —
would have taken a while to trace without the integration test catching it first.

## The commit that ate my staged files

While staging the feature files to commit, the working tree came back clean before
I ran `git commit`. The files had been committed — but under the wrong message,
absorbed into a background CI commit that IntelliJ's pre-commit hook was assembling
at the same time.

`git log` showed the feature work inside a commit titled "ci: use GH_PAT for
cross-repo repository_dispatch." Two resets and a force-push later, the history
was clean.

Workers can now call `WorkerExecutionContext.current().channels()` from inside
their function body and see every channel open for their case. The SPI is wired
end to end.
