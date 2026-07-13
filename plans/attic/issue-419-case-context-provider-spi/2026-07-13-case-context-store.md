# CaseContextStore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #419 — CaseContextProvider SPI — pluggable context backing store
**Issue group:** #419

**Goal:** Make CaseContext storage pluggable via a CaseContextStore SPI below CaseContextImpl, eliminate all `instanceof CaseContextImpl` checks via a MutableCaseContext interface, and demonstrate the full stack with an example module.

**Architecture:** CaseContextStore is a flat key-value storage SPI in `api/context/`. CaseContextStoreFactory (NamedStrategy) creates stores per layer per case. CaseContextImpl delegates to stores via WritableLayerImpl. MutableCaseContext extends CaseContext with engine-internal operations. EngineStrategyResolver discovers factory beans via CDI.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (ARC), Jackson, JUnit 5

## Global Constraints

- `CaseContextStore` and `CaseContextStoreFactory` live in `api/src/main/java/io/casehub/api/context/`
- `MutableCaseContext` lives in `api/src/main/java/io/casehub/api/context/`
- `InMemoryCaseContextStore` and `InMemoryCaseContextStoreFactory` live in `runtime/src/main/java/io/casehub/engine/internal/context/`
- Reuse existing `Subscription` and `ContextChangeEvent` — do NOT create new types
- `WritableLayer` interface is NOT modified — `engineSet()`/`engineUpdate()` stay on `WritableLayerImpl`
- `EpisodicLayerUpdater` casts `WritableLayer` → `WritableLayerImpl` for engine-bypass methods (localized, safe)
- `InMemoryCaseContextStore` wraps `LinkedHashMap` (not `ConcurrentHashMap`) — matching current `WritableLayerImpl` behavior
- All existing tests must pass after each task
- `mvn install -DskipTests -q` before module-specific tests; always include `TESTCONTAINERS_RYUK_DISABLED=true`

---

### Task 1: Core SPI — CaseContextStore, CaseContextStoreFactory, MutableCaseContext + InMemoryCaseContextStore

**Files:**
- Create: `api/src/main/java/io/casehub/api/context/CaseContextStore.java`
- Create: `api/src/main/java/io/casehub/api/context/CaseContextStoreFactory.java`
- Create: `api/src/main/java/io/casehub/api/context/MutableCaseContext.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/context/InMemoryCaseContextStore.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/context/InMemoryCaseContextStoreFactory.java`
- Test: `api/src/test/java/io/casehub/api/context/CaseContextStoreContractTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/context/InMemoryCaseContextStoreTest.java`

**Interfaces:**
- Consumes: `Subscription` (from `io.casehub.api.context`), `ContextChangeEvent` (from `io.casehub.api.context`), `NamedStrategy` (from `io.casehub.platform.api.routing`)
- Produces: `CaseContextStore`, `CaseContextStoreFactory`, `MutableCaseContext`, `InMemoryCaseContextStore`, `InMemoryCaseContextStoreFactory`

- [ ] **Step 1: Write CaseContextStore contract test**

```java
package io.casehub.api.context;

import static org.junit.jupiter.api.Assertions.*;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

public abstract class CaseContextStoreContractTest {

    protected abstract CaseContextStore createStore();

    private CaseContextStore store;

    @BeforeEach
    void setUp() { store = createStore(); }

    @Test void putAndGet() {
        assertNull(store.put("k", "v1"));
        assertEquals("v1", store.get("k"));
    }

    @Test void putReturnsPrevious() {
        store.put("k", "v1");
        assertEquals("v1", store.put("k", "v2"));
        assertEquals("v2", store.get("k"));
    }

    @Test void getMissing() { assertNull(store.get("absent")); }

    @Test void remove() {
        store.put("k", "v");
        assertEquals("v", store.remove("k"));
        assertNull(store.get("k"));
        assertFalse(store.containsKey("k"));
    }

    @Test void removeAbsent() { assertNull(store.remove("absent")); }

    @Test void containsKey() {
        assertFalse(store.containsKey("k"));
        store.put("k", "v");
        assertTrue(store.containsKey("k"));
    }

    @Test void keySet() {
        store.put("a", 1);
        store.put("b", 2);
        assertEquals(Set.of("a", "b"), store.keySet());
    }

    @Test void snapshot() {
        store.put("a", 1);
        store.put("b", 2);
        Map<String, Object> snap = store.snapshot();
        assertEquals(Map.of("a", 1, "b", 2), snap);
        assertThrows(UnsupportedOperationException.class, () -> snap.put("c", 3));
    }

    @Test void clear() {
        store.put("a", 1);
        store.put("b", 2);
        store.clear();
        assertTrue(store.isEmpty());
        assertEquals(0, store.size());
    }

    @Test void putAll() {
        store.putAll(Map.of("a", 1, "b", 2));
        assertEquals(1, store.get("a"));
        assertEquals(2, store.get("b"));
    }

    @Test void sizeAndIsEmpty() {
        assertTrue(store.isEmpty());
        assertEquals(0, store.size());
        store.put("k", "v");
        assertFalse(store.isEmpty());
        assertEquals(1, store.size());
    }

    @Test void closeIsIdempotent() throws Exception {
        store.close();
        store.close();
    }

    @Test void defaultsNoExternalChangeNotification() {
        assertFalse(store.supportsExternalChangeNotification());
    }

    @Test void defaultOnExternalChangeReturnsNoop() {
        Subscription sub = store.onExternalChange(e -> {});
        assertNotNull(sub);
        sub.cancel(); // no-op, should not throw
    }

    @Test void putNullValue() {
        store.put("k", null);
        assertNull(store.get("k"));
        assertTrue(store.containsKey("k"));
    }
}
```

- [ ] **Step 2: Run test — verify it fails (CaseContextStore does not exist)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseContextStoreContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: compilation failure — `CaseContextStore` not found.

- [ ] **Step 3: Create CaseContextStore interface**

Create `api/src/main/java/io/casehub/api/context/CaseContextStore.java`:

```java
package io.casehub.api.context;

import java.util.Map;
import java.util.Set;
import java.util.function.Consumer;

public interface CaseContextStore extends AutoCloseable {

    Object get(String key);

    Object put(String key, Object value);

    Object remove(String key);

    boolean containsKey(String key);

    Set<String> keySet();

    Map<String, Object> snapshot();

    void clear();

    default void putAll(Map<String, Object> entries) {
        entries.forEach(this::put);
    }

    int size();

    boolean isEmpty();

    @Override
    default void close() {}

    default boolean supportsExternalChangeNotification() {
        return false;
    }

    default Subscription onExternalChange(Consumer<ContextChangeEvent> listener) {
        return Subscription.NOOP;
    }
}
```

- [ ] **Step 4: Create MutableCaseContext interface**

Create `api/src/main/java/io/casehub/api/context/MutableCaseContext.java`:

```java
package io.casehub.api.context;

public interface MutableCaseContext extends CaseContext {

    WritableLayer writableLayer(String name);

    void freezeLayer(String name);
}
```

- [ ] **Step 5: Create CaseContextStoreFactory interface**

Create `api/src/main/java/io/casehub/api/context/CaseContextStoreFactory.java`:

```java
package io.casehub.api.context;

import io.casehub.platform.api.routing.NamedStrategy;
import java.util.UUID;

public interface CaseContextStoreFactory extends NamedStrategy {

    CaseContextStore createStore(String layerName, UUID caseId);

    default CaseContextStore loadStore(String layerName, UUID caseId) {
        return createStore(layerName, caseId);
    }

    default boolean isDurable() { return false; }
}
```

- [ ] **Step 6: Create InMemoryCaseContextStore**

Create `runtime/src/main/java/io/casehub/engine/internal/context/InMemoryCaseContextStore.java`:

```java
package io.casehub.engine.internal.context;

import io.casehub.api.context.CaseContextStore;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;

public class InMemoryCaseContextStore implements CaseContextStore {

    private final Map<String, Object> data;

    public InMemoryCaseContextStore() {
        this.data = new LinkedHashMap<>();
    }

    public InMemoryCaseContextStore(Map<String, Object> initial) {
        this.data = new LinkedHashMap<>(initial);
    }

    @Override public Object get(String key) { return data.get(key); }

    @Override public Object put(String key, Object value) { return data.put(key, value); }

    @Override public Object remove(String key) { return data.remove(key); }

    @Override public boolean containsKey(String key) { return data.containsKey(key); }

    @Override public Set<String> keySet() { return Set.copyOf(data.keySet()); }

    @Override public Map<String, Object> snapshot() {
        return Collections.unmodifiableMap(new LinkedHashMap<>(data));
    }

    @Override public void clear() { data.clear(); }

    @Override public void putAll(Map<String, Object> entries) { data.putAll(entries); }

    @Override public int size() { return data.size(); }

    @Override public boolean isEmpty() { return data.isEmpty(); }
}
```

- [ ] **Step 7: Create InMemoryCaseContextStoreFactory**

Create `runtime/src/main/java/io/casehub/engine/internal/context/InMemoryCaseContextStoreFactory.java`:

```java
package io.casehub.engine.internal.context;

import io.casehub.api.context.CaseContextStore;
import io.casehub.api.context.CaseContextStoreFactory;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class InMemoryCaseContextStoreFactory implements CaseContextStoreFactory {

    public static final InMemoryCaseContextStoreFactory INSTANCE =
        new InMemoryCaseContextStoreFactory();

    @Override
    public String id() { return "in-memory"; }

    @Override
    public CaseContextStore createStore(String layerName, UUID caseId) {
        return new InMemoryCaseContextStore();
    }
}
```

- [ ] **Step 8: Write InMemoryCaseContextStore test extending contract**

Create `runtime/src/test/java/io/casehub/engine/internal/context/InMemoryCaseContextStoreTest.java`:

```java
package io.casehub.engine.internal.context;

import io.casehub.api.context.CaseContextStore;
import io.casehub.api.context.CaseContextStoreContractTest;

class InMemoryCaseContextStoreTest extends CaseContextStoreContractTest {
    @Override
    protected CaseContextStore createStore() {
        return new InMemoryCaseContextStore();
    }
}
```

- [ ] **Step 9: Run tests**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseContextStoreContractTest -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=InMemoryCaseContextStoreTest -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass.

- [ ] **Step 10: Commit**

```
feat(#419): CaseContextStore SPI + MutableCaseContext + InMemoryCaseContextStore

Introduces the core SPI interfaces (CaseContextStore, CaseContextStoreFactory,
MutableCaseContext) and the default in-memory implementation with contract tests.

Refs #419
```

---

### Task 2: WritableLayerImpl — delegate to CaseContextStore

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/WritableLayerImpl.java`
- Test: existing `runtime/src/test/java/io/casehub/engine/internal/context/` tests must pass unchanged

**Interfaces:**
- Consumes: `CaseContextStore` (from Task 1), `InMemoryCaseContextStore` (from Task 1)
- Produces: `WritableLayerImpl(String layerName, CaseContextStore store)` constructor, `CaseContextStore getStore()` accessor

- [ ] **Step 1: Run existing WritableLayerImpl tests to establish baseline**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="io.casehub.engine.internal.context.*" -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass.

- [ ] **Step 2: Add new constructor to WritableLayerImpl**

Add a new constructor `WritableLayerImpl(String layerName, CaseContextStore store)` that takes a store. The existing `data` field (`Map<String, Object>`) is replaced by a `CaseContextStore` field. The existing constructors delegate to the new one by wrapping their initial data in `InMemoryCaseContextStore`.

Use `ide_insert_member` to add the new constructor, then `ide_edit_member` to refactor existing constructors and the `data` field.

Key changes in WritableLayerImpl:
- Field: `private final Map<String, Object> data` → `private final CaseContextStore store`
- Constructor `(String layerName)` → delegates to `this(layerName, new InMemoryCaseContextStore())`
- Constructor `(String layerName, Map<String, Object> initial)` → delegates to `this(layerName, new InMemoryCaseContextStore(initial))`
- New constructor `(String layerName, CaseContextStore store)`
- Private constructor `(String layerName, Map<String, Object> initial, boolean deepCopy)` — used by `deepCopy()`, creates `InMemoryCaseContextStore` from the deep-copied map
- New accessor: `CaseContextStore getStore() { return store; }` (package-private)
- All methods: `data.get(key)` → `store.get(key)`, `data.put(key, value)` → `store.put(key, value)`, etc.
- `modify()`: changes parameter from `BiFunction<Map<String, Object>, Runnable, R>` to `BiFunction<CaseContextStore, Runnable, R>`
- `getData()`: returns `store.snapshot()` (was `Collections.unmodifiableMap(new LinkedHashMap<>(data))`)
- `getKeys()`: returns `store.keySet()` (was `Set.copyOf(data.keySet())`)
- `asJsonNode()`: uses `store.snapshot()` (was `data` directly)
- `snapshot()`: `deepCopy()` creates a new `WritableLayerImpl` with `new InMemoryCaseContextStore(deepCopyMap(store.snapshot()))`
- Path-based operations (`setPath`, `getPathInternal`, `applyAndDiff`): first-level access via `store.get(parts[0])`, write-back via `store.put(parts[0], rootValue)` after nested mutation
- `applyDiff()`: uses `store.clear()` + `store.putAll(updated)` (was `data.clear()` + `data.putAll(updated)`)

- [ ] **Step 3: Run existing tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="io.casehub.engine.internal.context.*" -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass — behavior is identical, only the storage delegation changed.

- [ ] **Step 4: Commit**

```
refactor(#419): WritableLayerImpl delegates to CaseContextStore

Replaces the internal LinkedHashMap with a CaseContextStore. Existing
constructors wrap initial data in InMemoryCaseContextStore. All behavior
preserved — existing tests pass unchanged.

Refs #419
```

---

### Task 3: CaseContextImpl — implement MutableCaseContext, take factory in constructor

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java`
- Test: existing `runtime/src/test/java/io/casehub/engine/internal/context/CaseContextImpl*Test.java` must pass unchanged
- Test: `runtime/src/test/java/io/casehub/engine/internal/context/MutableCaseContextTest.java` (new)

**Interfaces:**
- Consumes: `MutableCaseContext` (from Task 1), `CaseContextStoreFactory` (from Task 1), `InMemoryCaseContextStoreFactory` (from Task 1), `WritableLayerImpl(String, CaseContextStore)` (from Task 2)
- Produces: `CaseContextImpl implements MutableCaseContext`, `CaseContextImpl(CaseContextStoreFactory, UUID)` constructor

- [ ] **Step 1: Write MutableCaseContext test**

```java
package io.casehub.engine.internal.context;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.context.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class MutableCaseContextTest {

    @Test void writableLayerReturnsWritableLayer() {
        MutableCaseContext ctx = new CaseContextImpl();
        WritableLayer layer = ctx.writableLayer(ContextLayer.WORKING);
        assertNotNull(layer);
        layer.set("k", "v");
        assertEquals("v", ctx.get("k"));
    }

    @Test void freezeLayerPreventsWrites() {
        MutableCaseContext ctx = new CaseContextImpl();
        ctx.writableLayer(ContextLayer.SEMANTIC).set("k", "v");
        ctx.freezeLayer(ContextLayer.SEMANTIC);
        assertThrows(IllegalStateException.class,
            () -> ctx.writableLayer(ContextLayer.SEMANTIC).set("k2", "v2"));
    }

    @Test void customFactory() {
        var factory = InMemoryCaseContextStoreFactory.INSTANCE;
        var ctx = new CaseContextImpl(factory, java.util.UUID.randomUUID());
        ctx.set("k", "v");
        assertEquals("v", ctx.get("k"));
    }

    @Test void onDemandLayerUsesFactory() {
        var factory = InMemoryCaseContextStoreFactory.INSTANCE;
        MutableCaseContext ctx = new CaseContextImpl(factory, null);
        WritableLayer custom = ctx.writableLayer("custom-layer");
        assertNotNull(custom);
        custom.set("k", "v");
        assertEquals("v", ctx.layer("custom-layer").get("k"));
    }

    @Test void backwardCompatibleConstructors() {
        var ctx1 = new CaseContextImpl();
        assertTrue(ctx1.isEmpty());

        var ctx2 = new CaseContextImpl(Map.of("k", "v"));
        assertEquals("v", ctx2.get("k"));
    }

    @Test void hybridObservationWiring() {
        // Verify CaseContextImpl subscribes to external changes from the working store
        // InMemoryCaseContextStore does not support external changes, so this is a no-op test
        var ctx = new CaseContextImpl();
        // No exception during construction = wiring code handles non-observable stores gracefully
        assertNotNull(ctx);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=MutableCaseContextTest -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: fails — CaseContextImpl does not yet implement MutableCaseContext.

- [ ] **Step 3: Modify CaseContextImpl**

Use `ide_edit_member` to change the class declaration:
```java
// From:
@JsonDeserialize(as = CaseContextImpl.class)
public class CaseContextImpl implements CaseContext {
// To:
@JsonDeserialize(as = CaseContextImpl.class)
public class CaseContextImpl implements MutableCaseContext {
```

Add fields and new constructors. Preserve existing constructors for backward compatibility (`MapCaseFile` extends CaseContextImpl and calls `super()` / `super(initial)`).

Key changes:
- Add fields: `private final CaseContextStoreFactory storeFactory;` and `private final UUID caseId;`
- Add constructor: `public CaseContextImpl(CaseContextStoreFactory storeFactory, UUID caseId)`
- Existing no-arg: `this(InMemoryCaseContextStoreFactory.INSTANCE, null)`
- Existing `(Map<String, Object> initial)`: creates with INSTANCE factory, populates working layer
- Existing `(Map<String, Object> initial, long ignoredVersion)`: delegates to `(initial)` (version is unused)
- Existing `(JsonNode asNode)`: creates with INSTANCE factory, populates from JSON
- `initBuiltinLayers()`: uses `storeFactory.createStore(layerName, caseId)` for each layer
- `layer()` and `writableLayer()`: use `storeFactory.createStore(n, caseId)` for on-demand layers
- `writableLayer()` now satisfies `MutableCaseContext` interface
- `freezeLayer()` now satisfies `MutableCaseContext` interface
- Hybrid observation: after creating working layer, check `getStore().supportsExternalChangeNotification()` and subscribe if true
- `modify()` calls in CaseContextImpl: change lambda parameter from `Map<String, Object>` → `CaseContextStore` (the `working().modify()` signature changed in Task 2)

- [ ] **Step 4: Run all context tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="io.casehub.engine.internal.context.*" -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass including new MutableCaseContextTest.

- [ ] **Step 5: Commit**

```
refactor(#419): CaseContextImpl implements MutableCaseContext with factory

CaseContextImpl now takes CaseContextStoreFactory in constructor and
delegates layer creation to the factory. Existing constructors preserved
for backward compatibility. MutableCaseContext.writableLayer() and
freezeLayer() satisfied by existing methods.

Refs #419
```

---

### Task 4: Eliminate all instanceof CaseContextImpl + wire factory through engine

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/EpisodicLayerUpdater.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add `contextStoreFactory` field)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (parse `context.storeFactory`)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java` (add `Instance<CaseContextStoreFactory>`)
- Test: existing integration tests must pass

**Interfaces:**
- Consumes: `MutableCaseContext` (Task 1), `CaseContextStoreFactory` (Task 1), `CaseContextImpl(CaseContextStoreFactory, UUID)` (Task 3)
- Produces: instanceof-free engine code, factory wiring through CaseDefinition → StrategyResolver → CaseHubRuntimeImpl

- [ ] **Step 1: Run full test suite to establish baseline**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass.

- [ ] **Step 2: Add contextStoreFactory to CaseDefinition**

Use `ide_insert_member` to add after the `defaultWorkerBridge` field (line 59):
```java
private String contextStoreFactory;
```

Add getter/setter:
```java
public String getContextStoreFactory() { return contextStoreFactory; }
public void setContextStoreFactory(String contextStoreFactory) { this.contextStoreFactory = contextStoreFactory; }
```

Add builder method in `CaseDefinition.Builder`:
```java
public Builder contextStoreFactory(String contextStoreFactory) {
    this.contextStoreFactory = contextStoreFactory;
    return this;
}
```

Wire in `Builder.build()` — add `def.setContextStoreFactory(contextStoreFactory)` alongside the other setters.

- [ ] **Step 3: Add CaseContextStoreFactory to EngineStrategyResolver**

Use `ide_edit_member` to update the `@Inject` constructor to add a new parameter:
```java
@Any Instance<CaseContextStoreFactory> contextStoreFactories
```

Add `registerStrategies(contextStoreFactories)` in the constructor body.

- [ ] **Step 4: Update EpisodicLayerUpdater — CaseContextImpl → MutableCaseContext**

Change all 4 method signatures from `CaseContextImpl ctx` to `MutableCaseContext ctx`. Each method internally does:
```java
WritableLayerImpl episodic = (WritableLayerImpl) ctx.writableLayer(ContextLayer.EPISODIC);
```
The cast from `WritableLayer` to `WritableLayerImpl` is safe — it's engine-internal code and `WritableLayerImpl` is the only implementation. The cast is localized to this one class instead of being scattered across 8 sites.

- [ ] **Step 5: Update CaseHubReactor — eliminate all instanceof**

Change `startCase()` and `buildInstance()` signatures to accept `MutableCaseContext` instead of `CaseContext`. Remove all `if (context instanceof CaseContextImpl ctx)` checks — call `ctx.writableLayer()` and `ctx.freezeLayer()` directly on the `MutableCaseContext` parameter.

Change `queryEpisodicMemory()` parameter from `CaseContextImpl` to `MutableCaseContext`.

Move UUID generation earlier in `buildInstance()`:
```java
UUID caseId = UUID.randomUUID();
```
Use the pre-generated `caseId` when setting `instance.setUuid(caseId)`.

- [ ] **Step 6: Update CaseHubRuntimeImpl — use factory**

Inject `StrategyResolver` into `CaseHubRuntimeImpl`. Resolve the factory from `CaseDefinition.getContextStoreFactory()`:

```java
CaseContextStoreFactory factory = strategyResolver.resolve(
    CaseContextStoreFactory.class,
    definition.getContextStoreFactory());
UUID caseId = UUID.randomUUID();
MutableCaseContext context = new CaseContextImpl(factory, caseId);
if (inputData != null) {
    context.setAll(toContextMap(inputData));
}
return reactor.startCase(definition, context);
```

Note: `CaseHubReactor.buildInstance()` currently generates the UUID — after this change, the UUID is generated in `CaseHubRuntimeImpl` and passed via the context. The reactor uses `context` directly and reads the UUID from it. This requires the `CaseContextImpl` to expose a `getCaseId()` method, or the UUID is threaded separately. Simplest: thread the UUID as a separate parameter alongside the context.

- [ ] **Step 7: Update WorkflowExecutionCompletedHandler — eliminate instanceof**

Two sites (lines 132 and 396): replace `instanceof CaseContextImpl ctx` with:
```java
if (caseInstance.getCaseContext() instanceof MutableCaseContext mctx) {
    EpisodicLayerUpdater.recordWorkerCompletion(mctx, worker.name(), "COMPLETED");
}
```

Since `CaseContextImpl` now implements `MutableCaseContext`, the instanceof check succeeds for all production contexts. `SnapshotCaseContext` does NOT implement `MutableCaseContext` — the check correctly skips it.

- [ ] **Step 8: Update DefaultWorkerExecutionRecoveryService — use factory + MutableCaseContext**

`rebuildStateContext()`: replace `new CaseContextImpl()` with factory-created context. The recovery service needs access to the `CaseDefinition` to resolve the factory. If `isDurable()`, use `loadStore()`; otherwise create empty and replay EventLog.

`applyTopLevelChanges()`: replace `instanceof CaseContextImpl c` with `instanceof MutableCaseContext mctx`.

- [ ] **Step 9: Update CaseDefinitionYamlMapper — parse context.storeFactory**

Add parsing for the `context:` YAML block. Read `storeFactory` from `context` node and call `builder.contextStoreFactory(value)`.

- [ ] **Step 10: Run full test suite**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass.

- [ ] **Step 11: Verify no instanceof CaseContextImpl remains in production code**

Use `ide_search_text` to search for `instanceof CaseContextImpl` across all production source files. The only remaining instances should be in `CaseContextImpl.equals()` (which is correct — it compares against its own type).

- [ ] **Step 12: Commit**

```
refactor(#419): eliminate instanceof CaseContextImpl — wire factory through engine

All 8 production sites now use MutableCaseContext. CaseDefinition gains
contextStoreFactory field. EngineStrategyResolver discovers factory beans.
CaseHubRuntimeImpl resolves factory and creates contexts with it.

Refs #419
```

---

### Task 5: Store lifecycle — close stores on case completion/eviction

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java` (add close method)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` (close on terminal status)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/cache/CaseInstanceCacheImpl.java` (close on eviction, if applicable)
- Test: `runtime/src/test/java/io/casehub/engine/internal/context/CaseContextStoreLifecycleTest.java` (new)

**Interfaces:**
- Consumes: `CaseContextStore.close()` (from Task 1)
- Produces: Store lifecycle management — stores are closed when cases complete or are evicted

- [ ] **Step 1: Write lifecycle test**

```java
package io.casehub.engine.internal.context;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.context.CaseContextStore;
import java.util.concurrent.atomic.AtomicBoolean;
import org.junit.jupiter.api.Test;

class CaseContextStoreLifecycleTest {

    @Test void closeCallsStoreClose() throws Exception {
        AtomicBoolean closed = new AtomicBoolean(false);
        CaseContextStore trackingStore = new InMemoryCaseContextStore() {
            @Override public void close() { closed.set(true); }
        };
        var layer = new WritableLayerImpl("test", trackingStore);
        var ctx = new CaseContextImpl();
        // Use the tracking store via a custom factory
        // ... verify close() is called on context disposal
    }
}
```

The exact test shape depends on how close() is wired — this will be refined during implementation.

- [ ] **Step 2: Add close support to CaseContextImpl**

Add a `close()` method (or implement `AutoCloseable`) that iterates all layers and calls `store.close()` on each.

- [ ] **Step 3: Wire close into CaseStatusChangedHandler**

When a case reaches terminal status (COMPLETED, FAULTED, CANCELLED), close the context's stores.

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="*Lifecycle*,*StatusChanged*" -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 5: Commit**

```
feat(#419): store lifecycle — close stores on case completion

CaseContextImpl.close() releases all layer stores. Called by
CaseStatusChangedHandler on terminal case status.

Refs #419
```

---

### Task 6: Example module — custom store + FuncDSL + sub-cases + tests

**Files:**
- Create: `examples/typed-context/pom.xml`
- Create: `examples/typed-context/src/main/java/io/casehub/examples/typedcontext/AuditingCaseContextStore.java`
- Create: `examples/typed-context/src/main/java/io/casehub/examples/typedcontext/AuditingCaseContextStoreFactory.java`
- Create: `examples/typed-context/src/main/java/io/casehub/examples/typedcontext/InvestigationContext.java`
- Create: `examples/typed-context/src/main/java/io/casehub/examples/typedcontext/InvestigationCase.java`
- Create: `examples/typed-context/src/test/java/io/casehub/examples/typedcontext/AuditingCaseContextStoreTest.java`
- Create: `examples/typed-context/src/test/java/io/casehub/examples/typedcontext/InvestigationCaseTest.java`
- Modify: root `pom.xml` — add examples/typed-context module

**Interfaces:**
- Consumes: `CaseContextStore` (Task 1), `CaseContextStoreFactory` (Task 1), `MutableCaseContext` (Task 1), all engine infrastructure from Tasks 2-5
- Produces: Working example demonstrating store pluggability with FuncDSL typed workers and sub-case propagation

- [ ] **Step 1: Create module POM**

Standard Quarkus test module with dependencies on `casehub-engine`, `casehub-blackboard`, `casehub-persistence-memory`, `quarkus-junit5`. Set `<maven.deploy.skip>true</maven.deploy.skip>`.

- [ ] **Step 2: Create AuditingCaseContextStore**

Wraps `InMemoryCaseContextStore`, records all writes to an audit log:

```java
package io.casehub.examples.typedcontext;

import io.casehub.api.context.CaseContextStore;
import io.casehub.api.context.ContextChangeEvent;
import io.casehub.engine.internal.context.InMemoryCaseContextStore;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

public class AuditingCaseContextStore implements CaseContextStore {
    private final InMemoryCaseContextStore delegate = new InMemoryCaseContextStore();
    private final List<ContextChangeEvent> auditLog = new CopyOnWriteArrayList<>();

    @Override public Object get(String key) { return delegate.get(key); }
    @Override public Object put(String key, Object value) {
        Object prev = delegate.put(key, value);
        auditLog.add(new ContextChangeEvent(key, prev, value));
        return prev;
    }
    @Override public Object remove(String key) {
        Object prev = delegate.remove(key);
        if (prev != null) auditLog.add(new ContextChangeEvent(key, prev, null));
        return prev;
    }
    @Override public boolean containsKey(String key) { return delegate.containsKey(key); }
    @Override public Set<String> keySet() { return delegate.keySet(); }
    @Override public Map<String, Object> snapshot() { return delegate.snapshot(); }
    @Override public void clear() { delegate.clear(); }
    @Override public void putAll(Map<String, Object> entries) { entries.forEach(this::put); }
    @Override public int size() { return delegate.size(); }
    @Override public boolean isEmpty() { return delegate.isEmpty(); }

    public List<ContextChangeEvent> getAuditLog() { return Collections.unmodifiableList(auditLog); }
}
```

- [ ] **Step 3: Create AuditingCaseContextStoreFactory**

```java
@ApplicationScoped
public class AuditingCaseContextStoreFactory implements CaseContextStoreFactory {
    @Override public String id() { return "auditing"; }
    @Override public CaseContextStore createStore(String layerName, UUID caseId) {
        return new AuditingCaseContextStore();
    }
}
```

- [ ] **Step 4: Write store contract test for AuditingCaseContextStore**

```java
class AuditingCaseContextStoreTest extends CaseContextStoreContractTest {
    @Override protected CaseContextStore createStore() { return new AuditingCaseContextStore(); }

    @Test void auditLogCapturesWrites() {
        var store = new AuditingCaseContextStore();
        store.put("k", "v1");
        store.put("k", "v2");
        assertEquals(2, store.getAuditLog().size());
        assertEquals("k", store.getAuditLog().get(0).key());
        assertNull(store.getAuditLog().get(0).oldValue());
        assertEquals("v1", store.getAuditLog().get(0).newValue());
        assertEquals("v1", store.getAuditLog().get(1).oldValue());
        assertEquals("v2", store.getAuditLog().get(1).newValue());
    }
}
```

- [ ] **Step 5: Create InvestigationContext record**

```java
package io.casehub.examples.typedcontext;

import java.util.List;

public record InvestigationContext(
    String suspectId,
    List<String> evidence,
    double riskScore,
    String verdict) {}
```

- [ ] **Step 6: Create InvestigationCase — CaseHub definition with FuncDSL + sub-cases**

A `CaseHub` subclass defining:
- Two capabilities: `assess-risk`, `render-verdict`
- Two typed workers using FuncDSL `.<InvestigationContext>fn().apply(...)`
- Goals: success when verdict is present
- Context store factory set to `"auditing"`

The exact implementation depends on existing CaseHub builder patterns and available FuncDSL support. If `<T>fn()` is not yet implemented (it's specified in the ContextBridge spec but may not be coded), use the existing `function(input -> ...)` pattern with Jackson deserialization as a placeholder — and create a GitHub issue to track the FuncDSL dependency.

- [ ] **Step 7: Write integration test**

`@QuarkusTest` that:
1. Starts the investigation case with initial context (`suspectId`, `evidence`)
2. Verifies the `assess-risk` worker runs and produces `riskScore`
3. Verifies the `render-verdict` worker runs and produces `verdict`
4. Verifies the case completes
5. Retrieves the `AuditingCaseContextStore` and verifies the audit log contains all writes
6. Verifies context propagation through the full engine stack

- [ ] **Step 8: Run tests**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl examples/typed-context -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 9: Run full project test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml
```
Expected: all pass across all modules.

- [ ] **Step 10: Commit**

```
feat(#419): example module — AuditingCaseContextStore + InvestigationCase

Demonstrates store pluggability with a custom auditing store, typed
workers (FuncDSL), and end-to-end case execution with context propagation
verification.

Refs #419
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ CaseContextStore interface — Task 1
- ✅ CaseContextStoreFactory — Task 1
- ✅ MutableCaseContext — Task 1
- ✅ InMemoryCaseContextStore — Task 1
- ✅ InMemoryCaseContextStoreFactory — Task 1
- ✅ WritableLayerImpl refactor — Task 2
- ✅ CaseContextImpl refactor — Task 3
- ✅ instanceof elimination (8 sites) — Task 4
- ✅ CaseDefinition integration — Task 4
- ✅ EngineStrategyResolver — Task 4
- ✅ CaseHubRuntimeImpl factory wiring — Task 4
- ✅ YAML parsing — Task 4
- ✅ EpisodicLayerUpdater migration — Task 4
- ✅ Store lifecycle — Task 5
- ✅ Example module — Task 6
- ✅ Hybrid observation wiring — Task 3
- ✅ modify() migration — Task 2 (signature change) + Task 3 (call-site update)
- ✅ Recovery model (isDurable) — referenced in Task 4 Step 8
- ⚠️ FuncDSL typed workers — depends on ContextBridge #203 implementation status; Task 6 notes the dependency

**Placeholder scan:** No TBDs. Task 5 lifecycle test is less detailed — will be refined during implementation.

**Type consistency:** `CaseContextStore`, `CaseContextStoreFactory`, `MutableCaseContext` used consistently across all tasks. `WritableLayerImpl` cast in `EpisodicLayerUpdater` is explicit and localized.

**Tooling safety:** No bash file operations on source files. All code changes via `ide_edit_member`, `ide_insert_member`, `ide_replace_member`.
