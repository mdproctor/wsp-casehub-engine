# Remove Serverlessworkflow SDK Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the serverlessworkflow SDK from all engine modules except flow, replacing the hardcoded dispatch in DefaultWorkerExecutor with a pluggable handler model.

**Architecture:** Two new SPIs decouple function construction (WorkerFunctionProvider) from function execution (WorkerFunctionHandler). The YAML mapper delegates SDK-dependent construction to providers. The executor delegates dispatch to handlers. FlowWorkerFunction moves to the flow module — the SDK never leaves.

**Tech Stack:** Java 21, Quarkus 3.32, CDI (Quarkus ARC), Mutiny, Jackson, Serverless Workflow SDK (flow module only)

## Global Constraints

- All changes on branch `issue-567-remove-serverlessworkflow-sdk` in both project (`/Users/mdproctor/claude/casehub/engine`) and workspace (`/Users/mdproctor/claude/public/casehub/engine`) repos
- Foundation-tier change to `casehub-worker-api` (`/Users/mdproctor/claude/casehub/worker`) must be installed locally before engine tasks
- Every commit references `Refs #567`
- Build with `mvn install -DskipTests -q` before running module-specific tests
- Always include `TESTCONTAINERS_RYUK_DISABLED=true` when running tests
- Use IntelliJ MCP for all code navigation — never bash grep/find for classes
- Use `java-dev` skill for all Java implementation work

## Cross-Repo Dependency

The `casehub-worker-api` change (Task 1) is in a separate repo (`/Users/mdproctor/claude/casehub/worker`). It must be committed, built, and installed locally before any engine tasks can compile. Engine depends on `casehub-worker-api:0.2-SNAPSHOT` which resolves from `~/.m2/repository`.

---

### Task 1: Foundation — WorkerFunction becomes marker interface

**Repo:** `/Users/mdproctor/claude/casehub/worker` (casehub-worker-api)

**Files:**
- Modify: `api/src/main/java/io/casehub/worker/api/WorkerFunction.java`
- Modify: `api/src/test/java/io/casehub/worker/api/WorkerFunctionTest.java` (if exists)

**Interfaces:**
- Produces: `WorkerFunction` marker interface (no `execute()` method), `WorkerFunction.Sync` record with `fn()` accessor only

- [ ] **Step 1: Write the failing test**

Create or update test to verify marker interface contract:

```java
package io.casehub.worker.api;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.Map;
import org.junit.jupiter.api.Test;

class WorkerFunctionTest {

  @Test
  void sync_is_workerFunction() {
    WorkerFunction fn = new WorkerFunction.Sync(input -> WorkerResult.of(Map.of()));
    assertThat(fn).isInstanceOf(WorkerFunction.class);
  }

  @Test
  void sync_fn_accessor_returns_function() {
    var sync = new WorkerFunction.Sync(input -> WorkerResult.of(Map.of("key", "value")));
    WorkerResult result = sync.fn().apply(Map.of());
    assertThat(result.output()).containsEntry("key", "value");
  }

  @Test
  void workerFunction_has_no_execute_method() throws Exception {
    assertThat(WorkerFunction.class.getDeclaredMethods()).isEmpty();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test -pl api -Dtest=WorkerFunctionTest -q`

Expected: FAIL — `WorkerFunction` still has `execute()` method, `workerFunction_has_no_execute_method` fails.

- [ ] **Step 3: Remove execute() from WorkerFunction and Sync**

Modify `api/src/main/java/io/casehub/worker/api/WorkerFunction.java`:

```java
package io.casehub.worker.api;

import java.util.Map;
import java.util.function.Function;

public interface WorkerFunction {

    record Sync(Function<Map<String, Object>, WorkerResult> fn) implements WorkerFunction {
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test -pl api -Dtest=WorkerFunctionTest -q`

Expected: PASS

- [ ] **Step 5: Install to local Maven repo**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml install -DskipTests -q`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add api/src/main/java/io/casehub/worker/api/WorkerFunction.java api/src/test/java/io/casehub/worker/api/WorkerFunctionTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "refactor(#567): WorkerFunction becomes marker interface — remove execute()

Refs casehubio/engine#567"
```

---

### Task 2: Delete execute() from AgentWorkerFunction and FlowWorkerFunction

**Repo:** `/Users/mdproctor/claude/casehub/engine`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java`
- Modify: `api/src/main/java/io/casehub/api/model/FlowWorkerFunction.java`
- Modify: `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/FlowWorkerFunctionTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/WorkerFunctionTest.java`

**Interfaces:**
- Consumes: `WorkerFunction` marker interface from Task 1
- Produces: `AgentWorkerFunction(Agent)` record (no `execute()`), `FlowWorkerFunction(Workflow)` record (no `execute()`) — FlowWorkerFunction still in api temporarily, moves in Task 7

- [ ] **Step 1: Delete execute() override from AgentWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.api.model.ai.Agent;
import io.casehub.worker.api.WorkerFunction;
import java.util.Objects;

public record AgentWorkerFunction(Agent agent) implements WorkerFunction {
  public AgentWorkerFunction {
    Objects.requireNonNull(agent);
  }
}
```

- [ ] **Step 2: Delete execute() override from FlowWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.worker.api.WorkerFunction;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Objects;

public record FlowWorkerFunction(Workflow workflow) implements WorkerFunction {
  public FlowWorkerFunction {
    Objects.requireNonNull(workflow);
  }
}
```

- [ ] **Step 3: Update tests**

Remove tests that assert `execute()` throws. Update `WorkerFunctionTest` to remove `instanceof` dispatch test (dead pattern — dispatch moves to handlers). Update `FlowWorkerFunctionTest` to remove `executeThrowsUnsupported` test. Update `AgentWorkerFunctionTest` to remove any `execute()` references.

- [ ] **Step 4: Build to verify compilation**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`

Expected: Compile errors in `DefaultWorkerExecutor.java` (references `execute()`-dependent patterns). Fix compilation: the switch in `DefaultWorkerExecutor` calls `sync.fn()`, `agent.agent()::execute`, and `flow.workflow()` — none of these call `WorkerFunction.execute()`, so no changes needed in the executor. Verify.

- [ ] **Step 5: Run api tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl api -q`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#567): delete execute() from AgentWorkerFunction and FlowWorkerFunction

Both threw UnsupportedOperationException — LSP violation. With
WorkerFunction as a marker interface, the dead overrides are removed.

Refs #567"
```

---

### Task 3: WorkerFunctionHandler SPI and SyncAgentWorkerFunctionHandler

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/WorkerFunctionHandler.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `WorkerFunction` (marker), `WorkerContext`, `ExecutionMetadata` from common
- Produces: `WorkerFunctionHandler` SPI interface, `SyncAgentWorkerFunctionHandler` handling `Sync` and `AgentWorkerFunction`

- [ ] **Step 1: Create WorkerFunctionHandler SPI in common**

```java
package io.casehub.engine.common.internal.executor;

import io.casehub.api.model.WorkerContext;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.smallrye.mutiny.Uni;
import java.util.Map;

public interface WorkerFunctionHandler {

  boolean supports(WorkerFunction function);

  Uni<WorkerResult> execute(
      WorkerFunction function,
      Map<String, Object> inputData,
      WorkerContext context,
      int timeoutMs,
      ExecutionMetadata metadata);
}
```

- [ ] **Step 2: Write failing test for SyncAgentWorkerFunctionHandler**

```java
package io.casehub.engine.internal.executor;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.WorkerContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SyncAgentWorkerFunctionHandlerTest {

  @Test
  void supports_sync() {
    WorkerFunctionHandler handler = createHandler();
    var sync = new WorkerFunction.Sync(input -> WorkerResult.of(Map.of()));
    assertThat(handler.supports(sync)).isTrue();
  }

  @Test
  void supports_agent() {
    WorkerFunctionHandler handler = createHandler();
    assertThat(handler.supports(new AgentWorkerFunction(testAgent()))).isTrue();
  }

  @Test
  void does_not_support_unknown() {
    WorkerFunctionHandler handler = createHandler();
    WorkerFunction unknown = new WorkerFunction() {};
    assertThat(handler.supports(unknown)).isFalse();
  }

  @Test
  void executes_sync_function() {
    WorkerFunctionHandler handler = createHandler();
    var sync = new WorkerFunction.Sync(
        input -> WorkerResult.of(Map.of("result", input.get("key"))));
    WorkerResult result = handler.execute(
            sync, Map.of("key", "value"),
            testContext(), 5000,
            new ExecutionMetadata("w1", "hash1"))
        .await().indefinitely();
    assertThat(result.output()).containsEntry("result", "value");
  }

  // Helper methods: createHandler(), testContext(), testAgent()
  // Use same test patterns as DefaultWorkerExecutorTimeoutTest
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl runtime -Dtest=SyncAgentWorkerFunctionHandlerTest -q`

Expected: FAIL — class not found

- [ ] **Step 4: Implement SyncAgentWorkerFunctionHandler**

Extract the `executeSync()` logic from `DefaultWorkerExecutor` into the handler. The handler:
1. Pattern-matches on `Sync` and `AgentWorkerFunction`
2. Runs the function on a virtual thread (`@VirtualThreads ExecutorService`)
3. Sets/clears `WorkerExecutionContext` around the call
4. Enforces timeout via `ifNoItem().after(Duration.ofMillis(timeoutMs)).fail()`
5. Recovers `TimeoutException` as `WorkerResult.expired()`

```java
package io.casehub.engine.internal.executor;

import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerExecutionContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.quarkus.virtual.threads.VirtualThreads;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.function.Function;

@ApplicationScoped
public class SyncAgentWorkerFunctionHandler implements WorkerFunctionHandler {

  private final ExecutorService virtualThreads;

  @Inject
  public SyncAgentWorkerFunctionHandler(
      @VirtualThreads ExecutorService virtualThreads) {
    this.virtualThreads = virtualThreads;
  }

  @Override
  public boolean supports(WorkerFunction function) {
    return function instanceof WorkerFunction.Sync
        || function instanceof AgentWorkerFunction;
  }

  @Override
  public Uni<WorkerResult> execute(
      WorkerFunction function,
      Map<String, Object> inputData,
      WorkerContext context,
      int timeoutMs,
      ExecutionMetadata metadata) {

    Function<Map<String, Object>, WorkerResult> fn =
        switch (function) {
          case WorkerFunction.Sync sync -> sync.fn()::apply;
          case AgentWorkerFunction agent -> agent.agent()::execute;
          default -> throw new UnsupportedOperationException(
              "Unsupported: " + function.getClass().getName());
        };

    return Uni.createFrom()
        .item(() -> {
          WorkerExecutionContext.set(context);
          try {
            return fn.apply(inputData);
          } finally {
            WorkerExecutionContext.clear();
          }
        })
        .runSubscriptionOn(virtualThreads)
        .ifNoItem()
        .after(Duration.ofMillis(timeoutMs))
        .fail()
        .onFailure(io.smallrye.mutiny.TimeoutException.class)
        .recoverWithItem(
            t -> WorkerResult.expired("Worker timed out after " + timeoutMs + "ms"));
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl runtime -Dtest=SyncAgentWorkerFunctionHandlerTest -q`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/main/java/io/casehub/engine/common/internal/executor/WorkerFunctionHandler.java runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#567): WorkerFunctionHandler SPI and SyncAgentWorkerFunctionHandler

Engine-internal SPI in common for pluggable function dispatch.
SyncAgentWorkerFunctionHandler handles Sync and AgentWorkerFunction
with virtual thread execution and timeout enforcement.

Refs #567"
```

---

### Task 4: Refactor DefaultWorkerExecutor to composite

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerExecutor.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerExecutorTimeoutTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerExecutorFlowContextTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionHandler` from Task 3, `WorkerExecutor` from common
- Produces: Composite `DefaultWorkerExecutor` that dispatches to handlers and applies output schema post-processing

- [ ] **Step 1: Write failing test for composite dispatch**

Add test to `DefaultWorkerExecutorTimeoutTest` (or a new test class) that verifies the composite dispatches to the handler and applies output schema:

```java
@Test
void dispatches_to_handler_and_applies_output_schema() {
    // Create a handler that returns a known result
    // Verify output schema is applied after handler returns
    // Verify handler.supports() is called before execute()
}
```

- [ ] **Step 2: Refactor DefaultWorkerExecutor**

Replace the switch-based dispatch with handler iteration. The `executeSync()` and `executeFlow()` private methods are removed. Output schema evaluation stays in the executor as `.map(result -> applyOutputSchema(result, outputSchema))`.

```java
@ApplicationScoped
public class DefaultWorkerExecutor implements WorkerExecutor {

  private final Instance<WorkerFunctionHandler> handlers;
  private final JQEvaluator jqEvaluator;

  @Inject
  public DefaultWorkerExecutor(
      Instance<WorkerFunctionHandler> handlers,
      JQEvaluator jqEvaluator) {
    this.handlers = handlers;
    this.jqEvaluator = jqEvaluator;
  }

  @Override
  public Uni<WorkerResult> execute(
      WorkerFunction function,
      Map<String, Object> inputData,
      WorkerContext context,
      int timeoutMs,
      String outputSchema,
      ExecutionMetadata metadata) {

    for (WorkerFunctionHandler handler : handlers) {
      if (handler.supports(function)) {
        return handler.execute(function, inputData, context, timeoutMs, metadata)
            .map(result -> applyOutputSchema(result, outputSchema));
      }
    }
    throw new UnsupportedOperationException(
        "No handler for: " + function.getClass().getName());
  }

  // applyOutputSchema() stays unchanged
}
```

- [ ] **Step 3: Delete NoOpWorkflowExecutor**

Delete `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkflowExecutor.java`. Without a flow handler on the classpath, the composite simply has no handler for flow functions — no no-op fallback needed.

- [ ] **Step 4: Build and fix compilation**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`

Fix any compilation errors. The `DefaultWorkerExecutor` no longer imports `FlowWorkerFunction`, `WorkflowExecutor`, or `VirtualThreads` directly. The existing tests (`DefaultWorkerExecutorTimeoutTest`, `DefaultWorkerExecutorFlowContextTest`) will need updates — they construct `DefaultWorkerExecutor` directly with constructor args that have changed.

- [ ] **Step 5: Run runtime tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl runtime -q`

Expected: PASS (some flow-specific tests may fail if they still depend on `NoOpWorkflowExecutor` — address in Task 8)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/ common/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#567): DefaultWorkerExecutor becomes composite over handlers

Replaces hardcoded switch dispatch with Instance<WorkerFunctionHandler>
iteration. Output schema evaluation applied as post-processing. Deletes
NoOpWorkflowExecutor — no no-op needed with handler model.

Refs #567"
```

---

### Task 5: WorkerFunctionProvider SPI and registry

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/WorkerFunctionProvider.java`
- Create: `api/src/main/java/io/casehub/api/spi/WorkerFunctionProviderRegistry.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/DefaultWorkerFunctionProviderRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerFunctionProviderRegistryTest.java`

**Interfaces:**
- Produces: `WorkerFunctionProvider` SPI, `WorkerFunctionProviderRegistry` interface, `DefaultWorkerFunctionProviderRegistry` CDI implementation

- [ ] **Step 1: Create WorkerFunctionProvider SPI**

```java
package io.casehub.api.spi;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.worker.api.WorkerFunction;

public interface WorkerFunctionProvider {
  boolean handles(JsonNode rawWorkerNode);
  WorkerFunction create(JsonNode rawWorkerNode);
}
```

- [ ] **Step 2: Create WorkerFunctionProviderRegistry interface**

```java
package io.casehub.api.spi;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.worker.api.WorkerFunction;

public interface WorkerFunctionProviderRegistry {
  WorkerFunction createFunction(JsonNode rawWorkerNode);
}
```

Returns `null` when no provider handles the node.

- [ ] **Step 3: Write failing test for DefaultWorkerFunctionProviderRegistry**

```java
package io.casehub.engine.internal.worker;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.spi.WorkerFunctionProvider;
import io.casehub.api.spi.WorkerFunctionProviderRegistry;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class DefaultWorkerFunctionProviderRegistryTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void returns_null_when_no_provider_handles() {
    WorkerFunctionProviderRegistry registry =
        new DefaultWorkerFunctionProviderRegistry(List.of());
    ObjectNode node = MAPPER.createObjectNode().put("name", "test");
    assertThat(registry.createFunction(node)).isNull();
  }

  @Test
  void delegates_to_matching_provider() {
    WorkerFunctionProvider provider = new WorkerFunctionProvider() {
      @Override
      public boolean handles(JsonNode raw) {
        return raw.has("do");
      }

      @Override
      public WorkerFunction create(JsonNode raw) {
        return new WorkerFunction.Sync(input -> WorkerResult.of(Map.of("flow", true)));
      }
    };

    WorkerFunctionProviderRegistry registry =
        new DefaultWorkerFunctionProviderRegistry(List.of(provider));

    ObjectNode node = MAPPER.createObjectNode();
    node.putArray("do");

    WorkerFunction fn = registry.createFunction(node);
    assertThat(fn).isNotNull().isInstanceOf(WorkerFunction.Sync.class);
  }
}
```

- [ ] **Step 4: Implement DefaultWorkerFunctionProviderRegistry**

```java
package io.casehub.engine.internal.worker;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.spi.WorkerFunctionProvider;
import io.casehub.api.spi.WorkerFunctionProviderRegistry;
import io.casehub.worker.api.WorkerFunction;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class DefaultWorkerFunctionProviderRegistry
    implements WorkerFunctionProviderRegistry {

  private final Iterable<WorkerFunctionProvider> providers;

  @Inject
  public DefaultWorkerFunctionProviderRegistry(
      Instance<WorkerFunctionProvider> providers) {
    this.providers = providers;
  }

  DefaultWorkerFunctionProviderRegistry(
      Iterable<WorkerFunctionProvider> providers) {
    this.providers = providers;
  }

  @Override
  public WorkerFunction createFunction(JsonNode rawWorkerNode) {
    for (WorkerFunctionProvider provider : providers) {
      if (provider.handles(rawWorkerNode)) {
        return provider.create(rawWorkerNode);
      }
    }
    return null;
  }
}
```

- [ ] **Step 5: Run test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl runtime -Dtest=DefaultWorkerFunctionProviderRegistryTest -q`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/WorkerFunctionProvider.java api/src/main/java/io/casehub/api/spi/WorkerFunctionProviderRegistry.java runtime/src/main/java/io/casehub/engine/internal/worker/DefaultWorkerFunctionProviderRegistry.java runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerFunctionProviderRegistryTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#567): WorkerFunctionProvider SPI and registry

WorkerFunctionProvider for YAML mapper delegation. Registry interface
in api/spi/, DefaultWorkerFunctionProviderRegistry in runtime with
Instance<WorkerFunctionProvider> iteration.

Refs #567"
```

---

### Task 6: Schema module — remove SDK from WorkerMarshaller and Worker

**Files:**
- Modify: `schema/src/main/java/io/casehub/model/Worker.java`
- Modify: `schema/src/main/java/io/casehub/model/marshaller/WorkerMarshaller.java`
- Modify: `schema/pom.xml`
- Modify: `schema/src/test/java/` (any tests referencing Workflow)

**Interfaces:**
- Produces: `Worker.hasWorkflowDefinition()`, `Worker.getWorkflowDefinition()` returning `JsonNode`

- [ ] **Step 1: Modify Worker.java**

Replace `Workflow`-typed methods with `JsonNode`:

```java
// Remove import: io.serverlessworkflow.api.types.Workflow

// Change workflow field type check methods:
public boolean hasWorkflowDefinition() {
    return workflow instanceof JsonNode;
}

public JsonNode getWorkflowDefinition() {
    return (JsonNode) workflow;
}

// Delete: isEmbeddedWorkflow(), getWorkflowAsEmbedded()
// Keep: isWorkflowRef(), getWorkflowAsRef() (String path — unchanged)
```

- [ ] **Step 2: Modify WorkerMarshaller.Deserializer**

Replace `WorkflowReader.readWorkflowFromString()` with raw `JsonNode` storage:

```java
// In Deserializer.deserialize():
// Replace the embedded workflow block:
} else if (root.has("do")) {
    // Store raw workflow fields as JsonNode instead of parsing to Workflow
    ObjectNode workflowFields = mapper.createObjectNode();
    if (root.has("document")) workflowFields.set("document", root.get("document"));
    if (root.has("do")) workflowFields.set("do", root.get("do"));
    if (root.has("input")) workflowFields.set("input", root.get("input"));
    if (root.has("output")) workflowFields.set("output", root.get("output"));
    if (root.has("use")) workflowFields.set("use", root.get("use"));
    if (root.has("schedule")) workflowFields.set("schedule", root.get("schedule"));
    if (root.has("timeout")) workflowFields.set("timeout", root.get("timeout"));

    if (!workflowFields.has("document")) {
        ObjectNode documentNode = mapper.createObjectNode();
        documentNode.put("dsl", "1.0.0");
        documentNode.put("name", "my-name");
        documentNode.put("namespace", "my-namespace");
        documentNode.put("version", "1.0.0");
        workflowFields.set("document", documentNode);
    }

    worker.setWorkflow(workflowFields);
}
```

- [ ] **Step 3: Simplify WorkerMarshaller.Serializer**

Replace `Workflow`-specific field extraction with raw `JsonNode` write:

```java
// In Serializer.serialize():
if (value.getWorkflow() != null) {
    if (value.isWorkflowRef()) {
        gen.writeStringField("workflow", value.getWorkflowAsRef());
    } else if (value.hasWorkflowDefinition()) {
        // Write raw JSON node fields inline
        JsonNode wfNode = value.getWorkflowDefinition();
        for (var it = wfNode.fields(); it.hasNext(); ) {
            var field = it.next();
            gen.writeObjectField(field.getKey(), field.getValue());
        }
    }
}
```

- [ ] **Step 4: Remove quarkus-flow from schema/pom.xml**

Remove the `quarkus-flow` dependency. Remove all `io.serverlessworkflow` imports from Worker.java and WorkerMarshaller.java.

- [ ] **Step 5: Build and run schema tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`
Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl schema -q`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add schema/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#567): schema stores workflow as raw JsonNode, drops quarkus-flow

WorkerMarshaller no longer parses do: blocks into Workflow objects.
Embedded workflow definitions stored as JsonNode — the flow module
deserializes them when it constructs the function.

Refs #567"
```

---

### Task 7: CaseDefinitionYamlMapper + YamlCaseHub — use provider registry

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/main/java/io/casehub/api/engine/YamlCaseHub.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`
- Delete: `api/src/main/java/io/casehub/api/model/FlowWorkerFunction.java`
- Delete: `api/src/test/java/io/casehub/api/model/FlowWorkerFunctionTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionProviderRegistry` from Task 5, schema `Worker.getWorkflowDefinition()` from Task 6
- Produces: Updated mapper signature: `load(InputStream, ObjectMapper, ExpressionEngineRegistry, WorkerFunctionProviderRegistry)`

- [ ] **Step 1: Add WorkerFunctionProviderRegistry to mapper load()**

Add a `WorkerFunctionProviderRegistry` parameter to both `load()` overloads. The no-arg overload uses an empty registry:

```java
private static final WorkerFunctionProviderRegistry EMPTY_PROVIDERS =
    rawWorkerNode -> null;

public static CaseDefinition load(final InputStream yamlStream) throws IOException {
    return load(yamlStream, new ObjectMapper(new YAMLFactory()), JQ_ONLY, EMPTY_PROVIDERS);
}

public static CaseDefinition load(
    final InputStream yamlStream,
    final ObjectMapper objectMapper,
    final ExpressionEngineRegistry registry,
    final WorkerFunctionProviderRegistry providerRegistry)
    throws IOException {
    // ... existing logic, pass providerRegistry to convertToApiModel
}
```

- [ ] **Step 2: Change worker construction to delegate to providers**

In `convertToApiModel()`, replace the hardcoded `FlowWorkerFunction` construction:

```java
// Try providers first (for SDK-dependent types like flow)
JsonNode rawWorkerNode = rawNode.get("spec").get("workers").get(workerIndex);
WorkerFunction function = providerRegistry.createFunction(rawWorkerNode);
if (function == null) {
    // API-local construction (no external SDK dependency)
    if (sw.getAgent() != null) {
        final io.casehub.api.model.ai.Agent apiAgent =
            AgentConverter.toApiAgent(sw.getAgent());
        function = new AgentWorkerFunction(apiAgent);
    } else {
        function = new WorkerFunction.Sync(
            input -> io.casehub.worker.api.WorkerResult.of(input));
    }
}
workerBuilder.function(function);
```

- [ ] **Step 3: Update YamlCaseHub to inject and pass the registry**

```java
public class YamlCaseHub extends CaseHub {
    @Inject ExpressionEngineRegistry expressionEngineRegistry;
    @Inject @YamlMapper ObjectMapper objectMapper;
    @Inject WorkerFunctionProviderRegistry workerFunctionProviderRegistry;

    // ...

    definition = CaseDefinitionYamlMapper.load(
        is, objectMapper, expressionEngineRegistry, workerFunctionProviderRegistry);
}
```

- [ ] **Step 4: Delete FlowWorkerFunction from api**

Delete `api/src/main/java/io/casehub/api/model/FlowWorkerFunction.java` and its test. Remove from `api/pom.xml` the `serverlessworkflow-experimental-fluent-func` dependency.

- [ ] **Step 5: Update WorkerFunctionTest to remove FlowWorkerFunction references**

Remove `flowWorkerFunction_implements_workerFunction` test and `instanceof` dispatch test from `WorkerFunctionTest.java`.

- [ ] **Step 6: Build and run api tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`
Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl api -q`

Expected: PASS (or compile errors from downstream modules referencing deleted FlowWorkerFunction — addressed in Tasks 8-9)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#567): YAML mapper delegates to WorkerFunctionProviderRegistry

CaseDefinitionYamlMapper.load() takes WorkerFunctionProviderRegistry.
Providers handle SDK-dependent construction (flow). Agent/Sync remain
inline. Deletes FlowWorkerFunction from api — serverlessworkflow dep
removed from engine-api.

Refs #567"
```

---

### Task 8: Flow module — FlowWorkerFunction, provider, and handler

**Files:**
- Create: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java`
- Create: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionProvider.java`
- Create: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java`
- Delete: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerExecutor.java`
- Modify: `flow/src/test/java/io/casehub/engine/flow/FlowWorkerExecutorTest.java` → rename/refactor to `FlowWorkerFunctionHandlerTest.java`
- Create: `flow/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionProviderTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionProvider` from Task 5, `WorkerFunctionHandler` from Task 3, `WorkflowApplication` (quarkus-flow CDI bean), `FlowExecutionRegistry` (existing flow class)
- Produces: `FlowWorkerFunction(Workflow)`, `FlowWorkerFunctionProvider`, `FlowWorkerFunctionHandler`

- [ ] **Step 1: Create FlowWorkerFunction in flow module**

```java
package io.casehub.engine.flow;

import io.casehub.worker.api.WorkerFunction;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Objects;

public record FlowWorkerFunction(Workflow workflow) implements WorkerFunction {
  public FlowWorkerFunction {
    Objects.requireNonNull(workflow);
  }
}
```

- [ ] **Step 2: Write failing test for FlowWorkerFunctionProvider**

```java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;

class FlowWorkerFunctionProviderTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void handles_workers_with_do_block() {
    var provider = new FlowWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    node.putArray("do");
    node.putObject("document").put("dsl", "1.0.0")
        .put("name", "test").put("namespace", "test").put("version", "1.0.0");
    assertThat(provider.handles(node)).isTrue();
  }

  @Test
  void does_not_handle_workers_without_do_block() {
    var provider = new FlowWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode().put("name", "test");
    assertThat(provider.handles(node)).isFalse();
  }

  @Test
  void creates_flow_worker_function() {
    var provider = new FlowWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    node.putObject("document").put("dsl", "1.0.0")
        .put("name", "test").put("namespace", "test").put("version", "1.0.0");
    node.putArray("do");

    var fn = provider.create(node);
    assertThat(fn).isInstanceOf(FlowWorkerFunction.class);
  }
}
```

- [ ] **Step 3: Implement FlowWorkerFunctionProvider**

```java
package io.casehub.engine.flow;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.spi.WorkerFunctionProvider;
import io.casehub.worker.api.WorkerFunction;
import io.serverlessworkflow.api.WorkflowFormat;
import io.serverlessworkflow.api.WorkflowReader;
import io.serverlessworkflow.api.types.Workflow;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class FlowWorkerFunctionProvider implements WorkerFunctionProvider {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Override
  public boolean handles(JsonNode rawWorkerNode) {
    return rawWorkerNode.has("do");
  }

  @Override
  public WorkerFunction create(JsonNode rawWorkerNode) {
    try {
      // Ensure document node exists for WorkflowReader
      if (!rawWorkerNode.has("document")) {
        var copy = rawWorkerNode.deepCopy();
        ((com.fasterxml.jackson.databind.node.ObjectNode) copy)
            .putObject("document")
            .put("dsl", "1.0.0")
            .put("name", "generated")
            .put("namespace", "generated")
            .put("version", "1.0.0");
        rawWorkerNode = copy;
      }
      Workflow workflow = WorkflowReader.readWorkflowFromString(
          MAPPER.writeValueAsString(rawWorkerNode), WorkflowFormat.YAML);
      return new FlowWorkerFunction(workflow);
    } catch (Exception e) {
      throw new IllegalArgumentException(
          "Failed to parse workflow definition from YAML", e);
    }
  }
}
```

- [ ] **Step 4: Write failing test for FlowWorkerFunctionHandler**

Adapt `FlowWorkerExecutorTest` patterns to test the handler interface.

- [ ] **Step 5: Implement FlowWorkerFunctionHandler**

Merge `FlowWorkerExecutor` logic into the handler:

```java
package io.casehub.engine.flow;

import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerExecutionContext;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.WorkerFunctionHandler;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.serverlessworkflow.impl.WorkflowApplication;
import io.serverlessworkflow.impl.WorkflowInstance;
import io.serverlessworkflow.impl.WorkflowModel;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

@ApplicationScoped
public class FlowWorkerFunctionHandler implements WorkerFunctionHandler {

  private final WorkflowApplication app;
  private final FlowExecutionRegistry registry;

  @Inject
  public FlowWorkerFunctionHandler(
      WorkflowApplication app, FlowExecutionRegistry registry) {
    this.app = app;
    this.registry = registry;
  }

  @Override
  public boolean supports(WorkerFunction function) {
    return function instanceof FlowWorkerFunction;
  }

  @Override
  public Uni<WorkerResult> execute(
      WorkerFunction function,
      Map<String, Object> inputData,
      WorkerContext context,
      int timeoutMs,
      ExecutionMetadata metadata) {

    FlowWorkerFunction flow = (FlowWorkerFunction) function;

    return Uni.createFrom()
        .completionStage(() -> {
          WorkerExecutionContext.set(context);
          try {
            return executeWorkflow(
                flow.workflow(), inputData, context.caseId(),
                metadata.workerName(), metadata.inputDataHash());
          } finally {
            WorkerExecutionContext.clear();
          }
        })
        .map(model -> WorkerResult.of(
            model.asMap().orElseThrow(() -> new RuntimeException(
                "Workflow produced non-serializable model for worker: "
                    + metadata.workerName()))));
  }

  private CompletableFuture<WorkflowModel> executeWorkflow(
      io.serverlessworkflow.api.types.Workflow workflow,
      Map<String, Object> inputData,
      java.util.UUID caseId,
      String workerName,
      String inputDataHash) {

    WorkflowInstance wfInstance =
        app.workflowDefinition(workflow).instance(inputData);
    String instanceId = wfInstance.id();

    registry.register(instanceId, caseId, workerName, inputDataHash);
    try {
      CompletableFuture<WorkflowModel> future = wfInstance.start();
      future.whenComplete((model, ex) -> registry.remove(instanceId));
      return future;
    } catch (RuntimeException e) {
      registry.remove(instanceId);
      throw e;
    }
  }
}
```

- [ ] **Step 6: Delete FlowWorkerExecutor**

Delete `flow/src/main/java/io/casehub/engine/flow/FlowWorkerExecutor.java`. Rename/update `FlowWorkerExecutorTest.java` to `FlowWorkerFunctionHandlerTest.java`.

- [ ] **Step 7: Run flow tests**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`
Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -pl flow -q`

Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add flow/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#567): flow module owns FlowWorkerFunction, provider, and handler

FlowWorkerFunction moves from api to flow. FlowWorkerFunctionProvider
handles do: block construction. FlowWorkerFunctionHandler replaces
FlowWorkerExecutor — merges execution logic into handler model.
Serverlessworkflow SDK stays exclusively in flow.

Refs #567"
```

---

### Task 9: Delete WorkflowExecutor, dependency cleanup, and runtime test migration

**Files:**
- Delete: `common/src/main/java/io/casehub/engine/common/internal/worker/WorkflowExecutor.java`
- Modify: `common/pom.xml` — remove `serverlessworkflow-api`, `serverlessworkflow-impl-core`
- Modify: `api/pom.xml` — remove `serverlessworkflow-experimental-fluent-func` (if not already done in Task 7)
- Modify: `runtime/pom.xml` — remove `quarkus-flow` and `serverlessworkflow-experimental-fluent-func` from production deps; keep `serverlessworkflow-experimental-fluent-func` as test dep only if needed
- Modify: `schema/pom.xml` — verify `quarkus-flow` removed (done in Task 6)
- Modify: `runtime/src/test/java/io/casehub/engine/SimpleCaseHubBean.java` — rewrite to `Sync`
- Modify: `runtime/src/test/java/io/casehub/engine/MultiWorkerPipelineBean.java` — rewrite to `Sync`
- Modify: `runtime/src/test/java/io/casehub/engine/AgentPipelineBean.java` — rewrite to `Sync`
- Modify: `runtime/src/test/java/io/casehub/engine/YamlSimpleCaseHubBeanTest.java` — add flow test dep
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/ServerlessWorkflowExecutor.java` — delete (test-only impl of deleted SPI)

**Interfaces:**
- Consumes: All prior tasks complete

- [ ] **Step 1: Delete WorkflowExecutor SPI from common**

Delete `common/src/main/java/io/casehub/engine/common/internal/worker/WorkflowExecutor.java`.

- [ ] **Step 2: Remove serverlessworkflow deps from common/pom.xml**

Remove `serverlessworkflow-api` and `serverlessworkflow-impl-core` dependencies.

- [ ] **Step 3: Remove SDK deps from runtime/pom.xml**

Remove `quarkus-flow` and `serverlessworkflow-experimental-fluent-func` from production dependencies. Add `casehub-engine-flow` as a test dependency (needed for `YamlSimpleCaseHubBeanTest` and any integration tests that test flow dispatch through the composite).

- [ ] **Step 4: Rewrite runtime test CaseHub beans to use WorkerFunction.Sync**

For `SimpleCaseHubBean`, `MultiWorkerPipelineBean`, `AgentPipelineBean`: replace `new FlowWorkerFunction(workflow(...).tasks(function(...)).build())` with `new WorkerFunction.Sync(input -> { /* same lambda body */ })`. The worker function body (the lambda inside the workflow step) moves directly to the `Sync` constructor.

Delete `ServerlessWorkflowExecutor.java` from runtime test sources (test-only impl of the deleted `WorkflowExecutor` SPI).

- [ ] **Step 5: Update YamlSimpleCaseHubBeanTest**

The test asserts `instanceof FlowWorkerFunction`. Update the import to `io.casehub.engine.flow.FlowWorkerFunction`. The flow module test dep added in Step 3 makes this available.

- [ ] **Step 6: Build and run full test suite**

Run: `/opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml install -DskipTests -q`
Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn -f /Users/mdproctor/claude/casehub/engine/pom.xml test -q`

Expected: PASS across all modules

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add .
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#567): delete WorkflowExecutor, clean deps, migrate runtime tests

Deletes WorkflowExecutor SPI from common. Removes serverlessworkflow
SDK from common, runtime, api production deps. Rewrites runtime test
CaseHub beans to use WorkerFunction.Sync. Adds casehub-engine-flow
as runtime test dep for YAML loading tests.

Closes #567"
```

---

## Task Dependency Graph

```
Task 1 (worker-api)
  ↓
Task 2 (delete execute overrides)
  ↓
Task 3 (handler SPI + SyncAgent handler) ──┐
  ↓                                         │
Task 4 (composite executor)                 │
  ↓                                         │
Task 5 (provider SPI + registry) ───────────┤
  ↓                                         │
Task 6 (schema SDK removal)                 │
  ↓                                         │
Task 7 (mapper + YamlCaseHub + delete Flow) │
  ↓                                         │
Task 8 (flow module: function + provider + handler)
  ↓
Task 9 (cleanup: delete WorkflowExecutor, deps, test migration)
```

Tasks 3 and 5 are independent of each other and could run in parallel. Tasks 6 and 7 are sequential (mapper depends on schema changes). Task 8 depends on Tasks 3, 5, 6, 7. Task 9 depends on all prior tasks.
