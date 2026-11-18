# Sync Handler Delegates to Worker Executor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #726 — refactor: engine Sync handler delegates to casehub-worker WorkerExecutor
**Issue group:** #726

**Goal:** Eliminate duplicated execution concerns by having the engine's Sync handler delegate to the worker repo's WorkerExecutor for timeout, retry, schema validation, and OTel tracing.

**Architecture:** Change `WorkerFunctionHandler.execute()` and engine's `WorkerExecutor.execute()` to accept `Worker` + `Capability` instead of decomposed fields. Split `SyncAgentWorkerFunctionHandler` into `SyncWorkerFunctionHandler` (delegates to `io.casehub.worker.runtime.WorkerExecutor`) and `AgentWorkerFunctionHandler` (keeps engine-local execution). The Sync handler wraps the Worker's function to inject engine-created `WorkerRuntime` before delegation.

**Tech Stack:** Java 21, Quarkus 3.32.2, SmallRye Mutiny, SmallRye Fault Tolerance, casehub-worker-runtime

## Global Constraints

- Quarkus version: `3.32.2` (from root pom.xml `version.quarkus.platform`)
- Cross-project version: `${version.io.casehub}` = `0.2-SNAPSHOT`
- Virtual-thread protocol (PP-20260723-c4c1cf): no `runSubscriptionOn()` wrappers — handlers use sequential blocking code, `Uni.await().indefinitely()` is correct
- All `WorkerFunctionHandler` implementations are `@ApplicationScoped` (complementary, not `@DefaultBean`)
- Output projection (JQ) stays in engine's `DefaultWorkerExecutor.applyOutputSchema()` — not moved to handlers
- Handlers must NOT import `io.casehub.worker.runtime.*` unless they delegate (only `SyncWorkerFunctionHandler` does)
- Use `ide_refactor_rename` for class renames, `ide_move_file` for moves — never bash mv/cp on .java files

---

## Batch 1: Worker Repo Prerequisite — ExchangeProcessor in liftToUni

### Task 1: Add ExchangeProcessor support to worker executor's liftToUni()

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/worker/runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java:132-140`
- Test: `/Users/mdproctor/claude/casehub/worker/runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java`

**Interfaces:**
- Consumes: `WorkerFunction.ExchangeProcessor` (worker-api), `Exchange` (worker-api)
- Produces: `liftToUni()` now handles `ExchangeProcessor` alongside `Sync`

- [ ] **Step 1: Write failing test for ExchangeProcessor execution**

```java
@Test
void execute_exchangeProcessor_returns_result() {
    var ep = new WorkerFunction.ExchangeProcessor<>(
        String.class, String.class,
        (exchange, scope) -> WorkerResult.of(exchange.withBody("processed: " + exchange.body())));
    Worker worker = Worker.builder()
        .name("ep-worker")
        .capabilityName("process")
        .function(ep)
        .executionPolicy(ExecutionPolicy.noRetry())
        .build();
    Capability capability = Capability.of("process", "{}", "{}");
    Exchange<String> input = Exchange.of("raw");

    WorkerResult result = workerExecutor.execute(worker, capability, input)
        .await().atMost(Duration.ofSeconds(5));

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
    Exchange<?> output = (Exchange<?>) result.output();
    assertThat(output.body()).isEqualTo("processed: raw");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=WorkerExecutorTest#execute_exchangeProcessor_returns_result -f /Users/mdproctor/claude/casehub/worker/pom.xml`

Expected: FAIL with `UnsupportedOperationException: Unsupported function type`

- [ ] **Step 3: Add ExchangeProcessor branch to liftToUni()**

In `DefaultWorkerExecutor.java`, modify the `liftToUni` method to add a branch for `ExchangeProcessor` after the existing `Sync` branch:

```java
@SuppressWarnings({"unchecked", "rawtypes"})
private Uni<WorkerResult> liftToUni(Worker worker, Object input) {
    if (worker.function() instanceof WorkerFunction.Sync sync) {
        return Uni.createFrom().item(() ->
            (WorkerResult) ((java.util.function.BiFunction) sync.fn()).apply(input, null));
    }
    if (worker.function() instanceof WorkerFunction.ExchangeProcessor ep) {
        return Uni.createFrom().item(() ->
            (WorkerResult) ((java.util.function.BiFunction) ep.fn()).apply(input, null));
    }
    throw new UnsupportedOperationException(
        "Unsupported function type: " + worker.function().getClass().getName());
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=WorkerExecutorTest#execute_exchangeProcessor_returns_result -f /Users/mdproctor/claude/casehub/worker/pom.xml`

Expected: PASS

- [ ] **Step 5: Run full worker test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/worker/pom.xml`

Expected: All tests pass

- [ ] **Step 6: Install worker SNAPSHOT locally**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/worker/pom.xml`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add runtime/src/main/java/io/casehub/worker/runtime/DefaultWorkerExecutor.java runtime/src/test/java/io/casehub/worker/runtime/WorkerExecutorTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "feat(#726): add ExchangeProcessor support to WorkerExecutor.liftToUni

Refs casehubio/engine#726"
```

---

## Batch 2: Interface Change — Worker + Capability on Handler and Executor

### Task 2: Change WorkerFunctionHandler interface signature

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerFunctionHandler.java:34-44`

**Interfaces:**
- Consumes: `Worker` (worker-api), `Capability` (worker-api)
- Produces: New `execute(Worker, Capability, Object, WorkerContext, ExecutionMetadata)` signature — all handler implementations depend on this

- [ ] **Step 1: Change the interface**

Use `ide_replace_member` to replace the `execute` method declaration:

```java
HandlerResult execute(
    Worker worker,
    Capability capability,
    Object inputData,
    WorkerContext context,
    ExecutionMetadata metadata);
```

Add imports for `io.casehub.worker.api.Worker` and `io.casehub.worker.api.Capability`.

- [ ] **Step 2: Verify the project does NOT compile**

Run: `mvn compile -pl common-core -q 2>&1 | head -5`

Expected: Compilation succeeds for common-core (interface-only), but downstream modules will fail. This confirms the interface changed.

- [ ] **Step 3: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerFunctionHandler.java
git commit -m "refactor(#726): WorkerFunctionHandler accepts Worker + Capability

Drop WorkerFunction and timeoutMs parameters — both available via
Worker aggregate. Refs #726"
```

### Task 3: Change engine WorkerExecutor interface and DefaultWorkerExecutor

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutor.java:30-39`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/DefaultWorkerExecutor.java:47-81`

**Interfaces:**
- Consumes: Updated `WorkerFunctionHandler.execute()` from Task 2
- Produces: `WorkerExecutor.execute(Worker, Capability, Object, WorkerContext, String outputProjection, ExecutionMetadata)` — called by `WorkerExecutionOrchestrator`

- [ ] **Step 1: Change WorkerExecutor interface**

Use `ide_replace_member` to replace the `execute` method:

```java
HandlerResult execute(
    Worker worker,
    Capability capability,
    Object inputData,
    WorkerContext context,
    String outputProjection,
    ExecutionMetadata metadata);
```

Add imports for `Worker` and `Capability`.

- [ ] **Step 2: Update DefaultWorkerExecutor to match**

Use `ide_replace_member` to update the `execute` method:

```java
@Override
public HandlerResult execute(
    Worker worker,
    Capability capability,
    Object inputData,
    WorkerContext context,
    String outputProjection,
    ExecutionMetadata metadata) {

  for (WorkerFunctionHandler handler : handlers) {
    if (handler.supports(worker.function())) {
      HandlerResult handlerResult =
          handler.execute(worker, capability, inputData, context, metadata);
      WorkerResult<?> result = applyOutputSchema(handlerResult.result(), outputProjection);
      return new HandlerResult(result, handlerResult.protocolMetadata());
    }
  }
  throw new UnsupportedOperationException("No handler for: " + worker.function().getClass().getName());
}
```

- [ ] **Step 3: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutor.java runtime/src/main/java/io/casehub/engine/runtime/executor/DefaultWorkerExecutor.java
git commit -m "refactor(#726): engine WorkerExecutor accepts Worker + Capability

DefaultWorkerExecutor extracts worker.function() for supports() check
and passes Worker + Capability through to handlers. Refs #726"
```

### Task 4: Update WorkerExecutionOrchestrator with ExecutionPolicy enrichment

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java:272,310-316`
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionConfig.java:18-29`

**Interfaces:**
- Consumes: Updated `WorkerExecutor.execute()` from Task 3
- Produces: Enriched Worker with effective timeout passed to executor

- [ ] **Step 1: Add withEffectiveTimeout to WorkerExecutionConfig**

Use `ide_insert_member` to add a method to `WorkerExecutionConfig`:

```java
public ExecutionPolicy withEffectiveTimeout(ExecutionPolicy policy) {
  return new ExecutionPolicy(
      getEffectiveTimeout(policy.timeoutMs()),
      policy.retries(),
      policy.circuitBreaker());
}
```

- [ ] **Step 2: Update the executeWorker call site**

In `WorkerExecutionOrchestrator.executeWorker()`, replace lines 272 and 310-316. Remove:

```java
int timeoutMs = executionConfig.getEffectiveTimeout(worker.executionPolicy().timeoutMs());
```

And replace the `workerExecutor.execute(...)` call:

```java
ExecutionPolicy effectivePolicy = executionConfig.withEffectiveTimeout(worker.executionPolicy());
Worker effectiveWorker = new Worker(worker.name(), worker.capabilities(),
    worker.function(), effectivePolicy, worker.description(), worker.definitionRef());

HandlerResult handlerResult =
    workerExecutor.execute(
        effectiveWorker,
        capability,
        typedInput,
        workerContext,
        capability.outputProjection(),
        metadata);
```

Add import for `io.casehub.platform.api.governance.ExecutionPolicy`.

- [ ] **Step 3: Add test for withEffectiveTimeout**

In `WorkerExecutionConfigTest`, add:

```java
@Test
void withEffectiveTimeout_applies_default_when_null() {
    var config = new WorkerExecutionConfig(30000);
    ExecutionPolicy policy = new ExecutionPolicy(null, null, null);
    ExecutionPolicy effective = config.withEffectiveTimeout(policy);
    assertThat(effective.timeoutMs()).isEqualTo(30000);
    assertThat(effective.retries()).isNull();
}

@Test
void withEffectiveTimeout_preserves_explicit_timeout() {
    var config = new WorkerExecutionConfig(30000);
    ExecutionPolicy policy = new ExecutionPolicy(5000, null, null);
    ExecutionPolicy effective = config.withEffectiveTimeout(policy);
    assertThat(effective.timeoutMs()).isEqualTo(5000);
}
```

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest=WorkerExecutionConfigTest`

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionConfig.java common-core/src/test/java/io/casehub/engine/common/internal/executor/WorkerExecutionConfigTest.java
git commit -m "refactor(#726): enrich ExecutionPolicy at orchestrator call site

Engine default timeout applied to Worker's ExecutionPolicy before
passing to executor. Handlers see effective Worker, not definition-time
Worker. Refs #726"
```

### Task 5: Mechanical signature update — all other handlers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandler.java:67-72`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/PersistentWorkerFunctionHandler.java:70-75`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:82+`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/mcp/McpWorkerFunctionHandler.java:60+`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/react/ReActWorkerFunctionHandler.java:74+`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/a2a/A2AWorkerFunctionHandler.java:67+`

**Interfaces:**
- Consumes: New `WorkerFunctionHandler.execute()` signature from Task 2
- Produces: All handlers compile with new signature

For each handler, use `ide_replace_member` to update the `execute` method signature. The parameter list changes from:

```java
(WorkerFunction<?, ?> function, Object inputData, WorkerContext context, int timeoutMs, ExecutionMetadata metadata)
```

to:

```java
(Worker worker, Capability capability, Object inputData, WorkerContext context, ExecutionMetadata metadata)
```

Inside each method body:
- Replace `function` with `worker.function()` (cast as needed)
- Replace `timeoutMs` with `worker.executionPolicy().timeoutMs()` (where used)
- Add imports for `io.casehub.worker.api.Worker` and `io.casehub.worker.api.Capability`

- [ ] **Step 1: Update SyncAgentWorkerFunctionHandler** (temporary — will be replaced in Batch 3)

Change signature. Replace `function` with `worker.function()`. Replace `timeoutMs` with `worker.executionPolicy().timeoutMs()`.

- [ ] **Step 2: Update PersistentWorkerFunctionHandler**

Change signature. Replace `(WorkerFunction.Persistent<?>) function` with `(WorkerFunction.Persistent<?>) worker.function()`. This handler ignores `timeoutMs`, so no timeout change needed.

- [ ] **Step 3: Update FlowWorkerFunctionHandler**

Change signature. Extract function from worker and cast. Replace any `timeoutMs` reference with `worker.executionPolicy().timeoutMs()`.

- [ ] **Step 4: Update McpWorkerFunctionHandler**

Same pattern as Step 3.

- [ ] **Step 5: Update ReActWorkerFunctionHandler**

Same pattern as Step 3.

- [ ] **Step 6: Update A2AWorkerFunctionHandler**

Same pattern as Step 3.

- [ ] **Step 7: Verify compilation**

Run: `mvn compile -DskipTests -q`

Expected: All modules compile successfully.

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandler.java runtime/src/main/java/io/casehub/engine/runtime/executor/PersistentWorkerFunctionHandler.java engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java engine-support-core/src/main/java/io/casehub/engine/mcp/McpWorkerFunctionHandler.java engine-support-core/src/main/java/io/casehub/engine/react/ReActWorkerFunctionHandler.java engine-support-core/src/main/java/io/casehub/engine/a2a/A2AWorkerFunctionHandler.java
git commit -m "refactor(#726): update all handler signatures to Worker + Capability

Mechanical change — extract worker.function() and
worker.executionPolicy().timeoutMs() where needed. Refs #726"
```

### Task 6: Update all handler tests to new signature

**Files:**
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandlerTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/DefaultWorkerExecutorTimeoutTest.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/mcp/McpWorkerFunctionHandlerTest.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/react/ReActWorkerFunctionHandlerTest.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/a2a/A2AWorkerFunctionHandlerTest.java` or `a2a/src/test/java/...`

**Interfaces:**
- Consumes: New handler/executor signatures from Tasks 2-5
- Produces: All tests pass with new signatures

For each test, update `handler.execute(...)` and `workerExecutor.execute(...)` calls:

- Create a `Worker` wrapping the test's `WorkerFunction`:
  ```java
  Worker worker = new Worker("test-worker", Set.of("cap"), syncFunction,
      new ExecutionPolicy(5000, null, null), null, null);
  Capability capability = Capability.of("cap", "{}", "{}");
  ```
- Replace `handler.execute(function, inputData, context, timeoutMs, metadata)` with `handler.execute(worker, capability, inputData, context, metadata)`
- Replace `workerExecutor.execute(function, inputData, context, timeoutMs, outputProjection, metadata)` with `workerExecutor.execute(worker, capability, inputData, context, outputProjection, metadata)`

- [ ] **Step 1: Update SyncAgentWorkerFunctionHandlerTest**

For each test method, wrap the test function in a Worker and create a Capability. E.g. for `executes_sync_function`:

```java
var sync = new WorkerFunction.Sync<>(Map.class, Map.class,
    (input, scope) -> WorkerResult.of(Map.of("result", input.get("key"))));
Worker worker = new Worker("test-worker", Set.of("cap"), sync,
    new ExecutionPolicy(5000, null, null), null, null);
Capability capability = Capability.of("cap", "{}", "{}");
WorkerResult<?> result = handler.execute(worker, capability,
    Map.of("key", "value"), testContext(), new ExecutionMetadata("w1", "hash1")).result();
```

For the timeout test, set timeout to 200 on the ExecutionPolicy:

```java
Worker worker = new Worker("test-worker", Set.of("cap"), slowWorker,
    new ExecutionPolicy(200, null, null), null, null);
```

- [ ] **Step 2: Update DefaultWorkerExecutorTimeoutTest**

```java
Worker worker = new Worker("test-worker", Set.of("cap"), slowWorker,
    new ExecutionPolicy(200, null, null), null, null);
Capability capability = Capability.of("cap", "{}", "{}");
var handlerResult = workerExecutor.execute(worker, capability,
    Map.of(), context, null, new ExecutionMetadata("test-worker", "hash-1"));
```

- [ ] **Step 3: Update remaining handler tests**

Same pattern for Flow, MCP, ReAct, A2A tests — wrap test functions in Worker, create Capability, update execute calls.

- [ ] **Step 4: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test`

Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/test/ engine-support-core/src/test/ a2a/src/test/
git commit -m "test(#726): update all handler and executor tests for Worker + Capability

Wrap test functions in Worker records, create Capability instances,
update execute() calls to new signature. Refs #726"
```

---

## Batch 3: Handler Split + Delegation

### Task 7: Add casehub-worker-runtime dependency

**Files:**
- Modify: `runtime/pom.xml`

**Interfaces:**
- Produces: `io.casehub.worker.runtime.WorkerExecutor` available for injection in runtime module

- [ ] **Step 1: Add dependency to runtime/pom.xml**

Add after the existing casehub dependencies:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-worker-runtime</artifactId>
    <version>${version.io.casehub}</version>
</dependency>
```

- [ ] **Step 2: Verify compilation**

Run: `mvn compile -pl runtime -q`

Expected: Compiles successfully, `io.casehub.worker.runtime.WorkerExecutor` resolvable.

- [ ] **Step 3: Commit**

```bash
git add runtime/pom.xml
git commit -m "build(#726): add casehub-worker-runtime dependency to engine runtime

Enables delegation to worker executor for timeout, retry, schema
validation, OTel. Refs #726"
```

### Task 8: Create SyncWorkerFunctionHandler with delegation

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/runtime/executor/SyncWorkerFunctionHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/executor/SyncWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `io.casehub.worker.runtime.WorkerExecutor` (worker-runtime), `WorkerRuntimeFactory` (engine), `ScopedWorkerRegistry` (engine)
- Produces: `WorkerFunctionHandler` that handles `Sync` and `ExchangeAwareFunction`, delegates to worker executor

- [ ] **Step 1: Write failing test — Sync function delegates and passes WorkerRuntime**

```java
@Test
void sync_function_receives_engine_runtime_despite_delegation() {
    var runtimeRef = new java.util.concurrent.atomic.AtomicReference<io.casehub.worker.api.WorkerScope>();
    var sync = new WorkerFunction.Sync<>(Map.class, Map.class,
        (input, scope) -> {
            runtimeRef.set(scope);
            return WorkerResult.of(Map.of("result", "ok"));
        });
    Worker worker = new Worker("test-worker", Set.of("cap"), sync,
        new ExecutionPolicy(5000, null, null), null, null);
    Capability capability = Capability.of("cap", "{}", "{}");

    handler.execute(worker, capability, Map.of(), testContext(),
        new ExecutionMetadata("test-worker", "hash"));

    assertThat(runtimeRef.get()).isNotNull();
    assertThat(runtimeRef.get()).isInstanceOf(io.casehub.api.engine.WorkerRuntime.class);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `SyncWorkerFunctionHandler` class doesn't exist yet.

- [ ] **Step 3: Implement SyncWorkerFunctionHandler**

```java
package io.casehub.engine.runtime.executor;

import io.casehub.api.model.WorkerContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.HandlerResult;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.engine.common.internal.worker.scope.ScopedWorkerRegistry;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class SyncWorkerFunctionHandler implements WorkerFunctionHandler {

  private final io.casehub.worker.runtime.WorkerExecutor workerExecutor;
  private final WorkerRuntimeFactory workerRuntimeFactory;
  private final ScopedWorkerRegistry scopedWorkerRegistry;

  @Inject
  public SyncWorkerFunctionHandler(
      io.casehub.worker.runtime.WorkerExecutor workerExecutor,
      WorkerRuntimeFactory workerRuntimeFactory,
      ScopedWorkerRegistry scopedWorkerRegistry) {
    this.workerExecutor = workerExecutor;
    this.workerRuntimeFactory = workerRuntimeFactory;
    this.scopedWorkerRegistry = scopedWorkerRegistry;
  }

  @Override
  public boolean supports(WorkerFunction<?, ?> function) {
    return function instanceof WorkerFunction.Sync
        || function instanceof io.casehub.worker.api.ExchangeAwareFunction;
  }

  @SuppressWarnings("unchecked")
  @Override
  public HandlerResult execute(
      Worker worker,
      Capability capability,
      Object inputData,
      WorkerContext context,
      ExecutionMetadata metadata) {

    if (inputData == null) {
      throw new io.casehub.api.context.BridgeTypeMismatchException(
          worker.function().inputType().getName(), "null");
    }
    if (!worker.function().inputType().isInstance(inputData)) {
      throw new io.casehub.api.context.BridgeTypeMismatchException(
          worker.function().inputType().getName(), inputData.getClass().getName());
    }

    Map<String, Object> accState = Map.of();
    ReentrantLock bindingLock = null;
    if (metadata.executionMode() == io.casehub.api.model.ExecutionMode.REINVOKED
        && metadata.bindingName() != null) {
      var scopeKey = new ScopedWorkerRegistry.ScopeKey(context.caseId(), metadata.bindingName());
      bindingLock = scopedWorkerRegistry.executionLock(scopeKey);
      bindingLock.lock();
      accState = scopedWorkerRegistry.get(context.caseId(), metadata.bindingName())
          .filter(io.casehub.engine.common.internal.worker.scope.ScopedWorkerSession.Reinvoked.class::isInstance)
          .map(s -> ((io.casehub.engine.common.internal.worker.scope.ScopedWorkerSession.Reinvoked) s)
              .accumulatedState().get())
          .orElse(Map.of());
    }

    io.casehub.api.engine.WorkerRuntime runtime =
        workerRuntimeFactory.create(context.caseId(), metadata.workerName(), context, accState);

    Worker effectiveWorker = wrapFunction(worker, runtime);

    try {
      WorkerResult<?> result = workerExecutor.execute(effectiveWorker, capability, inputData)
          .await().indefinitely();

      if (metadata.executionMode() == io.casehub.api.model.ExecutionMode.REINVOKED
          && metadata.bindingName() != null) {
        Map<String, Object> output = result.output() instanceof Map
            ? (Map<String, Object>) result.output() : null;
        if (output != null) {
          scopedWorkerRegistry.get(context.caseId(), metadata.bindingName())
              .filter(io.casehub.engine.common.internal.worker.scope.ScopedWorkerSession.Reinvoked.class::isInstance)
              .ifPresent(s -> ((io.casehub.engine.common.internal.worker.scope.ScopedWorkerSession.Reinvoked) s)
                  .accumulatedState().set(output));
        }
      }
      return new HandlerResult(result);
    } finally {
      if (bindingLock != null) {
        bindingLock.unlock();
      }
    }
  }

  @SuppressWarnings({"unchecked", "rawtypes"})
  private Worker wrapFunction(Worker worker, io.casehub.api.engine.WorkerRuntime runtime) {
    WorkerFunction<?, ?> wrapped;
    if (worker.function() instanceof WorkerFunction.Sync sync) {
      var originalFn = (java.util.function.BiFunction) sync.fn();
      wrapped = new WorkerFunction.Sync<>(sync.inputType(), sync.outputType(),
          (input, _ignored) -> originalFn.apply(input, runtime));
    } else if (worker.function() instanceof WorkerFunction.ExchangeProcessor ep) {
      var originalFn = (java.util.function.BiFunction) ep.fn();
      wrapped = new WorkerFunction.ExchangeProcessor<>(ep.inputType(), ep.outputType(),
          (input, _ignored) -> originalFn.apply(input, runtime));
    } else {
      throw new UnsupportedOperationException("Cannot wrap: " + worker.function().getClass().getName());
    }
    return new Worker(worker.name(), worker.capabilities(), wrapped,
        worker.executionPolicy(), worker.description(), worker.definitionRef());
  }
}
```

- [ ] **Step 4: Write the test class with mock worker executor**

```java
package io.casehub.engine.runtime.executor;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.WorkerContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.platform.api.governance.ExecutionPolicy;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.smallrye.mutiny.Uni;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicReference;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SyncWorkerFunctionHandlerTest {

  private WorkerFunctionHandler handler;
  private AtomicReference<Worker> capturedWorker;

  @BeforeEach
  void setUp() {
    capturedWorker = new AtomicReference<>();
    WorkerRuntimeFactory mockFactory = new WorkerRuntimeFactory(
        null, null, null, null, null, null, null, null, null, null) {
      @Override
      public io.casehub.api.engine.WorkerRuntime create(
          UUID caseId, String taskId, WorkerContext context,
          Map<String, Object> accumulatedState) {
        return new DefaultWorkerRuntime(
            caseId, taskId, context, accumulatedState,
            null, null, null, null, null, null);
      }
    };
    io.casehub.worker.runtime.WorkerExecutor mockExecutor =
        (worker, capability, input) -> {
          capturedWorker.set(worker);
          if (worker.function() instanceof WorkerFunction.Sync sync) {
            @SuppressWarnings({"unchecked", "rawtypes"})
            WorkerResult result = ((java.util.function.BiFunction) sync.fn()).apply(input, null);
            return Uni.createFrom().item(result);
          }
          if (worker.function() instanceof WorkerFunction.ExchangeProcessor ep) {
            @SuppressWarnings({"unchecked", "rawtypes"})
            WorkerResult result = ((java.util.function.BiFunction) ep.fn()).apply(input, null);
            return Uni.createFrom().item(result);
          }
          return Uni.createFrom().failure(new UnsupportedOperationException());
        };
    handler = new SyncWorkerFunctionHandler(
        mockExecutor, mockFactory,
        new io.casehub.engine.common.internal.worker.scope.ScopedWorkerRegistry());
  }

  @Test
  void supports_sync() {
    var sync = new WorkerFunction.Sync<>(Map.class, Map.class,
        (input, scope) -> WorkerResult.of(Map.of()));
    assertThat(handler.supports(sync)).isTrue();
  }

  @Test
  void supports_exchange_processor() {
    var ep = new WorkerFunction.ExchangeProcessor<>(String.class, String.class,
        (exchange, scope) -> WorkerResult.of(exchange.withBody("done")));
    assertThat(handler.supports(ep)).isTrue();
  }

  @Test
  void does_not_support_agent() {
    assertThat(handler.supports(
        new io.casehub.api.model.AgentWorkerFunction(null))).isFalse();
  }

  @Test
  void sync_function_receives_engine_runtime_despite_delegation() {
    var runtimeRef = new AtomicReference<io.casehub.worker.api.WorkerScope>();
    var sync = new WorkerFunction.Sync<>(Map.class, Map.class,
        (input, scope) -> {
          runtimeRef.set(scope);
          return WorkerResult.of(Map.of("result", "ok"));
        });
    Worker worker = new Worker("test-worker", Set.of("cap"), sync,
        new ExecutionPolicy(5000, null, null), null, null);
    Capability capability = Capability.of("cap", "{}", "{}");

    handler.execute(worker, capability, Map.of(), testContext(),
        new ExecutionMetadata("test-worker", "hash"));

    assertThat(runtimeRef.get()).isNotNull();
    assertThat(runtimeRef.get()).isInstanceOf(io.casehub.api.engine.WorkerRuntime.class);
  }

  @Test
  void executes_sync_function() {
    var sync = new WorkerFunction.Sync<>(Map.class, Map.class,
        (input, scope) -> WorkerResult.of(Map.of("result", input.get("key"))));
    Worker worker = new Worker("test-worker", Set.of("cap"), sync,
        new ExecutionPolicy(5000, null, null), null, null);
    Capability capability = Capability.of("cap", "{}", "{}");

    WorkerResult<?> result = handler.execute(worker, capability,
        Map.of("key", "value"), testContext(),
        new ExecutionMetadata("w1", "hash1")).result();

    assertThat((Map<String, Object>) result.output()).containsEntry("result", "value");
  }

  private WorkerContext testContext() {
    return new WorkerContext("test-worker", UUID.randomUUID(),
        null, null, io.casehub.api.context.PropagationContext.createRoot(), null);
  }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=SyncWorkerFunctionHandlerTest`

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/executor/SyncWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/runtime/executor/SyncWorkerFunctionHandlerTest.java
git commit -m "feat(#726): SyncWorkerFunctionHandler delegates to worker executor

Wraps Worker function to inject engine-created WorkerRuntime before
delegation. Worker executor provides timeout (Guard), retry, schema
validation, OTel. Engine handler manages scoped sessions. Refs #726"
```

### Task 9: Create AgentWorkerFunctionHandler

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `AgentWorkerFunction` (api), `WorkerFunctionHandler` (common-core)
- Produces: Handler for Agent functions — engine-local execution with invoker and protocol metadata

- [ ] **Step 1: Write failing test — Agent handler returns protocol metadata**

```java
@Test
void executes_agent_function_with_protocol_metadata() {
    WorkerAgentInvoker invoker = stubInvoker("{\"agentResult\": \"done\"}");
    AgentWorkerFunction agentFunction = new AgentWorkerFunction(invoker);
    Worker worker = new Worker("agent-w", Set.of("cap"), agentFunction,
        new ExecutionPolicy(5000, null, null), null, null);
    Capability capability = Capability.of("cap", "{}", "{}");

    var handlerResult = handler.execute(worker, capability,
        Map.of("input", "data"), testContext(),
        new ExecutionMetadata("agent-w", "hash2"));

    assertThat((Map<String, Object>) handlerResult.result().output())
        .containsEntry("agentResult", "done");
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `AgentWorkerFunctionHandler` class doesn't exist.

- [ ] **Step 3: Implement AgentWorkerFunctionHandler**

Extract the Agent execution path from `SyncAgentWorkerFunctionHandler`. The Agent handler keeps the virtual-thread-based execution with `future.get()` timeout — it does NOT delegate to the worker executor because `AgentWorkerFunction` has no `Sync` fn().

```java
package io.casehub.engine.runtime.executor;

import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.WorkerContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.HandlerResult;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.quarkus.virtual.threads.VirtualThreads;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.concurrent.ExecutorService;

@ApplicationScoped
public class AgentWorkerFunctionHandler implements WorkerFunctionHandler {

  private final ExecutorService virtualThreads;

  @Inject
  public AgentWorkerFunctionHandler(@VirtualThreads ExecutorService virtualThreads) {
    this.virtualThreads = virtualThreads;
  }

  @Override
  public boolean supports(WorkerFunction<?, ?> function) {
    return function instanceof AgentWorkerFunction;
  }

  @SuppressWarnings("unchecked")
  @Override
  public HandlerResult execute(
      Worker worker,
      Capability capability,
      Object inputData,
      WorkerContext context,
      ExecutionMetadata metadata) {

    AgentWorkerFunction agent = (AgentWorkerFunction) worker.function();
    int timeoutMs = worker.executionPolicy().timeoutMs() != null
        ? worker.executionPolicy().timeoutMs() : 30_000;

    var invocationResultRef =
        new java.util.concurrent.atomic.AtomicReference<
            io.casehub.platform.agent.AgentEvent.InvocationComplete>();

    try {
      var future = virtualThreads.submit(() -> {
        var response = agent.invoker().executeDetailed((Map<String, Object>) inputData);
        invocationResultRef.set(response.invocationResult());
        return response.result();
      });
      WorkerResult<?> result = future.get(timeoutMs, java.util.concurrent.TimeUnit.MILLISECONDS);

      io.casehub.platform.agent.AgentEvent.InvocationComplete ir = invocationResultRef.get();
      if (ir != null) {
        return new HandlerResult(result, Map.of(
            "inputTokens", ir.inputTokens(),
            "outputTokens", ir.outputTokens(),
            "thinkingTokens", ir.thinkingTokens(),
            "cacheReadTokens", ir.cacheReadTokens(),
            "cacheWriteTokens", ir.cacheWriteTokens(),
            "totalCostUsd", ir.totalCostUsd() != null ? ir.totalCostUsd() : 0.0,
            "durationMs", ir.durationMs(),
            "backendKey", ir.backendKey() != null ? ir.backendKey() : "",
            "instanceId", ir.instanceId() != null ? ir.instanceId() : ""));
      }
      return new HandlerResult(result);
    } catch (java.util.concurrent.TimeoutException e) {
      return new HandlerResult(WorkerResult.expired("Agent timed out after " + timeoutMs + "ms"));
    } catch (java.util.concurrent.ExecutionException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException re) { throw re; }
      throw new RuntimeException(cause);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new RuntimeException("Agent execution interrupted", e);
    }
  }
}
```

- [ ] **Step 4: Write full test class**

Port the agent-related tests from `SyncAgentWorkerFunctionHandlerTest`, adapted for the new class and signature. Include `supports_agent`, `does_not_support_sync`, `executes_agent_function`, `timeout_produces_expired_outcome`.

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest`

Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java
git commit -m "feat(#726): AgentWorkerFunctionHandler — engine-local agent execution

Keeps invoker, protocol metadata, token tracking. Own timeout via
future.get() — Agent path does not delegate to worker executor. Refs #726"
```

### Task 10: Delete SyncAgentWorkerFunctionHandler and its test

**Files:**
- Delete: `runtime/src/main/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandler.java`
- Delete: `runtime/src/test/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `SyncWorkerFunctionHandler` (Task 8) and `AgentWorkerFunctionHandler` (Task 9) now cover all function types that `SyncAgentWorkerFunctionHandler` handled

- [ ] **Step 1: Delete the old handler and test**

Use `ide_refactor_safe_delete` for `SyncAgentWorkerFunctionHandler`.
Use `ide_refactor_safe_delete` for `SyncAgentWorkerFunctionHandlerTest`.

- [ ] **Step 2: Verify no dangling references**

Run: `mvn compile -DskipTests -q`

Expected: Compiles — no code references the deleted class.

- [ ] **Step 3: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test`

Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git rm runtime/src/main/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/runtime/executor/SyncAgentWorkerFunctionHandlerTest.java
git commit -m "refactor(#726): delete SyncAgentWorkerFunctionHandler

Replaced by SyncWorkerFunctionHandler (delegates to worker executor)
and AgentWorkerFunctionHandler (engine-local). Refs #726"
```

---

## Batch 4: Agency Cross-Repo Update

### Task 11: Update PatternWorkerFunctionHandler in casehub-agency

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/agency/engine-adapter-core/src/main/java/io/casehub/engine/agentic/PatternWorkerFunctionHandler.java`
- Modify: `/Users/mdproctor/claude/casehub/agency/engine-adapter/src/test/java/io/casehub/engine/agentic/PatternWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: Updated `WorkerFunctionHandler` interface from engine common-core

- [ ] **Step 1: Update handler signature**

Same mechanical pattern as Task 5: change `execute()` parameter list, replace `function` with `worker.function()`, replace `timeoutMs` with `worker.executionPolicy().timeoutMs()`.

- [ ] **Step 2: Update test**

Same pattern as Task 6: wrap test functions in Worker, create Capability, update execute calls.

- [ ] **Step 3: Install engine SNAPSHOT**

Run: `mvn install -DskipTests -q` (from engine root)

- [ ] **Step 4: Compile and test agency**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/agency/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -f /Users/mdproctor/claude/casehub/agency/pom.xml`

Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/agency add engine-adapter-core/src/main/java/io/casehub/engine/agentic/PatternWorkerFunctionHandler.java engine-adapter/src/test/java/io/casehub/engine/agentic/PatternWorkerFunctionHandlerTest.java
git -C /Users/mdproctor/claude/casehub/agency commit -m "refactor(#726): PatternWorkerFunctionHandler accepts Worker + Capability

Mechanical signature update to match engine WorkerFunctionHandler
interface change. Refs casehubio/engine#726"
```

---

## References

- [2026-11-18-sync-handler-delegates-worker-executor-design.md] — design spec this plan implements
- `WorkerFunctionHandler.java:34-44` — current interface (changes in Task 2)
- `SyncAgentWorkerFunctionHandler.java` — current combined handler (split in Tasks 8-10)
- `DefaultWorkerExecutor.java` (engine, runtime) — composite executor (changes in Task 3)
- `DefaultWorkerExecutor.java` (worker repo) — Guard, SchemaValidator, OTel, liftToUni (changes in Task 1)
- `WorkerExecutionOrchestrator.java:227-316` — call site (changes in Task 4)
- `WorkerExecutionConfig.java:26-28` — effective timeout (changes in Task 4)
- `Worker.java`, `Capability.java` — record APIs for constructing test instances
- PP-20260723-c4c1cf — virtual-thread handler convention
- casehubio/engine#726 — focal issue
