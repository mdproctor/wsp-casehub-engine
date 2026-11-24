# Worker Backend Registration and CaseHub Auto-Creation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1360 — FlowWorkerFunctionProvider: register as backend for capabilities when workers have do: blocks
**Issue group:** #1360, #1361, #1362

**Goal:** Make YAML-defined workers with `do:` and `agent:` blocks fully functional at dispatch time, and auto-create `CaseHub` beans for auto-discovered YAML case definitions.

**Architecture:** Three independent fixes at the engine SPI level. #1360 diagnoses and fixes the parse→dispatch wiring for `FlowWorkerFunction`. #1361 widens `AgentWorkerFunctionProvider` to match all `agent:` block variants. #1362 adds `CaseHubRegistry` for auto-discovered YAML cases. The handler infrastructure (`FlowWorkerFunctionHandler`, `AgentWorkerFunctionHandler`) already works — these are registration/wiring fixes.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Arc), Serverless Workflow SDK, JUnit 5 + QuarkusTest

## Global Constraints

- Handlers are `@ApplicationScoped`, never `@DefaultBean` (PP-20260921-b7c277)
- `TESTCONTAINERS_RYUK_DISABLED=true` for all test commands
- `mvn install -DskipTests -q` before module-specific tests
- Test classes must be `*Test.java`, never `*IT.java`
- Inject repos/registries by SPI interface, never concrete class

---

## Batch 1: Flow Worker Backend Registration (#1360)

### Task 1: Diagnose and fix FlowWorkerFunction parse→dispatch wiring

**Files:**
- Investigate: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` (convertWorkers Phase 2)
- Investigate: `runtime-core/src/main/java/io/casehub/engine/runtime/worker/DefaultWorkerFunctionProviderRegistry.java`
- Investigate: `flow/src/main/java/io/casehub/engine/flow/quarkus/FlowBeans.java:51-55` (producer)
- Test: `flow/src/test/java/io/casehub/engine/flow/FlowWorkerBackendRegistrationTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionProviderRegistry.createFunction(JsonNode)`, `FlowWorkerFunctionProvider.handles(JsonNode)`, `FlowWorkerFunctionHandler.supports(WorkerFunction)`
- Produces: Integration test proving the full chain works; any fix to the wiring gap

- [ ] **Step 1: Write a failing integration test**

Create `flow/src/test/java/io/casehub/engine/flow/FlowWorkerBackendRegistrationTest.java`:

```java
package io.casehub.engine.flow;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.converter.CaseDefinitionYamlMapper;
import io.casehub.api.spi.WorkerFunctionProviderRegistry;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class FlowWorkerBackendRegistrationTest {

  @Inject WorkerFunctionProviderRegistry providerRegistry;
  @Inject ObjectMapper objectMapper;

  @Test
  void workerWithDoBlock_shouldHaveFlowWorkerFunction() throws Exception {
    String yaml =
        """
        name: test-flow-case
        namespace: test
        version: "1.0"
        workers:
          - name: notifier
            capabilities:
              - notify-customer
            do:
              - sendNotification:
                  call: http
                  with:
                    method: POST
                    endpoint:
                      uri: https://example.com/notify
        bindings:
          - name: on-notify
            when: "true"
            target:
              capability: notify-customer
        """;

    ObjectMapper yamlMapper = new ObjectMapper(new YAMLFactory());
    JsonNode root = yamlMapper.readTree(yaml);

    CaseDefinition definition =
        CaseDefinitionYamlMapper.load(root, objectMapper, null, providerRegistry, null);

    Worker notifier =
        definition.getWorkers().stream()
            .filter(w -> w.name().equals("notifier"))
            .findFirst()
            .orElseThrow();

    assertNotEquals(
        WorkerFunction.NONE,
        notifier.function(),
        "Worker with do: block should have a FlowWorkerFunction, not NONE");
    assertInstanceOf(FlowWorkerFunction.class, notifier.function());
  }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl flow -Dtest=FlowWorkerBackendRegistrationTest -DfailIfNoTests=false
```
Expected: FAIL — the worker's function is `WorkerFunction.NONE` (confirming the gap), OR PASS — the parse wiring works and the gap is elsewhere in the dispatch chain.

- [ ] **Step 3: Diagnose the gap based on test results**

If FAIL (function is NONE):
- Read `YamlCaseDefinitionConverter.convertWorkers()` Phase 2 to trace why the provider isn't invoked
- Check whether `FlowWorkerFunctionProvider` is in the CDI-resolved `Instance<WorkerFunctionProvider>` list
- Check whether the `rawWorkerNode` JSON structure passed to `handles()` contains the `"do"` key (it may be nested differently than expected)

If PASS (function is correct):
- The gap is in the dispatch chain, not parse. Write a second test that exercises the full dispatch:
  - Start a case with the definition
  - Trigger the binding
  - Verify `CompositeWorkerExecutionManager` can find a backend
  - Check `canExecute(worker.function())` on the available backends

- [ ] **Step 4: Implement the fix**

Apply the fix based on diagnosis. Likely scenarios:
- **Provider not in CDI:** Verify `FlowBeans.flowWorkerFunctionProvider()` is on the bean graph. Check the `flow` module is a dependency of the app module.
- **JSON structure mismatch:** The raw worker node may nest `do:` under a wrapper. Fix `handles()` to match the actual structure.
- **Registry iteration order:** Another provider may claim the node first. Add logging to `DefaultWorkerFunctionProviderRegistry.createFunction()` to trace which provider handles which node.

- [ ] **Step 5: Run the test to verify it passes**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl flow -Dtest=FlowWorkerBackendRegistrationTest -DfailIfNoTests=false
```
Expected: PASS

- [ ] **Step 6: Run the full flow module test suite**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl flow
```
Expected: All tests pass, no regressions.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "fix(#1360): wire FlowWorkerFunction through parse→dispatch chain

Refs #1360"
```

---

## Batch 2: Agent Worker Provider Coverage (#1361)

### Task 2: Widen AgentWorkerFunctionProvider to handle capability-based agent blocks

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java:28-44`
- Test: `engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java`

**Interfaces:**
- Consumes: `DeferredAgentConfig.CapabilityConfig(String capability, String domain, String model, ExpressionEvaluator inputEvaluator, ExpressionEvaluator outputEvaluator)`, `AgentWorkerFunction.ofCapability(CapabilityConfig)`
- Produces: `AgentWorkerFunctionProvider` that handles both `agent: { slot: ... }` and `agent: { capability: ... }` YAML structures

- [ ] **Step 1: Write failing tests for capability-based agent blocks**

Create `engine-support-core/src/test/java/io/casehub/engine/agent/AgentWorkerFunctionProviderTest.java`:

```java
package io.casehub.engine.agent;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.AgentWorkerFunction;
import io.casehub.api.model.ai.DeferredAgentConfig;
import io.casehub.worker.api.WorkerFunction;
import org.junit.jupiter.api.Test;

class AgentWorkerFunctionProviderTest {

  private final AgentWorkerFunctionProvider provider = new AgentWorkerFunctionProvider();
  private final ObjectMapper mapper = new ObjectMapper();

  @Test
  void handles_slotBased_returnsTrue() throws Exception {
    JsonNode node = mapper.readTree("""
        {"agent": {"slot": "classifier-agent"}}
        """);
    assertTrue(provider.handles(node));
  }

  @Test
  void handles_capabilityBased_returnsTrue() throws Exception {
    JsonNode node = mapper.readTree("""
        {"agent": {"capability": "text-classification", "domain": "support"}}
        """);
    assertTrue(provider.handles(node));
  }

  @Test
  void handles_noAgentBlock_returnsFalse() throws Exception {
    JsonNode node = mapper.readTree("""
        {"do": [{"step": {"call": "http"}}]}
        """);
    assertFalse(provider.handles(node));
  }

  @Test
  void create_slotBased_returnsSlotFunction() throws Exception {
    JsonNode node = mapper.readTree("""
        {"agent": {"slot": "classifier-agent", "model": "gpt-4o"}}
        """);
    WorkerFunction<?, ?> fn = provider.create(node);
    assertInstanceOf(AgentWorkerFunction.class, fn);
    AgentWorkerFunction agentFn = (AgentWorkerFunction) fn;
    assertTrue(agentFn.isDeferred());
    assertInstanceOf(DeferredAgentConfig.SlotConfig.class, agentFn.deferredConfig());
    DeferredAgentConfig.SlotConfig sc = (DeferredAgentConfig.SlotConfig) agentFn.deferredConfig();
    assertEquals("classifier-agent", sc.slot());
    assertEquals("gpt-4o", sc.model());
  }

  @Test
  void create_capabilityBased_returnsCapabilityFunction() throws Exception {
    JsonNode node = mapper.readTree("""
        {"agent": {"capability": "text-classification", "domain": "support", "model": "claude-sonnet"}}
        """);
    WorkerFunction<?, ?> fn = provider.create(node);
    assertInstanceOf(AgentWorkerFunction.class, fn);
    AgentWorkerFunction agentFn = (AgentWorkerFunction) fn;
    assertTrue(agentFn.isDeferred());
    assertInstanceOf(DeferredAgentConfig.CapabilityConfig.class, agentFn.deferredConfig());
    DeferredAgentConfig.CapabilityConfig cc =
        (DeferredAgentConfig.CapabilityConfig) agentFn.deferredConfig();
    assertEquals("text-classification", cc.capability());
    assertEquals("support", cc.domain());
    assertEquals("claude-sonnet", cc.model());
  }
}
```

- [ ] **Step 2: Run tests to verify capability-based tests fail**

Run:
```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=AgentWorkerFunctionProviderTest -DfailIfNoTests=false
```
Expected: `handles_capabilityBased_returnsTrue` and `create_capabilityBased_returnsCapabilityFunction` FAIL. Slot-based tests PASS.

- [ ] **Step 3: Widen handles() and create()**

Use `ide_replace_member` on `AgentWorkerFunctionProvider`:

**`handles()`** — change from requiring `slot` to accepting any `agent:` block:
```java
@Override
public boolean handles(JsonNode rawWorkerNode) {
  if (!rawWorkerNode.has("agent")) {
    return false;
  }
  JsonNode agentNode = rawWorkerNode.get("agent");
  return agentNode.has("slot") || agentNode.has("capability");
}
```

**`create()`** — add capability-based branch:
```java
@Override
public WorkerFunction<?, ?> create(JsonNode rawWorkerNode) {
  JsonNode agentNode = rawWorkerNode.get("agent");
  if (agentNode.has("slot")) {
    String slot = agentNode.get("slot").asText();
    String model = textOrNull(agentNode, "model");
    String userMessage = textOrNull(agentNode, "userMessage");
    String inputProjection = textOrNull(agentNode, "inputProjection");
    String outputProjection = textOrNull(agentNode, "outputProjection");
    return AgentWorkerFunction.ofSlot(
        new DeferredAgentConfig.SlotConfig(
            slot, model, userMessage, inputProjection, outputProjection));
  }
  if (agentNode.has("capability")) {
    String capability = agentNode.get("capability").asText();
    String domain = textOrNull(agentNode, "domain");
    String model = textOrNull(agentNode, "model");
    return AgentWorkerFunction.ofCapability(
        new DeferredAgentConfig.CapabilityConfig(capability, domain, model, null, null));
  }
  throw new IllegalArgumentException("agent: block requires 'slot' or 'capability'");
}
```

- [ ] **Step 4: Run tests to verify all pass**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=AgentWorkerFunctionProviderTest -DfailIfNoTests=false
```
Expected: All 5 tests PASS.

- [ ] **Step 5: Run the full engine-support-core test suite**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core
```
Expected: All tests pass, no regressions.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "fix(#1361): widen AgentWorkerFunctionProvider to handle capability-based agent blocks

AgentWorkerFunctionProvider.handles() now matches agent: blocks with
either slot: or capability:. create() produces AgentWorkerFunction.ofSlot()
or AgentWorkerFunction.ofCapability() accordingly.

Closes #1361"
```

---

## Batch 3: CaseHub Auto-Creation (#1362)

### Task 3: Add protected constructor to CaseHub for non-CDI usage

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/CaseHub.java:22-24`
- Test: `api/src/test/java/io/casehub/api/engine/CaseHubTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime` interface
- Produces: `CaseHub(CaseHubRuntime runtime)` protected constructor that sets the `runtime` field

- [ ] **Step 1: Write a failing test for constructor-based CaseHub**

Create `api/src/test/java/io/casehub/api/engine/CaseHubTest.java`:

```java
package io.casehub.api.engine;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import io.casehub.api.model.CaseDefinition;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CaseHubTest {

  @Test
  void constructorWithRuntime_delegatesStartCase() {
    CaseHubRuntime mockRuntime = mock(CaseHubRuntime.class);
    CaseDefinition mockDef = mock(CaseDefinition.class);
    UUID expectedId = UUID.randomUUID();
    when(mockRuntime.startCase(mockDef)).thenReturn(expectedId);

    CaseHub hub =
        new CaseHub(mockRuntime) {
          @Override
          public CaseDefinition getDefinition() {
            return mockDef;
          }
        };

    assertEquals(expectedId, hub.startCase());
    verify(mockRuntime).startCase(mockDef);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=CaseHubTest -DfailIfNoTests=false
```
Expected: FAIL — compilation error, no such constructor.

- [ ] **Step 3: Add protected constructor to CaseHub**

Use `ide_insert_member` to add after the `runtime` field (line 24):

```java
protected CaseHub() {}

protected CaseHub(CaseHubRuntime runtime) {
  this.runtime = runtime;
}
```

The no-arg constructor is implicit today (CDI uses it). Making it explicit keeps backward compatibility.

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=CaseHubTest -DfailIfNoTests=false
```
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1362): add protected CaseHub constructor accepting CaseHubRuntime

Enables non-CDI-managed CaseHub instances (e.g. from CaseHubRegistry)
while keeping the no-arg constructor for CDI subclasses.

Refs #1362"
```

### Task 4: Create CaseHubRegistry and wire into CaseYamlRegistrar

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/runtime/engine/CaseHubRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrar.java:38-87`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/engine/CaseHubRegistryTest.java`

**Interfaces:**
- Consumes: `CaseHub(CaseHubRuntime)` constructor (from Task 3), `CaseDefinition.getName()`, `CaseDefinitionRegistry.registerCaseDefinition(CaseDefinition)`, `CaseDefinitionRegistry.findByName(String)`
- Produces: `CaseHubRegistry` with `register(String, CaseHub)`, `get(String) → Optional<CaseHub>`, `all() → Collection<CaseHub>`

- [ ] **Step 1: Write failing tests for CaseHubRegistry**

Create `runtime/src/test/java/io/casehub/engine/runtime/engine/CaseHubRegistryTest.java`:

```java
package io.casehub.engine.runtime.engine;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.CaseDefinition;
import org.junit.jupiter.api.Test;

class CaseHubRegistryTest {

  @Test
  void registerAndRetrieve() {
    CaseHubRegistry registry = new CaseHubRegistry();
    CaseHub hub = createStubHub("test-case");

    registry.register("test-case", hub);

    assertTrue(registry.get("test-case").isPresent());
    assertEquals(hub, registry.get("test-case").get());
  }

  @Test
  void getUnknown_returnsEmpty() {
    CaseHubRegistry registry = new CaseHubRegistry();
    assertTrue(registry.get("nonexistent").isEmpty());
  }

  @Test
  void all_returnsAllRegistered() {
    CaseHubRegistry registry = new CaseHubRegistry();
    registry.register("case-a", createStubHub("case-a"));
    registry.register("case-b", createStubHub("case-b"));

    assertEquals(2, registry.all().size());
  }

  @Test
  void duplicateRegistration_secondWins() {
    CaseHubRegistry registry = new CaseHubRegistry();
    CaseHub hub1 = createStubHub("same-name");
    CaseHub hub2 = createStubHub("same-name");

    registry.register("same-name", hub1);
    registry.register("same-name", hub2);

    assertEquals(hub2, registry.get("same-name").get());
  }

  private static CaseHub createStubHub(String name) {
    CaseHubRuntime runtime = mock(CaseHubRuntime.class);
    CaseDefinition def = mock(CaseDefinition.class);
    when(def.getName()).thenReturn(name);
    return new CaseHub(runtime) {
      @Override
      public CaseDefinition getDefinition() {
        return def;
      }
    };
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=CaseHubRegistryTest -DfailIfNoTests=false
```
Expected: FAIL — `CaseHubRegistry` class does not exist.

- [ ] **Step 3: Create CaseHubRegistry**

Create `runtime/src/main/java/io/casehub/engine/runtime/engine/CaseHubRegistry.java`:

```java
package io.casehub.engine.runtime.engine;

import io.casehub.api.engine.CaseHub;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Collection;
import java.util.Collections;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

@ApplicationScoped
public class CaseHubRegistry {

  private final ConcurrentMap<String, CaseHub> hubs = new ConcurrentHashMap<>();

  public void register(String name, CaseHub hub) {
    hubs.put(name, hub);
  }

  public Optional<CaseHub> get(String name) {
    return Optional.ofNullable(hubs.get(name));
  }

  public Collection<CaseHub> all() {
    return Collections.unmodifiableCollection(hubs.values());
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=CaseHubRegistryTest -DfailIfNoTests=false
```
Expected: All 4 tests PASS.

- [ ] **Step 5: Wire CaseHubRegistry into CaseYamlRegistrar**

Use `ide_edit_member` on `CaseYamlRegistrar`:

Add fields:
```java
@Inject CaseHubRegistry caseHubRegistry;
@Inject CaseHubRuntime caseHubRuntime;
```

In `register()` method, after `caseDefinitionRegistry.registerCaseDefinition(definition)` (line 80), add:

```java
CaseHub hub =
    new CaseHub(caseHubRuntime) {
      private final CaseDefinition def = definition;

      @Override
      public CaseDefinition getDefinition() {
        return def;
      }
    };
caseHubRegistry.register(definition.getName(), hub);
```

- [ ] **Step 6: Verify coexistence with explicit CaseHub beans**

Both registration paths coexist:
- `CaseYamlRegistrar` (classpath bootstrap, priority 200) registers definitions AND creates registry CaseHubs
- `DefaultCaseDefinitionRegistry.registerKnownDefinitions()` (StartupEvent, priority 10) registers definitions from CDI `CaseHub` beans

`CaseDefinitionRegistry.registerCaseDefinition()` handles duplicate definitions (same namespace/name/version). The `CaseHubRegistry` only holds auto-discovered hubs — if an explicit `YamlCaseHub` subclass exists, consumers should use it directly via CDI injection, not via the registry. No dedup needed: the two lookups serve different consumers.

- [ ] **Step 7: Run the full runtime test suite**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```
Expected: All tests pass, no regressions.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#1362): add CaseHubRegistry and wire into CaseYamlRegistrar

CaseYamlRegistrar now creates a CaseHub instance for each auto-discovered
YAML case definition and registers it in CaseHubRegistry. Consumers inject
CaseHubRegistry and look up by name.

Closes #1362"
```

---

## References

- [2026-11-24-worker-backend-registration-design.md] — design spec this plan implements
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionProvider.java` — flow provider
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java` — flow handler (exists)
- `flow/src/main/java/io/casehub/engine/flow/quarkus/FlowBeans.java:51-64` — CDI producers for both
- `engine-support-core/src/main/java/io/casehub/engine/agent/AgentWorkerFunctionProvider.java` — agent provider to widen
- `api/src/main/java/io/casehub/api/model/ai/DeferredAgentConfig.java` — SlotConfig + CapabilityConfig
- `api/src/main/java/io/casehub/api/engine/CaseHub.java` — abstract base, needs constructor
- `runtime/src/main/java/io/casehub/engine/runtime/bootstrap/CaseYamlRegistrar.java` — wire registry
- `runtime-core/src/main/java/io/casehub/engine/runtime/worker/CompositeWorkerExecutionManager.java` — dispatch chain
- [PP-20260921-b7c277] — no @DefaultBean on multi-instance SPI
- [GitHub #1360, #1361, #1362] — focal issues
