---
title: "Wiring workflow steps to the engine"
date: 2026-06-04
author: mdp
tags: [casehub-engine, quarkus-flow, serverless-workflow, architecture]
projects: casehub-engine
---

Engine#206 has been on the list for a while: when a `Worker` runs a Serverless Workflow, the steps inside it have no way to dispatch other casehub workers. The workflow executes in quarkus-flow's execution environment, which knows nothing about the engine. You can build a YAML workflow that does HTTP calls, listens for events, and branches on conditions — but you can't call `analyze-document` and wait for the engine to schedule it, run it, and come back with a result. Sequential orchestration via workflow was structurally blocked.

This session closed that gap, shipping `casehub-engine-flow`: a new optional module that bridges the two execution environments.

---

## What was built

The module adds six production classes. The most important are `FlowWorkerExecutor` (the correct `WorkflowApplication` singleton implementation, fixing two bugs in the old `ServerlessWorkflowExecutor`), `FlowExecutionRegistry` (a `ConcurrentHashMap` keyed by workflow instance ID that lets workflow steps look up their engine context mid-execution), and `CasehubDispatch` (the single dispatch entrypoint for both YAML and Java FuncDSL paths).

From a YAML workflow, dispatching a capability now looks like this:

```yaml
do:
  - analyzeDocument:
      call: casehub:dispatch
      with:
        capability: analyze-document
  - generateReport:
      call: casehub:dispatch
      with:
        capability: generate-report
```

The `call: casehub:dispatch` step is handled by `CasehubCallableTaskBuilder`, registered via Java SPI. Each dispatch returns a `CompletableFuture<WorkflowModel>` — quarkus-flow sequences the steps, Worker A's output lands in the case context, Worker B reads from there. The case context is the data bus between steps. The steps themselves are fully async — Quartz returns immediately when a `Worker(Workflow)` starts; success or failure comes back through the event bus.

`WorkOrchestrator` — previously a concrete class in runtime — is now an interface in `casehub-engine-common/spi/`. This is what lets the flow module inject it without taking a compile dependency on runtime. `casehub-engine-flow` depends on `casehub-engine-common` only.

---

## Three rounds of spec review before a line of code

The design for this was reviewed three times before implementation started — not as methodology, but because each round uncovered something that would have been expensive to fix later.

The first review caught that `WorkRequest.input` was being silently ignored by `WorkOrchestrator.doSubmit()`. The spec had a YAML `input:` parameter that did nothing. The data flow model was wrong: the spec implied data passed step-to-step through the workflow, but the implementation always re-evaluated from the case context via `inputSchema`. That's actually the right design — the case context is the bus — but the spec hadn't made that explicit. The `input:` parameter was removed and the data flow section was written.

The second review caught that `WorkOrchestrator` couldn't go in `api/spi/` as originally planned. It takes `CaseInstance` as a parameter. `CaseInstance` is in `casehub-engine-common/internal/`. Putting the interface in `api/spi/` would create a circular dependency: `api` ← `common` ← `api`. So it went in `common/spi/`, alongside the persistence SPIs.

The third review caught that `maybeRescheduleJob(JobExecutionContext)` couldn't be reused for the async failure path, because the async path has no `JobExecutionContext` — Quartz has already returned. The retry logic had to be extracted into a `maybeRescheduleWorker(WorkerRetryContext)` method that takes a plain value object. Both the synchronous path (building it from the Quartz context) and the async path (building it from the `WorkflowExecutionFailed` event) call the same method.

---

## What the SDK investigation found

Before writing a line of implementation, we read the quarkus-flow and serverlessworkflow SDK sources. That's where the surprises were.

The extension point for `call: casehub:dispatch` isn't `TaskExecutorFactory` — that replaces the entire factory chain and requires delegating all built-in task types yourself. The correct extension is `CallableTaskBuilder<CallFunction>`, discovered via `ServiceLoader`, which handles only `call:` steps. `DefaultTaskExecutorFactory` uses it automatically.

`CallFunction` and `FunctionArguments` are in `io.serverlessworkflow.api.types` — not in the `io.serverlessworkflow.api.types.func` experimental subpackage. The `.func` subpackage is for `CallJava` and the Java-function call types. Both jars are on the classpath, which makes the wrong import look plausible. It fails silently with a compile error that doesn't suggest the correct package.

The quarkus-flow extension already provides `WorkflowApplication` as a CDI singleton via its recorder. Adding a `@Produces @ApplicationScoped WorkflowApplication` producer causes an `AmbiguousResolutionException` at startup — the extension's registration is invisible to CDI scanning and the error message names the type but not which class caused the conflict. Claude flagged this the moment the integration test failed; I'd assumed we needed a producer and had written one.

The thread pool used by quarkus-flow's task executors is `Executors.newCachedThreadPool()` — not Vert.x IO threads. This matters because `WorkOrchestrator.submit()` calls `.await().indefinitely()` internally. Blocking is safe inside `CasehubCallableTaskBuilder.build()`'s returned `CallableTask`.

One more: `ServiceLoader.Provider.get()` caches the SPI instance in Java 9+. `CasehubCallableTaskBuilder.init()` and `build()` are called sequentially on the same thread, but if two different workflow definitions are loaded concurrently — both have a `call: casehub:dispatch` step — they'd race on any mutable instance field. The fix was a `ThreadLocal<String>` to carry the capability between `init()` and `build()`, with `remove()` in `build()` to prevent leaks.

---

## The result

One commit: 30 files, 1560 insertions, 85 deletions. Twenty-four unit tests and three CDI wiring tests, all green. The integration test confirms that when `casehub-engine-flow` is on the classpath, `FlowWorkerExecutor` wins over the `NoOpWorkflowExecutor @DefaultBean` fallback and both `CasehubDispatch` and `FlowExecutionRegistry` are injectable.

Sequential workflow orchestration is now a first-class execution model. Worker A runs, its output lands in the case context, Worker B reads from there. The workflow engine handles sequencing; the engine handles scheduling, routing, and result delivery.
