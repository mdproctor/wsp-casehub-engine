# Wire CaseContextStoreFactory Through Recovery Path — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #732 — wire CaseContextStoreFactory through recovery path
**Issue group:** #1182, #732, #1183, #1184, #1185

**Goal:** Enable durable CaseContextStoreFactory implementations by wiring factory resolution through the recovery path and removing the durable guard.

**Architecture:** The recovery service (`DefaultWorkerExecutionRecoveryService`) gains factory resolution. For durable factories, it creates a `CaseContextImpl` via a new `loadFromStore()` static method that calls `factory.loadStore()` per layer. For volatile factories, the existing `CaseContextRecoveryStrategy` handles recovery. The durable guard in `CaseHubRuntimeImpl.resolveFactory()` is removed.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JUnit 5, AssertJ

## Global Constraints

- `CaseContextImpl.storeFactory` is `final` — recovery must create the context with the correct factory from the start
- `InMemoryCaseContextStoreFactory` is the default for volatile cases — no behavioral change for existing cases
- ERROR-level logging for factory resolution failures (corruption-shaped failure for durable cases)
- Snapshots continue to be written for all cases (belt-and-suspenders)
- No Flyway migrations — schema is managed by Hibernate `drop-and-create`

---

## Batch 1: CaseContextImpl.loadFromStore

### Task 1: Add `loadFromStore` static factory method to CaseContextImpl

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java:67-85`
- Test: `runtime/src/test/java/io/casehub/engine/internal/context/CaseContextImplLoadFromStoreTest.java`

**Interfaces:**
- Consumes: `CaseContextStoreFactory.loadStore(String layerName, UUID caseId)` (existing API)
- Produces: `CaseContextImpl.loadFromStore(CaseContextStoreFactory factory, UUID caseId)` — returns a `CaseContextImpl` with layers backed by `factory.loadStore()`. Used by Task 3.

- [ ] **Step 1: Write failing test — loadFromStore creates layers with loaded stores**

Create new test class:

```java
package io.casehub.engine.internal.context;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.context.CaseContext;
import io.casehub.api.context.CaseContextStore;
import io.casehub.api.context.CaseContextStoreFactory;
import io.casehub.api.context.ContextLayer;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.Test;

class CaseContextImplLoadFromStoreTest {

  @Test
  void loadFromStoreCallsLoadStoreNotCreateStore() {
    AtomicInteger loadCalls = new AtomicInteger();
    AtomicInteger createCalls = new AtomicInteger();

    CaseContextStoreFactory factory = new CaseContextStoreFactory() {
      @Override
      public String id() { return "tracking"; }

      @Override
      public CaseContextStore createStore(String layerName, UUID caseId) {
        createCalls.incrementAndGet();
        return new InMemoryCaseContextStore();
      }

      @Override
      public CaseContextStore loadStore(String layerName, UUID caseId) {
        loadCalls.incrementAndGet();
        InMemoryCaseContextStore store = new InMemoryCaseContextStore();
        store.put("loaded", true);
        return store;
      }

      @Override
      public boolean isDurable() { return true; }
    };

    UUID caseId = UUID.randomUUID();
    CaseContextImpl ctx = CaseContextImpl.loadFromStore(factory, caseId);

    assertEquals(3, loadCalls.get(), "loadStore called for working + semantic + episodic");
    assertEquals(0, createCalls.get(), "createStore must not be called");
    assertEquals(true, ctx.get("loaded"));
  }

  @Test
  void loadFromStoreUsesFactoryForNewLayers() {
    CaseContextStoreFactory factory = new CaseContextStoreFactory() {
      @Override
      public String id() { return "test"; }

      @Override
      public CaseContextStore createStore(String layerName, UUID caseId) {
        InMemoryCaseContextStore store = new InMemoryCaseContextStore();
        store.put("created-by", "factory");
        return store;
      }

      @Override
      public boolean isDurable() { return true; }
    };

    UUID caseId = UUID.randomUUID();
    CaseContextImpl ctx = CaseContextImpl.loadFromStore(factory, caseId);

    // Access a new layer not in the built-in set — should use factory.createStore()
    var customLayer = ctx.writableLayer("custom");
    assertEquals("factory", customLayer.get("created-by"));
  }
}
```

Use `ide_create_file` to create the test file.

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextImplLoadFromStoreTest" -q`
Expected: FAIL — `loadFromStore` method does not exist.

- [ ] **Step 3: Implement loadFromStore**

Refactor the existing factory constructor to share init logic with the new static method. Use `ide_replace_text_in_file` to replace the constructor block:

Replace the two constructors at lines 67-85:
```java
  public CaseContextImpl() {
    this(InMemoryCaseContextStoreFactory.INSTANCE, null);
  }

  // ... (Map constructor stays unchanged)

  public CaseContextImpl(CaseContextStoreFactory storeFactory, UUID caseId) {
    this.storeFactory = storeFactory;
    this.caseId = caseId;
    initBuiltinLayers();
  }
```

With:
```java
  public CaseContextImpl() {
    this(InMemoryCaseContextStoreFactory.INSTANCE, null, false);
  }

  // ... (Map constructor stays unchanged)

  public CaseContextImpl(CaseContextStoreFactory storeFactory, UUID caseId) {
    this(storeFactory, caseId, false);
  }

  private CaseContextImpl(CaseContextStoreFactory storeFactory, UUID caseId, boolean load) {
    this.storeFactory = storeFactory;
    this.caseId = caseId;
    for (String layer : java.util.List.of(ContextLayer.WORKING, ContextLayer.SEMANTIC, ContextLayer.EPISODIC)) {
      CaseContextStore store = load ? storeFactory.loadStore(layer, caseId)
                                    : storeFactory.createStore(layer, caseId);
      layers.put(layer, new WritableLayerImpl(layer, store));
    }
  }

  public static CaseContextImpl loadFromStore(CaseContextStoreFactory factory, UUID caseId) {
    return new CaseContextImpl(factory, caseId, true);
  }
```

Also remove the `initBuiltinLayers()` method (lines 105-118) — its logic is now in the private constructor.

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextImplLoadFromStoreTest" -q`
Expected: PASS

- [ ] **Step 5: Run existing CaseContextImpl tests to verify no regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseContextImpl*" -q`
Expected: all pass — the refactored constructor delegates to the same logic.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java \
       runtime/src/test/java/io/casehub/engine/internal/context/CaseContextImplLoadFromStoreTest.java
git commit -m "feat(#732): add CaseContextImpl.loadFromStore for durable factory recovery

Refactors factory constructor to share init logic via private constructor
with load flag. loadFromStore() calls factory.loadStore() per layer
instead of createStore().

Refs #732

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Factory-aware recovery + guard removal

### Task 2: Wire factory resolution into DefaultWorkerExecutionRecoveryService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryServiceTest.java`

**Interfaces:**
- Consumes: `CaseContextImpl.loadFromStore(CaseContextStoreFactory, UUID)` (from Task 1), `CaseDefinitionRegistry.getCaseDefinition(CaseMetaModel)` (existing), `StrategyResolver.resolve(Class, String)` (existing)
- Produces: `DefaultWorkerExecutionRecoveryService.loadOrRestoreCaseInstance(UUID)` now returns a CaseInstance with context backed by the correct factory for durable cases. Used by all callers of the recovery service.

- [ ] **Step 1: Write failing test — durable factory uses loadFromStore**

Create a new test class. The service needs mocking since it has CDI dependencies (`CrossTenantCaseInstanceRepository`, `CaseInstanceCache`, etc.). Use real in-memory implementations where available.

```java
package io.casehub.engine.internal.engine.recovery;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.CaseContextStore;
import io.casehub.api.context.CaseContextStoreFactory;
import io.casehub.api.context.ContextLayer;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.casehub.engine.internal.context.InMemoryCaseContextStore;
import io.casehub.persistence.memory.InMemoryCaseInstanceRepository;
import io.casehub.persistence.memory.InMemoryEventLogRepository;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultWorkerExecutionRecoveryServiceTest {

  private DefaultWorkerExecutionRecoveryService service;
  private InMemoryCaseInstanceRepository caseInstanceRepo;
  private AtomicInteger loadStoreCalls;
  private CaseContextStoreFactory durableFactory;

  @BeforeEach
  void setUp() {
    // Setup will be filled in Step 3 — this test drives the API shape
  }

  @Test
  void durableFactoryUsesLoadFromStore() {
    // Given a case instance with a CaseMetaModel that resolves to a durable factory
    CaseInstance instance = new CaseInstance();
    UUID caseId = UUID.randomUUID();
    instance.setUuid(caseId);
    instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setName("durable-case");
    instance.setCaseMetaModel(metaModel);
    // Store it
    caseInstanceRepo.save(instance, "tenant-1");

    // When recovered
    CaseInstance recovered = service.loadOrRestoreCaseInstance(caseId);

    // Then loadStore was called (not createStore or snapshot recovery)
    assertTrue(loadStoreCalls.get() >= 3, 
        "loadStore should be called for 3 built-in layers, got " + loadStoreCalls.get());
    assertNotNull(recovered.getCaseContext());
  }

  @Test
  void volatileFactoryUsesRecoveryStrategy() {
    // Given a case instance with no special factory (volatile, default)
    CaseInstance instance = new CaseInstance();
    UUID caseId = UUID.randomUUID();
    instance.setUuid(caseId);
    instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
    // No CaseMetaModel → factory resolution returns null → volatile path
    caseInstanceRepo.save(instance, "tenant-1");

    // When recovered
    CaseInstance recovered = service.loadOrRestoreCaseInstance(caseId);

    // Then recovery strategy was used (returns empty context, no loadStore calls)
    assertEquals(0, loadStoreCalls.get());
    assertNotNull(recovered.getCaseContext());
  }
}
```

Note: The full test setup requires wiring the service's dependencies. The test drives the API. Step 3 fills in the setup.

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="DefaultWorkerExecutionRecoveryServiceTest" -q`
Expected: FAIL — the service doesn't have factory resolution yet, and the test setup is incomplete.

- [ ] **Step 3: Implement factory resolution in the service**

Add dependencies and the `resolveFactory` method to `DefaultWorkerExecutionRecoveryService`. Use `ide_insert_member` for the new fields and method:

Add fields (after existing `@Inject` fields):
```java
@Inject CaseDefinitionRegistry caseDefinitionRegistry;
@Inject io.casehub.platform.api.routing.StrategyResolver strategyResolver;
```

Add private method `resolveFactory`:
```java
private io.casehub.api.context.CaseContextStoreFactory resolveFactory(CaseInstance instance) {
    CaseMetaModel metaModel = instance.getCaseMetaModel();
    if (metaModel == null) {
        LOG.errorf("Cannot resolve factory for caseId=%s — CaseMetaModel is null. "
            + "Falling back to volatile recovery. If this case uses a durable factory, "
            + "writes since last snapshot will be lost on next restart.", instance.getUuid());
        return null;
    }
    io.casehub.api.CaseDefinition definition = caseDefinitionRegistry.getCaseDefinition(metaModel);
    if (definition == null) {
        LOG.errorf("Cannot resolve factory for caseId=%s — CaseDefinition not registered "
            + "for metaModel '%s'. Falling back to volatile recovery.",
            instance.getUuid(), metaModel.getName());
        return null;
    }
    String factoryName = definition.getContextStoreFactory();
    if (factoryName == null || factoryName.isBlank()) {
        return null;
    }
    try {
        return strategyResolver.resolve(
            io.casehub.api.context.CaseContextStoreFactory.class, factoryName);
    } catch (Exception e) {
        LOG.errorf(e, "Cannot resolve CaseContextStoreFactory '%s' for caseId=%s. "
            + "Falling back to volatile recovery.", factoryName, instance.getUuid());
        return null;
    }
}
```

Modify `loadOrRestoreCaseInstance` body with `ide_replace_member`:
```java
    CaseInstance cached = caseInstanceCache.get(caseId);
    if (cached != null) {
        return cached;
    }

    CaseInstance instance =
        caseInstanceRepository
            .findByUuid(caseId)
            .orElseThrow(
                () -> new IllegalStateException("CaseInstance not found for caseId=" + caseId));

    io.casehub.api.context.CaseContextStoreFactory factory = resolveFactory(instance);
    CaseContext stateContext;
    if (factory != null && factory.isDurable()) {
        stateContext = CaseContextImpl.loadFromStore(factory, instance.getUuid());
    } else {
        stateContext = recoveryStrategy.recover(instance);
    }

    instance.setCaseContext(stateContext);
    caseInstanceCache.put(instance);
    return instance;
```

Complete the test setUp with the necessary wiring (using real in-memory implementations and test doubles for the registry and resolver).

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="DefaultWorkerExecutionRecoveryServiceTest" -q`
Expected: PASS

- [ ] **Step 5: Run existing recovery tests to verify no regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="SnapshotRecoveryStrategyTest,CaseContextRecoveryIntegrationTest,EventLogReplayRecoveryStrategyTest" -q`
Expected: all pass — volatile path is unchanged.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java \
       runtime/src/test/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryServiceTest.java
git commit -m "feat(#732): wire factory resolution into recovery service

Resolves CaseContextStoreFactory from CaseInstance → CaseMetaModel →
CaseDefinition. Durable factories bypass CaseContextRecoveryStrategy
and use CaseContextImpl.loadFromStore(). Volatile factories use existing
strategy path. Factory resolution failures log ERROR and fall back to
volatile recovery.

Refs #732

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Remove durable guard from CaseHubRuntimeImpl

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java:116-131`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/CaseHubRuntimeImplTest.java`

**Interfaces:**
- Consumes: none (self-contained change)
- Produces: `CaseHubRuntimeImpl.resolveFactory(CaseDefinition)` no longer throws for durable factories. All `startCase()` overloads accept durable factories.

- [ ] **Step 1: Write failing test — durable factory does not throw**

Check existing `CaseHubRuntimeImplTest` for a test that asserts the `UnsupportedOperationException`. If present, update it to expect success. If absent, add:

```java
@Test
void resolveFactoryAcceptsDurableFactory() {
    // Given a CaseDefinition that resolves to a durable factory
    // When resolveFactory is called
    // Then no exception is thrown
    assertDoesNotThrow(() -> /* call startCase with durable factory definition */);
}
```

The exact setup depends on the existing test infrastructure — use `ide_file_structure` to read the test class structure.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `UnsupportedOperationException` thrown.

- [ ] **Step 3: Remove the guard**

Use `ide_replace_member` on `CaseHubRuntimeImpl.resolveFactory`:

```java
    return strategyResolver.resolve(
        io.casehub.api.context.CaseContextStoreFactory.class,
        definition.getContextStoreFactory());
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseHubRuntimeImplTest" -q`
Expected: PASS

- [ ] **Step 5: Run all recovery + runtime tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="*Recovery*Test,*CaseHubRuntime*,CaseContextImpl*,SnapshotRecovery*,EventLogReplay*,ContextStoreFactory*" -q`
Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java \
       runtime/src/test/java/io/casehub/engine/internal/engine/CaseHubRuntimeImplTest.java
git commit -m "feat(#732): remove durable factory guard from CaseHubRuntimeImpl

Recovery path now supports durable factories via loadFromStore(). The
guard that rejected isDurable()=true factories is no longer needed.

Closes #732

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- `specs/issue-1182-snapshot-recovery-fallback/2026-09-27-factory-wired-recovery-design.md` — design spec
- `specs/issue-1182-snapshot-recovery-fallback/decisions.md` — D1-D4
- `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java:52` — final storeFactory field
- `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java:81` — factory constructor
- `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java:63` — recovery entry point
- `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubRuntimeImpl.java:116` — durable guard
- `api/src/main/java/io/casehub/api/context/CaseContextStoreFactory.java:35` — loadStore API
- `common-core/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java:67` — getCaseDefinition
- `docs/specs/2026-07-13-case-context-store-design.md` — CaseContextStore SPI design
- `docs/specs/2026-07-14-context-store-factory-wiring-design.md` — factory wiring design
- GitHub #725 — CaseContextStoreFactory wiring through startCase
- GitHub #732 — this issue
