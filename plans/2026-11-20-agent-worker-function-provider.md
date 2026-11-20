# Agent Worker Function Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1333 — call: agent worker function provider for eidos dispatch
**Issue group:** #1333

**Goal:** Promote agent workers from an inline converter fallback to a first-class `WorkerFunctionProvider` SPI participant, add `AgentSlotConfig` for deferred slot-based resolution, and add `AgentTaskTarget` as a new `BindingTarget` permit for capability-based agent dispatch.

**Architecture:** New `AgentWorkerFunctionProvider` in `engine-support-core` handles `agent:` YAML key and creates `AgentWorkerFunction` with slot-based deferred resolution via `AgentSlotConfig`. The existing `AgentWorkerFunctionHandler` in `runtime` is extended to resolve slots at execution time through `AgentRegistry` → `SystemPromptRenderer` → `AgentProvider`. A new `AgentTaskTarget` sealed permit on `BindingTarget` enables capability-based agent matching via `AgentRegistry.find(AgentQuery)`.

**Tech Stack:** Java 21 (sealed interfaces, records, pattern matching), Quarkus 3.32.2, CDI, eidos-api (`AgentRegistry`, `AgentQuery`, `AgentMatch`, `SystemPromptRenderer`), platform-agent-api (`AgentProvider`)

## Global Constraints

- YAML and Java are peer representations — every YAML field wraps existing Java API
- `AgentWorkerFunction` must remain backward-compatible (existing eager invoker path unchanged)
- Module placement: provider in `engine-support-core`, target in `api`, handler changes in `runtime`
- Tests use `InMemoryAgentRegistry` or `NoOpAgentRegistry` — no live eidos required
- `BindingTarget` is sealed — adding `AgentTaskTarget` causes compiler errors at all exhaustive switch sites; fix them all

---

## Batch 1: Foundation — AgentSlotConfig, AgentWorkerFunction dual-mode, and AgentTaskTarget

### Task 1: AgentSlotConfig record and AgentWorkerFunction dual-mode construction

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/ai/AgentSlotConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java`
- Test: `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java`

**Interfaces:**
- Consumes: `WorkerAgentInvoker` (existing)
- Produces: `AgentSlotConfig(String slot, String model, String userMessageTemplate, String inputProjection, String outputProjection)` record. `AgentWorkerFunction.eager(WorkerAgentInvoker)` and `AgentWorkerFunction.ofSlot(AgentSlotConfig)` factory methods. `AgentWorkerFunction.isDeferred()`, `AgentWorkerFunction.slotConfig()`, `AgentWorkerFunction.invoker()` accessors.

- [ ] **Step 1: Write failing tests for AgentSlotConfig and dual-mode AgentWorkerFunction**

Add to existing `AgentWorkerFunctionTest.java`:

```java
@Test
void ofSlot_createsDeferred() {
    var config = new AgentSlotConfig("sentiment-analyst", null, null, null, null);
    var fn = AgentWorkerFunction.ofSlot(config);
    assertThat(fn.isDeferred()).isTrue();
    assertThat(fn.slotConfig()).isEqualTo(config);
    assertThat(fn.invoker()).isNull();
}

@Test
void eager_createsImmediate() {
    var fn = AgentWorkerFunction.eager(testInvoker());
    assertThat(fn.isDeferred()).isFalse();
    assertThat(fn.invoker()).isNotNull();
    assertThat(fn.slotConfig()).isNull();
}

@Test
void slotConfig_withOverrides() {
    var config = new AgentSlotConfig("my-slot", "claude-sonnet-5-5", "Analyse: {{text}}", ".input", ".output");
    assertThat(config.slot()).isEqualTo("my-slot");
    assertThat(config.model()).isEqualTo("claude-sonnet-5-5");
    assertThat(config.userMessageTemplate()).isEqualTo("Analyse: {{text}}");
    assertThat(config.inputProjection()).isEqualTo(".input");
    assertThat(config.outputProjection()).isEqualTo(".output");
}

@Test
void ofSlot_nullSlot_throws() {
    assertThatThrownBy(() -> new AgentSlotConfig(null, null, null, null, null))
        .isInstanceOf(NullPointerException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentWorkerFunctionTest -q`
Expected: compilation failures — `AgentSlotConfig` and factory methods don't exist yet

- [ ] **Step 3: Create AgentSlotConfig record**

Create `api/src/main/java/io/casehub/api/model/ai/AgentSlotConfig.java`:

```java
package io.casehub.api.model.ai;

import java.util.Objects;

public record AgentSlotConfig(
    String slot,
    String model,
    String userMessageTemplate,
    String inputProjection,
    String outputProjection) {
  public AgentSlotConfig {
    Objects.requireNonNull(slot, "slot is required");
  }
}
```

- [ ] **Step 4: Modify AgentWorkerFunction for dual-mode**

Change `AgentWorkerFunction` from `record AgentWorkerFunction(WorkerAgentInvoker invoker)` to support both modes. Keep backward compatibility — the existing `new AgentWorkerFunction(invoker)` constructor must still work.

```java
@SuppressWarnings("unchecked")
public record AgentWorkerFunction(WorkerAgentInvoker invoker, AgentSlotConfig slotConfig)
    implements WorkerFunction<java.util.Map<String, Object>, java.util.Map<String, Object>> {

  public static AgentWorkerFunction eager(WorkerAgentInvoker invoker) {
    return new AgentWorkerFunction(java.util.Objects.requireNonNull(invoker), null);
  }

  public static AgentWorkerFunction ofSlot(AgentSlotConfig slotConfig) {
    return new AgentWorkerFunction(null, java.util.Objects.requireNonNull(slotConfig));
  }

  public AgentWorkerFunction(WorkerAgentInvoker invoker) {
    this(invoker, null);
  }

  public boolean isDeferred() {
    return slotConfig != null;
  }

  @Override
  public Class<java.util.Map<String, Object>> inputType() {
    return (Class) java.util.Map.class;
  }

  @Override
  public Class<java.util.Map<String, Object>> outputType() {
    return (Class) java.util.Map.class;
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentWorkerFunctionTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/ai/AgentSlotConfig.java api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java
git commit -m "feat(#1333): add AgentSlotConfig and dual-mode AgentWorkerFunction

Refs #1333"
```

### Task 2: AgentTaskTarget binding target

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/AgentTaskTarget.java`
- Modify: `api/src/main/java/io/casehub/api/model/BindingTarget.java:27-33` — add `AgentTaskTarget` to sealed permits
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java` — add `agentTask()` convenience builder method
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/routing/BindingExecutorResolver.java:34-50` — add `AgentTaskTarget` case
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CaseDefinitionService.java:83-103` — add `AgentTaskTarget` cases to both switch methods
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java:54-63` — add `AgentTaskTarget` case
- Test: `api/src/test/java/io/casehub/api/model/AgentTaskTargetTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/BindingTest.java:67-77` — add `AgentTaskTarget` to sealed hierarchy test

**Interfaces:**
- Consumes: `BindingTarget` sealed interface (existing), `ExpressionEvaluator` (existing)
- Produces: `AgentTaskTarget` with builder: `.capability(String)`, `.domain(String)`, `.inputMapping(String)`, `.outputMapping(String)`, `.build()`. Accessors: `capability()`, `domain()`, `inputMapping()`, `outputMapping()`.

- [ ] **Step 1: Write failing tests for AgentTaskTarget**

Create `api/src/test/java/io/casehub/api/model/AgentTaskTargetTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class AgentTaskTargetTest {

  @Test
  void builder_requiredFieldsOnly() {
    AgentTaskTarget target = AgentTaskTarget.builder()
        .capability("sentiment-analysis")
        .build();
    assertThat(target.capability()).isEqualTo("sentiment-analysis");
    assertThat(target.domain()).isNull();
    assertThat(target.inputMapping()).isNull();
    assertThat(target.outputMapping()).isNull();
    assertThat(target).isInstanceOf(BindingTarget.class);
  }

  @Test
  void builder_allFields() {
    AgentTaskTarget target = AgentTaskTarget.builder()
        .capability("sentiment-analysis")
        .domain("customer-support")
        .inputMapping(".ticket")
        .outputMapping(".analysis")
        .build();
    assertThat(target.capability()).isEqualTo("sentiment-analysis");
    assertThat(target.domain()).isEqualTo("customer-support");
    assertThat(target.inputMapping()).isNotNull();
    assertThat(target.outputMapping()).isNotNull();
  }

  @Test
  void builder_nullCapability_throws() {
    assertThatThrownBy(() -> AgentTaskTarget.builder().build())
        .isInstanceOf(NullPointerException.class);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentTaskTargetTest -q`
Expected: compilation failure — `AgentTaskTarget` doesn't exist

- [ ] **Step 3: Create AgentTaskTarget class**

Create `api/src/main/java/io/casehub/api/model/AgentTaskTarget.java`:

```java
package io.casehub.api.model;

import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.Objects;

public final class AgentTaskTarget implements BindingTarget {

  private final String capability;
  private final String domain;
  private final ExpressionEvaluator inputMapping;
  private final ExpressionEvaluator outputMapping;

  private AgentTaskTarget(Builder builder) {
    this.capability = Objects.requireNonNull(builder.capability, "capability is required");
    this.domain = builder.domain;
    this.inputMapping = builder.inputMapping;
    this.outputMapping = builder.outputMapping;
  }

  public String capability() { return capability; }
  public String domain() { return domain; }
  public ExpressionEvaluator inputMapping() { return inputMapping; }
  public ExpressionEvaluator outputMapping() { return outputMapping; }

  public static Builder builder() { return new Builder(); }

  public static final class Builder {
    private String capability;
    private String domain;
    private ExpressionEvaluator inputMapping;
    private ExpressionEvaluator outputMapping;

    public Builder capability(String capability) { this.capability = capability; return this; }
    public Builder domain(String domain) { this.domain = domain; return this; }
    public Builder inputMapping(String jq) { this.inputMapping = new JQExpressionEvaluator(jq); return this; }
    public Builder inputMapping(ExpressionEvaluator e) { this.inputMapping = e; return this; }
    public Builder outputMapping(String jq) { this.outputMapping = new JQExpressionEvaluator(jq); return this; }
    public Builder outputMapping(ExpressionEvaluator e) { this.outputMapping = e; return this; }
    public AgentTaskTarget build() { return new AgentTaskTarget(this); }
  }
}
```

- [ ] **Step 4: Add AgentTaskTarget to BindingTarget sealed permits**

In `BindingTarget.java`, add `AgentTaskTarget` to the permits list:

```java
public sealed interface BindingTarget
    permits CapabilityTarget,
        SubCaseTarget,
        HumanTaskTarget,
        JudgmentTarget,
        SignalTarget,
        AgentTaskTarget,
        ExtensionTarget {}
```

- [ ] **Step 5: Fix all exhaustive switch sites**

The compiler will now fail at every exhaustive switch on `BindingTarget`. Add `case AgentTaskTarget` to each:

**`BindingExecutorResolver.resolve()`** (`common-core`):
```java
case AgentTaskTarget at -> ExecutorRef.of(at.capability());
```

**`CaseDefinitionService.targetType()`** (`rest-core`):
```java
case AgentTaskTarget ignored -> "agentTask";
```

**`CaseDefinitionService.targetName()`** (`rest-core`):
```java
case AgentTaskTarget at -> at.capability();
```

**`CompensationGraphProjection.targetTypeName()`** (`graphql`):
```java
case AgentTaskTarget ignored -> "agent-task";
```

Use `ide_find_references` on `BindingTarget` and `mvn compile` to find any additional switch sites beyond these four.

- [ ] **Step 6: Add Binding.Builder convenience method**

Add to `Binding.Builder`:
```java
public Builder agentTask(AgentTaskTarget agentTask) {
    this.target = agentTask;
    return this;
}
```

- [ ] **Step 7: Update BindingTest sealed hierarchy test**

Add `AgentTaskTarget` assertion to `target_sealedHierarchy_allPermitsReachable`:
```java
AgentTaskTarget agentTask = AgentTaskTarget.builder().capability("test").build();
assertThat(agentTask).isInstanceOf(BindingTarget.class);
```

- [ ] **Step 8: Run all tests to verify**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common-core,rest-core,graphql -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/AgentTaskTarget.java api/src/main/java/io/casehub/api/model/BindingTarget.java api/src/main/java/io/casehub/api/model/Binding.java common-core/src/main/java/io/casehub/engine/common/internal/routing/BindingExecutorResolver.java rest-core/src/main/java/io/casehub/engine/rest/service/CaseDefinitionService.java graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java api/src/test/java/io/casehub/api/model/AgentTaskTargetTest.java api/src/test/java/io/casehub/api/model/BindingTest.java
git commit -m "feat(#1333): add AgentTaskTarget as sealed BindingTarget permit

Refs #1333"
```

---

## Batch 2: Provider and Handler — AgentWorkerFunctionProvider and slot resolution

### Task 3: AgentWorkerFunctionProvider

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java`
- Test: `engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionProvider` SPI (existing), `AgentWorkerFunction.ofSlot(AgentSlotConfig)` (from Task 1), `AgentSlotConfig` (from Task 1)
- Produces: `AgentWorkerFunctionProvider` — CDI-discovered bean. `handles(JsonNode)` returns true when node has `agent` key with a `slot` child. `create(JsonNode)` parses slot + optional overrides and returns `AgentWorkerFunction`.

- [ ] **Step 1: Write failing tests**

Create `engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java`:

```java
package io.casehub.engine.agent;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.yaml.jackson.YamlMappers;
import org.junit.jupiter.api.Test;

class AgentWorkerFunctionProviderTest {

  private final AgentWorkerFunctionProvider provider = new AgentWorkerFunctionProvider();
  private final ObjectMapper yaml = YamlMappers.create();

  @Test
  void handles_returnsTrueForAgentBlock() throws Exception {
    var node = yaml.readTree("""
        name: analyst
        agent:
          slot: sentiment-analyst
        """);
    assertThat(provider.handles(node)).isTrue();
  }

  @Test
  void handles_returnsFalseWithoutAgentBlock() throws Exception {
    var node = yaml.readTree("""
        name: local-worker
        do:
          - step: something
        """);
    assertThat(provider.handles(node)).isFalse();
  }

  @Test
  void handles_returnsFalseForAgentWithoutSlot() throws Exception {
    var node = yaml.readTree("""
        name: inline-agent
        agent:
          systemPrompt: "You are a helper"
        """);
    assertThat(provider.handles(node)).isFalse();
  }

  @Test
  void create_parsesSlotOnly() throws Exception {
    var node = yaml.readTree("""
        name: analyst
        agent:
          slot: sentiment-analyst
        """);
    var function = (AgentWorkerFunction) provider.create(node);
    assertThat(function.isDeferred()).isTrue();
    assertThat(function.slotConfig().slot()).isEqualTo("sentiment-analyst");
    assertThat(function.slotConfig().model()).isNull();
  }

  @Test
  void create_parsesAllOverrides() throws Exception {
    var node = yaml.readTree("""
        name: analyst
        agent:
          slot: sentiment-analyst
          model: claude-sonnet-5-5
          userMessage: "Analyse: {{text}}"
          inputProjection: ".ticket"
          outputProjection: ".sentiment"
        """);
    var function = (AgentWorkerFunction) provider.create(node);
    assertThat(function.slotConfig().slot()).isEqualTo("sentiment-analyst");
    assertThat(function.slotConfig().model()).isEqualTo("claude-sonnet-5-5");
    assertThat(function.slotConfig().userMessageTemplate()).isEqualTo("Analyse: {{text}}");
    assertThat(function.slotConfig().inputProjection()).isEqualTo(".ticket");
    assertThat(function.slotConfig().outputProjection()).isEqualTo(".sentiment");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=AgentWorkerFunctionProviderTest -q`
Expected: compilation failure — `AgentWorkerFunctionProvider` doesn't exist

- [ ] **Step 3: Implement AgentWorkerFunctionProvider**

Create `engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java`:

```java
package io.casehub.engine.agent;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.ai.AgentSlotConfig;
import io.casehub.api.spi.WorkerFunctionProvider;
import io.casehub.worker.api.WorkerFunction;

public class AgentWorkerFunctionProvider implements WorkerFunctionProvider {

  @Override
  public boolean handles(JsonNode rawWorkerNode) {
    return rawWorkerNode.has("agent") && rawWorkerNode.get("agent").has("slot");
  }

  @Override
  public WorkerFunction<?, ?> create(JsonNode rawWorkerNode) {
    JsonNode agentNode = rawWorkerNode.get("agent");
    String slot = agentNode.get("slot").asText();
    String model = textOrNull(agentNode, "model");
    String userMessage = textOrNull(agentNode, "userMessage");
    String inputProjection = textOrNull(agentNode, "inputProjection");
    String outputProjection = textOrNull(agentNode, "outputProjection");
    return AgentWorkerFunction.ofSlot(
        new AgentSlotConfig(slot, model, userMessage, inputProjection, outputProjection));
  }

  private static String textOrNull(JsonNode node, String field) {
    return node.has(field) ? node.get(field).asText() : null;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=AgentWorkerFunctionProviderTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java
git commit -m "feat(#1333): add AgentWorkerFunctionProvider for slot-based agent YAML

Refs #1333"
```

### Task 4: Extend AgentWorkerFunctionHandler for slot-based resolution

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `AgentWorkerFunction.isDeferred()`, `AgentWorkerFunction.slotConfig()` (from Task 1), `AgentRegistry.findById(slot, tenancyId)` (eidos-api), `SystemPromptRenderer.render(descriptor, context)` (eidos-api), `AgentPromptContext.forFormat(RenderFormat)` (eidos-api), `AgentProvider` (platform-agent-api), `WorkerAgentInvoker` (existing)
- Produces: Extended `execute()` method that checks `isDeferred()` and resolves slot → descriptor → invoker at execution time. Falls back to existing eager invoker path when not deferred.

- [ ] **Step 1: Write failing tests for slot-based resolution**

Add to existing `AgentWorkerFunctionHandlerTest.java`. Add imports for `AgentSlotConfig`, `AgentDescriptor`, `InMemoryAgentRegistry`, `SystemPromptRenderer`, `AgentPromptContext`. Add test helpers:

```java
private WorkerContext testContext(String tenancyId) {
    return new WorkerContext("test-worker", UUID.randomUUID(), tenancyId, null,
        io.casehub.api.context.PropagationContext.createRoot(), null);
}

private AgentProvider stubAgentProvider(String jsonResponse) {
    return new AgentProvider() {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            return Multi.createFrom().items(
                new AgentEvent.TextDelta(jsonResponse),
                new AgentEvent.InvocationComplete(0, 0, 0, 0, 0, null, 0L, 0L, null, 0, false, null, null));
        }
        @Override
        public AgentSession openSession(AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    };
}
```

Tests:

```java
@Test
void executes_deferred_agent_resolves_from_registry() {
    AgentDescriptor descriptor = AgentDescriptor.builder()
        .agentId("agent-1").name("Test Agent").slot("test-slot").tenancyId("t1")
        .build();
    var registry = new io.casehub.eidos.memory.InMemoryAgentRegistry(
        new io.casehub.eidos.memory.TestAgentRegistryEvents());
    registry.register(descriptor);

    SystemPromptRenderer renderer = (desc, ctx) ->
        new SystemPromptRenderer.RenderedPrompt("You are a test agent",
            SystemPromptRenderer.RenderFormat.MARKDOWN, "h1", "h2", false);

    AgentProvider provider = stubAgentProvider("{\"result\": \"resolved\"}");

    var deferredHandler = new AgentWorkerFunctionHandler(
        Executors.newVirtualThreadPerTaskExecutor(), registry, renderer, provider);

    var config = new AgentSlotConfig("test-slot", null, null, null, null);
    var agentFunction = AgentWorkerFunction.ofSlot(config);

    Worker worker = new Worker("agent-w", Set.of("cap"), agentFunction,
        new ExecutionPolicy(5000, null, null), null, null);

    HandlerResult result = deferredHandler.execute(worker, Capability.of("cap", "{}", "{}"),
        Map.of("input", "data"), testContext("t1"), new ExecutionMetadata("agent-w", "hash"));

    assertThat((Map<String, Object>) result.result().output()).containsEntry("result", "resolved");
}

@Test
void deferred_agent_missing_slot_returns_failed() {
    var registry = new io.casehub.eidos.memory.InMemoryAgentRegistry(
        new io.casehub.eidos.memory.TestAgentRegistryEvents());
    SystemPromptRenderer renderer = (desc, ctx) ->
        new SystemPromptRenderer.RenderedPrompt("", SystemPromptRenderer.RenderFormat.MARKDOWN, "", "", false);
    AgentProvider provider = stubAgentProvider("{}");

    var deferredHandler = new AgentWorkerFunctionHandler(
        Executors.newVirtualThreadPerTaskExecutor(), registry, renderer, provider);

    var config = new AgentSlotConfig("nonexistent-slot", null, null, null, null);
    var agentFunction = AgentWorkerFunction.ofSlot(config);

    Worker worker = new Worker("agent-w", Set.of("cap"), agentFunction,
        new ExecutionPolicy(5000, null, null), null, null);

    HandlerResult result = deferredHandler.execute(worker, Capability.of("cap", "{}", "{}"),
        Map.of(), testContext("t1"), new ExecutionMetadata("agent-w", "hash"));

    assertThat(result.result().outcome()).isInstanceOf(WorkerOutcome.Failed.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: compilation failure — constructor doesn't accept registry/renderer/provider

- [ ] **Step 3: Extend AgentWorkerFunctionHandler**

Add new constructor and slot resolution logic. Keep the existing single-arg constructor for backward compatibility:

```java
@ApplicationScoped
public class AgentWorkerFunctionHandler implements WorkerFunctionHandler {

  private final ExecutorService virtualThreads;
  private final AgentRegistry agentRegistry;
  private final SystemPromptRenderer systemPromptRenderer;
  private final AgentProvider agentProvider;

  @Inject
  public AgentWorkerFunctionHandler(
      @VirtualThreads ExecutorService virtualThreads,
      AgentRegistry agentRegistry,
      SystemPromptRenderer systemPromptRenderer,
      AgentProvider agentProvider) {
    this.virtualThreads = virtualThreads;
    this.agentRegistry = agentRegistry;
    this.systemPromptRenderer = systemPromptRenderer;
    this.agentProvider = agentProvider;
  }

  public AgentWorkerFunctionHandler(ExecutorService virtualThreads) {
    this(virtualThreads, null, null, null);
  }
  // ... supports() unchanged ...

  @Override
  public HandlerResult execute(...) {
    AgentWorkerFunction agent = (AgentWorkerFunction) worker.function();
    if (agent.isDeferred()) {
      return executeDeferred(agent.slotConfig(), worker, capability, inputData, context, metadata);
    }
    return executeEager(agent.invoker(), worker, capability, inputData, context, metadata);
  }
}
```

The `executeDeferred` method:
1. Gets `tenancyId` from `context.tenancyId()`
2. Calls `agentRegistry.findById(slotConfig.slot(), tenancyId)` — if empty, return `HandlerResult(WorkerResult.failed("Agent not found for slot: " + slot))`
3. Renders system prompt via `systemPromptRenderer.render(descriptor, AgentPromptContext.forFormat(RenderFormat.MARKDOWN))`
4. Constructs `WorkerAgentInvoker` with the resolved `agentProvider`, rendered system prompt, and any overrides from `slotConfig` (model, userMessageTemplate, inputProjection → `JQExpressionEvaluator` → `UnaryOperator<JsonNode>`, outputProjection similarly)
5. Delegates to the same execution logic as `executeEager`

The `executeEager` method extracts the existing execution logic (lines 66-121 of the current handler).

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: PASS

- [ ] **Step 5: Run existing tests to verify backward compatibility**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q`
Expected: PASS — existing eager-mode tests unchanged

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java
git commit -m "feat(#1333): extend AgentWorkerFunctionHandler for slot-based deferred resolution

Refs #1333"
```

---

## Batch 3: Migration — Remove inline agent fallback

### Task 5: Remove inline agent fallback from converter

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java:549-558` — remove `yw.agent()` fallback
- Modify: `schema/src/main/resources/examples/agent-worker-example.yaml` — migrate to slot-based shape

**Interfaces:**
- Consumes: `AgentWorkerFunctionProvider` (from Task 3) — now handles `agent:` through SPI
- Produces: Cleaner converter without special-case agent handling

- [ ] **Step 1: Verify provider handles the agent case by running full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,engine-support-core -q`
Expected: PASS

- [ ] **Step 2: Remove the inline fallback**

In `YamlCaseDefinitionConverter.java` around line 549-558, the fallback reads:
```java
if (function == null) {
    if (yw.agent() != null) {
        function = buildAgentFunction(yw, agentProvider);
    } else if ...
}
```

Remove the `yw.agent()` branch. The `buildAgentFunction` private method (line ~617) can also be removed if it's no longer called from anywhere else. Verify with `ide_find_references` on `buildAgentFunction` before removing.

- [ ] **Step 3: Update agent-worker-example.yaml**

Migrate `schema/src/main/resources/examples/agent-worker-example.yaml` from inline `agent:` with `systemPrompt`/`model` to the new slot-based shape:

```yaml
workers:
  - name: sentiment-analyst
    agent:
      slot: "sentiment-analyst"
    capabilities:
      - name: analyse-sentiment
        inputSchema: ".ticket.description"
        outputSchema: ".sentiment"
```

- [ ] **Step 4: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java schema/src/main/resources/examples/agent-worker-example.yaml
git commit -m "refactor(#1333): remove inline agent fallback from YamlCaseDefinitionConverter

The AgentWorkerFunctionProvider now handles agent: through the SPI.

Closes #1333"
```

---

## References

- [2026-11-20-agent-worker-function-provider-design.md] — design spec this plan implements
- `A2AWorkerFunctionProvider.java` — SPI pattern reference for Task 3
- `A2AWorkerFunctionProviderTest.java` — test pattern reference for Task 3
- `AgentWorkerFunction.java:22` — existing record to extend in Task 1
- `AgentWorkerFunctionHandler.java:52` — existing handler to extend in Task 4
- `AgentWorkerFunctionHandlerTest.java:47` — existing test to extend in Task 4
- `BindingTarget.java:27` — sealed interface to extend in Task 2
- `BindingExecutorResolver.java:34` — exhaustive switch to update in Task 2
- `CaseDefinitionService.java:83-103` — switch sites to update in Task 2
- `CompensationGraphProjection.java:54` — switch site to update in Task 2
- `HumanTaskTarget.java:44` — existing task target pattern reference for Task 2
- `WorkerAgentInvoker.java:32` — invoker constructed at resolution time in Task 4
- `AgentConverter.java` — existing converter utility retained
- `YamlCaseDefinitionConverter.java:549` — inline fallback removed in Task 5
- `AgentRegistry.java` (eidos-api) — slot lookup in Task 4
- `AgentQuery` (eidos-api) — `bySlot(slot, tenancyId)` factory
- `AgentMatch` (eidos-api) — `descriptor()` accessor
- `AgentPromptContext` (eidos-api) — `forFormat(RenderFormat)` factory
- `SystemPromptRenderer.java` (eidos-api) — prompt rendering in Task 4
- `NoOpAgentRegistry.java` — existing no-op for tests
- `InMemoryAgentRegistry` (eidos persistence-memory) — test registry
- GitHub #1333 — focal issue
