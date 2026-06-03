# FlowWorker ↔ WorkOrchestrator Bridge

**Issue:** casehubio/engine#206  
**Branch:** issue-206-flowworker-bridge  
**Date:** 2026-06-03  
**Status:** approved

## Problem

A `Worker` backed by a Serverless Workflow (`Worker(Workflow)`) has no way to dispatch other casehub workers from within a workflow step. `WorkOrchestrator` and `CaseInstance` are invisible inside the workflow execution context. Sequential multi-worker orchestration — Worker A → Worker B → Worker C — is structurally impossible today.

Two compounding bugs make the existing path broken even for purely static workflows:

1. `ServerlessWorkflowExecutor` creates a brand-new `WorkflowApplication` via `ServiceLoader` on every execution — expensive and incorrect.
2. The `try-with-resources` closes the `WorkflowApplication` before the returned `CompletableFuture` resolves, cancelling the in-flight workflow.

## Goals

- Enable workflow steps (YAML or Java FuncDSL) to dispatch casehub workers and await their results reactively.
- Fix the `WorkflowApplication` lifecycle.
- Make FlowWorker execution non-blocking — Quartz threads must not be held for workflow duration.
- Emit `WORKFLOW_STEP_DISPATCHED` ledger events for lineage.

## Out of scope (follow-on)

- Sub-workflow lineage (`SUBWORKFLOW_STARTED` etc.) — requires verifying quarkus-flow 7.22.1 `WorkflowExecutionListener` sub-workflow hooks.
- Full workflow persistence on JVM restart — tracked in engine#404.
- `WorkflowContext` input injection via `onWorkflowStarted` (confirmed non-functional for FuncDSL task input by engine#213 Q1).

---

## Module Structure

New module: **`casehub-engine-flow`**

```
casehub-engine-common
        ↑
casehub-engine-api
        ↑
casehub-engine (runtime)              ← WorkOrchestrator, handlers
        ↑
casehub-engine-flow                   ← NEW
        ↑ (classpath activation)
casehub-engine-scheduler-quartz       ← QuartzWorkerExecutionJob
```

`casehub-engine-flow` has a compile-time dependency on `casehub-engine` (runtime) — needed to inject `WorkOrchestrator`. `scheduler-quartz` activates `FlowWorkerExecutor` via CDI at runtime; no compile-time dependency on `casehub-engine-flow` required from `scheduler-quartz`.

`ServerlessWorkflowExecutor` moves from `runtime` to `casehub-engine-flow` (renamed `FlowWorkerExecutor`). `runtime` gets a `NoOpWorkflowExecutor @DefaultBean @ApplicationScoped` that throws `UnsupportedOperationException` with a clear message — safe fallback for deployments that omit the flow module.

---

## `WorkflowExecutor` Interface Change

```java
// casehub-engine-common
public interface WorkflowExecutor {
    CompletableFuture<WorkflowModel> execute(
        Workflow workflow,
        Map<String, Object> inputData,
        CaseInstance caseInstance       // ← new parameter
    );
}
```

`CaseInstance` is already present at every call site (`QuartzWorkerExecutionJob`). The executor needs it to register the execution context before the workflow starts. Breaking change to one interface with one implementation — cost is mechanical.

---

## Components in `casehub-engine-flow`

### `WorkflowApplicationProducer`

`@ApplicationScoped @Produces WorkflowApplication`. Creates the singleton once at startup; `@Disposes` closes it on shutdown. `CasehubDispatchTaskExecutorFactory` is registered via `META-INF/services/io.serverlessworkflow.impl.executors.TaskExecutorFactory` and picked up automatically by `ServiceLoader` when the application is built — no explicit registration in the producer needed.

### `FlowExecutionRegistry`

`@ApplicationScoped`. `ConcurrentHashMap<String, FlowExecution>` keyed by **workflow instance ID** (assigned by `WorkflowApplication.instance()` before `start()` is called).

```java
record FlowExecution(CaseInstance caseInstance, WorkOrchestrator orchestrator) {}

void register(String instanceId, CaseInstance instance, WorkOrchestrator orchestrator)
FlowExecution get(String instanceId)
void remove(String instanceId)
```

Thread-locals are not used. Workflow steps run on quarkus-flow's own thread pool — thread-locals set on the Quartz thread are invisible there. Registry lookup by instance ID is correct for cross-thread access.

### `FlowWorkerExecutor`

`@ApplicationScoped implements WorkflowExecutor`. Replaces `ServerlessWorkflowExecutor`.

```java
public CompletableFuture<WorkflowModel> execute(
        Workflow workflow, Map<String, Object> inputData, CaseInstance instance) {
    var wfInstance = app.workflowDefinition(workflow).instance(inputData);
    String instanceId = wfInstance.instanceData().id();
    registry.register(instanceId, instance, orchestrator);
    CompletableFuture<WorkflowModel> future = wfInstance.start();
    future.whenComplete((model, ex) -> registry.remove(instanceId));
    return future;
}
```

### `CasehubDispatch`

`@ApplicationScoped`. The single dispatch entrypoint for both YAML and Java FuncDSL paths.

```java
public CompletableFuture<Map<String, Object>> dispatch(
        String workflowInstanceId, String capability, Map<String, Object> input) {
    FlowExecution execution = registry.get(workflowInstanceId);
    appendDispatchLog(execution.caseInstance(), capability, workflowInstanceId);
    return execution.orchestrator()
        .submit(execution.caseInstance(), WorkRequest.of(capability, input))
        .thenApply(WorkResult::output)
        .toCompletableFuture();
}
```

`appendDispatchLog` emits a `WORKFLOW_STEP_DISPATCHED` event (see Lineage section).

### `CasehubDispatchTaskExecutorFactory`

Plain class (not a CDI bean). Java SPI registered via `META-INF/services/io.serverlessworkflow.impl.executors.TaskExecutorFactory`.

```java
public TaskExecutorBuilder<?> getTaskExecutor(
        WorkflowContext ctx, Task task, WorkflowDefinition def) {
    // inspect task — return handler if this is a casehub:dispatch call, else null
    if (!isCasehubDispatch(task)) return null;
    return new CasehubDispatchTaskExecutorBuilder(task);
}
```

Handler's `apply(workflowContext, taskContext, input)`:

```java
String instanceId  = workflowContext.instanceData().id();
String capability  = input.get("capability").toString();
Map    stepInput   = (Map) input.get("input");

return Arc.container().instance(CasehubDispatch.class).get()
    .dispatch(instanceId, capability, stepInput)
    .thenApply(output ->
        workflowContext.definition().application().modelFactory().fromMap(output));
```

**Implementation risk:** `TaskExecutorFactory` was documented against 7.13.8 (GE-20260414-14d244). Verify exact API, registration mechanism, and task inspection approach against 7.22.1 before writing this class. If the interface changed, use the equivalent extension point.

---

## YAML Authoring

```yaml
document:
  dsl: '1.0.0'
  namespace: io.casehub
  name: analyze-and-report

do:
  - analyzeDocument:
      call: casehub:dispatch
      with:
        capability: analyze-document
        input: ${.}
  - generateReport:
      call: casehub:dispatch
      with:
        capability: generate-report
        input: ${.}
```

`Worker(Workflow)` — same constructor, no API change for YAML-authored workers.

---

## Java FuncDSL Authoring

Users write quarkus-flow FuncDSL steps directly. Inside each step, `WorkflowContext` provides the instance ID:

```java
// FuncDSL step — exact signature verified against 7.22.1 during implementation
.function("analyzeDocument", (workflowContext, taskContext, input) -> {
    String instanceId = workflowContext.instanceData().id();
    return Arc.container().instance(CasehubDispatch.class).get()
        .dispatch(instanceId, "analyze-document", input.asMap().orElse(Map.of()))
        .thenApply(output ->
            workflowContext.definition().application().modelFactory().fromMap(output));
})
```

The resulting `Workflow` object is passed to `Worker.builder().function(Workflow)` — same constructor as YAML. Both paths execute through the same `FlowWorkerExecutor` and `WorkflowApplication`.

**Implementation risk:** Verify the FuncDSL step function signature in quarkus-flow 7.22.1. The design assumes `WorkflowContext` is accessible in step lambdas (consistent with `CallableTask.apply(WorkflowContext, ...)` in the SPI path).

---

## Non-Blocking Execution

`QuartzWorkerExecutionJob.workflow()` currently blocks the Quartz thread:

```java
// BEFORE — blocks Quartz thread
WorkflowModel model = cf.get(timeoutMs, TimeUnit.MILLISECONDS);
```

New path for `Workflow` workers:

```java
// AFTER — fire and return
workflowExecutor.execute(workflow, inputData, instance)
    .thenApply(model -> model.asMap().orElse(Map.of()))
    .thenApply(output -> evalJqAsMap(output, capability.getOutputSchema()))
    .whenComplete((output, ex) -> {
        if (ex != null) {
            // publish WORKER_RETRIES_EXHAUSTED or equivalent failure event
        } else {
            eventBus.publish(WORKER_EXECUTION_FINISHED,
                new WorkflowExecutionCompleted(instance, worker, inputDataHash, output));
        }
    });
// Quartz thread returns here
```

**Known limitation:** Quartz marks the job complete when `execute()` returns. JVM restart while a workflow is in flight loses intermediate dispatch progress. `WorkerExecutionRecoveryService` will attempt to re-run from the beginning. Full persistence (quarkus-flow JPA/Redis module) is the fix — deferred to engine#404.

---

## Lineage

New `CaseHubEventType`: **`WORKFLOW_STEP_DISPATCHED`**

Emitted by `CasehubDispatch.dispatch()` before calling `WorkOrchestrator.submit()`.

| Field | Value |
|-------|-------|
| `caseId` | from `CaseInstance` |
| `workerId` | the FlowWorker's name |
| `metadata.capability` | dispatched capability name |
| `metadata.workflowInstanceId` | for correlation to the enclosing workflow |
| `metadata.parentInputDataHash` | FlowWorker's `inputDataHash` (links step to parent execution) |

This produces the ledger chain:
```
WORKER_SCHEDULED (FlowWorker)
  └─ WORKFLOW_STEP_DISPATCHED (step 1)
       └─ WORKER_SCHEDULED (dispatched worker)
            └─ WORKER_EXECUTION_COMPLETED (dispatched worker)
  └─ WORKFLOW_STEP_DISPATCHED (step 2)
       └─ ...
WORKER_EXECUTION_COMPLETED (FlowWorker)
```

---

## Error Handling

- `FlowExecutionRegistry.get()` with an unknown instance ID: throw `IllegalStateException` — indicates a programming error (dispatch called outside a workflow step or after cleanup).
- `CasehubDispatch.dispatch()` with an unknown capability: `WorkOrchestrator.submit()` returns a failed `CompletableFuture` — propagates as a step failure in quarkus-flow.
- Worker failure (retries exhausted): `WorkOrchestrator.submit()` future completes exceptionally — the workflow step fails, quarkus-flow applies its configured error handling (retry, compensate, terminate).
- Listener exceptions in `onWorkflowCompleted`: wrap all listener logic in try/catch and log — exceptions are silently swallowed by `LifecycleEventsUtils.publishEvent()` (GE-20260430-84bef2).

---

## Testing

- **Unit:** `CasehubDispatch` with mock `FlowExecutionRegistry` and mock `WorkOrchestrator`.
- **Integration (`@QuarkusTest`):** `FlowWorkerIntegrationTest` — define a `CaseHub` subclass with a YAML FlowWorker that dispatches a single capability; use `persistence-memory` and a mock worker function; assert `WORKFLOW_STEP_DISPATCHED` + `WORKER_EXECUTION_COMPLETED` in the event log.
- **FuncDSL path:** separate test with a programmatically built `Workflow` using quarkus-flow FuncDSL; assert same lineage.
- **`TaskExecutorFactory` API verification:** write a minimal test against 7.22.1 before implementing `CasehubDispatchTaskExecutorFactory` — verify task inspection, handler signature, and return type.

---

## Files Created / Modified

| Action | Path |
|--------|------|
| Create | `casehub-engine-flow/pom.xml` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/WorkflowApplicationProducer.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/FlowExecutionRegistry.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/FlowWorkerExecutor.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/CasehubDispatch.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/CasehubDispatchTaskExecutorFactory.java` |
| Create | `casehub-engine-flow/src/main/resources/META-INF/services/io.serverlessworkflow.impl.executors.TaskExecutorFactory` |
| Modify | `casehub-engine-common/src/main/java/io/casehub/engine/common/internal/worker/WorkflowExecutor.java` — add `CaseInstance` parameter |
| Move+delete | `runtime/src/main/java/io/casehub/engine/internal/worker/ServerlessWorkflowExecutor.java` → `casehub-engine-flow/FlowWorkerExecutor.java` |
| Create | `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkflowExecutor.java` |
| Modify | `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java` — non-blocking `Workflow` path |
| Modify | `casehub-engine-common` — add `WORKFLOW_STEP_DISPATCHED` to `CaseHubEventType` |
| Modify | `pom.xml` — add `casehub-engine-flow` module |
