# Design: Engine Sync Handler Delegates to casehub-worker WorkerExecutor

**Issue:** casehubio/engine#726
**Epic:** casehubio/engine#203 (ContextBridge protocol)
**Date:** 2026-11-18

## Problem

The engine's `SyncAgentWorkerFunctionHandler` reimplements execution concerns that `casehub-worker`'s `DefaultWorkerExecutor` already owns:

- **Timeout:** engine uses `future.get(timeoutMs)` on a virtual thread; worker executor uses SmallRye Guard built from `ExecutionPolicy`
- **Input type validation:** duplicated in both
- **Schema validation:** worker executor validates input/output against Capability projections; engine does not
- **OTel tracing:** worker executor instruments with spans; engine does not
- **Retry:** worker executor supports retry with backoff/jitter from ExecutionPolicy; engine has no handler-level retry
- **Exception → result conversion:** both convert exceptions to `WorkerResult.expired()`/`WorkerResult.failed()`

Additionally, the `WorkerFunctionHandler` interface decomposes the Worker aggregate into individual fields (`WorkerFunction`, `int timeoutMs`), stripping information handlers need for delegation. The `SyncAgentWorkerFunctionHandler` combines three distinct execution paths (Sync, Agent, ExchangeProcessor) in a single class with internal dispatch, defeating the handler chain's polymorphic design.

## Adaptation from Original Issue

Issue #726 was written before the virtual-thread migration (engine#770). The following adjustments apply:

| Original proposal | Adapted design |
|---|---|
| Wrap in `Uni.createFrom().item().runSubscriptionOn(virtualThreads)` | Call `.await().indefinitely()` on virtual thread (PP-20260723-c4c1cf) |
| `WorkerExecutionContext` set/clear | Replaced by `WorkerScope` parameter passing — no ThreadLocal needed |
| `PolicyEnforcer` handles timeout | Worker executor uses SmallRye Guard directly |
| Worker executor is "blocking" | Worker executor returns `Uni<WorkerResult>` |

## Design

### 1. Interface Changes

**`WorkerFunctionHandler`** (common-core) — new signature:

```java
public interface WorkerFunctionHandler {
    boolean supports(WorkerFunction<?, ?> function);

    HandlerResult execute(
        Worker worker,
        Capability capability,
        Object inputData,
        WorkerContext context,
        ExecutionMetadata metadata);
}
```

Changes: adds `Worker` and `Capability`, drops `int timeoutMs` (available via `worker.executionPolicy().timeoutMs()`), drops `WorkerFunction<?,?>` (available via `worker.function()`).

**Engine's `WorkerExecutor`** (common-core) — new signature:

```java
public interface WorkerExecutor {
    HandlerResult execute(
        Worker worker,
        Capability capability,
        Object inputData,
        WorkerContext context,
        String outputProjection,
        ExecutionMetadata metadata);
}
```

Changes: same as handler but retains `outputProjection` — the composite executor owns JQ evaluation as a cross-cutting concern applied after handler returns.

**`DefaultWorkerExecutor`** (runtime) — the composite extracts `worker.function()` for the `supports()` check and passes `Worker` + `Capability` through to handlers.

### 2. Handler Split

`SyncAgentWorkerFunctionHandler` splits into two classes:

**`SyncWorkerFunctionHandler`** (runtime):
- `supports()`: `WorkerFunction.Sync` or `ExchangeAwareFunction`
- Delegates to `io.casehub.worker.runtime.WorkerExecutor`
- Wraps Worker function to provide engine-created `WorkerRuntime`
- Manages scoped worker sessions (REINVOKED mode)

**`AgentWorkerFunctionHandler`** (runtime):
- `supports()`: `AgentWorkerFunction`
- Keeps engine-local execution (invoker, protocol metadata, token tracking)
- Own timeout via `future.get(worker.executionPolicy().timeoutMs())`
- Does NOT delegate to worker executor — `AgentWorkerFunction` has no `WorkerFunction.Sync` fn()

### 3. Delegation Mechanics

`SyncWorkerFunctionHandler.execute()` sequence:

```
1. If REINVOKED mode: acquire binding lock from ScopedWorkerRegistry, load accumulated state
2. Create WorkerRuntime via WorkerRuntimeFactory.create(caseId, workerName, context, accState)
3. Wrap Worker's function:
   - Extract original Sync/ExchangeProcessor function
   - Build new function whose BiFunction captures the engine-created runtime
   - Construct effective Worker with wrapped function (same name, capabilities, policy)
4. Delegate: workerExecutor.execute(effectiveWorker, capability, inputData)
                            .await().indefinitely()
5. If REINVOKED mode: update accumulated state in scoped session
6. Return HandlerResult(result)
```

The function wrapping is the key mechanism — it bridges the engine's `WorkerRuntime` into the worker executor's execution path without changing the worker executor API. The worker executor's `liftToUni()` calls `sync.fn().apply(input, null)`, but the wrapped function ignores the null scope and uses the captured runtime instead.

### 4. Timeout Reconciliation

The engine applies a platform-wide default timeout via `WorkerExecutionConfig.getEffectiveTimeout()` when `Worker.executionPolicy().timeoutMs()` is null. If passed through unmodified, the worker executor's Guard would skip timeout entirely.

Resolution: `WorkerExecutionOrchestrator` enriches the Worker's ExecutionPolicy before passing to the executor:

```java
ExecutionPolicy effective = new ExecutionPolicy(
    executionConfig.getEffectiveTimeout(worker.executionPolicy().timeoutMs()),
    worker.executionPolicy().retries(),
    worker.executionPolicy().circuitBreaker());
Worker effectiveWorker = new Worker(worker.name(), worker.capabilities(),
    worker.function(), effective, worker.description(), worker.definitionRef());
```

Handlers see the effective Worker with a concrete timeout, not the definition-time Worker. This is analogous to how `timeoutMs` is resolved today — the enrichment just moves from inline resolution to Worker construction.

### 5. Cross-Repo Changes

**casehub-worker** (prerequisite):
- `DefaultWorkerExecutor.liftToUni()` — add `ExchangeProcessor` branch:
  ```java
  if (worker.function() instanceof WorkerFunction.ExchangeProcessor ep) {
      return Uni.createFrom().item(() ->
          (WorkerResult) ((BiFunction) ep.fn()).apply(input, null));
  }
  ```

**casehub-engine** (main change):
- `WorkerFunctionHandler` interface — new signature (common-core)
- `WorkerExecutor` interface — new signature (common-core)
- `DefaultWorkerExecutor` — updated composite (runtime)
- `SyncWorkerFunctionHandler` — new class, delegates to worker executor (runtime)
- `AgentWorkerFunctionHandler` — new class, engine-local agent execution (runtime)
- `SyncAgentWorkerFunctionHandler` — deleted
- `WorkerExecutionOrchestrator` — ExecutionPolicy enrichment, pass Worker+Capability
- `WorkerExecutionConfig` — add `withEffectiveTimeout(ExecutionPolicy)` convenience
- All other handlers: `PersistentWorkerFunctionHandler`, `FlowWorkerFunctionHandler`, `McpWorkerFunctionHandler`, `ReActWorkerFunctionHandler`, `A2AWorkerFunctionHandler` — mechanical signature updates
- `runtime/pom.xml` — add `casehub-worker-runtime` dependency

**casehub-agency** (follow-on):
- `PatternWorkerFunctionHandler` — mechanical signature update

### 6. Maven Dependency

Add to `runtime/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-worker-runtime</artifactId>
    <version>${version.io.casehub}</version>
</dependency>
```

This brings in `WorkerExecutor`, `DefaultWorkerExecutor`, `SchemaValidator`, and SmallRye Fault Tolerance transitively.

### 7. What Does NOT Change

- **Output projection (JQ):** stays in engine's `DefaultWorkerExecutor.applyOutputSchema()` — applied after handler returns, not inside the handler
- **Handler chain architecture:** `DefaultWorkerExecutor` still iterates `Instance<WorkerFunctionHandler>` and delegates to the first matching handler
- **WorkerRuntimeFactory:** still an engine concern — creates `DefaultWorkerRuntime` with signal/observation/neighbor/rule/stigmergy spaces
- **Scoped worker sessions:** still an engine concern — binding locks and accumulated state managed by `ScopedWorkerRegistry`
- **RetryOrchestrator:** engine's higher-level retry (re-dispatch on failure) is unrelated to the worker executor's Guard-level retry (retries within a single execution)

### 8. Ordering

1. **casehub-worker:** extend `liftToUni()` with ExchangeProcessor (1 commit, publish SNAPSHOT)
2. **casehub-engine:** interface change + handler split + delegation + timeout enrichment + tests (multiple commits)
3. **casehub-agency:** PatternWorkerFunctionHandler signature update (1 commit)

### 9. Testing

- Split `SyncAgentWorkerFunctionHandlerTest` into `SyncWorkerFunctionHandlerTest` + `AgentWorkerFunctionHandlerTest`
- `SyncWorkerFunctionHandlerTest`: verify WorkerRuntime is passed through to worker function despite delegation; verify timeout via Guard (not future.get); verify REINVOKED mode with binding lock and accumulated state
- `AgentWorkerFunctionHandlerTest`: verify invoker execution, protocol metadata extraction, timeout
- `DefaultWorkerExecutorTimeoutTest`: update to pass Worker + Capability
- All handler tests: mechanical signature updates
- New integration test: verify default timeout enrichment when `executionPolicy().timeoutMs()` is null

## References

- `WorkerFunctionHandler.java:34-44` — current interface
- `SyncAgentWorkerFunctionHandler.java` — current combined handler
- `DefaultWorkerExecutor.java` (engine) — composite executor
- `DefaultWorkerExecutor.java` (worker) — Guard, SchemaValidator, OTel, liftToUni
- `WorkerExecutionOrchestrator.java:227-316` — call site with Worker/Capability resolution
- `WorkerExecutionConfig.java:26-28` — effective timeout resolution
- `WorkerRuntimeFactory.java` — engine-specific scope creation
- `PersistentWorkerFunctionHandler.java` — precedent for split handler
- PP-20260723-c4c1cf — virtual-thread handler convention
- casehubio/engine#203 — ContextBridge protocol epic
- casehubio/casehub-worker#10 — worker executor accepts Object input (prerequisite, completed)
