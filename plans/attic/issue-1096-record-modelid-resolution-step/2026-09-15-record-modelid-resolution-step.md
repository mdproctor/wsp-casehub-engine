# Record modelId in ResolutionStep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1096 — Record modelId in ResolutionStep.parameters() for CBR model-aware routing
**Issue group:** #1096

**Goal:** When the engine records a ResolutionStep after an agent invocation, include the resolved model name in the step's parameters map under key `modelId`, so blocks' `ModelPreferenceSignalProvider` can score agents by `(workerId, modelId)`.

**Architecture:** Add a nullable `modelId` field to `Agent`, wire it from YAML via `AgentConverter`, and look it up at CBR retain time from `CaseDefinition.getWorkers()` in `CbrCaseRetainObserver.toResolutionStep()`.

**Tech Stack:** Java 21, Quarkus, JUnit 5, AssertJ

## Global Constraints

- `Agent` constructor change must be backward-compatible (existing callers pass `null`)
- Non-agent workers (A2A, MCP, ReAct, Sync) get `Map.of()` parameters (unchanged behavior)
- `ResolutionStep` record is in neocortex (external dep) — no changes needed, its `parameters` field already exists

---

## Batch 1: Add modelId to Agent and populate in CBR retain

### Task 1: Add modelId field to Agent and AgentBuilder

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/ai/Agent.java`
- Modify: `api/src/main/java/io/casehub/api/model/ai/AgentBuilder.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/AgentConverter.java`
- Modify: `api/src/test/java/io/casehub/api/model/ai/AgentTest.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java`

**Interfaces:**
- Produces: `Agent.modelId()` — returns `@Nullable String`, the declared LLM model name

- [ ] **Step 1: Write failing test — Agent.modelId() accessor**

In `AgentTest.java`, add:

```java
@Test
void modelId_returns_configured_value() {
  Agent agent = Agent.builder()
      .systemPrompt("test")
      .model(fixedResponseModel("{\"result\":\"ok\"}"))
      .modelId("claude-sonnet-4-20250514")
      .build();
  assertThat(agent.modelId()).isEqualTo("claude-sonnet-4-20250514");
}

@Test
void modelId_defaults_to_null() {
  Agent agent = Agent.builder()
      .systemPrompt("test")
      .model(fixedResponseModel("{\"result\":\"ok\"}"))
      .build();
  assertThat(agent.modelId()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentTest#modelId_returns_configured_value+modelId_defaults_to_null -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: compilation failure — `modelId()` method does not exist

- [ ] **Step 3: Implement modelId on Agent and AgentBuilder**

In `Agent.java`, add field and accessor:

```java
private final String modelId;
```

Update constructor to accept `modelId` as the last parameter:

```java
Agent(
    String systemPrompt,
    String userMessageTemplate,
    UnaryOperator<JsonNode> inputTransformer,
    UnaryOperator<JsonNode> outputTransformer,
    ChatModel model,
    JsonSchema responseSchema,
    Function<Map<String, Object>, PlannedAction> plannedActionExtractor,
    String modelId) {
  // ... existing assignments ...
  this.modelId = modelId;
}

public String modelId() {
  return modelId;
}
```

In `AgentBuilder.java`, add field and setter:

```java
private String modelId;

public AgentBuilder modelId(String modelId) {
  this.modelId = modelId;
  return this;
}
```

Update `build()` — pass `modelId` as the last argument to the `Agent` constructor:

```java
return new Agent(
    systemPrompt,
    userMessageTemplate,
    resolvedInput,
    resolvedOutput,
    resolvedModel,
    responseSchema,
    plannedActionExtractor,
    modelId);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all PASS

- [ ] **Step 5: Write failing test — AgentConverter wires modelName as modelId**

In `AgentConverterTest.java`, add:

```java
@Test
void toApiAgent_openai_setsModelId() throws Exception {
  JsonNode node =
      JSON.readTree(
          """
      {"model":"openai","modelName":"gpt-4","apiKey":"sk-test",
       "systemPrompt":"You are a test agent"}""");
  Agent result = AgentConverter.toApiAgent(node);
  assertThat(result).isNotNull();
  assertThat(result.modelId()).isEqualTo("gpt-4");
}

@Test
void toApiAgent_noModelName_modelIdIsNull() throws Exception {
  JsonNode node =
      JSON.readTree(
          """
      {"model":"openai","apiKey":"sk-test",
       "systemPrompt":"You are a test agent"}""");
  Agent result = AgentConverter.toApiAgent(node);
  assertThat(result).isNotNull();
  assertThat(result.modelId()).isNull();
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentConverterTest#toApiAgent_openai_setsModelId+toApiAgent_noModelName_modelIdIsNull -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — `modelId()` returns null

- [ ] **Step 7: Wire modelName in AgentConverter**

In `AgentConverter.toApiAgent()`, after the existing `modelProvider` resolution and before `builder.build()`, add:

```java
String modelName = agentNode.has("modelName") ? agentNode.get("modelName").asText() : null;
```

(This variable already exists inside `toChatModelProviderFromNode` but isn't accessible here. Read it at the `toApiAgent` level.)

Then add to the builder chain:

```java
if (modelName != null) {
  builder.modelId(modelName);
}
```

Note: `modelName` is read from the YAML node at the same level as `model`. For the `model: { openai: { modelName: ... } }` object syntax, it comes from `providerConfigNode`. For the flat syntax `model: openai, modelName: gpt-4`, it comes from `agentNode`. The existing code already reads `modelName` inside `toChatModelProviderFromNode` from `node` (which is `providerConfigNode` for object syntax, `agentNode` for flat syntax). We should read from the same source — use `providerConfigNode` to match the existing logic:

```java
String modelNameForId = providerConfigNode.has("modelName")
    ? providerConfigNode.get("modelName").asText() : null;
if (modelNameForId != null) {
  builder.modelId(modelNameForId);
}
```

- [ ] **Step 8: Run all AgentConverter tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentConverterTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/model/ai/Agent.java api/src/main/java/io/casehub/api/model/ai/AgentBuilder.java api/src/main/java/io/casehub/api/model/converter/AgentConverter.java api/src/test/java/io/casehub/api/model/ai/AgentTest.java api/src/test/java/io/casehub/api/model/converter/AgentConverterTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: add modelId field to Agent for CBR model-aware routing Refs #1096"
```

### Task 2: Populate modelId in CbrCaseRetainObserver parameters

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java`

**Interfaces:**
- Consumes: `Agent.modelId()` — `@Nullable String` from Task 1
- Consumes: `CaseDefinition.getWorkers()` — `List<Worker>`, already available in `doRetain()`
- Consumes: `AgentWorkerFunction.agent()` — `Agent`, from `Worker.function()`

- [ ] **Step 1: Write failing test — parameters contain modelId for agent workers**

In `CbrCaseRetainObserverTest.java`, add a new helper method and test:

```java
private CaseDefinition defWithJqCbrAndWorker(
    String name, String domain, Map<String, String> features,
    Binding binding, Worker worker) {
  var configBuilder = CbrConfig.builder();
  features.forEach(configBuilder::feature);
  if (domain != null) {
    configBuilder.domain(domain);
  }
  return CaseDefinition.builder()
      .name(name)
      .namespace("test")
      .version("1.0.0")
      .cbrConfig(configBuilder.build())
      .bindings(binding)
      .workers(worker)
      .build();
}
```

```java
@Test
void resolution_step_parameters_contain_modelId_for_agent_worker() {
  Agent agent = Agent.builder()
      .systemPrompt("test")
      .model(new dev.langchain4j.model.chat.ChatModel() {
        @Override
        public dev.langchain4j.model.chat.response.ChatResponse doChat(
            dev.langchain4j.model.chat.request.ChatRequest request) {
          return dev.langchain4j.model.chat.response.ChatResponse.builder()
              .aiMessage(dev.langchain4j.data.message.AiMessage.from("{}"))
              .build();
        }
      })
      .modelId("claude-sonnet-4-20250514")
      .build();
  Worker worker = Worker.builder()
      .name("agent-1")
      .capabilityName("risk-assessment")
      .function(new AgentWorkerFunction(agent))
      .build();
  registry.register(
      defWithJqCbrAndWorker(
          "model-case", "dom", Map.of("k", ".k"),
          capBinding("assess", "risk-assessment"), worker));
  planItemStore.items = List.of(planItem("assess", "agent-1", TaskStatus.COMPLETED));

  observer.onOutcome(event("model-case", "COMPLETED", Map.of("k", "v")));

  assertThat(store.storedCases).hasSize(1);
  var step = store.storedCases.get(0).resolutionStep().get(0);
  assertThat(step.parameters()).containsEntry("modelId", "claude-sonnet-4-20250514");
}

@Test
void resolution_step_parameters_empty_for_non_agent_worker() {
  registry.register(
      defWithJqCbr("plain-case", "dom", Map.of("k", ".k"), capBinding("b1", "cap1")));
  planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.COMPLETED));

  observer.onOutcome(event("plain-case", "COMPLETED", Map.of("k", "v")));

  assertThat(store.storedCases).hasSize(1);
  var step = store.storedCases.get(0).resolutionStep().get(0);
  assertThat(step.parameters()).isEmpty();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CbrCaseRetainObserverTest#resolution_step_parameters_contain_modelId_for_agent_worker -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — parameters is empty `Map.of()`

- [ ] **Step 3: Implement parameter resolution in CbrCaseRetainObserver**

Add a private method `resolveParameters`:

```java
private Map<String, Object> resolveParameters(
    PlanItemRecord record, CaseDefinition definition) {
  if (record.executorName() == null) {
    return Map.of();
  }
  return definition.getWorkers().stream()
      .filter(w -> w.name().equals(record.executorName()))
      .findFirst()
      .filter(w -> w.function() instanceof AgentWorkerFunction)
      .map(w -> ((AgentWorkerFunction) w.function()).agent().modelId())
      .filter(java.util.Objects::nonNull)
      .map(id -> Map.<String, Object>of("modelId", id))
      .orElse(Map.of());
}
```

Add import for `AgentWorkerFunction`:
```java
import io.casehub.api.model.AgentWorkerFunction;
```

Update `toResolutionStep` to call `resolveParameters` instead of `Map.of()`:

```java
private ResolutionStep toResolutionStep(
    PlanItemRecord record, Map<String, String> capabilityNameMap, int priority,
    CaseDefinition definition) {
  return new ResolutionStep(
      record.bindingName(),
      capabilityNameMap.get(record.bindingName()),
      record.executorName(),
      OUTCOME_MAP.getOrDefault(record.status(), RoutingOutcome.FAILURE).name(),
      priority,
      resolveParameters(record, definition),
      record.variantId());
}
```

Update the call site in `doRetain()` (around line 161) to pass `definition`:

```java
traces.add(toResolutionStep(sorted.get(i), capabilityNameMap, i, definition));
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CbrCaseRetainObserverTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all PASS

- [ ] **Step 5: Run full test suite for affected modules**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: populate modelId in ResolutionStep parameters at CBR retain time Closes #1096"
```

## References

- [2026-09-15-record-modelid-design.md] — design spec this plan implements
- [CbrCaseRetainObserver.java:335-344] — recording site for ResolutionStep
- [CbrRetrievalService.java:714-725] — passthrough mapping to ExperiencePlanStep
- [Agent.java] — model holder, gains modelId field
- [AgentConverter.java] — YAML-to-Agent wiring
- [ModelPreferenceSignalProvider.java:63-70] — downstream consumer in blocks
- [GitHub #1096] — focal issue
