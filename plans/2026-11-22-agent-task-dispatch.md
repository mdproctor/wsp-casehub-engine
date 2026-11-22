# Agent-Task Runtime Dispatch Handler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1339 — agent-task: runtime dispatch handler for AgentRegistry capability matching
**Issue group:** #1339

**Goal:** Wire the runtime dispatch path so `AgentTaskTarget` bindings query `AgentRegistry` by capability+domain, select via the platform's `AgentSelector` SPI, and execute through `AgentWorkerFunctionHandler`.

**Architecture:** Introduce a sealed `DeferredAgentConfig` hierarchy (SlotConfig, CapabilityConfig) replacing `AgentSlotConfig`. Refactor `AgentWorkerFunction` to use it. Extend `AgentWorkerFunctionHandler` with a capability-based dispatch path that delegates to `AgentSelector` for health-aware, trust-informed selection. Create synthetic Workers for `AgentTaskTarget` bindings at CaseDefinition load time so the execution pipeline is unchanged.

**Tech Stack:** Java 21 (sealed interfaces, pattern matching), Quarkus 3.32.2, CDI, eidos API (`AgentRegistry`, `AgentSelector`, `SystemPromptRenderer`)

## Global Constraints

- Quarkus version: `3.32.2`
- Test classes must be `*Test.java`, never `*IT.java`
- Inject repos by SPI interface, not concrete class
- Workers return `WorkerResult`, not `Map`
- `@ObservesAsync` unreliable in `@QuarkusTest` — call observer directly
- All `@ConsumeEvent` handlers: `@RunOnVirtualThread` + void
- Plan-definition types in `engine-api`; execution types in `engine-common`
- IntelliJ MCP for all code navigation and structural editing

---

## Batch 1: Foundation — DeferredAgentConfig type hierarchy refactor

### Task 1: Create DeferredAgentConfig sealed interface

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/ai/DeferredAgentConfig.java`
- Test: `api/src/test/java/io/casehub/api/model/ai/DeferredAgentConfigTest.java`

**Interfaces:**
- Consumes: `ExpressionEvaluator` (platform-api, existing)
- Produces: `DeferredAgentConfig` sealed interface with `SlotConfig(String slot, String model, String userMessageTemplate, String inputProjection, String outputProjection)` and `CapabilityConfig(String capability, String domain, String model, ExpressionEvaluator inputEvaluator, ExpressionEvaluator outputEvaluator)`

- [ ] **Step 1: Write failing tests for DeferredAgentConfig**

```java
package io.casehub.api.model.ai;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import org.junit.jupiter.api.Test;

class DeferredAgentConfigTest {

  @Test
  void slotConfig_requiresSlot() {
    assertThrows(NullPointerException.class,
        () -> new DeferredAgentConfig.SlotConfig(null, null, null, null, null));
  }

  @Test
  void slotConfig_acceptsSlotOnly() {
    var config = new DeferredAgentConfig.SlotConfig("my-slot", null, null, null, null);
    assertEquals("my-slot", config.slot());
    assertNull(config.model());
  }

  @Test
  void slotConfig_acceptsAllFields() {
    var config = new DeferredAgentConfig.SlotConfig(
        "my-slot", "claude-sonnet-5-5", "Analyse: {{text}}", ".input", ".output");
    assertEquals("my-slot", config.slot());
    assertEquals("claude-sonnet-5-5", config.model());
    assertEquals("Analyse: {{text}}", config.userMessageTemplate());
    assertEquals(".input", config.inputProjection());
    assertEquals(".output", config.outputProjection());
  }

  @Test
  void capabilityConfig_requiresCapability() {
    assertThrows(NullPointerException.class,
        () -> new DeferredAgentConfig.CapabilityConfig(null, null, null, null, null));
  }

  @Test
  void capabilityConfig_acceptsCapabilityOnly() {
    var config = new DeferredAgentConfig.CapabilityConfig("risk-assessment", null, null, null, null);
    assertEquals("risk-assessment", config.capability());
    assertNull(config.domain());
    assertNull(config.model());
  }

  @Test
  void capabilityConfig_acceptsAllFields() {
    var inputEval = new JQExpressionEvaluator(".input");
    var outputEval = new JQExpressionEvaluator(".output");
    var config = new DeferredAgentConfig.CapabilityConfig(
        "risk-assessment", "aml", "claude-opus-5-5", inputEval, outputEval);
    assertEquals("risk-assessment", config.capability());
    assertEquals("aml", config.domain());
    assertEquals("claude-opus-5-5", config.model());
    assertSame(inputEval, config.inputEvaluator());
    assertSame(outputEval, config.outputEvaluator());
  }

  @Test
  void patternMatch_exhaustive() {
    DeferredAgentConfig slot = new DeferredAgentConfig.SlotConfig("s", null, null, null, null);
    DeferredAgentConfig cap = new DeferredAgentConfig.CapabilityConfig("c", null, null, null, null);

    assertEquals("slot:s", describe(slot));
    assertEquals("cap:c", describe(cap));
  }

  private String describe(DeferredAgentConfig config) {
    return switch (config) {
      case DeferredAgentConfig.SlotConfig sc -> "slot:" + sc.slot();
      case DeferredAgentConfig.CapabilityConfig cc -> "cap:" + cc.capability();
    };
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=DeferredAgentConfigTest -q`
Expected: FAIL — `DeferredAgentConfig` does not exist

- [ ] **Step 3: Implement DeferredAgentConfig**

Create `api/src/main/java/io/casehub/api/model/ai/DeferredAgentConfig.java`:

```java
package io.casehub.api.model.ai;

import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.Objects;

public sealed interface DeferredAgentConfig
    permits DeferredAgentConfig.SlotConfig, DeferredAgentConfig.CapabilityConfig {

  record SlotConfig(
      String slot,
      String model,
      String userMessageTemplate,
      String inputProjection,
      String outputProjection) implements DeferredAgentConfig {
    public SlotConfig {
      Objects.requireNonNull(slot, "slot is required");
    }
  }

  record CapabilityConfig(
      String capability,
      String domain,
      String model,
      ExpressionEvaluator inputEvaluator,
      ExpressionEvaluator outputEvaluator) implements DeferredAgentConfig {
    public CapabilityConfig {
      Objects.requireNonNull(capability, "capability is required");
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=DeferredAgentConfigTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add api/src/main/java/io/casehub/api/model/ai/DeferredAgentConfig.java api/src/test/java/io/casehub/api/model/ai/DeferredAgentConfigTest.java
git commit -m "feat(#1339): add DeferredAgentConfig sealed hierarchy

SlotConfig preserves existing AgentSlotConfig contract.
CapabilityConfig adds capability+domain with ExpressionEvaluator mappings.

Refs #1339"
```

---

### Task 2: Refactor AgentWorkerFunction and update all callers

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java:25-53`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java:38-41`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java:100-113`
- Modify: `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java:498`

**Interfaces:**
- Consumes: `DeferredAgentConfig` sealed interface (from Task 1)
- Produces: `AgentWorkerFunction(WorkerAgentInvoker invoker, DeferredAgentConfig deferredConfig)` with `deferredConfig()`, `ofSlot(DeferredAgentConfig.SlotConfig)`, `ofCapability(DeferredAgentConfig.CapabilityConfig)`, `isDeferred()`, `eager(WorkerAgentInvoker)`

- [ ] **Step 1: Update AgentWorkerFunction record**

Replace `AgentSlotConfig slotConfig` with `DeferredAgentConfig deferredConfig`. Update the record signature, constructor, factory methods, and `isDeferred()`. Add `ofCapability()` factory.

```java
@SuppressWarnings("unchecked")
public record AgentWorkerFunction(WorkerAgentInvoker invoker, DeferredAgentConfig deferredConfig)
    implements WorkerFunction<Map<String, Object>, Map<String, Object>> {

  public AgentWorkerFunction(WorkerAgentInvoker invoker) {
    this(Objects.requireNonNull(invoker), null);
  }

  public static AgentWorkerFunction eager(WorkerAgentInvoker invoker) {
    return new AgentWorkerFunction(Objects.requireNonNull(invoker), null);
  }

  public static AgentWorkerFunction ofSlot(DeferredAgentConfig.SlotConfig config) {
    return new AgentWorkerFunction(null, Objects.requireNonNull(config));
  }

  public static AgentWorkerFunction ofCapability(DeferredAgentConfig.CapabilityConfig config) {
    return new AgentWorkerFunction(null, Objects.requireNonNull(config));
  }

  public boolean isDeferred() {
    return deferredConfig != null;
  }

  @Override
  public Class<Map<String, Object>> inputType() {
    return (Class) Map.class;
  }

  @Override
  public Class<Map<String, Object>> outputType() {
    return (Class) Map.class;
  }
}
```

- [ ] **Step 2: Update AgentWorkerFunctionProvider**

In `AgentWorkerFunctionProvider.create()`, replace `new AgentSlotConfig(...)` with `new DeferredAgentConfig.SlotConfig(...)` and `AgentWorkerFunction.ofSlot(config)` now takes `DeferredAgentConfig.SlotConfig`.

- [ ] **Step 3: Update AgentWorkerFunctionHandler**

In `execute()` method, replace `agent.slotConfig()` with `agent.deferredConfig()`. Rename `executeDeferred` to `executeBySlot` and update its parameter type from `AgentSlotConfig` to `DeferredAgentConfig.SlotConfig`. The `execute()` body becomes:

```java
AgentWorkerFunction agent = (AgentWorkerFunction) worker.function();
if (agent.isDeferred()) {
  return switch (agent.deferredConfig()) {
    case DeferredAgentConfig.SlotConfig sc -> executeBySlot(sc, worker, inputData, metadata);
    case DeferredAgentConfig.CapabilityConfig cc ->
        throw new UnsupportedOperationException("Capability dispatch not yet implemented");
  };
}
return executeWithInvoker(agent.invoker(), worker, inputData);
```

- [ ] **Step 4: Update all test files**

- `AgentWorkerFunctionTest.java`: Replace `slotConfig()` calls with `deferredConfig()`. Replace `new AgentSlotConfig(...)` with `new DeferredAgentConfig.SlotConfig(...)`.
- `AgentWorkerFunctionProviderTest.java`: Replace `function.slotConfig()` with `((DeferredAgentConfig.SlotConfig) function.deferredConfig())`.
- `AgentWorkerFunctionHandlerTest.java`: Replace `new AgentSlotConfig(...)` with `new DeferredAgentConfig.SlotConfig(...)`.
- `CaseDefinitionYamlMapperTest.java:498`: Replace `agentFn.slotConfig()` with `((DeferredAgentConfig.SlotConfig) agentFn.deferredConfig())`.

- [ ] **Step 5: Run full test suite for affected modules**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,engine-support-core,runtime -q`
Expected: PASS — all existing behavior preserved, just type changes

- [ ] **Step 6: Commit**

```
git add api/ engine-support-core/ runtime/
git commit -m "refactor(#1339): AgentWorkerFunction slotConfig → deferredConfig

Replace AgentSlotConfig with sealed DeferredAgentConfig hierarchy.
AgentWorkerFunction.slotConfig() → deferredConfig().
Handler uses pattern matching for slot vs capability dispatch.
Capability path throws UnsupportedOperationException (implemented in Task 4).

Refs #1339"
```

---

## Batch 2: Capability dispatch pipeline

### Task 3: Add model field to AgentTaskTarget

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/AgentTaskTarget.java:22-97`
- Modify: `api/src/test/java/io/casehub/api/model/AgentTaskTargetTest.java`

**Interfaces:**
- Consumes: none
- Produces: `AgentTaskTarget.model()` (String, nullable), `AgentTaskTarget.Builder.model(String)` method

- [ ] **Step 1: Write failing test**

Add to `AgentTaskTargetTest.java`:

```java
@Test
void builder_withModel() {
  AgentTaskTarget target = AgentTaskTarget.builder()
      .capability("risk-assessment")
      .model("claude-sonnet-5-5")
      .build();
  assertEquals("risk-assessment", target.capability());
  assertEquals("claude-sonnet-5-5", target.model());
}

@Test
void builder_withoutModel_isNull() {
  AgentTaskTarget target = AgentTaskTarget.builder()
      .capability("risk-assessment")
      .build();
  assertNull(target.model());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentTaskTargetTest -q`
Expected: FAIL — `model()` method does not exist

- [ ] **Step 3: Implement model field**

Add to `AgentTaskTarget`:
- Field: `private final String model;`
- Constructor: `this.model = builder.model;`
- Accessor: `public String model() { return model; }`
- Builder field: `private String model;`
- Builder method: `public Builder model(String model) { this.model = model; return this; }`

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentTaskTargetTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add api/
git commit -m "feat(#1339): add model override field to AgentTaskTarget

Parity with slot-based mode where AgentSlotConfig.model() overrides
the agent descriptor's model at dispatch time.

Refs #1339"
```

---

### Task 4: Implement executeByCapability in AgentWorkerFunctionHandler

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java:63-199`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `DeferredAgentConfig.CapabilityConfig` (from Task 1), `AgentRegistry.find(AgentQuery)` (eidos-api), `AgentSelector.select(List<AgentMatch>, SelectionContext)` (eidos-api), `SystemPromptRenderer.render(AgentDescriptor, AgentPromptContext)` (eidos-api), `ExpressionEngineRegistry.transform(ExpressionEvaluator, JsonNode)` (platform-api), `AgentProvider` (platform-agent)
- Produces: `AgentWorkerFunctionHandler.executeByCapability(CapabilityConfig, Worker, Object, ExecutionMetadata) → HandlerResult`

- [ ] **Step 1: Write failing tests for capability dispatch**

Add to `AgentWorkerFunctionHandlerTest.java`:

```java
@Test
void executeByCapability_selected_invokesAgent() {
  // Setup: register agent in InMemoryAgentRegistry with capability "risk-assessment"
  // Setup: mock AgentSelector returning Selected(descriptor, null, 0.9, "trust")
  // Setup: mock SystemPromptRenderer returning "You are a risk assessor"
  // Setup: mock AgentProvider returning a canned response

  var config = new DeferredAgentConfig.CapabilityConfig(
      "risk-assessment", "aml", null, null, null);
  var function = AgentWorkerFunction.ofCapability(config);
  var worker = Worker.builder()
      .name("__agent-task__check-risk")
      .function(function)
      .build();

  // Execute and verify: HandlerResult with successful WorkerResult
}

@Test
void executeByCapability_noneQualified_returnsFailed() {
  // Setup: AgentSelector returns NoneQualified("no candidates")
  var config = new DeferredAgentConfig.CapabilityConfig(
      "nonexistent-cap", null, null, null, null);
  var function = AgentWorkerFunction.ofCapability(config);
  var worker = Worker.builder()
      .name("__agent-task__check-unknown")
      .function(function)
      .build();

  // Execute and verify: WorkerResult.failed with reason
}

@Test
void executeByCapability_escalated_returnsFailed() {
  // Setup: AgentSelector returns Escalated("risk-assessment", TRUST, "below threshold")
  // Execute and verify: WorkerResult.failed with "Escalated: below threshold"
}

@Test
void executeByCapability_modelOverride_usesConfigModel() {
  // Setup: CapabilityConfig with model="claude-sonnet-5-5"
  // Setup: agent descriptor has model family "opus"
  // Verify: WorkerAgentInvoker built with "claude-sonnet-5-5"
}

@Test
void executeByCapability_evaluatorTransforms_appliedViaRegistry() {
  // Setup: CapabilityConfig with inputEvaluator = JQExpressionEvaluator(".ticket")
  // Setup: ExpressionEngineRegistry.transform returns transformed node
  // Verify: input is transformed before invoker call
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: FAIL — `executeByCapability` throws UnsupportedOperationException

- [ ] **Step 3: Add AgentSelector and ExpressionEngineRegistry to handler constructor**

Add fields and constructor parameters:

```java
private final AgentSelector agentSelector;
private final ExpressionEngineRegistry expressionEngineRegistry;

@Inject
public AgentWorkerFunctionHandler(
    @VirtualThreads ExecutorService virtualThreads,
    AgentRegistry agentRegistry,
    SystemPromptRenderer systemPromptRenderer,
    AgentProvider agentProvider,
    AgentSelector agentSelector,
    ExpressionEngineRegistry expressionEngineRegistry) {
  this.virtualThreads = virtualThreads;
  this.agentRegistry = agentRegistry;
  this.systemPromptRenderer = systemPromptRenderer;
  this.agentProvider = agentProvider;
  this.agentSelector = agentSelector;
  this.expressionEngineRegistry = expressionEngineRegistry;
}
```

Update the test-only constructor accordingly.

- [ ] **Step 4: Implement executeByCapability**

```java
private HandlerResult executeByCapability(
    DeferredAgentConfig.CapabilityConfig config,
    Worker worker,
    Object inputData,
    ExecutionMetadata metadata) {

  String tenancyId = metadata.tenancyId();
  var matches = agentRegistry.find(
      AgentQuery.byCapabilityAndDomain(config.capability(), config.domain(), tenancyId));

  var selection = agentSelector.select(
      matches, SelectionContext.of(tenancyId, config.capability(), config.domain()));

  return switch (selection) {
    case AgentSelection.NoneQualified nq ->
        new HandlerResult(WorkerResult.failed(nq.reason()));
    case AgentSelection.Escalated esc ->
        new HandlerResult(WorkerResult.failed("Escalated: " + esc.reason()));
    case AgentSelection.Selected sel -> {
      AgentDescriptor descriptor = sel.agent();
      SystemPromptRenderer.RenderedPrompt rendered =
          systemPromptRenderer.render(
              descriptor,
              AgentPromptContext.forFormat(SystemPromptRenderer.RenderFormat.MARKDOWN));

      UnaryOperator<JsonNode> inputTransformer = config.inputEvaluator() != null
          ? node -> expressionEngineRegistry.transform(config.inputEvaluator(), node).get(0)
          : UnaryOperator.identity();
      UnaryOperator<JsonNode> outputTransformer = config.outputEvaluator() != null
          ? node -> expressionEngineRegistry.transform(config.outputEvaluator(), node).get(0)
          : UnaryOperator.identity();

      String effectiveModel = config.model() != null ? config.model() : descriptor.modelFamily();

      WorkerAgentInvoker invoker = new WorkerAgentInvoker(
          agentProvider, rendered.content(), null,
          inputTransformer, outputTransformer, null, effectiveModel, null);

      yield executeWithInvoker(invoker, worker, inputData);
    }
  };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```
git add runtime/
git commit -m "feat(#1339): implement executeByCapability in AgentWorkerFunctionHandler

Queries AgentRegistry.find(byCapabilityAndDomain), delegates selection
to AgentSelector SPI (health, trust, routing), renders system prompt,
builds invoker with ExpressionEngineRegistry-based transforms and
optional model override.

Handles Selected, NoneQualified, and Escalated selection outcomes.

Refs #1339"
```

---

### Task 5: BindingExecutorResolver + synthetic Worker creation + collision protection

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/routing/BindingExecutorResolver.java:50`
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/routing/SyntheticAgentTaskWorker.java`
- Modify: `common-core/src/test/java/io/casehub/engine/common/internal/routing/BindingExecutorResolverTest.java`
- Create: `common-core/src/test/java/io/casehub/engine/common/internal/routing/SyntheticAgentTaskWorkerTest.java`

**Interfaces:**
- Consumes: `AgentTaskTarget` (api), `DeferredAgentConfig.CapabilityConfig` (from Task 1), `AgentWorkerFunction.ofCapability()` (from Task 2), `Worker.builder()` (worker-api)
- Produces: `SyntheticAgentTaskWorker.create(String bindingName, AgentTaskTarget target) → Worker`, `SyntheticAgentTaskWorker.RESERVED_PREFIX` constant, `SyntheticAgentTaskWorker.validateNoCollision(List<Worker>, Set<String> syntheticNames)` validator

- [ ] **Step 1: Write failing tests for synthetic Worker creation**

Create `SyntheticAgentTaskWorkerTest.java`:

```java
package io.casehub.engine.common.internal.routing;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.AgentTaskTarget;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.ai.DeferredAgentConfig;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.worker.api.Worker;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;

class SyntheticAgentTaskWorkerTest {

  @Test
  void create_setsCorrectName() {
    var target = AgentTaskTarget.builder().capability("risk-assessment").build();
    Worker worker = SyntheticAgentTaskWorker.create("check-risk", target);
    assertEquals("__agent-task__check-risk", worker.name());
  }

  @Test
  void create_setsCapabilityConfig() {
    var target = AgentTaskTarget.builder()
        .capability("risk-assessment")
        .domain("aml")
        .model("claude-sonnet-5-5")
        .inputMapping(".ticket")
        .build();
    Worker worker = SyntheticAgentTaskWorker.create("check-risk", target);

    var fn = (AgentWorkerFunction) worker.function();
    assertTrue(fn.isDeferred());
    var config = (DeferredAgentConfig.CapabilityConfig) fn.deferredConfig();
    assertEquals("risk-assessment", config.capability());
    assertEquals("aml", config.domain());
    assertEquals("claude-sonnet-5-5", config.model());
    assertNotNull(config.inputEvaluator());
    assertNull(config.outputEvaluator());
  }

  @Test
  void validateNoCollision_passesForSyntheticOnly() {
    var syntheticNames = Set.of("__agent-task__check-risk");
    var workers = List.of(
        Worker.builder().name("my-worker").build(),
        Worker.builder().name("__agent-task__check-risk")
            .function(AgentWorkerFunction.ofCapability(
                new DeferredAgentConfig.CapabilityConfig("c", null, null, null, null)))
            .build());

    assertDoesNotThrow(
        () -> SyntheticAgentTaskWorker.validateNoCollision(workers, syntheticNames));
  }

  @Test
  void validateNoCollision_throwsForUserWorkerWithReservedPrefix() {
    var syntheticNames = Set.of("__agent-task__check-risk");
    var workers = List.of(
        Worker.builder().name("__agent-task__user-worker").build());

    assertThrows(IllegalArgumentException.class,
        () -> SyntheticAgentTaskWorker.validateNoCollision(workers, syntheticNames));
  }
}
```

- [ ] **Step 2: Write failing test for BindingExecutorResolver**

Add to `BindingExecutorResolverTest.java`:

```java
@Test
void resolvesAgentTaskTargetToSyntheticName() {
  var target = AgentTaskTarget.builder().capability("risk-assessment").build();
  Binding binding = Binding.builder().name("check-risk").target(target).build();
  CaseDefinition definition = new CaseDefinition();

  ExecutorRef result = BindingExecutorResolver.resolve(binding, definition);
  assertEquals("__agent-task__check-risk", result.name());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest="SyntheticAgentTaskWorkerTest,BindingExecutorResolverTest" -q`
Expected: FAIL

- [ ] **Step 4: Implement SyntheticAgentTaskWorker**

Create `common-core/src/main/java/io/casehub/engine/common/internal/routing/SyntheticAgentTaskWorker.java`:

```java
package io.casehub.engine.common.internal.routing;

import io.casehub.api.model.AgentTaskTarget;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.ai.DeferredAgentConfig;
import io.casehub.worker.api.Worker;
import java.util.List;
import java.util.Set;

public final class SyntheticAgentTaskWorker {

  public static final String RESERVED_PREFIX = "__agent-task__";

  private SyntheticAgentTaskWorker() {}

  public static Worker create(String bindingName, AgentTaskTarget target) {
    var config = new DeferredAgentConfig.CapabilityConfig(
        target.capability(),
        target.domain(),
        target.model(),
        target.inputMapping(),
        target.outputMapping());

    return Worker.builder()
        .name(RESERVED_PREFIX + bindingName)
        .description("Synthetic agent-task worker for capability: " + target.capability())
        .function(AgentWorkerFunction.ofCapability(config))
        .build();
  }

  public static void validateNoCollision(List<Worker> workers, Set<String> syntheticNames) {
    for (Worker w : workers) {
      if (w.name().startsWith(RESERVED_PREFIX) && !syntheticNames.contains(w.name())) {
        throw new IllegalArgumentException(
            "Worker name '" + w.name() + "' uses reserved prefix '" + RESERVED_PREFIX + "'");
      }
    }
  }
}
```

- [ ] **Step 5: Update BindingExecutorResolver**

Change line 50 from:

```java
case AgentTaskTarget at -> ExecutorRef.of(at.capability());
```

to:

```java
case AgentTaskTarget at -> ExecutorRef.of(
    SyntheticAgentTaskWorker.RESERVED_PREFIX + binding.getName());
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest="SyntheticAgentTaskWorkerTest,BindingExecutorResolverTest" -q`
Expected: PASS

- [ ] **Step 7: Run full test suite for affected modules**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common-core,engine-support-core,runtime,planning-core -q`
Expected: PASS — all modules compile and pass

- [ ] **Step 8: Commit**

```
git add common-core/
git commit -m "feat(#1339): BindingExecutorResolver + synthetic Worker creation

BindingExecutorResolver resolves AgentTaskTarget to synthetic worker
name (__agent-task__<bindingName>). SyntheticAgentTaskWorker utility
creates Workers with CapabilityConfig and validates no user-authored
Worker uses the reserved __agent-task__ prefix.

Closes #1339"
```

---

## References

- [2026-11-22-agent-task-dispatch-design.md] — design spec this plan implements
- `AgentTaskTarget.java:22-97` — existing binding target
- `AgentWorkerFunction.java:25-53` — record being refactored
- `AgentSlotConfig.java:20-29` — existing config becoming SlotConfig
- `AgentWorkerFunctionHandler.java:63-199` — handler gaining capability path
- `AgentWorkerFunctionProvider.java:24-46` — provider using SlotConfig
- `BindingExecutorResolver.java:30-55` — resolver gaining synthetic name
- `AgentSelector.java:5-7` — selection SPI
- `EngineAwareAgentSelector.java:42-113` — production selector
- `AgentSelection.java:3-22` — selection result types
- `SelectionContext.java:5-21` — selection context
- `ExpressionEngineRegistry` — expression evaluation
- casehubio/engine#1339 — focal issue
- casehubio/engine#1333 — predecessor (agent-task binding model)
