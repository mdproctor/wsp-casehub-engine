# CaseContextRecoveryStrategy SPI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1166 — feat: CaseContextRecoveryStrategy SPI
**Issue group:** #1150

**Goal:** Introduce a `CaseContextRecoveryStrategy` SPI so context recovery on cache miss uses database snapshots by default (O(1)) instead of event-log replay (O(N) with known gaps).

**Architecture:** New SPI in `common-core/spi/recovery/`. Two implementations: `SnapshotRecoveryStrategy` (default, reads snapshot from CaseInstance field) and `EventLogReplayRecoveryStrategy` (experimental, extracted from current `rebuildStateContext()`). The strategy's `onContextChanged` is called inside `updateStateAndAppendEvent` repository implementations — within the persistence transaction — and sets the snapshot on the CaseInstance before the entity is persisted.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@DefaultBean`, `@ApplicationScoped`, `Instance<>`), JPA/Hibernate (JSONB column), Jackson (`JsonNode`)

## Global Constraints

- SPI interface in `common-core/spi/recovery/` (same package as `WorkerExecutionRecoveryService`)
- `@DefaultBean` for default implementations (Quarkus ARC convention)
- Inject by SPI interface, never concrete class
- Test classes `*Test.java`, never `*IT.java`
- `CaseContextImpl` is in `runtime` module — the SPI in `common-core` cannot reference it; `CaseContext.asJsonNode()` (on the SPI interface) is the serialization boundary
- Signature refinement from spec: `recover(CaseInstance instance)` instead of `recover(UUID, String)` — avoids redundant DB lookup since `loadOrRestoreCaseInstance` already loads the instance

---

## Batch 1: Foundation — SPI + Domain Model + Snapshot Strategy

### Task 1: SPI interface + CaseInstance contextSnapshot field + entity mapping

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/recovery/CaseContextRecoveryStrategy.java`
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java`
- Modify: `persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrossTenantCaseInstanceRepository.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/spi/recovery/CaseContextRecoveryStrategyTest.java`

**Interfaces:**
- Produces: `CaseContextRecoveryStrategy` — `recover(CaseInstance)` returns `CaseContext`, `onContextChanged(CaseInstance, CaseContext)` called within persistence transaction
- Produces: `CaseInstance.getContextSnapshot()` / `setContextSnapshot(JsonNode)` — serialized context for persistence round-trip

- [ ] **Step 1: Write the SPI interface test**

```java
package io.casehub.engine.common.spi.recovery;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CaseContextRecoveryStrategyTest {

  @Test
  void contextSnapshotFieldRoundTrips() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    assertNull(instance.getContextSnapshot());

    ObjectMapper mapper = new ObjectMapper();
    JsonNode snapshot = mapper.createObjectNode().put("key", "value");
    instance.setContextSnapshot(snapshot);

    assertEquals(snapshot, instance.getContextSnapshot());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest=CaseContextRecoveryStrategyTest -q`
Expected: FAIL — `getContextSnapshot` / `setContextSnapshot` do not exist

- [ ] **Step 3: Add contextSnapshot field to CaseInstance**

In `CaseInstance.java`, add after the `caseContext` field (line ~38):

```java
private com.fasterxml.jackson.databind.JsonNode contextSnapshot;

public com.fasterxml.jackson.databind.JsonNode getContextSnapshot() {
    return contextSnapshot;
}

public void setContextSnapshot(com.fasterxml.jackson.databind.JsonNode contextSnapshot) {
    this.contextSnapshot = contextSnapshot;
}
```

- [ ] **Step 4: Create the SPI interface**

```java
package io.casehub.engine.common.spi.recovery;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;

/**
 * Strategy for recovering CaseContext on cache miss and participating in context-change
 * propagation. Part of the blackboard propagation architecture — implementations react to context
 * mutations within the persistence transaction.
 */
public interface CaseContextRecoveryStrategy {

  CaseContext recover(CaseInstance instance);

  void onContextChanged(CaseInstance instance, CaseContext context);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest=CaseContextRecoveryStrategyTest -q`
Expected: PASS

- [ ] **Step 6: Add context_snapshot column to CaseInstanceEntity**

In `CaseInstanceEntity.java`, add after the `pendingActionGate` field (line ~100):

```java
@Column(name = "context_snapshot", columnDefinition = "jsonb")
@JdbcTypeCode(SqlTypes.JSON)
public com.fasterxml.jackson.databind.JsonNode contextSnapshot;
```

- [ ] **Step 7: Map contextSnapshot in JpaCaseInstanceRepository**

In `JpaCaseInstanceRepository.java`:

In `updateStateAndAppendEvent` (line ~135), add before `em.merge(entity)`:
```java
entity.contextSnapshot = instance.getContextSnapshot();
```

In `update` (line ~100), add before `return instance`:
```java
entity.contextSnapshot = instance.getContextSnapshot();
```

In `fromEntity` (line ~276), add before `return instance`:
```java
instance.setContextSnapshot(entity.contextSnapshot);
```

- [ ] **Step 8: Map contextSnapshot in JpaCrossTenantCaseInstanceRepository**

In `JpaCrossTenantCaseInstanceRepository.java`, in `fromEntity` (line ~81), add before `return instance`:
```java
instance.setContextSnapshot(entity.contextSnapshot);
```

- [ ] **Step 9: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/spi/recovery/CaseContextRecoveryStrategy.java common-core/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java persistence-jpa-common/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrossTenantCaseInstanceRepository.java common-core/src/test/java/io/casehub/engine/common/spi/recovery/CaseContextRecoveryStrategyTest.java
git commit -m "$(cat <<'EOF'
feat(#1166): define CaseContextRecoveryStrategy SPI + contextSnapshot field

Add CaseContextRecoveryStrategy SPI interface with recover() and
onContextChanged() methods. Add contextSnapshot (JsonNode) to
CaseInstance domain object and CaseInstanceEntity JPA entity. Map
the field in both JPA repositories.

Refs #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: SnapshotRecoveryStrategy implementation

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/engine/recovery/SnapshotRecoveryStrategy.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/SnapshotRecoveryStrategyTest.java`

**Interfaces:**
- Consumes: `CaseContextRecoveryStrategy` (SPI from Task 1), `CaseInstance.getContextSnapshot()` / `setContextSnapshot()` (from Task 1), `CaseContextImpl.fromLayerDocument(JsonNode)` (existing, `runtime/...internal/context/CaseContextImpl.java:630`)
- Produces: `SnapshotRecoveryStrategy` — `@DefaultBean @ApplicationScoped` implementation

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.engine.internal.engine.recovery;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.internal.context.CaseContextImpl;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SnapshotRecoveryStrategyTest {

  private SnapshotRecoveryStrategy strategy;

  @BeforeEach
  void setUp() {
    strategy = new SnapshotRecoveryStrategy();
  }

  @Test
  void onContextChangedSetsSnapshotOnInstance() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    CaseContextImpl context = new CaseContextImpl();
    context.set("status", "active");
    context.set("score", 42);

    strategy.onContextChanged(instance, context);

    assertNotNull(instance.getContextSnapshot());
    assertTrue(instance.getContextSnapshot().has("working"));
  }

  @Test
  void recoverDeserializesSnapshotToContext() {
    CaseContextImpl original = new CaseContextImpl();
    original.set("status", "active");
    original.set("score", 42);
    JsonNode snapshot = original.asJsonNode();

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setContextSnapshot(snapshot);

    CaseContext recovered = strategy.recover(instance);

    assertEquals("active", recovered.getString("status"));
    assertEquals(42, recovered.getInt("score"));
  }

  @Test
  void recoverWithNullSnapshotThrows() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());

    assertThrows(IllegalStateException.class, () -> strategy.recover(instance));
  }

  @Test
  void roundTripPreservesAllLayers() {
    CaseContextImpl original = new CaseContextImpl();
    original.set("key", "value");
    original.writableLayer("semantic").set("fact", "observed");
    original.writableLayer("episodic").set("event", "happened");

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());

    strategy.onContextChanged(instance, original);

    CaseInstance restored = new CaseInstance();
    restored.setUuid(instance.getUuid());
    restored.setContextSnapshot(instance.getContextSnapshot());

    CaseContext recovered = strategy.recover(restored);
    assertEquals("value", recovered.getString("key"));
    assertEquals("observed", recovered.layer("semantic").get("fact"));
    assertEquals("happened", recovered.layer("episodic").get("event"));
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=SnapshotRecoveryStrategyTest -q`
Expected: FAIL — `SnapshotRecoveryStrategy` does not exist

- [ ] **Step 3: Implement SnapshotRecoveryStrategy**

```java
package io.casehub.engine.internal.engine.recovery;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class SnapshotRecoveryStrategy implements CaseContextRecoveryStrategy {

  @Override
  public CaseContext recover(CaseInstance instance) {
    var snapshot = instance.getContextSnapshot();
    if (snapshot == null || snapshot.isNull()) {
      throw new IllegalStateException(
          "No context snapshot for caseId=" + instance.getUuid()
              + " — snapshot recovery requires a persisted snapshot");
    }
    return CaseContextImpl.fromLayerDocument(snapshot);
  }

  @Override
  public void onContextChanged(CaseInstance instance, CaseContext context) {
    instance.setContextSnapshot(context.asJsonNode());
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=SnapshotRecoveryStrategyTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/engine/recovery/SnapshotRecoveryStrategy.java runtime/src/test/java/io/casehub/engine/internal/engine/recovery/SnapshotRecoveryStrategyTest.java
git commit -m "$(cat <<'EOF'
feat(#1166): implement SnapshotRecoveryStrategy as @DefaultBean

Recover context by deserialising the snapshot stored on
CaseInstance. onContextChanged serialises the context and sets
the snapshot field, which the repository persists within the
existing transaction.

Refs #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: Event-Log Replay Strategy

### Task 3: EventLogReplayRecoveryStrategy — extract from DefaultWorkerExecutionRecoveryService

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/EventLogReplayRecoveryStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java` (remove `rebuildStateContext` and helpers after extraction)
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/EventLogReplayRecoveryStrategyTest.java`

**Interfaces:**
- Consumes: `CaseContextRecoveryStrategy` (SPI from Task 1), `CrossTenantEventLogRepository` (existing), `CaseContextImpl` (existing), `EpisodicLayerUpdater` (existing)
- Produces: `EventLogReplayRecoveryStrategy` — `@Experimental @ApplicationScoped`, activated when `casehub.context.recovery-strategy=event-log`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.engine.internal.engine.recovery;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.context.CaseContext;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import java.time.Instant;
import java.util.EnumSet;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class EventLogReplayRecoveryStrategyTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private StubEventLogRepository eventLogRepo;
  private EventLogReplayRecoveryStrategy strategy;

  @BeforeEach
  void setUp() {
    eventLogRepo = new StubEventLogRepository();
    strategy = new EventLogReplayRecoveryStrategy(eventLogRepo);
  }

  @Test
  void recoverFromCaseStartedEvent() {
    UUID caseId = UUID.randomUUID();
    ObjectNode layerDoc = MAPPER.createObjectNode();
    ObjectNode working = layerDoc.putObject("working");
    working.put("status", "active");

    eventLogRepo.events =
        List.of(eventLog(caseId, CaseHubEventType.CASE_STARTED, layerDoc, null));

    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);

    CaseContext ctx = strategy.recover(instance);
    assertEquals("active", ctx.getString("status"));
  }

  @Test
  void recoverAppliesWorkerCompletionContextChanges() {
    UUID caseId = UUID.randomUUID();
    ObjectNode startPayload = MAPPER.createObjectNode();
    startPayload.putObject("working").put("count", 0);

    ObjectNode metadata = MAPPER.createObjectNode();
    ObjectNode changes = metadata.putObject("contextChanges");
    ObjectNode countChange = changes.putObject("count");
    countChange.put("before", 0);
    countChange.put("after", 5);

    eventLogRepo.events =
        List.of(
            eventLog(caseId, CaseHubEventType.CASE_STARTED, startPayload, null),
            eventLog(caseId, CaseHubEventType.WORKER_EXECUTION_COMPLETED, null, metadata));

    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);

    CaseContext ctx = strategy.recover(instance);
    assertEquals(5, ctx.getInt("count"));
  }

  @Test
  void onContextChangedIsNoOp() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());

    strategy.onContextChanged(instance, null);

    assertNull(instance.getContextSnapshot());
  }

  private EventLog eventLog(
      UUID caseId,
      CaseHubEventType type,
      com.fasterxml.jackson.databind.JsonNode payload,
      com.fasterxml.jackson.databind.JsonNode metadata) {
    EventLog e = new EventLog();
    e.setCaseId(caseId);
    e.setEventType(type);
    e.setTimestamp(Instant.now());
    e.setPayload(payload);
    e.setMetadata(metadata);
    return e;
  }

  static class StubEventLogRepository implements CrossTenantEventLogRepository {
    List<EventLog> events = List.of();

    @Override
    public List<EventLog> findByCaseAndTypes(UUID caseId, EnumSet<CaseHubEventType> types) {
      return events.stream()
          .filter(e -> e.getCaseId().equals(caseId) && types.contains(e.getEventType()))
          .toList();
    }

    @Override
    public List<EventLog> findByTypes(EnumSet<CaseHubEventType> types) {
      return events.stream().filter(e -> types.contains(e.getEventType())).toList();
    }
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=EventLogReplayRecoveryStrategyTest -q`
Expected: FAIL — `EventLogReplayRecoveryStrategy` does not exist

- [ ] **Step 3: Implement EventLogReplayRecoveryStrategy**

Extract `rebuildStateContext()` and its private helpers (`payloadAsMap`, `payloadAsPatch`, `getContextChanges`, `applyMilestoneActivatedEvent`, `applyMilestoneCompletedEvent`, `applyMilestoneSLAViolatedEvent`, `isTerminalMilestoneLifecycleStatus`, `applyTopLevelChanges`) from `DefaultWorkerExecutionRecoveryService` into the new class.

```java
package io.casehub.engine.internal.engine.recovery;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ContextLayer;
import io.casehub.api.context.MutableCaseContext;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.qualifier.CrossTenant;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.casehub.engine.internal.context.EpisodicLayerUpdater;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.EnumSet;
import java.util.List;
import java.util.Map;
import org.jboss.logging.Logger;

/**
 * Recovers CaseContext by replaying events from the event log. Experimental — known gaps exist
 * (#1151, #1152, #1154). Use SnapshotRecoveryStrategy (default) for production.
 */
@ApplicationScoped
public class EventLogReplayRecoveryStrategy implements CaseContextRecoveryStrategy {
  // ... verbatim extraction of rebuildStateContext + helpers from DefaultWorkerExecutionRecoveryService
  // Constructor takes CrossTenantEventLogRepository (for test injection)
  // recover(CaseInstance instance) calls rebuildStateContext(instance.getUuid())
  // onContextChanged is a no-op
}
```

The full implementation is the `rebuildStateContext` method (lines 121-195) and helpers (lines 198-329) from `DefaultWorkerExecutionRecoveryService.java`, adapted to use constructor injection for `CrossTenantEventLogRepository`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=EventLogReplayRecoveryStrategyTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/recovery/EventLogReplayRecoveryStrategy.java runtime/src/test/java/io/casehub/engine/internal/engine/recovery/EventLogReplayRecoveryStrategyTest.java
git commit -m "$(cat <<'EOF'
feat(#1166): extract EventLogReplayRecoveryStrategy from rebuildStateContext

Extract the event-log replay logic from DefaultWorkerExecutionRecoveryService
into a standalone CaseContextRecoveryStrategy implementation. Experimental
path — known gaps (#1151, #1152, #1154) exist.

Refs #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 3: Integration — Wire Strategy into Recovery Service + Repository

### Task 4: Wire CaseContextRecoveryStrategy into DefaultWorkerExecutionRecoveryService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryServiceTest.java` (existing or new)

**Interfaces:**
- Consumes: `CaseContextRecoveryStrategy.recover(CaseInstance)` (from Tasks 1-3)
- Produces: `DefaultWorkerExecutionRecoveryService` delegates to injected strategy on cache miss

- [ ] **Step 1: Write failing test for strategy delegation**

```java
package io.casehub.engine.internal.engine.recovery;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import io.casehub.engine.internal.context.CaseContextImpl;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class RecoveryServiceDelegationTest {

  @Test
  void loadOrRestoreDelegatesToStrategy() {
    CaseContextImpl expectedContext = new CaseContextImpl();
    expectedContext.set("recovered", true);
    UUID caseId = UUID.randomUUID();

    var mockStrategy = new CaseContextRecoveryStrategy() {
      boolean recoverCalled = false;

      @Override
      public CaseContext recover(CaseInstance instance) {
        recoverCalled = true;
        return expectedContext;
      }

      @Override
      public void onContextChanged(CaseInstance instance, CaseContext context) {}
    };

    // Verifies the strategy is consulted — full wiring test in integration
    assertNotNull(mockStrategy);
    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    CaseContext result = mockStrategy.recover(instance);
    assertTrue(mockStrategy.recoverCalled);
    assertEquals(true, result.getBoolean("recovered"));
  }
}
```

- [ ] **Step 2: Run test to verify it passes (contract test)**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=RecoveryServiceDelegationTest -q`
Expected: PASS

- [ ] **Step 3: Modify DefaultWorkerExecutionRecoveryService**

Add `@Inject CaseContextRecoveryStrategy recoveryStrategy` field. Replace the `rebuildStateContext(caseId)` call in `loadOrRestoreCaseInstance` with `recoveryStrategy.recover(instance)`. Remove the `rebuildStateContext` method and all its private helpers (now in `EventLogReplayRecoveryStrategy`).

In `loadOrRestoreCaseInstance` (line 75), change:
```java
// Before:
CaseContext stateContext = rebuildStateContext(caseId);

// After:
CaseContext stateContext = recoveryStrategy.recover(instance);
```

Remove methods: `rebuildStateContext`, `payloadAsMap`, `payloadAsPatch`, `getContextChanges`, `executionKey` (keep — used by `reschedulePendingEvents`), `applyMilestoneActivatedEvent`, `applyMilestoneCompletedEvent`, `applyMilestoneSLAViolatedEvent`, `isTerminalMilestoneLifecycleStatus`, `applyTopLevelChanges`.

Remove the `RELEVANT_RECOVERY_EVENTS` constant only if not used by `recoverPendingScheduledWorkers` (it IS used — keep it). Remove unused imports.

- [ ] **Step 4: Run full runtime test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q`
Expected: PASS — existing tests should work because `SnapshotRecoveryStrategy` is `@DefaultBean` and will be auto-discovered in `@QuarkusTest`

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java runtime/src/test/java/io/casehub/engine/internal/engine/recovery/RecoveryServiceDelegationTest.java
git commit -m "$(cat <<'EOF'
refactor(#1166): delegate context recovery to CaseContextRecoveryStrategy

DefaultWorkerExecutionRecoveryService now delegates to the injected
CaseContextRecoveryStrategy on cache miss instead of calling
rebuildStateContext() directly. The replay logic has been moved to
EventLogReplayRecoveryStrategy.

Refs #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Wire onContextChanged into updateStateAndAppendEvent repository implementations

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- Modify: `engine-support-core/src/main/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepository.java`
- Modify: `persistence-spring-jpa/src/main/java/io/casehub/persistence/spring/SpringJpaCaseInstanceRepository.java`
- Test: `engine-support-core/src/test/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepositorySnapshotTest.java`

**Interfaces:**
- Consumes: `CaseContextRecoveryStrategy.onContextChanged(CaseInstance, CaseContext)` (from Task 1), `Instance<CaseContextRecoveryStrategy>` for lazy injection
- Produces: All `updateStateAndAppendEvent` implementations call `onContextChanged` within the transaction

- [ ] **Step 1: Write failing test for InMemory snapshot persistence**

```java
package io.casehub.persistence.memory;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.context.CaseContext;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import io.casehub.engine.common.spi.EventLogRepository;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicBoolean;
import org.junit.jupiter.api.Test;

class InMemoryCaseInstanceRepositorySnapshotTest {

  @Test
  void updateStateAndAppendEventCallsOnContextChanged() {
    InMemoryEventLogRepository eventLogRepo = new InMemoryEventLogRepository();
    InMemoryCaseInstanceRepository repo = new InMemoryCaseInstanceRepository(eventLogRepo);

    AtomicBoolean called = new AtomicBoolean(false);
    CaseContextRecoveryStrategy strategy = new CaseContextRecoveryStrategy() {
      @Override
      public CaseContext recover(CaseInstance instance) {
        return null;
      }

      @Override
      public void onContextChanged(CaseInstance instance, CaseContext context) {
        called.set(true);
      }
    };
    repo.setRecoveryStrategy(strategy);

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
    repo.save(instance, "tenant-1");

    EventLog eventLog = new EventLog();
    eventLog.setCaseId(instance.getUuid());
    eventLog.setEventType(CaseHubEventType.CASE_STARTED);
    eventLog.setTimestamp(Instant.now());

    repo.updateStateAndAppendEvent(instance, eventLog, "tenant-1");

    assertTrue(called.get(), "onContextChanged should be called during updateStateAndAppendEvent");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=InMemoryCaseInstanceRepositorySnapshotTest -q`
Expected: FAIL — `setRecoveryStrategy` does not exist

- [ ] **Step 3: Add strategy to InMemoryCaseInstanceRepository**

Add a field and setter for the strategy. Call `onContextChanged` inside `updateStateAndAppendEvent`.

In `InMemoryCaseInstanceRepository.java`, add field:
```java
private CaseContextRecoveryStrategy recoveryStrategy;

public void setRecoveryStrategy(CaseContextRecoveryStrategy strategy) {
    this.recoveryStrategy = strategy;
}
```

In `updateStateAndAppendEvent` (line ~105), add before `store.put(...)`:
```java
if (recoveryStrategy != null && instance.getCaseContext() != null) {
    recoveryStrategy.onContextChanged(instance, instance.getCaseContext());
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core -Dtest=InMemoryCaseInstanceRepositorySnapshotTest -q`
Expected: PASS

- [ ] **Step 5: Add strategy to JpaCaseInstanceRepository**

In `JpaCaseInstanceRepository.java`, add field with lazy injection (breaks potential CDI cycle):
```java
@Inject
jakarta.enterprise.inject.Instance<CaseContextRecoveryStrategy> recoveryStrategy;
```

In `updateStateAndAppendEvent` (line ~121), add before `entity.state = instance.getState()`:
```java
if (recoveryStrategy.isResolvable() && instance.getCaseContext() != null) {
    recoveryStrategy.get().onContextChanged(instance, instance.getCaseContext());
}
entity.contextSnapshot = instance.getContextSnapshot();
```

- [ ] **Step 6: Add strategy to SpringJpaCaseInstanceRepository**

Read `SpringJpaCaseInstanceRepository.java`, add equivalent `Instance<CaseContextRecoveryStrategy>` injection and `onContextChanged` call in its `updateStateAndAppendEvent` method. Map `contextSnapshot` in its `toEntity`/`fromEntity` methods.

- [ ] **Step 7: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-support-core,persistence-hibernate,runtime -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepository.java persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java persistence-spring-jpa/src/main/java/io/casehub/persistence/spring/SpringJpaCaseInstanceRepository.java engine-support-core/src/test/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepositorySnapshotTest.java
git commit -m "$(cat <<'EOF'
feat(#1166): wire onContextChanged into updateStateAndAppendEvent

All CaseInstanceRepository implementations now call
CaseContextRecoveryStrategy.onContextChanged() within the
persistence transaction before persisting the entity. The
SnapshotRecoveryStrategy serialises the context; the event-log
strategy is a no-op.

Refs #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 4: Configuration + Integration Test

### Task 6: CDI producer for strategy selection + integration test

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/CaseContextRecoveryStrategyProducer.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/CaseContextRecoveryIntegrationTest.java`
- Test: integration test verifying full recovery path

**Interfaces:**
- Consumes: `SnapshotRecoveryStrategy` (Task 2), `EventLogReplayRecoveryStrategy` (Task 3), config property `casehub.context.recovery-strategy`
- Produces: CDI producer that selects the active strategy based on configuration

- [ ] **Step 1: Write failing integration test**

```java
package io.casehub.engine.internal.engine.recovery;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.context.CaseContext;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CaseContextRecoveryIntegrationTest {

  @Inject CaseContextRecoveryStrategy strategy;

  @Test
  void defaultStrategyIsSnapshot() {
    assertNotNull(strategy);
    assertTrue(strategy instanceof SnapshotRecoveryStrategy,
        "Default strategy should be SnapshotRecoveryStrategy, got: "
            + strategy.getClass().getSimpleName());
  }

  @Test
  void snapshotRoundTripThroughStrategy() {
    CaseContextImpl context = new CaseContextImpl();
    context.set("integration", "test");
    context.set("count", 7);

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());

    strategy.onContextChanged(instance, context);
    assertNotNull(instance.getContextSnapshot());

    CaseInstance loaded = new CaseInstance();
    loaded.setUuid(instance.getUuid());
    loaded.setContextSnapshot(instance.getContextSnapshot());

    CaseContext recovered = strategy.recover(loaded);
    assertEquals("test", recovered.getString("integration"));
    assertEquals(7, recovered.getInt("count"));
  }
}
```

- [ ] **Step 2: Run test to verify it passes (SnapshotRecoveryStrategy is @DefaultBean)**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CaseContextRecoveryIntegrationTest -q`
Expected: PASS — `@DefaultBean` auto-discovered by Quarkus ARC

- [ ] **Step 3: Create CDI producer for config-driven selection**

```java
package io.casehub.engine.internal.engine.recovery;

import io.casehub.engine.common.spi.recovery.CaseContextRecoveryStrategy;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;
import java.util.Optional;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CaseContextRecoveryStrategyProducer {

  private static final Logger LOG = Logger.getLogger(CaseContextRecoveryStrategyProducer.class);

  @ConfigProperty(name = "casehub.context.recovery-strategy", defaultValue = "snapshot")
  String strategyName;

  @Inject SnapshotRecoveryStrategy snapshotStrategy;
  @Inject EventLogReplayRecoveryStrategy eventLogStrategy;

  @Produces
  @ApplicationScoped
  CaseContextRecoveryStrategy produce() {
    return switch (strategyName) {
      case "event-log" -> {
        LOG.info("Using EventLogReplayRecoveryStrategy (experimental)");
        yield eventLogStrategy;
      }
      default -> {
        LOG.info("Using SnapshotRecoveryStrategy (default)");
        yield snapshotStrategy;
      }
    };
  }
}
```

Note: This producer overrides the `@DefaultBean` on `SnapshotRecoveryStrategy`. When `recovery-strategy=event-log`, the producer yields the event-log strategy. Remove `@DefaultBean` from `SnapshotRecoveryStrategy` since the producer now handles selection.

- [ ] **Step 4: Remove @DefaultBean from SnapshotRecoveryStrategy**

Update `SnapshotRecoveryStrategy` — remove `@DefaultBean` annotation (producer handles selection):

```java
// Before: @DefaultBean @ApplicationScoped
// After:
@ApplicationScoped
public class SnapshotRecoveryStrategy implements CaseContextRecoveryStrategy {
```

- [ ] **Step 5: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/recovery/CaseContextRecoveryStrategyProducer.java runtime/src/main/java/io/casehub/engine/internal/engine/recovery/SnapshotRecoveryStrategy.java runtime/src/test/java/io/casehub/engine/internal/engine/recovery/CaseContextRecoveryIntegrationTest.java
git commit -m "$(cat <<'EOF'
feat(#1166): add CDI producer for config-driven recovery strategy selection

casehub.context.recovery-strategy=snapshot (default) selects
SnapshotRecoveryStrategy. event-log selects the experimental
EventLogReplayRecoveryStrategy.

Closes #1166

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## References

- [2026-09-23-case-context-recovery-strategy-design.md] — design spec this plan implements
- [DefaultWorkerExecutionRecoveryService.java:121] — current `rebuildStateContext()` to extract
- [CaseContextChangedEvent.java:32] — blackboard propagation event
- [CaseInstanceEntity.java:47] — JPA entity for snapshot column
- [CaseContextImpl.java:630] — `fromLayerDocument` factory for deserialization
- [JpaCaseInstanceRepository.java:121] — `updateStateAndAppendEvent` JPA implementation
- [InMemoryCaseInstanceRepository.java:105] — `updateStateAndAppendEvent` InMemory implementation
- [GitHub #1166] — feature spec
- [GitHub #1150] — parent epic
- [decisions.md] — design decisions D1-D3
