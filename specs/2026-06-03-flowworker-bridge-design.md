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
- Emit `WORKFLOW_STEP_DISPATCHED` and `WORKFLOW_STEP_COMPLETED` ledger events for full step lineage.

## Out of scope (follow-on)

- Sub-workflow lineage (`SUBWORKFLOW_STARTED` etc.) — requires verifying quarkus-flow 0.6.0 (`serverlessworkflow-impl-core` 7.13.4.Final) `WorkflowExecutionListener` sub-workflow hooks.
- Full workflow persistence on JVM restart — tracked in engine#404.
- `WorkflowContext` input injection via `onWorkflowStarted` (confirmed non-functional for FuncDSL task input by engine#213 Q1).
- Workflow execution timeout enforcement in the non-blocking path — deferred to engine#404.

---

## Module Structure

New module: **`casehub-engine-flow`**

```
casehub-engine-api
        ↑
casehub-engine-common       ← WorkOrchestrator interface (NEW, in common/spi/)
        ↑                                      ↑
casehub-engine (runtime)    casehub-engine-flow (NEW)
        ↑
casehub-engine-scheduler-quartz
```

**Key dependency rule:** `casehub-engine-flow` depends on `casehub-engine-common` — not on runtime. `WorkOrchestrator` lives in `common/spi/` (alongside `CaseInstanceRepository`, `EventLogRepository`, etc.) because it uses `CaseInstance` from `common/internal/`. Placing it in `api/spi/` would create a circular dependency (`api` ← `common` ← `api`). The concrete implementation stays in `runtime`. CDI resolves the implementation at startup. `scheduler-quartz` has no compile-time dependency on `casehub-engine-flow`; it activates `FlowWorkerExecutor` via CDI at runtime.

`ServerlessWorkflowExecutor` moves from `runtime` to `casehub-engine-flow` (renamed `FlowWorkerExecutor`). `runtime` gets a `NoOpWorkflowExecutor @DefaultBean @ApplicationScoped` that throws `UnsupportedOperationException` with a clear message — safe fallback for deployments that omit the flow module.

---

## WorkOrchestrator Interface (new in `common/spi/`)

`WorkOrchestrator` currently lives as a concrete class in `runtime`. Extract a two-method interface into `common/spi/` (same package as `CaseInstanceRepository`, `EventLogRepository` — all use `CaseInstance`):

```java
// casehub-engine-common/src/main/java/io/casehub/engine/common/spi/WorkOrchestrator.java
public interface WorkOrchestrator {
    CompletionStage<WorkResult> submit(CaseInstance instance, WorkRequest request);
    CompletionStage<WorkResult> submitAndWait(CaseInstance instance, WorkRequest request);
}
```

The existing `WorkOrchestrator` class in `runtime` is renamed to `DefaultWorkOrchestrator implements WorkOrchestrator` and stays `@ApplicationScoped`. All existing injection sites in `runtime` continue to work via CDI. `casehub-engine-flow` injects `WorkOrchestrator` (the interface from `common/spi/`).

---

## `WorkflowExecutor` Interface Change

```java
// casehub-engine-common
public interface WorkflowExecutor {
    CompletableFuture<WorkflowModel> execute(
        Workflow workflow,
        Map<String, Object> inputData,
        CaseInstance caseInstance,   // for FlowExecutionRegistry
        String workerName,           // for WORKFLOW_STEP_DISPATCHED lineage
        String inputDataHash         // for WORKFLOW_STEP_DISPATCHED lineage
    );
}
```

All five parameters are available at the sole call site (`QuartzWorkerExecutionJob`). Breaking change to one interface with one implementation — cost is mechanical.

---

## Components in `casehub-engine-flow`

### `WorkflowApplicationProducer`

`@ApplicationScoped @Produces WorkflowApplication`. Creates the singleton once at startup; `@Disposes` closes it on shutdown. `CasehubCallableTaskBuilder` is registered via `META-INF/services/io.serverlessworkflow.impl.executors.CallableTaskBuilder` and is discovered automatically by `DefaultTaskExecutorFactory`'s `ServiceLoader` call — no explicit registration in the producer needed.

**Thread safety confirmed:** `WorkflowApplication.workflowDefinition()` uses `ConcurrentHashMap.computeIfAbsent` internally — concurrent calls from different Quartz threads are safe.

### `FlowExecutionRegistry`

`@ApplicationScoped`. `ConcurrentHashMap<String, FlowExecution>` keyed by **workflow instance ID**.

```java
record FlowExecution(
    CaseInstance caseInstance,
    String workerName,      // for WORKFLOW_STEP_DISPATCHED.workerId
    String inputDataHash    // for WORKFLOW_STEP_DISPATCHED.metadata.parentInputDataHash
) {}

void register(String instanceId, CaseInstance instance, String workerName, String inputDataHash)
FlowExecution get(String instanceId)   // throws IllegalStateException if not found
void remove(String instanceId)
```

Thread-locals are not used. Workflow steps run on quarkus-flow's own thread pool — thread-locals set on the Quartz thread are invisible there. Registry lookup by instance ID is correct for cross-thread access.

**Implementation note:** quarkus-flow 0.6.0 / `serverlessworkflow-impl-core` 7.13.4.Final does not provide user-attachable execution context on `WorkflowContext`. The global `ConcurrentHashMap` registry is the correct mechanism.

### `FlowWorkerExecutor`

`@ApplicationScoped implements WorkflowExecutor`. Plain `@ApplicationScoped` (no `@DefaultBean`) — wins over `NoOpWorkflowExecutor @DefaultBean` per CDI resolution.

```java
public CompletableFuture<WorkflowModel> execute(
        Workflow workflow, Map<String, Object> inputData,
        CaseInstance instance, String workerName, String inputDataHash) {

    var wfInstance = app.workflowDefinition(workflow).instance(inputData);
    // id() is assigned in WorkflowMutableInstance constructor — available before start()
    String instanceId = wfInstance.id();

    registry.register(instanceId, instance, workerName, inputDataHash);
    try {
        CompletableFuture<WorkflowModel> future = wfInstance.start();
        future.whenComplete((model, ex) -> registry.remove(instanceId));
        return future;
    } catch (RuntimeException e) {
        registry.remove(instanceId);
        throw e;
    }
}
```

### `CasehubDispatch`

`@ApplicationScoped`. Single dispatch entrypoint for both YAML and FuncDSL paths. Injects `WorkOrchestrator` (the interface from `common/spi/`) directly — not through `FlowExecution`.

```java
@Inject WorkOrchestrator orchestrator;
@Inject FlowExecutionRegistry registry;
@Inject EventLogRepository eventLogRepository;

public CompletableFuture<Map<String, Object>> dispatch(
        String workflowInstanceId, String capability) {

    FlowExecution execution = registry.get(workflowInstanceId);
    appendStepDispatchedLog(execution, capability, workflowInstanceId);

    return orchestrator
        .submit(execution.caseInstance(), WorkRequest.of(capability, Map.of()))
        .whenComplete((result, ex) -> {
            // whenComplete always fires — success and failure both get a terminal step event.
            // Logging exceptions here do not alter the completion propagated to quarkus-flow.
            if (ex != null) {
                appendStepLog(execution, capability, workflowInstanceId,
                    WORKFLOW_STEP_FAILED, null);
            } else {
                appendStepLog(execution, capability, workflowInstanceId,
                    WORKFLOW_STEP_COMPLETED, result.output());
            }
        })
        .thenApply(WorkResult::output)
        .toCompletableFuture();
}
```

`appendStepDispatchedLog` and `appendStepLog` emit ledger events fire-and-forget via `eventLogRepository.appendAndReturnId(...).subscribe().with(...)` — the same pattern used by `WorkOrchestrator.doSubmit()` for `WORK_SUBMITTED`. No external transaction required; the reactive implementation creates its own via `withTenantTransaction`.

**Why `whenComplete` and not `thenApply` for logging:** `thenApply` is skipped when `submit()` returns a failed future (unknown capability, routing failure, worker exhaustion). `whenComplete` always fires regardless of outcome, ensuring every dispatched step has a terminal ledger event. A logging exception inside `thenApply` would also abort a successful dispatch — `whenComplete` isolates logging failures from the completion value.

### `CasehubFlow`

`CasehubFlow` is a static utility class in `casehub-engine-flow` for FuncDSL step authors. It blocks on the dispatch future — safe because quarkus-flow runs steps on a `newCachedThreadPool()` (not Vert.x IO threads).

```java
public final class CasehubFlow {
    private CasehubFlow() {}

    /** Dispatches a casehub capability and blocks until the result is available.
     *  Safe to call from FuncDSL steps — quarkus-flow uses a cached thread pool. */
    public static Map<String, Object> dispatch(WorkflowContextData ctx, String capability) {
        return Arc.container().instance(CasehubDispatch.class).get()
            .dispatch(ctx.instanceData().id(), capability)
            .toCompletableFuture()
            .join();
    }
}
```

### `CasehubCallableTaskBuilder`

Plain class (not a CDI bean). Java SPI registered via `META-INF/services/io.serverlessworkflow.impl.executors.CallableTaskBuilder`. `DefaultTaskExecutorFactory` discovers all `CallableTaskBuilder` implementations via `ServiceLoader` when routing `call:` tasks. In the SDK, `call: casehub:dispatch` in YAML is parsed as `CallFunction` with `call = "casehub:dispatch"` and `with = FunctionArguments` (an `@AdditionalProperties` map). `DefaultTaskExecutorFactory` calls `findCallTask(CallFunction.class)` which picks up our builder.

```java
public class CasehubCallableTaskBuilder implements CallableTaskBuilder<CallFunction> {
    private String capability;

    @Override
    public boolean accept(Class<? extends TaskBase> clazz) {
        return CallFunction.class.isAssignableFrom(clazz);
    }

    @Override
    public void init(CallFunction task, WorkflowDefinition definition,
                     WorkflowMutablePosition position) {
        if (!"casehub:dispatch".equals(task.getCall())) {
            throw new UnsupportedOperationException(
                "Unsupported call function: " + task.getCall()
                + ". Only 'casehub:dispatch' is supported.");
        }
        Map<String, Object> args = task.getWith() != null
            ? task.getWith().getAdditionalProperties() : Map.of();
        Object capabilityArg = args.get("capability");
        if (capabilityArg == null) {
            throw new IllegalArgumentException(
                "casehub:dispatch step is missing required 'capability' argument");
        }
        this.capability = capabilityArg.toString();
    }

    @Override
    public CallableTask build() {
        String cap = this.capability;
        // CallableTask.apply() returns CompletableFuture<WorkflowModel> — fully async, no blocking
        return (workflowContext, taskContext, input) -> {
            String instanceId = workflowContext.instanceData().id();
            return Arc.container().instance(CasehubDispatch.class).get()
                .dispatch(instanceId, cap)
                .thenApply(output ->
                    workflowContext.definition().application().modelFactory().fromMap(output));
        };
    }
}
```

**API facts (confirmed from `serverlessworkflow-impl-core` 7.13.4.Final source):**
- `CallableTask` interface: `CompletableFuture<WorkflowModel> apply(WorkflowContext, TaskContext, WorkflowModel)` — the YAML dispatch path is fully async.
- `CallFunction` and `FunctionArguments` are in `io.serverlessworkflow.api.types` (the standard types jar, not the experimental-types jar). The `func` subpackage (`io.serverlessworkflow.api.types.func`) is for the experimental `CallJava`/`CallTaskJava` types.
- `task.getWith().getAdditionalProperties()` is the correct accessor for `FunctionArguments` (it wraps a `LinkedHashMap<String, Object>`).
- `workflowContext.instanceData()` returns the live `WorkflowMutableInstance` which directly implements `id()`.
- Thread pool: `DefaultExecutorServiceFactory` uses `Executors.newCachedThreadPool()` — not Vert.x IO threads.
- `accept()` returning true for all `CallFunction.class` instances is correct — no other SDK `CallableTaskBuilder` handles `CallFunction`. Unknown function names fail fast in `init()`.
- `ServicePriority` (extended by `CallableTaskBuilder`) defaults to 0 — no ordering issues with an empty set of competitors.

---

## Data Flow Between Steps

The case context is the data bus between workflow steps. There is no direct step-to-step data passing in the workflow data model.

The sequence for a two-step workflow:

1. Worker A runs as a dispatched worker. Its output is returned as `WorkResult.output()`.
2. `CasehubDispatch.dispatch()` resolves with the output map.
3. `WorkflowExecutionCompletedHandler` (which ran when Worker A's `WORKER_EXECUTION_FINISHED` fired) has already merged Worker A's output into the case context via `applyOutputWithConflictResolution()`.
4. Worker B runs. Its input is computed by `WorkOrchestrator.submit()` via `evalJqAsMap(caseContext, capability.inputSchema)` — reading from the now-updated case context.

This means:
- The `capability:` field in the YAML `with:` block is the only required dispatch parameter.
- Worker B sees Worker A's output through the case context, shaped by Worker B's `inputSchema` JQ expression.
- Workers dispatched from workflow steps should define `inputSchema` to select the relevant fields from the case context, exactly as they would for normal binding-triggered execution.

The `dispatch()` future resolves when `PendingWorkRegistry.complete(correlationKey)` is called — triggered by `WorkflowExecutionCompletedHandler` processing the dispatched worker's `WORKER_EXECUTION_FINISHED` event. This is the mechanism that sequences workflow steps: step N+1 cannot begin until step N's dispatch future resolves and quarkus-flow receives the resolved value.

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
  - generateReport:
      call: casehub:dispatch
      with:
        capability: generate-report
```

Worker input is always drawn from the case context via each capability's `inputSchema` — not from the workflow data model. `Worker(Workflow)` constructor unchanged; no API change for YAML-authored workers.

---

## Java FuncDSL Authoring

Use `JavaFilterFunction<Map, Map>` (confirmed API: `(T input, WorkflowContextData wfCtx, TaskContextData taskCtx) → R`) for context-aware steps. `CasehubFlow.dispatch()` blocks on the dispatch future — safe because quarkus-flow uses a cached thread pool, not Vert.x IO threads.

```java
// Via FuncTaskItemListBuilder.function(name, Consumer<FuncCallTaskBuilder>)
list.function("analyzeDocument", b -> b.function(
    (JavaFilterFunction<Map, Map>)(input, wfCtx, taskCtx) ->
        CasehubFlow.dispatch(wfCtx, "analyze-document"),
    Map.class));
```

`CasehubFlow.dispatch(WorkflowContextData, String)` blocks via `.join()` and returns `Map<String, Object>`. The resulting `Workflow` is passed to `Worker.builder().function(Workflow)` — same constructor as YAML. Both paths execute through the same `FlowWorkerExecutor` and `WorkflowApplication`.

---

## Non-Blocking Execution

`QuartzWorkerExecutionJob.workflow()` currently blocks the Quartz thread. The new path replaces it inline in the `execute()` method where `worker` and `capability` are already resolved:

```java
// Inside execute(), after worker, capability, instance, inputDataHash, workerContext are resolved:

if (worker.getFunction().getValue() instanceof Workflow workflow) {

    workflowExecutor.execute(workflow, inputData, instance, worker.getName(), inputDataHash)
        .thenApply(model -> model.asMap()
            .orElseThrow(() -> new RuntimeException(
                "Workflow produced non-serializable model: " + worker.getName())))
        .thenApply(output -> evalJqAsMap(output, capability.getOutputSchema()))
        .whenComplete((output, ex) -> {
            if (ex != null) {
                // Failure handling — see below
                handleWorkflowFailure(instance, worker, ex);
            } else {
                eventBus.publish(WORKER_EXECUTION_FINISHED,
                    new WorkflowExecutionCompleted(instance, worker, inputDataHash, output));
            }
        });

    // Quartz thread returns here — job marked complete by Quartz
    return;
}
// ... function, agent paths continue synchronously as before
```

**Failure handling in the non-blocking path:**

The Quartz thread has already returned when the workflow future resolves exceptionally. Re-throwing a `JobExecutionException` is not possible from `whenComplete`. Two options:

1. **Publish `WORKER_EXECUTION_FAILED` (new event type):** Handled by a new handler that triggers Quartz retry scheduling directly (bypassing the job re-queue path). Cleaner but requires a new event type and handler.
2. **Directly re-schedule the Quartz job via `WorkerExecutionManager.rescheduleRetry()`:** Replicates Quartz's built-in retry logic; complex and fragile.

**Spec decision (option 1):** `WORKER_EXECUTION_FAILED` already exists in `CaseHubEventType`. The async failure path routes through `QuartzWorkerExecutionJobListener` via the event bus, reusing its retry and exhaustion logic — but this requires a real refactoring, not a free reuse.

**Required refactoring of `QuartzWorkerExecutionJobListener`:**

`maybeRescheduleJob(JobExecutionContext)` and `rescheduleJob(JobExecutionContext, ...)` read all retry data from the Quartz job context — `caseHubInstanceUuid`, `workerId`, `inputDataHash`, `tenancyId`, and the full `JobDataMap` which `rescheduleJob()` copies verbatim to the retry job via `.usingJobData(context.getMergedJobDataMap())`. In the async path, the Quartz thread has already returned and there is no `JobExecutionContext`. These methods cannot be called as-is.

Fix: Extract a new `WorkerRetryContext` record:
```java
record WorkerRetryContext(
    UUID caseId, String workerId, String inputDataHash, String tenancyId, String eventLogId)
```

Refactor `maybeRescheduleJob` and `rescheduleJob` to accept `WorkerRetryContext` instead of `JobExecutionContext`. `rescheduleJob` reconstructs the `JobDataMap` from the record's fields (the five fields cover everything the map contains). The sync path in `jobWasExecuted` builds a `WorkerRetryContext` from its `JobExecutionContext` before calling the refactored method; the async path builds it from the `WorkflowExecutionFailed` event record.

**Sequencing — async `@ConsumeEvent` handler:**

The handler must follow the same ordering as the sync path in `jobWasExecuted`:
1. Persist `WORKER_EXECUTION_FAILED` event log entry (fire-and-forget reactive subscribe).
2. In the success callback of that subscribe, call the refactored `maybeRescheduleWorker(WorkerRetryContext)`.

`countFailedAttempts` queries `WORKER_EXECUTION_FAILED` events with matching `inputDataHash` to determine retry count. If the failure event is not written first, the count is one short and every workflow failure gets one extra unearned retry.

`handleWorkflowFailure()` publishes a `WorkflowExecutionFailed` event record on the Vert.x event bus carrying: `instance`, `worker`, `capability`, `inputDataHash`, `eventLogId`. From `instance`, `tenancyId` and `caseId` are available; from `worker`, `workerId`. The full `WorkerRetryContext` can be constructed.

No new handler class in `runtime`. The new `@ConsumeEvent(EventBusAddresses.WORKFLOW_EXECUTION_FAILED)` method lives in `QuartzWorkerExecutionJobListener` (in `scheduler-quartz`). A new `WorkflowExecutionFailed` event record and `WorkerRetryContext` record are required; `WorkerRetryContext` lives in `scheduler-quartz` (only that module constructs or interprets it).

**Known limitation:** Quartz marks the job complete when `execute()` returns. JVM restart while a workflow is in flight loses intermediate dispatch progress. `WorkerExecutionRecoveryService` will attempt to re-run from the beginning (see Constraints section). Full persistence is deferred to engine#404.

**Timeout:** `executionConfig.getEffectiveTimeout()` is not applied to the non-blocking workflow path — the Quartz thread returns before the workflow completes, so the timeout has no enforcement point. Workflow timeout is deferred to engine#404.

---

## Lineage

New `CaseHubEventType` values: **`WORKFLOW_STEP_DISPATCHED`**, **`WORKFLOW_STEP_COMPLETED`**, and **`WORKFLOW_STEP_FAILED`**

`WORKFLOW_STEP_DISPATCHED` is emitted before `submit()`. `WORKFLOW_STEP_COMPLETED` or `WORKFLOW_STEP_FAILED` is emitted in the `whenComplete` callback after `submit()` resolves — success or failure respectively. Every dispatched step has exactly one terminal event.

| Field | `WORKFLOW_STEP_DISPATCHED` | `WORKFLOW_STEP_COMPLETED` | `WORKFLOW_STEP_FAILED` |
|-------|--------------------------|--------------------------|------------------------|
| `caseId` | from `CaseInstance` | from `CaseInstance` | from `CaseInstance` |
| `workerId` | FlowWorker name (from `FlowExecution.workerName`) | FlowWorker name | FlowWorker name |
| `metadata.capability` | dispatched capability name | dispatched capability name | dispatched capability name |
| `metadata.workflowInstanceId` | for correlation | for correlation | for correlation |
| `metadata.parentInputDataHash` | FlowWorker's `inputDataHash` | FlowWorker's `inputDataHash` | FlowWorker's `inputDataHash` |
| `metadata.outputSummary` | — | serialized output keys (not full payload) | — |
| `metadata.errorMessage` | — | — | exception message |

Ledger chain (step execution sequencing in parentheses):

```
WORKER_SCHEDULED (FlowWorker)
  └─ WORKFLOW_STEP_DISPATCHED (step 1)
       └─ WORKER_SCHEDULED (dispatched worker 1)
            └─ WORKER_EXECUTION_COMPLETED (dispatched worker 1)
                  [context updated → PendingWorkRegistry.complete() → dispatch() future resolves]
       └─ WORKFLOW_STEP_COMPLETED (step 1)
  └─ WORKFLOW_STEP_DISPATCHED (step 2)
       └─ WORKER_SCHEDULED (dispatched worker 2)
            └─ WORKER_EXECUTION_COMPLETED (dispatched worker 2)
                  [context updated → PendingWorkRegistry.complete() → dispatch() future resolves]
       └─ WORKFLOW_STEP_COMPLETED (step 2)
WORKER_EXECUTION_COMPLETED (FlowWorker)
```

---

## Constraints and Known Behaviours

**Re-entrant binding evaluation:** When a dispatched worker completes, its output is merged into the case context and `CONTEXT_CHANGED` fires. `CaseContextChangedEventHandler` re-evaluates all bindings. If the dispatched capability has a guard-based binding in the same case definition, the binding evaluator may schedule a second execution of that capability.

Mitigation: Capabilities dispatched from workflow steps must not simultaneously have guard-based bindings in the enclosing case definition. If a capability is workflow-controlled, it should not also be binding-controlled. PlanItem-backed suppression (making the binding evaluator aware of in-flight dispatched work) is a follow-on concern.

**Capability scope:** Dispatched capabilities must be defined in the same `CaseDefinition` as the FlowWorker. `WorkOrchestrator.submit()` looks up capabilities from the enclosing case definition. Attempting to dispatch a capability from a different case definition returns a failed `CompletableFuture` with `"No capability found for name: ..."`.

**JVM restart with in-flight dispatched workers:** If a FlowWorker is in flight when the JVM restarts, `WorkerExecutionRecoveryService` re-runs it from the beginning. If a sub-worker (step 1) completed before restart, its output is already in the case context. Re-running the FlowWorker will re-dispatch the same capability — the sub-worker executes again. Sub-workers dispatched from FlowWorkers should be idempotent where possible. Full deduplication requires engine#404.

**Classpath activation:** `FlowWorkerExecutor` is plain `@ApplicationScoped` (no `@DefaultBean`). CDI resolution prefers it over `NoOpWorkflowExecutor @DefaultBean` when `casehub-engine-flow` is on the classpath.

---

## Error Handling

- `FlowExecutionRegistry.get()` with unknown instance ID: throw `IllegalStateException` — indicates dispatch called outside a workflow step or after cleanup.
- `CasehubDispatch.dispatch()` with unknown capability: `WorkOrchestrator.submit()` returns a failed `CompletableFuture` — propagates as step failure in quarkus-flow.
- `wfInstance.start()` throws synchronously: `FlowWorkerExecutor.execute()` removes the registry entry in the catch block before re-throwing — no leak.
- `WorkflowModel.asMap()` returns empty: throw `RuntimeException("Workflow produced non-serializable model")` — no silent data loss.
- Workflow future completes exceptionally: `handleWorkflowFailure()` publishes `WORKER_EXECUTION_FAILED` — retried or escalated by the failure handler.
- Listener exceptions in `onWorkflowCompleted`: wrap all listener logic in try/catch and log — exceptions are silently swallowed by `LifecycleEventsUtils.publishEvent()` (GE-20260430-84bef2).

---

## Testing

- **Unit — `CasehubDispatch`:** mock `FlowExecutionRegistry` and mock `WorkOrchestrator`; assert `WORKFLOW_STEP_DISPATCHED` and `WORKFLOW_STEP_COMPLETED` both emitted; assert dispatch failure propagates as failed future.
- **Unit — `FlowExecutionRegistry`:** register → get → remove lifecycle; concurrent register/get under different instance IDs (assert no cross-contamination); get for unknown ID throws `IllegalStateException`.
- **Integration (`@QuarkusTest`) — `FlowWorkerIntegrationTest`:** define a `CaseHub` subclass with a YAML FlowWorker that dispatches a single capability; use `persistence-memory` and a mock worker function; assert `WORKFLOW_STEP_DISPATCHED` + `WORKFLOW_STEP_COMPLETED` + `WORKER_EXECUTION_FINISHED` in event log in correct order.
- **Integration — two-step sequential dispatch:** assert step 2 does not begin until step 1's dispatched worker completes; assert step 1 output appears in case context before step 2 dispatches.
- **Integration — FuncDSL path:** programmatically built `Workflow` via quarkus-flow FuncDSL using `CasehubFlow.dispatch()`; assert same lineage as YAML path.
- **Integration — CONTEXT_CHANGED re-entrancy (behavioural documentation):** define a case with a FlowWorker that dispatches capability X, and separately a guard binding for capability X; assert that capability X executes twice — once from the workflow dispatch and once from the binding evaluator after context update. This test documents the known double-execution behaviour described in Constraints; it is not a correctness assertion.
- **Integration — dispatch step failure / lineage:** mock `WorkOrchestrator.submit()` to return a failed future; assert `WORKFLOW_STEP_DISPATCHED` emitted, then `WORKFLOW_STEP_FAILED` emitted; assert `WORKFLOW_STEP_COMPLETED` is not emitted.
- **Integration — dispatch step success / lineage:** assert `WORKFLOW_STEP_DISPATCHED` then `WORKFLOW_STEP_COMPLETED` emitted; assert `WORKFLOW_STEP_FAILED` is not emitted.
- **Integration — workflow failure / retry:** mock worker function throws; assert `WORKER_EXECUTION_FAILED` event log written before `maybeRescheduleWorker()` is called; assert `WORKFLOW_EXECUTION_FAILED` event bus message published; assert Quartz job re-scheduled (via `QuartzWorkerSchedulerService`); assert `WORKER_EXECUTION_FINISHED` is not published.
- **Integration — workflow failure / exhaustion:** mock worker function throws; set retry policy `maxAttempts=1`; after one attempt assert `WORKER_RETRIES_EXHAUSTED` published and no further reschedule.
- **Unit — `QuartzWorkerExecutionJobListener` refactoring:** assert that `maybeRescheduleWorker(WorkerRetryContext)` produces identical scheduling behaviour when called from both the sync path (built from `JobExecutionContext`) and the async path (built from `WorkflowExecutionFailed` event).
- **Unit — `CasehubCallableTaskBuilder`:** `accept(CallFunction.class)` returns true; `accept(CallHTTP.class)` returns false; `init()` with missing capability throws `IllegalArgumentException`; `init()` with wrong call name throws `UnsupportedOperationException`; `build()` produces a `CallableTask` that calls `CasehubDispatch.dispatch()` with the correct instance ID and capability.
- **Concurrent execution:** two simultaneous FlowWorker executions (different cases) — assert no cross-contamination in `FlowExecutionRegistry`.

---

## API Facts (Confirmed from Source)

All items below are confirmed from `serverlessworkflow-impl-core` 7.13.4.Final and `quarkus-flow` 0.6.0 source. No further verification needed.

| Fact | Value |
|------|-------|
| `wfInstance.id()` available before `start()` | ✅ Assigned in `WorkflowMutableInstance` constructor |
| `workflowDefinition()` thread-safe | ✅ `ConcurrentHashMap.computeIfAbsent` |
| Task executor thread pool | ✅ `Executors.newCachedThreadPool()` — blocking safe |
| Vert.x IO thread concern | ✅ Not applicable — separate thread pool |
| Extension point for `call: casehub:dispatch` | ✅ `CallableTaskBuilder<CallFunction>` SPI |
| SPI registration file | ✅ `META-INF/services/io.serverlessworkflow.impl.executors.CallableTaskBuilder` |
| YAML `with:` argument access | ✅ `task.getWith().getAdditionalProperties().get("capability")` |
| `CallableTask.apply()` signature | ✅ `CompletableFuture<WorkflowModel> apply(WorkflowContext, TaskContext, WorkflowModel)` |
| FuncDSL context function type | ✅ `JavaFilterFunction<T, R>`: `(T, WorkflowContextData, TaskContextData) → R` |
| User-attachable workflow context | ✅ Not available in 7.13.4.Final — global registry is correct mechanism |

---

## Files Created / Modified

| Action | Path |
|--------|------|
| Create | `casehub-engine-common/src/main/java/io/casehub/engine/common/spi/WorkOrchestrator.java` |
| Rename+modify | `runtime/.../orchestration/WorkOrchestrator.java` → `DefaultWorkOrchestrator.java` (implements interface) |
| Create | `casehub-engine-flow/pom.xml` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/WorkflowApplicationProducer.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/FlowExecutionRegistry.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/FlowWorkerExecutor.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/CasehubDispatch.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/CasehubFlow.java` |
| Create | `casehub-engine-flow/src/main/java/io/casehub/engine/flow/CasehubCallableTaskBuilder.java` |
| Create | `casehub-engine-flow/src/main/resources/META-INF/services/io.serverlessworkflow.impl.executors.CallableTaskBuilder` |
| Modify | `casehub-engine-common/.../worker/WorkflowExecutor.java` — add `workerName`, `inputDataHash` parameters |
| Move+delete | `runtime/.../worker/ServerlessWorkflowExecutor.java` → `casehub-engine-flow/FlowWorkerExecutor.java` |
| Create | `runtime/.../worker/NoOpWorkflowExecutor.java` |
| Modify | `scheduler-quartz/.../QuartzWorkerExecutionJob.java` — non-blocking `Workflow` path; pass `workerName`+`inputDataHash` to `execute()` |
| Modify | `casehub-engine-common` — add `WORKFLOW_STEP_DISPATCHED`, `WORKFLOW_STEP_COMPLETED`, `WORKFLOW_STEP_FAILED` to `CaseHubEventType` (`WORKER_EXECUTION_FAILED` already exists) |
| Create | `casehub-engine-common/.../event/WorkflowExecutionFailed.java` — event record: `(CaseInstance instance, Worker worker, Capability capability, String inputDataHash, String eventLogId)` |
| Create | `casehub-engine-common/.../event/EventBusAddresses.WORKFLOW_EXECUTION_FAILED` — new address constant |
| Modify | `scheduler-quartz/.../QuartzWorkerExecutionJobListener.java` — (1) extract `maybeRescheduleJob(JobExecutionContext)` and `rescheduleJob(JobExecutionContext, ...)` into `maybeRescheduleWorker(WorkerRetryContext)` and `rescheduleWorker(WorkerRetryContext, ...)` that take a plain value object; sync path builds `WorkerRetryContext` from `JobExecutionContext`; (2) add `@ConsumeEvent(WORKFLOW_EXECUTION_FAILED)` handler that persists `WORKER_EXECUTION_FAILED` event log first, then calls `maybeRescheduleWorker(WorkerRetryContext)` |
| Create | `scheduler-quartz/.../WorkerRetryContext.java` — record: `(UUID caseId, String workerId, String inputDataHash, String tenancyId, String eventLogId)` |
| Modify | `pom.xml` — add `casehub-engine-flow` module |