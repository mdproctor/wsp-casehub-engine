# CBR Routing Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #706 — epic: CBR-informed routing pipeline
**Issue group:** #703, #505 (#705 closed as superseded)

**Goal:** Wire the end-to-end CBR pipeline: cases close → outcomes stored → similar cases retrieved → strategies consume retrieved experiences.

**Architecture:** Two issues on a single branch. #703 adds `CbrCaseRetainObserver` in engine runtime — stores `PlanCbrCase` entries on case terminal state. #505 refactors blocks strategies (`CbrAgentRoutingStrategy`, `CbrRoutingPromptSection`) to read `context.experiences()` instead of querying the store directly, and removes the now-orphaned `CbrRoutingOutcomeRecorder`, `RoutingFeatureExtractor`, and `TextOnlyFeatureExtractor`.

**Tech Stack:** Java 21, Quarkus 3.32, CDI, jackson-jq, neocortex-memory-cbr API

## Global Constraints

- Pre-release — breaking changes cost nothing
- IntelliJ MCP mandatory for all .java edits — use `ide_edit_member`, `ide_replace_member`, `ide_insert_member`, `ide_refactor_rename`, `ide_move_file`, `ide_refactor_safe_delete`
- Workspace project path: use sub-project paths (e.g. `/Users/mdproctor/claude/casehub/engine/api`, `/Users/mdproctor/claude/casehub/engine/runtime`, `/Users/mdproctor/claude/casehub/blocks`)
- Tests use `@QuarkusTest` in runtime; plain JUnit in api
- `CbrCaseMemoryStore` is from `casehub-neocortex-memory-api` (external dependency, already on classpath)
- `NoOpCbrCaseMemoryStore` is `@DefaultBean` — always resolvable, absorbs calls when no real store
- All engine SPI changes follow the SPI evolution protocol (default methods for additions)

---

### Task 1: Enrich CaseOutcomeEvent with tenancyId

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/CaseOutcomeEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` (method `fireOutcomeObservers`)
- Modify: `runtime/src/test/java/io/casehub/engine/CaseOutcomeObserverTest.java`

**Interfaces:**
- Consumes: `CaseInstance.tenancyId` (public field)
- Produces: `CaseOutcomeEvent.tenancyId()` — used by Task 2

- [ ] **Step 1: Update existing test to assert tenancyId**

In `CaseOutcomeObserverTest.caseOutcomeObserver_called_when_case_completes()`, add assertion after the existing assertions:

```java
assertThat(event.tenancyId()).isEqualTo("test-tenant");
```

Use `ide_replace_member` on the test method to add this line after `assertThat(event.closedAt()).isNotNull();`.

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CaseOutcomeObserverTest#caseOutcomeObserver_called_when_case_completes -q
```

Expected: compilation error — `tenancyId()` not on `CaseOutcomeEvent`.

- [ ] **Step 3: Add tenancyId to CaseOutcomeEvent record**

Use `ide_edit_member` on `CaseOutcomeEvent` to add `tenancyId` as the second parameter:

```java
public record CaseOutcomeEvent(
    String caseType,
    String tenancyId,
    UUID caseId,
    Map<String, Object> caseFileSnapshot,
    String outcomeLabel,
    Instant closedAt,
    Map<String, Object> metadata) {}
```

- [ ] **Step 4: Update fireOutcomeObservers to pass tenancyId**

In `CaseStatusChangedHandler.fireOutcomeObservers()`, use `ide_replace_member` to update the `CaseOutcomeEvent` construction to include `caseInstance.tenancyId`:

```java
final CaseOutcomeEvent outcomeEvent =
    new CaseOutcomeEvent(
        caseType,
        caseInstance.tenancyId,
        caseInstance.getUuid(),
        snapshot,
        newState.name(),
        Instant.now(),
        outcomeMetadata);
```

- [ ] **Step 5: Run test to verify it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CaseOutcomeObserverTest -q
```

Expected: all 3 tests PASS.

- [ ] **Step 6: Build full project to catch compilation errors**

```bash
/opt/homebrew/bin/mvn install -DskipTests -q
```

Use `ide_diagnostics` with `includeBuildErrors=true` to check for any compilation errors from callers of `CaseOutcomeEvent`.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/spi/CaseOutcomeEvent.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java runtime/src/test/java/io/casehub/engine/CaseOutcomeObserverTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#703): add tenancyId to CaseOutcomeEvent

Refs #703"
```

---

### Task 2: Create SnapshotCaseContext adapter

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/memory/SnapshotCaseContext.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/memory/SnapshotCaseContextTest.java`

**Interfaces:**
- Consumes: `CaseContext` interface (`io.casehub.api.context.CaseContext`), `ReadableLayer` interface, `ContextLayer.WORKING`
- Produces: `SnapshotCaseContext(Map<String, Object>)` constructor — used by Task 3

- [ ] **Step 1: Write tests for SnapshotCaseContext**

Create `runtime/src/test/java/io/casehub/engine/internal/memory/SnapshotCaseContextTest.java`:

```java
package io.casehub.engine.internal.memory;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.casehub.api.context.ContextLayer;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SnapshotCaseContextTest {

  @Test
  void working_layer_returns_snapshot_data() {
    var snapshot = Map.<String, Object>of("txn", "SAR-001", "amount", 50000);
    var ctx = new SnapshotCaseContext(snapshot);

    assertThat(ctx.layer(ContextLayer.WORKING).getData()).isEqualTo(snapshot);
    assertThat(ctx.layer(ContextLayer.WORKING).get("txn")).isEqualTo("SAR-001");
    assertThat(ctx.layer(ContextLayer.WORKING).get("amount")).isEqualTo(50000);
  }

  @Test
  void working_layer_asJsonNode_returns_json() {
    var ctx = new SnapshotCaseContext(Map.of("key", "value"));
    var node = ctx.layer(ContextLayer.WORKING).asJsonNode();

    assertThat(node.has("key")).isTrue();
    assertThat(node.get("key").asText()).isEqualTo("value");
  }

  @Test
  void non_working_layers_return_empty() {
    var ctx = new SnapshotCaseContext(Map.of("key", "value"));

    assertThat(ctx.layer(ContextLayer.SEMANTIC).getData()).isEmpty();
    assertThat(ctx.layer(ContextLayer.EPISODIC).getData()).isEmpty();
    assertThat(ctx.layer("unknown").getData()).isEmpty();
  }

  @Test
  void getData_returns_snapshot() {
    var data = Map.<String, Object>of("a", 1);
    var ctx = new SnapshotCaseContext(data);
    assertThat(ctx.getData()).isEqualTo(data);
  }

  @Test
  void get_delegates_to_snapshot() {
    var ctx = new SnapshotCaseContext(Map.of("k", "v"));
    assertThat(ctx.get("k")).isEqualTo("v");
    assertThat(ctx.get("missing")).isNull();
  }

  @Test
  void mutation_methods_throw() {
    var ctx = new SnapshotCaseContext(Map.of());
    assertThatThrownBy(() -> ctx.set("k", "v"))
        .isInstanceOf(UnsupportedOperationException.class);
    assertThatThrownBy(() -> ctx.remove("k"))
        .isInstanceOf(UnsupportedOperationException.class);
    assertThatThrownBy(() -> ctx.clear())
        .isInstanceOf(UnsupportedOperationException.class);
    assertThatThrownBy(() -> ctx.setAll(Map.of()))
        .isInstanceOf(UnsupportedOperationException.class);
  }

  @Test
  void contains_and_size() {
    var ctx = new SnapshotCaseContext(Map.of("a", 1, "b", 2));
    assertThat(ctx.contains("a")).isTrue();
    assertThat(ctx.contains("c")).isFalse();
    assertThat(ctx.size()).isEqualTo(2);
    assertThat(ctx.isEmpty()).isFalse();
    assertThat(ctx.getKeys()).containsExactlyInAnyOrder("a", "b");
  }

  @Test
  void empty_snapshot() {
    var ctx = new SnapshotCaseContext(Map.of());
    assertThat(ctx.isEmpty()).isTrue();
    assertThat(ctx.size()).isZero();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=SnapshotCaseContextTest -q
```

Expected: compilation error — `SnapshotCaseContext` does not exist.

- [ ] **Step 3: Implement SnapshotCaseContext**

Create `runtime/src/main/java/io/casehub/engine/internal/memory/SnapshotCaseContext.java`:

```java
package io.casehub.engine.internal.memory;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ContextLayer;
import io.casehub.api.context.ReadableLayer;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.function.Consumer;
import java.util.function.Function;

/**
 * Read-only CaseContext backed by a Map snapshot from CaseOutcomeEvent.caseFileSnapshot().
 *
 * <p>Only the WORKING layer is populated. All other layers return empty data. Mutation methods
 * throw UnsupportedOperationException. Lambda feature extractors that access non-WORKING layers
 * will receive empty results — this is inherent to retain-time context where only the final
 * working layer snapshot is available after case close.
 */
final class SnapshotCaseContext implements CaseContext {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  private final Map<String, Object> data;
  private final ReadableLayer workingLayer;
  private final ReadableLayer emptyLayer;

  SnapshotCaseContext(Map<String, Object> snapshot) {
    this.data = Map.copyOf(Objects.requireNonNull(snapshot));
    this.workingLayer = new SnapshotReadableLayer(ContextLayer.WORKING, this.data);
    this.emptyLayer = new SnapshotReadableLayer("empty", Map.of());
  }

  @Override
  public ReadableLayer layer(String name) {
    return ContextLayer.WORKING.equals(name) ? workingLayer : emptyLayer;
  }

  @Override
  public Map<String, Object> getData() {
    return data;
  }

  @Override
  public Object get(String key) {
    return data.get(key);
  }

  @Override
  public <T> T getAs(String key, Class<T> type) {
    return type.cast(data.get(key));
  }

  @Override
  public <T> T getOrDefault(String key, T defaultValue) {
    @SuppressWarnings("unchecked")
    T value = (T) data.get(key);
    return value != null ? value : defaultValue;
  }

  @Override
  public String getString(String key) {
    Object v = data.get(key);
    return v != null ? v.toString() : null;
  }

  @Override
  public Integer getInt(String key) {
    Object v = data.get(key);
    return v instanceof Number n ? n.intValue() : null;
  }

  @Override
  public Long getLong(String key) {
    Object v = data.get(key);
    return v instanceof Number n ? n.longValue() : null;
  }

  @Override
  public Double getDouble(String key) {
    Object v = data.get(key);
    return v instanceof Number n ? n.doubleValue() : null;
  }

  @Override
  public Boolean getBoolean(String key) {
    Object v = data.get(key);
    return v instanceof Boolean b ? b : null;
  }

  @Override
  public <T> List<T> getList(String key, Class<T> elementType) {
    return List.of();
  }

  @Override
  public Object getPath(String path) {
    return null;
  }

  @Override
  public String getPathAsString(String path) {
    return null;
  }

  @Override
  public boolean contains(String key) {
    return data.containsKey(key);
  }

  @Override
  public Set<String> getKeys() {
    return data.keySet();
  }

  @Override
  public boolean isEmpty() {
    return data.isEmpty();
  }

  @Override
  public int size() {
    return data.size();
  }

  @Override
  public JsonNode asJsonNode() {
    return MAPPER.valueToTree(data);
  }

  @Override
  public Map<String, Object> getAll(String... keys) {
    return Map.of();
  }

  @Override
  public long getVersion() {
    return 0;
  }

  @Override
  public CaseContext snapshot() {
    return this;
  }

  @Override
  public JsonNode diff(CaseContext other) {
    return MAPPER.createObjectNode();
  }

  @Override
  public CaseContext merge(CaseContext other) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  // --- Mutation methods: all throw ---

  @Override
  public CaseContext set(String key, Object value) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public Object computeIfAbsent(String key, Function<String, Object> fn) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public Object putIfAbsent(String key, Object value) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public boolean compareAndSet(String key, Object expected, Object newValue) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public CaseContext update(String key, Function<Object, Object> fn) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public CaseContext setPath(String path, Object value) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public Optional<JsonNode> applyAndDiff(String path, Object value) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public CaseContext setAll(Map<String, Object> values) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public CaseContext remove(String key) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public CaseContext clear() {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  @Override
  public void applyDiff(JsonNode diff) {
    throw new UnsupportedOperationException("SnapshotCaseContext is read-only");
  }

  private record SnapshotReadableLayer(String layerName, Map<String, Object> data)
      implements ReadableLayer {

    @Override
    public boolean isReadOnly() {
      return true;
    }

    @Override
    public Object get(String key) {
      return data.get(key);
    }

    @Override
    public <T> T getAs(String key, Class<T> type) {
      return type.cast(data.get(key));
    }

    @Override
    public <T> T getOrDefault(String key, T defaultValue) {
      @SuppressWarnings("unchecked")
      T value = (T) data.get(key);
      return value != null ? value : defaultValue;
    }

    @Override
    public String getString(String key) {
      Object v = data.get(key);
      return v != null ? v.toString() : null;
    }

    @Override
    public Integer getInt(String key) {
      Object v = data.get(key);
      return v instanceof Number n ? n.intValue() : null;
    }

    @Override
    public Long getLong(String key) {
      Object v = data.get(key);
      return v instanceof Number n ? n.longValue() : null;
    }

    @Override
    public Double getDouble(String key) {
      Object v = data.get(key);
      return v instanceof Number n ? n.doubleValue() : null;
    }

    @Override
    public Boolean getBoolean(String key) {
      Object v = data.get(key);
      return v instanceof Boolean b ? b : null;
    }

    @Override
    public <T> List<T> getList(String key, Class<T> elementType) {
      return List.of();
    }

    @Override
    public boolean contains(String key) {
      return data.containsKey(key);
    }

    @Override
    public Set<String> getKeys() {
      return data.keySet();
    }

    @Override
    public Map<String, Object> getData() {
      return data;
    }

    @Override
    public Map<String, Object> getAll(String... keys) {
      return Map.of();
    }

    @Override
    public Object getPath(String path) {
      return null;
    }

    @Override
    public String getPathAsString(String path) {
      return null;
    }

    @Override
    public boolean isEmpty() {
      return data.isEmpty();
    }

    @Override
    public int size() {
      return data.size();
    }

    @Override
    public long getVersion() {
      return 0;
    }

    @Override
    public JsonNode asJsonNode() {
      return MAPPER.valueToTree(data);
    }

    @Override
    public ReadableLayer snapshot() {
      return this;
    }

    @Override
    public String layerName() {
      return layerName;
    }
  }
}
```

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=SnapshotCaseContextTest -q
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/memory/SnapshotCaseContext.java runtime/src/test/java/io/casehub/engine/internal/memory/SnapshotCaseContextTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#703): add SnapshotCaseContext read-only adapter

Read-only CaseContext wrapping caseFileSnapshot Map for Lambda feature
extraction at retain time. Only WORKING layer populated.

Refs #703"
```

---

### Task 3: Implement CbrCaseRetainObserver

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java`

**Interfaces:**
- Consumes: `CaseOutcomeObserver` SPI, `CaseOutcomeEvent.tenancyId()` (Task 1), `SnapshotCaseContext` (Task 2), `CaseDefinitionRegistry.findByName()`, `PlanItemStore.findByCaseId()`, `CbrCaseMemoryStore.store()`, `JQEvaluator.eval()`, `CbrConfig`, `FeatureExtractor` (sealed: `JqFeatureExtractor`, `LambdaFeatureExtractor`), `PlanItemRecord`, `PlanCbrCase`, `PlanTrace`, `TaskStatus.isTerminal()`, `CapabilityTarget`
- Produces: `CbrCaseRetainObserver` (`@ApplicationScoped`, implements `CaseOutcomeObserver`) — called by `CaseStatusChangedHandler` on terminal case state

- [ ] **Step 1: Write tests**

Create `runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java`:

```java
package io.casehub.engine.internal.memory;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.HumanTaskTarget;
import io.casehub.api.model.TaskStatus;
import io.casehub.api.model.cbr.CbrConfig;
import io.casehub.api.model.cbr.JqFeatureExtractor;
import io.casehub.api.model.cbr.LambdaFeatureExtractor;
import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.engine.common.internal.jq.JQEvaluator;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.internal.engine.DefaultCaseDefinitionRegistry;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.cbr.CbrQuery;
import io.casehub.neocortex.memory.cbr.EraseRequest;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import io.casehub.worker.api.Capability;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CbrCaseRetainObserverTest {

  private RecordingCbrStore store;
  private StubRegistry registry;
  private StubPlanItemStore planItemStore;
  private JQEvaluator jqEvaluator;
  private CbrCaseRetainObserver observer;

  @BeforeEach
  void setUp() {
    store = new RecordingCbrStore();
    registry = new StubRegistry();
    planItemStore = new StubPlanItemStore();
    jqEvaluator = new JQEvaluator();
    observer = new CbrCaseRetainObserver(store, registry, planItemStore, jqEvaluator);
  }

  @Test
  void stores_plan_cbr_case_on_completed_case() {
    var def = definitionWithJqCbr("test-case", "test-domain",
        Map.of("amount", ".transaction.amount"));
    def.addBinding(capabilityBinding("assess-risk", "risk-assessment"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("assess-risk", "worker-1", TaskStatus.COMPLETED));

    observer.onOutcome(event("test-case", "COMPLETED",
        Map.of("transaction", Map.of("amount", 50000))));

    assertThat(store.storedCases).hasSize(1);
    PlanCbrCase stored = store.storedCases.get(0);
    assertThat(stored.problem()).isEqualTo("test-case");
    assertThat(stored.outcome()).isEqualTo("COMPLETED");
    assertThat(stored.features()).containsEntry("amount", 50000);
    assertThat(stored.planTrace()).hasSize(1);
    assertThat(stored.planTrace().get(0).bindingName()).isEqualTo("assess-risk");
    assertThat(stored.planTrace().get(0).capabilityName()).isEqualTo("risk-assessment");
    assertThat(stored.planTrace().get(0).workerName()).isEqualTo("worker-1");
    assertThat(stored.planTrace().get(0).stepOutcome()).isEqualTo("SUCCESS");
  }

  @Test
  void stores_with_correct_store_parameters() {
    var def = definitionWithJqCbr("my-case", "my-domain", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);
    planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.COMPLETED));

    observer.onOutcome(event("my-case", "COMPLETED", Map.of("k", "v"),
        "tenant-42", UUID.fromString("00000000-0000-0000-0000-000000000001")));

    assertThat(store.lastCaseType).isEqualTo("my-case");
    assertThat(store.lastEntityId).isEqualTo("case-retain");
    assertThat(store.lastDomain.name()).isEqualTo("my-domain");
    assertThat(store.lastTenantId).isEqualTo("tenant-42");
    assertThat(store.lastCaseId).isEqualTo("00000000-0000-0000-0000-000000000001");
  }

  @Test
  void no_store_call_when_definition_not_found() {
    observer.onOutcome(event("unknown-case", "COMPLETED", Map.of()));
    assertThat(store.storedCases).isEmpty();
  }

  @Test
  void no_store_call_when_no_cbr_config() {
    var def = CaseDefinition.builder().name("no-cbr").namespace("test").build();
    registry.register(def);

    observer.onOutcome(event("no-cbr", "COMPLETED", Map.of()));
    assertThat(store.storedCases).isEmpty();
  }

  @Test
  void no_store_call_when_domain_unresolvable() {
    var def = definitionWithJqCbr("d-case", null, Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);

    observer.onOutcome(event("d-case", "COMPLETED", Map.of("k", "v")));
    assertThat(store.storedCases).isEmpty();
  }

  @Test
  void no_store_call_when_features_empty() {
    var def = definitionWithJqCbr("e-case", "dom", Map.of("k", ".nonexistent"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);
    planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.COMPLETED));

    observer.onOutcome(event("e-case", "COMPLETED", Map.of("other", "val")));
    assertThat(store.storedCases).isEmpty();
  }

  @Test
  void no_store_call_when_filtered_trace_empty() {
    var def = definitionWithJqCbr("t-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);
    planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.PENDING));

    observer.onOutcome(event("t-case", "COMPLETED", Map.of("k", "v")));
    assertThat(store.storedCases).isEmpty();
  }

  @Test
  void filters_non_capability_bindings() {
    var def = definitionWithJqCbr("f-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("cap-bind", "cap1"));
    def.addBinding(humanTaskBinding("ht-bind"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("cap-bind", "w1", TaskStatus.COMPLETED),
        planItem("ht-bind", "w2", TaskStatus.COMPLETED));

    observer.onOutcome(event("f-case", "COMPLETED", Map.of("k", "v")));

    assertThat(store.storedCases).hasSize(1);
    assertThat(store.storedCases.get(0).planTrace()).hasSize(1);
    assertThat(store.storedCases.get(0).planTrace().get(0).bindingName()).isEqualTo("cap-bind");
  }

  @Test
  void filters_non_terminal_plan_items() {
    var def = definitionWithJqCbr("nt-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    def.addBinding(capabilityBinding("b2", "cap2"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("b1", "w1", TaskStatus.COMPLETED),
        planItem("b2", "w2", TaskStatus.RUNNING));

    observer.onOutcome(event("nt-case", "COMPLETED", Map.of("k", "v")));

    assertThat(store.storedCases.get(0).planTrace()).hasSize(1);
    assertThat(store.storedCases.get(0).planTrace().get(0).bindingName()).isEqualTo("b1");
  }

  @Test
  void filters_null_executor_name() {
    var def = definitionWithJqCbr("ne-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    def.addBinding(capabilityBinding("b2", "cap2"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("b1", "w1", TaskStatus.COMPLETED),
        planItem("b2", null, TaskStatus.CANCELLED));

    observer.onOutcome(event("ne-case", "CANCELLED", Map.of("k", "v")));

    assertThat(store.storedCases.get(0).planTrace()).hasSize(1);
  }

  @Test
  void maps_task_status_to_outcome_strings() {
    var def = definitionWithJqCbr("os-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "c1"));
    def.addBinding(capabilityBinding("b2", "c2"));
    def.addBinding(capabilityBinding("b3", "c3"));
    def.addBinding(capabilityBinding("b4", "c4"));
    def.addBinding(capabilityBinding("b5", "c5"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("b1", "w1", TaskStatus.COMPLETED),
        planItem("b2", "w2", TaskStatus.FAULTED),
        planItem("b3", "w3", TaskStatus.REJECTED),
        planItem("b4", "w4", TaskStatus.CANCELLED),
        planItem("b5", "w5", TaskStatus.OBSOLETE));

    observer.onOutcome(event("os-case", "FAULTED", Map.of("k", "v")));

    var traces = store.storedCases.get(0).planTrace();
    assertThat(traces).extracting("stepOutcome")
        .containsExactly("SUCCESS", "FAILURE", "DECLINED", "CANCELLED", "OBSOLETE");
  }

  @Test
  void solution_synthesis() {
    var def = definitionWithJqCbr("sol-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("assess", "cap1"));
    def.addBinding(capabilityBinding("review", "cap2"));
    registry.register(def);
    planItemStore.items = List.of(
        planItem("assess", "agent-1", TaskStatus.COMPLETED),
        planItem("review", "agent-2", TaskStatus.FAULTED));

    observer.onOutcome(event("sol-case", "FAULTED", Map.of("k", "v")));

    assertThat(store.storedCases.get(0).solution())
        .isEqualTo("assess→agent-1(SUCCESS), review→agent-2(FAILURE)");
  }

  @Test
  void lambda_feature_extraction() {
    var lambda = new LambdaFeatureExtractor(ctx -> Map.of("extracted", ctx.get("raw")));
    var config = CbrConfig.builder().featureExtractor(lambda).domain("dom").build();
    var def = CaseDefinition.builder().name("lambda-case").namespace("test")
        .cbrConfig(config).build();
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);
    planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.COMPLETED));

    observer.onOutcome(event("lambda-case", "COMPLETED", Map.of("raw", "data")));

    assertThat(store.storedCases.get(0).features()).containsEntry("extracted", "data");
  }

  @Test
  void store_exception_does_not_propagate() {
    var def = definitionWithJqCbr("err-case", "dom", Map.of("k", ".k"));
    def.addBinding(capabilityBinding("b1", "cap1"));
    registry.register(def);
    planItemStore.items = List.of(planItem("b1", "w1", TaskStatus.COMPLETED));
    store.throwOnStore = new RuntimeException("store down");

    assertThatCode(() ->
        observer.onOutcome(event("err-case", "COMPLETED", Map.of("k", "v")))
    ).doesNotThrowAnyException();
  }

  // --- Helpers ---

  private CaseOutcomeEvent event(String caseType, String outcome, Map<String, Object> snapshot) {
    return event(caseType, outcome, snapshot, "test-tenant", UUID.randomUUID());
  }

  private CaseOutcomeEvent event(String caseType, String outcome, Map<String, Object> snapshot,
      String tenancyId, UUID caseId) {
    return new CaseOutcomeEvent(caseType, tenancyId, caseId, snapshot, outcome,
        Instant.now(), Map.of());
  }

  private CaseDefinition definitionWithJqCbr(String name, String domain,
      Map<String, String> features) {
    var jq = JqFeatureExtractor.of(features);
    var configBuilder = CbrConfig.builder().featureExtractor(jq);
    if (domain != null) configBuilder.domain(domain);
    var def = CaseDefinition.builder().name(name).namespace("test")
        .cbrConfig(configBuilder.build()).build();
    return def;
  }

  private Binding capabilityBinding(String bindingName, String capabilityName) {
    return Binding.builder()
        .name(bindingName)
        .target(new CapabilityTarget(new Capability(capabilityName)))
        .build();
  }

  private Binding humanTaskBinding(String bindingName) {
    return Binding.builder()
        .name(bindingName)
        .target(HumanTaskTarget.builder().build())
        .build();
  }

  private PlanItemRecord planItem(String bindingName, String executorName, TaskStatus status) {
    return new PlanItemRecord(
        UUID.randomUUID(), UUID.randomUUID().toString(), bindingName, status,
        Instant.now(), TargetType.CAPABILITY, null, "test-tenant",
        bindingName + " description", executorName, null);
  }

  // --- Stubs ---

  static class StubRegistry implements DefaultCaseDefinitionRegistry {
    private final Map<String, CaseDefinition> defs = new java.util.HashMap<>();

    void register(CaseDefinition def) {
      defs.put(def.getName(), def);
    }

    @Override
    public Optional<CaseDefinition> findByName(String name) {
      return Optional.ofNullable(defs.get(name));
    }
  }

  static class StubPlanItemStore implements PlanItemStore {
    List<PlanItemRecord> items = List.of();

    @Override
    public List<PlanItemRecord> findByCaseId(UUID caseId, String tenancyId) {
      return items;
    }

    @Override public void save(io.casehub.engine.common.internal.model.PlanItemSaveRequest r, String t) {}
    @Override public void updateStatus(String id, TaskStatus s) {}
    @Override public List<PlanItemRecord> findDelegated(UUID id) { return List.of(); }
    @Override public List<PlanItemRecord> findAllDelegated() { return List.of(); }
  }

  static class RecordingCbrStore implements CbrCaseMemoryStore {
    final List<PlanCbrCase> storedCases = new ArrayList<>();
    String lastCaseType, lastEntityId, lastTenantId, lastCaseId;
    MemoryDomain lastDomain;
    RuntimeException throwOnStore;

    @Override
    public String store(CbrCase c, String ct, String eid, MemoryDomain d, String tid, String cid) {
      if (throwOnStore != null) throw throwOnStore;
      storedCases.add((PlanCbrCase) c);
      lastCaseType = ct;
      lastEntityId = eid;
      lastDomain = d;
      lastTenantId = tid;
      lastCaseId = cid;
      return "stored-id";
    }

    @Override public void registerSchema(CbrFeatureSchema s) {}
    @Override public <C extends CbrCase> List<ScoredCbrCase<C>> retrieveSimilar(CbrQuery q, Class<C> t) { return List.of(); }
    @Override public Integer erase(io.casehub.neocortex.memory.EraseRequest r) { return 0; }
    @Override public Integer eraseEntity(String eid, String tid) { return 0; }
  }
}
```

**Note:** The `StubRegistry` class above is a placeholder — the actual stub needs to match `CaseDefinitionRegistry`'s interface. During implementation, verify the interface and adjust. `CaseDefinitionRegistry` is an interface; the stub implements it directly.

- [ ] **Step 2: Run tests to verify they fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CbrCaseRetainObserverTest -q
```

Expected: compilation error — `CbrCaseRetainObserver` does not exist.

- [ ] **Step 3: Implement CbrCaseRetainObserver**

Create `runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java`.

The observer:
1. Implements `CaseOutcomeObserver`
2. Injects `CbrCaseMemoryStore` (direct — NoOp default absorbs), `CaseDefinitionRegistry`, `PlanItemStore`, `JQEvaluator`
3. `onOutcome()` flow follows spec steps 1-9 exactly
4. Maps `TaskStatus` → outcome string: COMPLETED→"SUCCESS", FAULTED→"FAILURE", REJECTED→"DECLINED", CANCELLED→"CANCELLED", OBSOLETE→"OBSOLETE"
5. Builds capability name lookup map from bindings filtered to `CapabilityTarget`
6. Filters PlanItemRecords to terminal + has capability + non-null executorName
7. Returns early on: definition not found, no CbrConfig, domain null, empty features, empty filtered trace
8. All exceptions caught and logged

Full implementation code is in the spec at §Issue #703 — CbrCaseRetainObserver flow steps 1-9.

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CbrCaseRetainObserverTest -q
```

Expected: all tests PASS.

- [ ] **Step 5: Run full runtime test suite to verify no regressions**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -q
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/memory/CbrCaseRetainObserver.java runtime/src/test/java/io/casehub/engine/internal/memory/CbrCaseRetainObserverTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#703): add CbrCaseRetainObserver — stores PlanCbrCase on case close

Implements CaseOutcomeObserver. Extracts features via CbrConfig,
reconstructs plan trace from PlanItemStore, stores PlanCbrCase in
CbrCaseMemoryStore. Filters to terminal CapabilityTarget items with
assigned workers. Empty features/traces return early. Exceptions caught
and logged — never blocks case progression.

Closes #703"
```

---

### Task 4: Refactor CbrAgentRoutingStrategy to consume context.experiences()

**Files:**
- Modify: `src/main/java/io/casehub/blocks/routing/agent/CbrAgentRoutingStrategy.java` (in blocks repo)
- Modify: `src/test/java/io/casehub/blocks/routing/agent/CbrAgentRoutingStrategyTest.java` (in blocks repo)

**Interfaces:**
- Consumes: `AgentRoutingContext.experiences()` → `List<RetrievedExperience>`, `RetrievedExperience.planTrace()` → `List<ExperiencePlanStep>`, `ExperiencePlanStep.capabilityName()`, `ExperiencePlanStep.workerName()`, `ExperiencePlanStep.stepOutcome()`
- Produces: `CbrAgentRoutingStrategy` — unchanged external interface (`AgentRoutingStrategy.select()`)

- [ ] **Step 1: Update/write tests for experience-based selection**

Update tests in `CbrAgentRoutingStrategyTest` to provide experiences via `AgentRoutingContext` constructor instead of mocking a store. Test cases:

1. Empty experiences → falls through to graph/trust (unresolvable if both absent)
2. Experiences with plan traces matching capability → selects highest success rate worker
3. Experiences with no matching capability → unresolvable
4. Multiple workers — tie-break by observation count

- [ ] **Step 2: Run tests to verify they fail**

```bash
/opt/homebrew/bin/mvn test -pl . -Dtest=CbrAgentRoutingStrategyTest -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 3: Refactor CbrAgentRoutingStrategy**

Remove constructor parameters: `CbrCaseMemoryStore`, `RoutingFeatureExtractor`, `topK`, `minSimilarity`.

Keep: `AgentGraphQuery`, `TrustCandidateClassifier`, `TrustScoreSource`, `TrustRoutingPolicyProvider`.

Replace `tryCbrStore()` with analysis of `context.experiences()`:

```java
private @Nullable String analyseExperiences(
    List<RetrievedExperience> experiences, Set<String> eligibleIds, String capabilityName) {
  Map<String, double[]> workerStats = new HashMap<>();
  for (var exp : experiences) {
    for (var step : exp.planTrace()) {
      if (capabilityName.equals(step.capabilityName())
          && step.workerName() != null
          && eligibleIds.contains(step.workerName())) {
        var stats = workerStats.computeIfAbsent(step.workerName(), k -> new double[]{0.0, 0});
        stats[1]++;
        stats[0] += OUTCOME_WEIGHTS.getOrDefault(step.stepOutcome(), 0.0);
      }
    }
  }
  // ... same best-worker selection logic ...
}
```

- [ ] **Step 4: Run tests**

```bash
/opt/homebrew/bin/mvn test -pl . -Dtest=CbrAgentRoutingStrategyTest -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add src/main/java/io/casehub/blocks/routing/agent/CbrAgentRoutingStrategy.java src/test/java/io/casehub/blocks/routing/agent/CbrAgentRoutingStrategyTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#505): CbrAgentRoutingStrategy consumes context.experiences()

Remove direct CbrCaseMemoryStore query. Analyse RetrievedExperience plan
traces from context instead. Same worker selection logic, adapted to
engine-owned types. Removes RoutingFeatureExtractor, CbrCaseMemoryStore,
topK/minSimilarity injections.

Refs casehubio/engine#505"
```

---

### Task 5: Refactor CbrRoutingPromptSection to consume context.experiences()

**Files:**
- Modify: `src/main/java/io/casehub/blocks/routing/agent/CbrRoutingPromptSection.java` (in blocks repo)
- Modify: `src/test/java/io/casehub/blocks/routing/agent/CbrRoutingPromptSectionTest.java` (in blocks repo)

**Interfaces:**
- Consumes: `AgentRoutingContext.experiences()`, `RetrievedExperience`, `ExperiencePlanStep`
- Produces: `CbrRoutingPromptSection` — unchanged external interface (`RoutingPromptSection.render()`)

- [ ] **Step 1: Update/write tests for experience-based rendering**

1. Empty experiences → null
2. Experiences with matching plan traces → formatted output with agent outcomes and case details
3. Experiences with no matching capability → null

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Refactor CbrRoutingPromptSection**

Remove: `CbrCaseMemoryStore`, `RoutingFeatureExtractor`, `topK`, `minSimilarity`.

`render()` reads `context.experiences()` directly. Same formatting logic, adapted from `ScoredCbrCase<CbrCase>` to `RetrievedExperience` and from `PlanTrace` to `ExperiencePlanStep`. `scored.score()` maps to `exp.similarityScore()`.

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add src/main/java/io/casehub/blocks/routing/agent/CbrRoutingPromptSection.java src/test/java/io/casehub/blocks/routing/agent/CbrRoutingPromptSectionTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#505): CbrRoutingPromptSection consumes context.experiences()

Remove direct CbrCaseMemoryStore query. Format from RetrievedExperience
list instead. Same output structure, adapted to engine-owned types.

Refs casehubio/engine#505"
```

---

### Task 6: Remove CbrRoutingOutcomeRecorder, RoutingFeatureExtractor, TextOnlyFeatureExtractor

**Files:**
- Delete: `src/main/java/io/casehub/blocks/routing/agent/CbrRoutingOutcomeRecorder.java` (use `ide_refactor_safe_delete`)
- Delete: `src/test/java/io/casehub/blocks/routing/agent/CbrRoutingOutcomeRecorderTest.java`
- Delete: `src/main/java/io/casehub/blocks/routing/agent/RoutingFeatureExtractor.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/blocks/routing/agent/TextOnlyFeatureExtractor.java` (use `ide_refactor_safe_delete`)
- Delete: `src/test/java/io/casehub/blocks/routing/agent/TextOnlyFeatureExtractorTest.java`

**Interfaces:**
- Consumes: Tasks 4 and 5 must be complete (all consumers removed)
- Produces: nothing — pure deletion

- [ ] **Step 1: Verify zero remaining consumers**

Use `ide_find_references` on `CbrRoutingOutcomeRecorder`, `RoutingFeatureExtractor`, and `TextOnlyFeatureExtractor` to confirm no production references remain after Tasks 4-5.

- [ ] **Step 2: Safe-delete CbrRoutingOutcomeRecorder**

Use `ide_refactor_safe_delete` on `CbrRoutingOutcomeRecorder`. If usages remain, investigate before forcing.

- [ ] **Step 3: Safe-delete RoutingFeatureExtractor and TextOnlyFeatureExtractor**

Use `ide_refactor_safe_delete` on both. Delete test files for both.

- [ ] **Step 4: Verify RoutingOutcomeRecorder SPI still exists**

Use `ide_find_class` for `RoutingOutcomeRecorder` in engine-api — confirm the SPI interface itself is untouched (other implementations may use it).

- [ ] **Step 5: Build blocks to verify no compilation errors**

```bash
/opt/homebrew/bin/mvn compile -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 6: Run blocks test suite**

```bash
/opt/homebrew/bin/mvn test -f /Users/mdproctor/claude/casehub/blocks/pom.xml -q
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add -A
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#505): remove orphaned CBR recording and feature extraction

CbrRoutingOutcomeRecorder entries used incompatible domain/caseType —
not retrievable by CbrRetrievalService after strategies switched to
context.experiences(). RoutingFeatureExtractor and TextOnlyFeatureExtractor
have zero remaining consumers.

Closes casehubio/engine#505"
```

---

### Task 7: Close #705 and update CLAUDE.md

**Files:**
- Modify: engine `CLAUDE.md` — update CBR Retrieval Bridge section to document the retain observer

**Interfaces:**
- Consumes: All prior tasks complete
- Produces: Updated documentation, closed issue

- [ ] **Step 1: Close #705 as superseded**

```bash
gh issue close 705 --repo casehubio/engine --reason "not planned" --comment "Superseded — after #505 removed CbrRoutingOutcomeRecorder and strategy store injections, zero consumers of RoutingFeatureExtractor remain. Both the interface and TextOnlyFeatureExtractor were deleted as dead code in #505."
```

- [ ] **Step 2: Update CLAUDE.md CBR section**

Add `CbrCaseRetainObserver` to the CBR Retrieval Bridge section in CLAUDE.md. Document:
- Location: `runtime/internal/memory/`
- CDI: `@ApplicationScoped`, implements `CaseOutcomeObserver`
- Stores `PlanCbrCase` on case terminal state
- Feature extraction from `CbrConfig.featureExtractor()` against `caseFileSnapshot`
- Plan trace from `PlanItemStore`, filtered to terminal CapabilityTarget items

Also update the CaseOutcomeObserver SPI section to note `tenancyId` on `CaseOutcomeEvent`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/engine commit -m "docs: update CLAUDE.md with CBR retain observer and CaseOutcomeEvent tenancyId

Refs #706"
```
