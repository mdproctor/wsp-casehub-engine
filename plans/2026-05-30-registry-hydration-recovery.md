# BlackboardRegistry Hydration + HumanTask Recovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two post-restart failure modes — in-flight HumanTask completions silently dropped because the registry is empty (#274), and WorkItems that completed while the JVM was down permanently orphaning cases (#398).

**Architecture:** `BlackboardRegistry.get()` lazily hydrates from `PlanItemStore.findDelegated()` on first miss after restart, so completion handlers find their PlanItems without any startup ordering. A separate `HumanTaskRecoveryService` scans `findAllDelegated()` at startup and catches up WorkItems that already terminated during downtime.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, blocking JPA (work-adapter), Panache reactive (persistence-hibernate), AssertJ, JUnit 5, in-memory store alternatives for tests.

---

## File Map

| Action | File | What changes |
|--------|------|-------------|
| Create | `common/.../model/TargetType.java` | new enum |
| Create | `common/.../model/PlanItemSaveRequest.java` | new value object |
| Modify | `common/.../model/PlanItemRecord.java` | add `targetType`, `outputMappingExpression` |
| Modify | `common/.../spi/PlanItemStore.java` | `save(PlanItemSaveRequest)`, `findDelegated(UUID)`, `findAllDelegated()` |
| Modify | `common/.../spi/ReactivePlanItemStore.java` | same three methods, Uni-wrapped |
| Modify | `blackboard/.../store/NoOpPlanItemStore.java` | implement new methods |
| Modify | `persistence-memory/.../MemoryPlanItemStore.java` | implement new methods |
| Modify | `persistence-memory/.../MemoryReactivePlanItemStore.java` | delegate new methods |
| Modify | `common/.../spi/PlanItemStoreContractTest.java` | 3 new contract tests, update existing |
| Modify | `work-adapter/.../WorkAdapterPlanItemEntity.java` | add `targetType`, `outputMappingExpression` columns |
| Modify | `persistence-hibernate/.../PlanItemEntity.java` | same two columns |
| Modify | `work-adapter/.../JpaPlanItemStore.java` | implement new SPI methods |
| Modify | `persistence-hibernate/.../JpaReactivePlanItemStore.java` | implement new SPI methods |
| Modify | `work-adapter/.../HumanTaskScheduleHandler.java` | update `save()` call sites |
| Modify | `blackboard/.../plan/PlanItem.java` | add `restore()` static factory + new private constructor |
| Modify | `blackboard/.../plan/DefaultCasePlanModel.java` | add `restorePlanItem()` |
| Create | `blackboard/.../registry/PlanItemRestorer.java` | package-private, converts records → PlanItems |
| Modify | `blackboard/.../registry/BlackboardRegistry.java` | lazy `get()` + inject `PlanItemStore` + `PlanItemRestorer` |
| Create | `blackboard/.../registry/BlackboardRegistryLazyHydrationTest.java` | new test |
| Modify | `blackboard/.../handler/PlanItemCompletionHandler.java` | `blocking=true`, `void` return |
| Modify | `blackboard/.../handler/WorkerRetryExhaustionHandler.java` | `blocking=true`, `void` return |
| Modify | `blackboard/.../handler/PlanItemFaultHandler.java` | `blocking=true`, `void` return |
| Modify | `work-adapter/.../WorkItemLifecycleAdapter.java` | null-guard on `applyOutputMapping()` |
| Create | `work-adapter/.../PlanItemCompletionApplier.java` | shared status-transition logic |
| Modify | `work-adapter/.../WorkItemLifecycleAdapter.java` | refactor to use `PlanItemCompletionApplier` |
| Modify | `~/casehub/work/.../WorkItemService.java` | add `findByCallerRef()` |
| Create | `work-adapter/.../recovery/HumanTaskRecoveryServiceTest.java` | new test |
| Create | `work-adapter/.../recovery/HumanTaskRecoveryService.java` | startup recovery for #398 |

---

### Task 1: Add TargetType, PlanItemSaveRequest, update PlanItemRecord

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/model/TargetType.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java`

- [ ] **Step 1: Create TargetType enum**

```java
// common/src/main/java/io/casehub/engine/common/internal/model/TargetType.java
package io.casehub.engine.common.internal.model;

public enum TargetType {
  HUMAN_TASK, CAPABILITY, SUB_CASE, EXTENSION
}
```

- [ ] **Step 2: Create PlanItemSaveRequest value object**

```java
// common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java
package io.casehub.engine.common.internal.model;

import java.time.Instant;
import java.util.UUID;

public record PlanItemSaveRequest(
    UUID caseId,
    String planItemId,
    String bindingName,
    PlanItemStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression) {}
```

- [ ] **Step 3: Add two fields to PlanItemRecord**

Replace the entire `PlanItemRecord.java`:

```java
// common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java
package io.casehub.engine.common.internal.model;

import java.time.Instant;
import java.util.UUID;

/** Lightweight read model returned by {@link io.casehub.engine.common.spi.PlanItemStore#findByCaseId}. */
public record PlanItemRecord(
    UUID caseId,
    String planItemId,
    String bindingName,
    PlanItemStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression) {}
```

- [ ] **Step 4: Build common module to verify compile**

```bash
mvn --batch-mode install -DskipTests -pl common -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: add TargetType enum, PlanItemSaveRequest value object, extend PlanItemRecord with targetType+outputMappingExpression

Refs #274, #398"
```

---

### Task 2: Update PlanItemStore + ReactivePlanItemStore SPIs

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/ReactivePlanItemStore.java`

- [ ] **Step 1: Update PlanItemStore interface**

Replace the entire file:

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import java.util.List;
import java.util.UUID;

/**
 * Blocking SPI for durable PlanItem status persistence.
 *
 * <p>Used by HumanTaskScheduleHandler (blocking context) to write PlanItem status in the same JTA
 * transaction as WorkItem creation. The default no-op implementation (NoOpPlanItemStore) is active
 * when no real store is on the classpath.
 */
public interface PlanItemStore {

  /** Record a new PlanItem. */
  void save(PlanItemSaveRequest request);

  /** Update the stored status for an existing PlanItem. */
  void updateStatus(String planItemId, PlanItemStatus status);

  /** Return all PlanItemRecords for the given case. */
  List<PlanItemRecord> findByCaseId(UUID caseId);

  /** Return all DELEGATED PlanItemRecords for the given case. Used by BlackboardRegistry lazy hydration. */
  List<PlanItemRecord> findDelegated(UUID caseId);

  /** Return all DELEGATED PlanItemRecords across all cases. Used by HumanTaskRecoveryService at startup. */
  List<PlanItemRecord> findAllDelegated();
}
```

- [ ] **Step 2: Update ReactivePlanItemStore interface**

Replace the entire file:

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.smallrye.mutiny.Uni;
import java.util.List;
import java.util.UUID;

/**
 * Reactive mirror of PlanItemStore — method signatures identical, return types wrapped in Uni.
 * For engine runtime handlers on Vert.x IO threads.
 */
public interface ReactivePlanItemStore {

  Uni<Void> save(PlanItemSaveRequest request);

  Uni<Void> updateStatus(String planItemId, PlanItemStatus status);

  Uni<List<PlanItemRecord>> findByCaseId(UUID caseId);

  Uni<List<PlanItemRecord>> findDelegated(UUID caseId);

  Uni<List<PlanItemRecord>> findAllDelegated();
}
```

- [ ] **Step 3: Build common module**

```bash
mvn --batch-mode install -DskipTests -pl common -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
```

Expected: `BUILD SUCCESS` (implementations will fail until Task 3)

---

### Task 3: Implement new SPI methods in NoOp and Memory stores

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/store/NoOpPlanItemStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java`

- [ ] **Step 1: Update NoOpPlanItemStore**

Replace the entire file:

```java
package io.casehub.blackboard.store;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.spi.PlanItemStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.UUID;

/**
 * No-op PlanItemStore — active when no real store is on the classpath.
 * PlanItem status is tracked in-memory only via PlanItem itself.
 */
@DefaultBean
@ApplicationScoped
public class NoOpPlanItemStore implements PlanItemStore {

  @Override
  public void save(PlanItemSaveRequest request) {}

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {}

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return List.of();
  }

  @Override
  public List<PlanItemRecord> findDelegated(UUID caseId) {
    return List.of();
  }

  @Override
  public List<PlanItemRecord> findAllDelegated() {
    return List.of();
  }
}
```

- [ ] **Step 2: Update MemoryPlanItemStore**

Replace the entire file:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.spi.PlanItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * In-memory PlanItemStore for engine unit tests.
 * Activated via quarkus.arc.selected-alternatives — never active in production.
 */
@Alternative
@ApplicationScoped
public class MemoryPlanItemStore implements PlanItemStore {

  private final ConcurrentHashMap<String, PlanItemRecord> records = new ConcurrentHashMap<>();

  public void clear() {
    records.clear();
  }

  @Override
  public void save(PlanItemSaveRequest request) {
    records.put(
        request.planItemId(),
        new PlanItemRecord(
            request.caseId(),
            request.planItemId(),
            request.bindingName(),
            request.status(),
            request.createdAt(),
            request.targetType(),
            request.outputMappingExpression()));
  }

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {
    records.computeIfPresent(
        planItemId,
        (k, r) ->
            new PlanItemRecord(
                r.caseId(), r.planItemId(), r.bindingName(), status, r.createdAt(),
                r.targetType(), r.outputMappingExpression()));
  }

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return records.values().stream()
        .filter(r -> caseId.equals(r.caseId()))
        .collect(Collectors.toList());
  }

  @Override
  public List<PlanItemRecord> findDelegated(UUID caseId) {
    return records.values().stream()
        .filter(r -> caseId.equals(r.caseId()) && r.status() == PlanItemStatus.DELEGATED)
        .collect(Collectors.toList());
  }

  @Override
  public List<PlanItemRecord> findAllDelegated() {
    return records.values().stream()
        .filter(r -> r.status() == PlanItemStatus.DELEGATED)
        .collect(Collectors.toList());
  }
}
```

- [ ] **Step 3: Update MemoryReactivePlanItemStore**

Replace the entire file:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.spi.ReactivePlanItemStore;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.List;
import java.util.UUID;

/**
 * Reactive wrapper around MemoryPlanItemStore for engine handlers on Vert.x IO threads.
 * Activated via quarkus.arc.selected-alternatives — never active in production.
 */
@Alternative
@ApplicationScoped
public class MemoryReactivePlanItemStore implements ReactivePlanItemStore {

  @Inject MemoryPlanItemStore delegate;

  @Override
  public Uni<Void> save(PlanItemSaveRequest request) {
    delegate.save(request);
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> updateStatus(String planItemId, PlanItemStatus status) {
    delegate.updateStatus(planItemId, status);
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId) {
    return Uni.createFrom().item(delegate.findByCaseId(caseId));
  }

  @Override
  public Uni<List<PlanItemRecord>> findDelegated(UUID caseId) {
    return Uni.createFrom().item(delegate.findDelegated(caseId));
  }

  @Override
  public Uni<List<PlanItemRecord>> findAllDelegated() {
    return Uni.createFrom().item(delegate.findAllDelegated());
  }
}
```

- [ ] **Step 4: Build blackboard and persistence-memory**

```bash
mvn --batch-mode install -DskipTests -pl blackboard,persistence-memory -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
```

Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/ persistence-memory/ common/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: update PlanItemStore SPI — PlanItemSaveRequest, findDelegated(), findAllDelegated(); implement in NoOp and Memory stores

Refs #274, #398"
```

---

### Task 4: Update contract tests for PlanItemStore

**Files:**
- Modify: `common/src/test/java/io/casehub/engine/common/spi/PlanItemStoreContractTest.java`

- [ ] **Step 1: Update existing tests and add three new contract tests**

Replace the entire file:

```java
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.internal.model.TargetType;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/** Abstract contract test — extend with a concrete impl to verify the Store SPI. */
public abstract class PlanItemStoreContractTest {

  protected abstract PlanItemStore store();

  private PlanItemSaveRequest req(UUID caseId, String planItemId, String binding, PlanItemStatus status) {
    return new PlanItemSaveRequest(caseId, planItemId, binding, status, Instant.now(), null, null);
  }

  @Test
  void save_and_findByCaseId() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store().save(new PlanItemSaveRequest(caseId, planItemId, "my-binding",
        PlanItemStatus.PENDING, Instant.now(), null, null));
    List<PlanItemRecord> results = store().findByCaseId(caseId);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).planItemId()).isEqualTo(planItemId);
    assertThat(results.get(0).status()).isEqualTo(PlanItemStatus.PENDING);
  }

  @Test
  void updateStatus_changes_stored_status() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store().save(req(caseId, planItemId, "my-binding", PlanItemStatus.PENDING));
    store().updateStatus(planItemId, PlanItemStatus.RUNNING);
    List<PlanItemRecord> results = store().findByCaseId(caseId);
    assertThat(results.get(0).status()).isEqualTo(PlanItemStatus.RUNNING);
  }

  @Test
  void findDelegated_returnsOnlyDelegatedForCase() {
    UUID caseId = UUID.randomUUID();
    UUID otherCaseId = UUID.randomUUID();
    String piPending = UUID.randomUUID().toString();
    String piDelegated = UUID.randomUUID().toString();
    String piCompleted = UUID.randomUUID().toString();
    String piOther = UUID.randomUUID().toString();

    store().save(req(caseId, piPending, "b1", PlanItemStatus.PENDING));
    store().save(new PlanItemSaveRequest(caseId, piDelegated, "b2", PlanItemStatus.DELEGATED,
        Instant.now(), TargetType.HUMAN_TASK, ".result.approval"));
    store().save(new PlanItemSaveRequest(caseId, piCompleted, "b3", PlanItemStatus.COMPLETED,
        Instant.now(), TargetType.HUMAN_TASK, null));
    store().save(new PlanItemSaveRequest(otherCaseId, piOther, "b4", PlanItemStatus.DELEGATED,
        Instant.now(), TargetType.HUMAN_TASK, null));

    List<PlanItemRecord> result = store().findDelegated(caseId);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).planItemId()).isEqualTo(piDelegated);
    assertThat(result.get(0).targetType()).isEqualTo(TargetType.HUMAN_TASK);
    assertThat(result.get(0).outputMappingExpression()).isEqualTo(".result.approval");
  }

  @Test
  void findAllDelegated_returnsAcrossAllCases() {
    UUID c1 = UUID.randomUUID();
    UUID c2 = UUID.randomUUID();
    store().save(new PlanItemSaveRequest(c1, UUID.randomUUID().toString(), "a",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, null));
    store().save(new PlanItemSaveRequest(c2, UUID.randomUUID().toString(), "b",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, null));
    store().save(new PlanItemSaveRequest(c1, UUID.randomUUID().toString(), "c",
        PlanItemStatus.COMPLETED, Instant.now(), TargetType.HUMAN_TASK, null));

    assertThat(store().findAllDelegated()).hasSize(2);
  }

  @Test
  void save_storesTargetTypeAndExpression() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store().save(new PlanItemSaveRequest(caseId, planItemId, "binding",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, ".result.value"));

    List<PlanItemRecord> results = store().findByCaseId(caseId);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).targetType()).isEqualTo(TargetType.HUMAN_TASK);
    assertThat(results.get(0).outputMappingExpression()).isEqualTo(".result.value");
  }
}
```

- [ ] **Step 2: Run contract tests via MemoryPlanItemStore (already implements the contract)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl persistence-memory -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL|ERROR" | tail -15
```

Expected: all tests pass. Find the concrete test class that extends `PlanItemStoreContractTest` in `persistence-memory` first:

```bash
find /Users/mdproctor/claude/casehub/engine/persistence-memory/src/test -name "*PlanItemStore*"
```

If no concrete class exists yet, skip running tests until Task 5.

- [ ] **Step 3: Add ReactivePlanItemStoreContractTest to common** (spec requirement)

Check if `common/src/test/java/io/casehub/engine/common/spi/ReactivePlanItemStoreContractTest.java` exists. If not, create it:

```java
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.internal.model.TargetType;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/** Abstract contract test for ReactivePlanItemStore — extend with a concrete impl. */
public abstract class ReactivePlanItemStoreContractTest {

  protected abstract ReactivePlanItemStore store();

  private static final Duration TIMEOUT = Duration.ofSeconds(5);

  @Test
  void findDelegated_returnsOnlyDelegatedForCase() {
    UUID caseId = UUID.randomUUID();
    String piDelegated = UUID.randomUUID().toString();
    String piPending = UUID.randomUUID().toString();

    store().save(new PlanItemSaveRequest(caseId, piDelegated, "b1",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, ".result.x"))
        .await().atMost(TIMEOUT);
    store().save(new PlanItemSaveRequest(caseId, piPending, "b2",
        PlanItemStatus.PENDING, Instant.now(), null, null))
        .await().atMost(TIMEOUT);

    List<PlanItemRecord> result = store().findDelegated(caseId).await().atMost(TIMEOUT);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).planItemId()).isEqualTo(piDelegated);
    assertThat(result.get(0).targetType()).isEqualTo(TargetType.HUMAN_TASK);
    assertThat(result.get(0).outputMappingExpression()).isEqualTo(".result.x");
  }

  @Test
  void findAllDelegated_returnsAcrossAllCases() {
    UUID c1 = UUID.randomUUID();
    UUID c2 = UUID.randomUUID();
    store().save(new PlanItemSaveRequest(c1, UUID.randomUUID().toString(), "a",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, null))
        .await().atMost(TIMEOUT);
    store().save(new PlanItemSaveRequest(c2, UUID.randomUUID().toString(), "b",
        PlanItemStatus.DELEGATED, Instant.now(), TargetType.HUMAN_TASK, null))
        .await().atMost(TIMEOUT);

    assertThat(store().findAllDelegated().await().atMost(TIMEOUT)).hasSize(2);
  }
}
```

Then create `persistence-memory/src/test/java/io/casehub/persistence/memory/MemoryReactivePlanItemStoreContractTest.java`:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.common.spi.ReactivePlanItemStore;
import io.casehub.engine.common.spi.ReactivePlanItemStoreContractTest;

class MemoryReactivePlanItemStoreContractTest extends ReactivePlanItemStoreContractTest {

  private final MemoryPlanItemStore backing = new MemoryPlanItemStore();
  private final MemoryReactivePlanItemStore store = new MemoryReactivePlanItemStore();

  public MemoryReactivePlanItemStoreContractTest() {
    store.delegate = backing;
  }

  @Override
  protected ReactivePlanItemStore store() {
    return store;
  }
}
```

Note: `MemoryReactivePlanItemStore.delegate` is `@Inject` — set it reflectively or make it package-scoped for tests, or restructure using a constructor. If the field injection causes issues, set it via reflection in the test.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/
git -C /Users/mdproctor/claude/casehub/engine commit -m "test: update PlanItemStoreContractTest — use PlanItemSaveRequest, add findDelegated/findAllDelegated/targetType contract tests; add ReactivePlanItemStoreContractTest

Refs #274, #398"
```

---

### Task 5: Update JPA entities and JPA store implementations

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`

- [ ] **Step 1: Add targetType + outputMappingExpression to WorkAdapterPlanItemEntity**

Add these two fields to the entity (after `createdAt`):

```java
// Add import at top of file:
import io.casehub.engine.common.internal.model.TargetType;

// Add these two fields to the entity class body:
@Enumerated(EnumType.STRING)
@Column(name = "target_type", length = 20)
public TargetType targetType;

@Column(name = "output_mapping_expression", length = 1000)
public String outputMappingExpression;
```

- [ ] **Step 2: Add same two fields to PlanItemEntity in persistence-hibernate**

Add import and fields identically:

```java
// Add import:
import io.casehub.engine.common.internal.model.TargetType;

// Add fields after createdAt:
@Enumerated(EnumType.STRING)
@Column(name = "target_type", length = 20)
public TargetType targetType;

@Column(name = "output_mapping_expression", length = 1000)
public String outputMappingExpression;
```

- [ ] **Step 3: Update JpaPlanItemStore — full replacement**

Replace the entire `JpaPlanItemStore.java`:

```java
package io.casehub.workadapter;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.spi.PlanItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

/**
 * Blocking JPA PlanItemStore for use in the work-adapter context.
 *
 * <p>Uses the same blocking persistence unit as casehub-work, so writes participate in the same JTA
 * transaction as WorkItemService. This is the key atomicity guarantee: planItemStore.updateStatus()
 * and workItemService.create() either both commit or both roll back.
 */
@ApplicationScoped
public class JpaPlanItemStore implements PlanItemStore {

  @Inject EntityManager em;

  @Override
  public void save(PlanItemSaveRequest request) {
    WorkAdapterPlanItemEntity e = new WorkAdapterPlanItemEntity();
    e.caseId = request.caseId();
    e.planItemId = request.planItemId();
    e.bindingName = request.bindingName();
    e.status = request.status();
    e.createdAt = request.createdAt();
    e.targetType = request.targetType();
    e.outputMappingExpression = request.outputMappingExpression();
    em.persist(e);
  }

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {
    em.flush();
    em.createQuery(
            "UPDATE WorkAdapterPlanItemEntity e SET e.status = :status WHERE e.planItemId = :planItemId")
        .setParameter("status", status)
        .setParameter("planItemId", planItemId)
        .executeUpdate();
    em.clear();
  }

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return em.createQuery(
            "SELECT e FROM WorkAdapterPlanItemEntity e WHERE e.caseId = :caseId",
            WorkAdapterPlanItemEntity.class)
        .setParameter("caseId", caseId)
        .getResultList()
        .stream()
        .map(this::toRecord)
        .collect(Collectors.toList());
  }

  @Override
  public List<PlanItemRecord> findDelegated(UUID caseId) {
    return em.createQuery(
            "SELECT e FROM WorkAdapterPlanItemEntity e WHERE e.caseId = :caseId AND e.status = :status",
            WorkAdapterPlanItemEntity.class)
        .setParameter("caseId", caseId)
        .setParameter("status", PlanItemStatus.DELEGATED)
        .getResultList()
        .stream()
        .map(this::toRecord)
        .collect(Collectors.toList());
  }

  @Override
  public List<PlanItemRecord> findAllDelegated() {
    return em.createQuery(
            "SELECT e FROM WorkAdapterPlanItemEntity e WHERE e.status = :status",
            WorkAdapterPlanItemEntity.class)
        .setParameter("status", PlanItemStatus.DELEGATED)
        .getResultList()
        .stream()
        .map(this::toRecord)
        .collect(Collectors.toList());
  }

  private PlanItemRecord toRecord(WorkAdapterPlanItemEntity e) {
    return new PlanItemRecord(
        e.caseId, e.planItemId, e.bindingName, e.status, e.createdAt,
        e.targetType, e.outputMappingExpression);
  }
}
```

- [ ] **Step 4: Update JpaReactivePlanItemStore — full replacement**

Replace `JpaReactivePlanItemStore.java`:

```java
package io.casehub.persistence.jpa;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.spi.ReactivePlanItemStore;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.quarkus.panache.common.Parameters;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class JpaReactivePlanItemStore extends AbstractJpaRepository
    implements ReactivePlanItemStore {

  @Override
  public Uni<Void> save(PlanItemSaveRequest request) {
    return withSafeContext(
        () ->
            Panache.withTransaction(
                () -> {
                  PlanItemEntity e = new PlanItemEntity();
                  e.caseId = request.caseId();
                  e.planItemId = request.planItemId();
                  e.bindingName = request.bindingName();
                  e.status = request.status();
                  e.createdAt = request.createdAt();
                  e.targetType = request.targetType();
                  e.outputMappingExpression = request.outputMappingExpression();
                  return e.persist().replaceWithVoid();
                }));
  }

  @Override
  public Uni<Void> updateStatus(String planItemId, PlanItemStatus status) {
    return withSafeContext(
        () ->
            Panache.withTransaction(
                () ->
                    PlanItemEntity.getSession()
                        .chain(session -> session.flush())
                        .chain(
                            () ->
                                PlanItemEntity.update(
                                    "status = :status WHERE planItemId = :planItemId",
                                    Parameters.with("status", status)
                                        .and("planItemId", planItemId)))
                        .replaceWithVoid()));
  }

  @Override
  public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId) {
    return withSafeContext(
        () ->
            Panache.withSession(
                () ->
                    PlanItemEntity.<PlanItemEntity>find("caseId", caseId)
                        .list()
                        .map(list -> list.stream().map(this::toRecord).collect(Collectors.toList()))));
  }

  @Override
  public Uni<List<PlanItemRecord>> findDelegated(UUID caseId) {
    return withSafeContext(
        () ->
            Panache.withSession(
                () ->
                    PlanItemEntity.<PlanItemEntity>find(
                            "caseId = ?1 AND status = ?2", caseId, PlanItemStatus.DELEGATED)
                        .list()
                        .map(list -> list.stream().map(this::toRecord).collect(Collectors.toList()))));
  }

  @Override
  public Uni<List<PlanItemRecord>> findAllDelegated() {
    return withSafeContext(
        () ->
            Panache.withSession(
                () ->
                    PlanItemEntity.<PlanItemEntity>find("status", PlanItemStatus.DELEGATED)
                        .list()
                        .map(list -> list.stream().map(this::toRecord).collect(Collectors.toList()))));
  }

  private PlanItemRecord toRecord(PlanItemEntity e) {
    return new PlanItemRecord(
        e.caseId, e.planItemId, e.bindingName, e.status, e.createdAt,
        e.targetType, e.outputMappingExpression);
  }
}
```

- [ ] **Step 5: Build all modules**

```bash
mvn --batch-mode install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/ persistence-hibernate/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: add target_type/output_mapping_expression columns to JPA entities; implement findDelegated/findAllDelegated in JPA stores

Refs #274, #398"
```

---

### Task 6: Update HumanTaskScheduleHandler save() call sites

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java`

- [ ] **Step 1: Add import and extraction helper**

Add to the imports section:
```java
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.TargetType;
```

Add this private static method at the bottom of the class (before the closing `}`):

```java
private static String extractOutputMappingExpression(HumanTaskTarget target) {
  if (target == null || target.outputMapping() == null) return null;
  if (target.outputMapping() instanceof JQExpressionEvaluator jq) return jq.expression();
  return null;
}
```

- [ ] **Step 2: Update inline mode save() call**

Find in `handleInlineMode()` (around line 150):
```java
planItemStore.save(
    event.caseId(),
    item.getPlanItemId(),
    item.getBindingName(),
    PlanItemStatus.DELEGATED,
    item.getCreatedAt());
```

Replace with:
```java
planItemStore.save(
    new PlanItemSaveRequest(
        event.caseId(),
        item.getPlanItemId(),
        item.getBindingName(),
        PlanItemStatus.DELEGATED,
        item.getCreatedAt(),
        TargetType.HUMAN_TASK,
        extractOutputMappingExpression(event.target())));
```

- [ ] **Step 3: Update template mode save() call**

Find in `handleTemplateMode()` (around line 137):
```java
planItemStore.save(
    event.caseId(),
    item.getPlanItemId(),
    item.getBindingName(),
    PlanItemStatus.DELEGATED,
    item.getCreatedAt());
```

Replace with:
```java
planItemStore.save(
    new PlanItemSaveRequest(
        event.caseId(),
        item.getPlanItemId(),
        item.getBindingName(),
        PlanItemStatus.DELEGATED,
        item.getCreatedAt(),
        TargetType.HUMAN_TASK,
        extractOutputMappingExpression(event.target())));
```

- [ ] **Step 4: Build and run work-adapter tests**

```bash
mvn --batch-mode install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL|ERROR" | tail -15
```

Expected: `BUILD SUCCESS`, existing tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: update HumanTaskScheduleHandler to store targetType and outputMappingExpression via PlanItemSaveRequest

Refs #274, #398"
```

---

### Task 7: Add PlanItem.restore() and DefaultCasePlanModel.restorePlanItem()

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java`

- [ ] **Step 1: Write failing test for PlanItem.restore()**

Add to `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java`:

```java
@Test
void restore_createsPlanItemWithGivenStatusAndId() {
    String planItemId = UUID.randomUUID().toString();
    PlanItem item = PlanItem.restore(planItemId, "my-binding", null, PlanItemStatus.DELEGATED, Instant.now());
    assertThat(item.getPlanItemId()).isEqualTo(planItemId);
    assertThat(item.getBindingName()).isEqualTo("my-binding");
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.DELEGATED);
    assertThat(item.getTarget()).isNull();
}

@Test
void restore_rejectsInvalidStatus() {
    assertThatThrownBy(() ->
        PlanItem.restore(UUID.randomUUID().toString(), "b", null, PlanItemStatus.PENDING, Instant.now()))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("PENDING");
}
```

Add import at the top of `PlanItemTest.java`:
```java
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import java.time.Instant;
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=PlanItemTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR" | tail -5
```

Expected: compilation failure (method not found).

- [ ] **Step 3: Add private constructor and restore() to PlanItem**

Add this new private constructor (after the existing private constructor):

```java
/** For restoration from persistent store. Allows setting a specific planItemId and status. */
private PlanItem(
    String planItemId,
    String bindingName,
    BindingTarget target,
    PlanItemStatus status,
    Instant createdAt) {
  this.planItemId = planItemId;
  this.bindingName = bindingName;
  this.workerName = null;
  this.priority = 0;
  this.target = target;
  this.status = status;
  this.createdAt = createdAt;
  this.parentStageId = null;
}
```

Add the static factory after the existing `create()` methods:

```java
/**
 * Restores a PlanItem from persistent store after a JVM restart.
 *
 * <p>Only RUNNING and DELEGATED items are valid for restoration. PENDING items are re-created by
 * evaluation; terminal items must not re-enter the live plan.
 *
 * @throws IllegalArgumentException if status is not RUNNING or DELEGATED
 */
public static PlanItem restore(
    String planItemId,
    String bindingName,
    BindingTarget target,
    PlanItemStatus status,
    Instant createdAt) {
  if (status != PlanItemStatus.RUNNING && status != PlanItemStatus.DELEGATED) {
    throw new IllegalArgumentException(
        "restore() only valid for RUNNING or DELEGATED status, got: " + status);
  }
  return new PlanItem(planItemId, bindingName, target, status, createdAt);
}
```

- [ ] **Step 4: Run test to confirm it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=PlanItemTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|BUILD" | tail -5
```

Expected: `Tests run: N, Failures: 0, Errors: 0`

- [ ] **Step 5: Write failing test for DefaultCasePlanModel.restorePlanItem()**

Add to `blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java`:

```java
@Test
void restorePlanItem_makesItemFindableByIdAndBinding() {
    UUID caseId = UUID.randomUUID();
    DefaultCasePlanModel model = new DefaultCasePlanModel(caseId);
    String planItemId = UUID.randomUUID().toString();
    PlanItem item = PlanItem.restore(planItemId, "task-binding", null, PlanItemStatus.DELEGATED, Instant.now());

    model.restorePlanItem(item);

    assertThat(model.getPlanItem(planItemId)).contains(item);
    assertThat(model.getPlanItemByBindingName("task-binding")).contains(item);
}

@Test
void restorePlanItem_doesNotAddToAgenda() {
    UUID caseId = UUID.randomUUID();
    DefaultCasePlanModel model = new DefaultCasePlanModel(caseId);
    PlanItem item = PlanItem.restore(UUID.randomUUID().toString(), "task", null, PlanItemStatus.DELEGATED, Instant.now());

    model.restorePlanItem(item);

    // agenda only shows PENDING items
    assertThat(model.getAgenda()).isEmpty();
}
```

Add import at the top of `DefaultCasePlanModelTest.java`:
```java
import java.time.Instant;
```

- [ ] **Step 6: Run test to confirm it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=DefaultCasePlanModelTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|BUILD" | tail -5
```

Expected: compilation failure (method not found).

- [ ] **Step 7: Add restorePlanItem() to DefaultCasePlanModel**

Add after the `addPlanItemIfAbsent()` method:

```java
/**
 * Restores a PlanItem from persistent store into the live plan after a JVM restart.
 *
 * <p>Adds the item to itemsById and activeByBinding so completion handlers can find it, but
 * does NOT add it to the agenda — restored items are not pending dispatch.
 */
public void restorePlanItem(PlanItem item) {
  itemsById.put(item.getPlanItemId(), item);
  activeByBinding.put(item.getBindingName(), item);
}
```

- [ ] **Step 8: Run test to confirm it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=DefaultCasePlanModelTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|BUILD" | tail -5
```

Expected: `Tests run: N, Failures: 0, Errors: 0`

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: add PlanItem.restore() static factory with status validation; add DefaultCasePlanModel.restorePlanItem() for post-restart registry population

Refs #274"
```

---

### Task 8: Create PlanItemRestorer + lazy hydration in BlackboardRegistry (TDD)

**Files:**
- Create: `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryLazyHydrationTest.java`
- Create: `blackboard/src/main/java/io/casehub/blackboard/registry/PlanItemRestorer.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`

- [ ] **Step 1: Write failing integration test**

Create `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryLazyHydrationTest.java`:

```java
package io.casehub.blackboard.registry;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.HumanTaskTarget;
import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.persistence.memory.MemoryPlanItemStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies that BlackboardRegistry.get() lazily hydrates DELEGATED PlanItems from PlanItemStore
 * on first miss after restart. Uses MemoryPlanItemStore as a stand-in for the real store.
 */
@QuarkusTest
class BlackboardRegistryLazyHydrationTest {

  @Inject BlackboardRegistry registry;
  @Inject PlanItemStore planItemStore;

  @BeforeEach
  void setUp() {
    if (planItemStore instanceof MemoryPlanItemStore mem) {
      mem.clear();
    }
  }

  @Test
  void get_returnsEmptyWhenStoreHasNoRecordsForCase() {
    UUID caseId = UUID.randomUUID();
    assertThat(registry.get(caseId)).isEmpty();
  }

  @Test
  void get_hydratesDelegatedPlanItemFromStore() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();

    planItemStore.save(new PlanItemSaveRequest(
        caseId, planItemId, "review-binding",
        PlanItemStatus.DELEGATED, Instant.now(),
        TargetType.HUMAN_TASK, ".result.decision"));

    Optional<CasePlanModel> result = registry.get(caseId);

    assertThat(result).isPresent();
    assertThat(result.get().getPlanItem(planItemId)).isPresent();
    PlanItem item = result.get().getPlanItem(planItemId).get();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.DELEGATED);
    assertThat(item.getBindingName()).isEqualTo("review-binding");
    assertThat(item.getTarget()).isInstanceOf(HumanTaskTarget.class);
    HumanTaskTarget ht = (HumanTaskTarget) item.getTarget();
    assertThat(ht.outputMapping()).isNotNull();
  }

  @Test
  void get_hydratesDelegatedPlanItemWithNullExpression() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();

    planItemStore.save(new PlanItemSaveRequest(
        caseId, planItemId, "approve-binding",
        PlanItemStatus.DELEGATED, Instant.now(),
        TargetType.HUMAN_TASK, null));

    Optional<CasePlanModel> result = registry.get(caseId);

    assertThat(result).isPresent();
    PlanItem item = result.get().getPlanItem(planItemId).get();
    assertThat(item.getTarget()).isInstanceOf(HumanTaskTarget.class);
    HumanTaskTarget ht = (HumanTaskTarget) item.getTarget();
    assertThat(ht.outputMapping()).isNull();
  }

  @Test
  void get_onlyHydratesDelegatedNotPendingOrCompleted() {
    UUID caseId = UUID.randomUUID();

    planItemStore.save(new PlanItemSaveRequest(
        caseId, UUID.randomUUID().toString(), "pending-binding",
        PlanItemStatus.PENDING, Instant.now(), null, null));
    planItemStore.save(new PlanItemSaveRequest(
        caseId, UUID.randomUUID().toString(), "completed-binding",
        PlanItemStatus.COMPLETED, Instant.now(), TargetType.HUMAN_TASK, null));

    Optional<CasePlanModel> result = registry.get(caseId);

    // Only DELEGATED items are hydrated — no DELEGATED items means empty registry
    assertThat(result).isEmpty();
  }
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=BlackboardRegistryLazyHydrationTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: test `get_hydratesDelegatedPlanItemFromStore` fails (registry returns empty).

- [ ] **Step 3: Create PlanItemRestorer (package-private, same package as BlackboardRegistry)**

Create `blackboard/src/main/java/io/casehub/blackboard/registry/PlanItemRestorer.java`:

```java
package io.casehub.blackboard.registry;

import io.casehub.api.model.HumanTaskTarget;
import io.casehub.api.model.BindingTarget;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.TargetType;

/**
 * Package-private utility — converts PlanItemRecord to PlanItem during registry hydration.
 * Keeps BlackboardRegistry free of HumanTaskTarget and JQExpressionEvaluator imports.
 */
class PlanItemRestorer {

  PlanItem restore(PlanItemRecord r) {
    BindingTarget target =
        r.targetType() == TargetType.HUMAN_TASK ? buildHumanTaskTarget(r.outputMappingExpression()) : null;
    return PlanItem.restore(r.planItemId(), r.bindingName(), target, r.status(), r.createdAt());
  }

  private HumanTaskTarget buildHumanTaskTarget(String expr) {
    HumanTaskTarget.Builder b = HumanTaskTarget.inline();
    if (expr != null) {
      b = b.outputMapping(expr);
    }
    return b.build();
  }
}
```

- [ ] **Step 4: Update BlackboardRegistry with lazy hydration**

Replace the entire `BlackboardRegistry.java`:

```java
package io.casehub.blackboard.registry;

import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.plan.DefaultCasePlanModel;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Shared registry of per-case CasePlanModel instances and the worker-name-to-PlanItemId
 * completion index.
 *
 * <p>All per-case state is co-located in a single CaseEntry, making eviction atomic — one map
 * removal instead of three. See casehubio/engine#292.
 *
 * <p>On first get() miss after a JVM restart, DELEGATED PlanItems are lazily restored from
 * PlanItemStore so completion handlers can find their PlanItems without any startup ordering
 * constraint. RUNNING items and completionIndex are not persisted — Quartz-only case recovery
 * is a separate concern. See casehubio/engine#274.
 */
@ApplicationScoped
public class BlackboardRegistry {

  private static final class CaseEntry {
    final CasePlanModel planModel;
    final ConcurrentHashMap<String, String> completionIndex = new ConcurrentHashMap<>();
    final AtomicBoolean configured = new AtomicBoolean(false);

    CaseEntry(UUID caseId) {
      this.planModel = new DefaultCasePlanModel(caseId);
    }
  }

  private final ConcurrentHashMap<UUID, CaseEntry> entries = new ConcurrentHashMap<>();
  private final PlanItemRestorer restorer = new PlanItemRestorer();

  @Inject PlanItemStore planItemStore;

  /**
   * Returns the CasePlanModel for the given case, creating it if absent. Only
   * PlanningStrategyLoopControl should call this method — all other components should use get().
   */
  public CasePlanModel getOrCreate(UUID caseId) {
    return entries.computeIfAbsent(caseId, CaseEntry::new).planModel;
  }

  /**
   * Returns the CasePlanModel for the given case, or empty if absent.
   *
   * <p>On miss: queries PlanItemStore for DELEGATED records and hydrates the registry before
   * returning. Two concurrent misses for the same case each query the store; the loser finds the
   * entry already populated and re-applies restorePlanItem() (idempotent — same planItemId, same
   * data). The cost is at most one duplicate DB call per concurrent miss, acceptable post-restart.
   *
   * <p>If planItemStore is not available (null in unit tests), returns empty immediately.
   */
  public Optional<CasePlanModel> get(UUID caseId) {
    CaseEntry e = entries.get(caseId);
    if (e != null) return Optional.of(e.planModel);

    if (planItemStore == null) return Optional.empty();

    List<PlanItemRecord> records = planItemStore.findDelegated(caseId);
    if (records.isEmpty()) return Optional.empty();

    CaseEntry hydrated = entries.computeIfAbsent(caseId, CaseEntry::new);
    records.forEach(r -> hydrated.planModel.restorePlanItem(restorer.restore(r)));
    return Optional.of(hydrated.planModel);
  }

  public void indexForCompletion(UUID caseId, String workerName, String planItemId) {
    CaseEntry e = entries.get(caseId);
    if (e != null) {
      e.completionIndex.put(workerName, planItemId);
    }
  }

  public Optional<String> getPlanItemId(UUID caseId, String workerName) {
    CaseEntry e = entries.get(caseId);
    return e == null ? Optional.empty() : Optional.ofNullable(e.completionIndex.get(workerName));
  }

  /**
   * Atomically marks a case as configured by BlackboardPlanConfigurer(s). Returns true only the
   * first time this method is called for the given case. This guarantees configurers are invoked
   * exactly once per case instance.
   *
   * <p>Contract: BlackboardPlanConfigurer implementations must be idempotent with respect to
   * pre-populated plan items — addPlanItemIfAbsent() correctly rejects re-dispatch of DELEGATED
   * bindings already in activeByBinding.
   */
  public boolean markConfigured(UUID caseId) {
    CaseEntry e = entries.get(caseId);
    return e != null && e.configured.compareAndSet(false, true);
  }

  /**
   * Atomically evicts the plan model, completion index, and configured marker for a completed or
   * terminated case. See casehubio/engine#84.
   */
  public void evict(UUID caseId) {
    entries.remove(caseId);
  }
}
```

- [ ] **Step 5: Run test to confirm it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -Dtest=BlackboardRegistryLazyHydrationTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: `Tests run: 4, Failures: 0, Errors: 0`

- [ ] **Step 6: Run all blackboard tests to catch regressions**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -15
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: BlackboardRegistry lazy get() — hydrates DELEGATED PlanItems from PlanItemStore on first miss after restart; PlanItemRestorer converts records to PlanItems

Closes #274"
```

---

### Task 9: Fix four blocking @ConsumeEvent handlers

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemFaultHandler.java`

`BlackboardRegistry.get()` now makes a blocking JDBC call on first miss. All three handlers call `registry.get()` from non-blocking Vert.x IO threads. Add `blocking = true` and change return type to `void`.

- [ ] **Step 1: Update PlanItemCompletionHandler**

Change both `@ConsumeEvent` methods and the private helper. Find and replace:

```java
// BEFORE: onWorkerFinished
@ConsumeEvent(EventBusAddresses.WORKER_EXECUTION_FINISHED)
public Uni<Void> onWorkerFinished(WorkflowExecutionCompleted event) {
  return completePlanItemByKey(event.caseInstance().getUuid(), event.worker().getName());
}

// AFTER:
@ConsumeEvent(value = EventBusAddresses.WORKER_EXECUTION_FINISHED, blocking = true)
public void onWorkerFinished(WorkflowExecutionCompleted event) {
  completePlanItemByKey(event.caseInstance().getUuid(), event.worker().getName());
}
```

```java
// BEFORE: onSubCaseFinished
@ConsumeEvent(BlackboardEventBusAddresses.SUBCASE_EXECUTION_COMPLETED)
public Uni<Void> onSubCaseFinished(SubCaseExecutionCompleted event) {
  return completePlanItemByKey(event.parentCaseId(), event.childCaseId().toString());
}

// AFTER:
@ConsumeEvent(value = BlackboardEventBusAddresses.SUBCASE_EXECUTION_COMPLETED, blocking = true)
public void onSubCaseFinished(SubCaseExecutionCompleted event) {
  completePlanItemByKey(event.parentCaseId(), event.childCaseId().toString());
}
```

```java
// BEFORE: private completePlanItemByKey
private Uni<Void> completePlanItemByKey(UUID caseId, String trackingKey) {
  CasePlanModel plan = registry.get(caseId).orElse(null);
  if (plan == null) return Uni.createFrom().voidItem();
  // ... (body unchanged up to end)
  return Uni.createFrom().voidItem();
}

// AFTER: (change signature, remove returns)
private void completePlanItemByKey(UUID caseId, String trackingKey) {
  CasePlanModel plan = registry.get(caseId).orElse(null);
  if (plan == null) return;

  String planItemId = registry.getPlanItemId(caseId, trackingKey).orElse(null);
  if (planItemId == null) {
    LOG.debugf(
        "No PlanItem indexed for key '%s' in case %s — pure choreography or already evicted",
        trackingKey, caseId);
    return;
  }

  plan.getPlanItem(planItemId)
      .ifPresent(
          item -> {
            if (!COMPLETABLE.contains(item.getStatus())) {
              LOG.debugf(
                  "PlanItem %s for key '%s' in case %s has status %s — not completable, skipping",
                  planItemId, trackingKey, caseId, item.getStatus());
              return;
            }
            item.markCompleted();
            stageAutocompleteEvaluator.evaluate(caseId, plan, planItemId);
            planItemCompletedEvents.fireAsync(
                new PlanItemCompletedEvent(caseId, planItemId, trackingKey));
          });
}
```

Remove the `import io.smallrye.mutiny.Uni;` import if it's no longer used after this change.

- [ ] **Step 2: Update WorkerRetryExhaustionHandler**

Change `@ConsumeEvent` method signature:

```java
// BEFORE:
@ConsumeEvent(EventBusAddresses.WORKER_RETRIES_EXHAUSTED)
public Uni<Void> onWorkerRetriesExhausted(final WorkerRetriesExhaustedEvent event) {
  final CasePlanModel plan = registry.get(event.caseId()).orElse(null);
  if (plan == null) return Uni.createFrom().voidItem();
  // ...
  return Uni.createFrom().voidItem();
}

// AFTER:
@ConsumeEvent(value = EventBusAddresses.WORKER_RETRIES_EXHAUSTED, blocking = true)
public void onWorkerRetriesExhausted(final WorkerRetriesExhaustedEvent event) {
  final CasePlanModel plan = registry.get(event.caseId()).orElse(null);
  if (plan == null) return;

  final String planItemId = registry.getPlanItemId(event.caseId(), event.workerId()).orElse(null);
  if (planItemId == null) {
    LOG.debugf(
        "No PlanItem indexed for worker '%s' in case %s — guard-blocked or already evicted",
        event.workerId(), event.caseId());
    return;
  }

  plan.getPlanItem(planItemId)
      .ifPresent(
          item -> {
            if (item.getStatus() != PlanItemStatus.RUNNING) {
              LOG.debugf(
                  "PlanItem %s for worker '%s' in case %s has status %s — not RUNNING, skipping",
                  planItemId, event.workerId(), event.caseId(), item.getStatus());
              return;
            }
            item.markFaulted();
            stageAutocompleteEvaluator.evaluate(event.caseId(), plan, planItemId);
            LOG.warnf(
                "PlanItem %s marked FAULTED — worker '%s' retries exhausted in case %s",
                planItemId, event.workerId(), event.caseId());
          });
}
```

Remove `import io.smallrye.mutiny.Uni;` if no longer used.

- [ ] **Step 3: Update PlanItemFaultHandler**

Apply the same pattern to `PlanItemFaultHandler.onWorkerRetriesExhausted()` — add `blocking = true`, change `Uni<Void>` → `void`, remove `return Uni.createFrom().voidItem()` statements, change `if (plan == null) return ...` to `if (plan == null) return;`. Read the full file first to get exact content:

```bash
cat /Users/mdproctor/claude/casehub/engine/blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemFaultHandler.java
```

Apply the same `blocking = true` + `void` transformation to the `@ConsumeEvent` handler method.

- [ ] **Step 4: Build and run blackboard tests**

```bash
mvn --batch-mode install -DskipTests -pl blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -15
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix: add blocking=true to PlanItemCompletionHandler, WorkerRetryExhaustionHandler, PlanItemFaultHandler — required because BlackboardRegistry.get() now makes a blocking JDBC call on post-restart cache miss

Refs #274"
```

---

### Task 10: Add null-guard to WorkItemLifecycleAdapter.applyOutputMapping()

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`

- [ ] **Step 1: Add null-guard at the top of applyOutputMapping()**

Find the `applyOutputMapping` method. After the null-check for `instance.getCaseContext()`, add:

```java
// Add this block immediately after the caseContext null-check:
if (item.getTarget() == null) {
  LOG.warnf(
      "PlanItem %s has no target (recovered without target info) — outputMapping skipped",
      item.getPlanItemId());
  return;
}
```

The method should look like this at the start:

```java
private void applyOutputMapping(PlanItem item, WorkItem workItem, CaseInstance instance) {
  if (instance.getCaseContext() == null) {
    LOG.warnf(
        "CaseInstance %s has no CaseContext — outputMapping skipped for PlanItem %s",
        instance.getUuid(), item.getPlanItemId());
    return;
  }
  if (item.getTarget() == null) {
    LOG.warnf(
        "PlanItem %s has no target (recovered without target info) — outputMapping skipped",
        item.getPlanItemId());
    return;
  }
  HumanTaskTarget ht =
      switch (item.getTarget()) {
        ...
      };
```

- [ ] **Step 2: Build and run work-adapter tests**

```bash
mvn --batch-mode install -DskipTests -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix: add null-guard to WorkItemLifecycleAdapter.applyOutputMapping() — PlanItems restored without target info skip outputMapping gracefully

Refs #274"
```

---

### Task 11: Extract PlanItemCompletionApplier + refactor WorkItemLifecycleAdapter

**Files:**
- Create: `work-adapter/src/main/java/io/casehub/workadapter/PlanItemCompletionApplier.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`

- [ ] **Step 1: Create PlanItemCompletionApplier**

Create `work-adapter/src/main/java/io/casehub/workadapter/PlanItemCompletionApplier.java`:

```java
package io.casehub.workadapter;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.HumanTaskTarget;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.common.internal.event.CaseContextChangedEvent;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.jq.JQEvaluator;
import io.casehub.engine.common.internal.jq.ValidationResult;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Applies a terminal WorkItemStatus to a PlanItem and fires CONTEXT_CHANGED.
 *
 * <p>Shared between WorkItemLifecycleAdapter (normal flow) and HumanTaskRecoveryService
 * (startup catch-up). Declares @Transactional — REQUIRED semantics means the transaction
 * propagates from callers that already have one, and a new one is opened when called without.
 */
@ApplicationScoped
class PlanItemCompletionApplier {

  private static final Logger LOG = Logger.getLogger(PlanItemCompletionApplier.class);
  private static final Duration TIMEOUT = Duration.ofSeconds(5);
  private static final ObjectMapper MAPPER = new ObjectMapper();
  private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};

  @Inject BlackboardRegistry registry;
  @Inject CaseInstanceRepository caseInstanceRepository;
  @Inject EventBus eventBus;
  @Inject JQEvaluator jqEvaluator;

  /**
   * Applies the terminal WorkItemStatus to the PlanItem, runs outputMapping if configured,
   * loads the CaseInstance, and publishes CONTEXT_CHANGED.
   *
   * <p>If the PlanItem is already terminal, logs DEBUG and returns — idempotent for recovery.
   *
   * @param caseId the case containing the PlanItem
   * @param planItemId the PlanItem to transition
   * @param status the terminal WorkItemStatus to apply
   * @param workItem the source WorkItem (for outputMapping resolution JSON)
   */
  @Transactional
  public void apply(UUID caseId, String planItemId, WorkItemStatus status, WorkItem workItem) {
    PlanItem item =
        registry.get(caseId).flatMap(plan -> plan.getPlanItem(planItemId)).orElse(null);

    if (item == null) {
      LOG.warnf("PlanItem %s not found in case %s — completion not applied", planItemId, caseId);
      return;
    }

    if (!applyStatus(item, status)) {
      return; // already terminal or invalid transition — idempotent skip
    }

    CaseInstance instance =
        caseInstanceRepository.findByUuid(caseId).await().atMost(TIMEOUT);
    if (instance == null) {
      LOG.warnf("CaseInstance not found for caseId=%s — CONTEXT_CHANGED not fired", caseId);
      return;
    }

    applyOutputMapping(item, workItem, instance);
    eventBus.publish(
        EventBusAddresses.CONTEXT_CHANGED,
        new CaseContextChangedEvent(instance, instance.getCaseContext().asJsonNode()));
  }

  private boolean applyStatus(PlanItem item, WorkItemStatus status) {
    try {
      switch (status) {
        case COMPLETED -> item.markCompleted();
        case REJECTED -> item.markRejected();
        case EXPIRED -> item.markFaulted();
        case CANCELLED -> item.markCancelled();
        default -> {
          return false;
        }
      }
      return true;
    } catch (IllegalStateException e) {
      LOG.debugf(
          "PlanItem %s already terminal (status=%s) — skipping for WorkItemStatus %s",
          item.getPlanItemId(), item.getStatus(), status);
      return false;
    }
  }

  private void applyOutputMapping(PlanItem item, WorkItem workItem, CaseInstance instance) {
    if (instance.getCaseContext() == null) return;
    if (item.getTarget() == null) return;
    if (!(item.getTarget() instanceof HumanTaskTarget ht)) return;
    if (ht.outputMapping() == null) return;
    if (workItem == null || workItem.resolution == null) return;

    if (!(ht.outputMapping() instanceof JQExpressionEvaluator jq)) {
      LOG.warnf(
          "Unsupported outputMapping evaluator type '%s' for PlanItem %s — skipping",
          ht.outputMapping().getClass().getName(), item.getPlanItemId());
      return;
    }

    try {
      JsonNode resolutionNode = MAPPER.readTree(workItem.resolution);
      ValidationResult vr = jqEvaluator.eval(jq.expression(), resolutionNode);
      if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) {
        LOG.warnf(
            "outputMapping jq expression returned no result for PlanItem %s: %s",
            item.getPlanItemId(), vr.error());
        return;
      }
      List<JsonNode> output = vr.output();
      Map<String, Object> updates = MAPPER.convertValue(output.get(0), MAP_TYPE);
      instance.getCaseContext().setAll(updates);
    } catch (Exception e) {
      LOG.warnf(
          e,
          "outputMapping failed for PlanItem %s — CONTEXT_CHANGED fires without output update",
          item.getPlanItemId());
    }
  }
}
```

- [ ] **Step 2: Refactor WorkItemLifecycleAdapter to inject and use PlanItemCompletionApplier**

Add `@Inject PlanItemCompletionApplier applier;` field.

Replace the `onWorkItemLifecycle()` body where it currently calls `applyStatus()`, `applyOutputMapping()`, loads CaseInstance, and fires CONTEXT_CHANGED — delegate to `applier.apply()`:

```java
// After finding CallerRef ref:
CallerRef ref = CallerRef.parse(workItem.callerRef);
if (ref == null) return;

// Replace the rest of the method body (registry.get → applyStatus → applyOutputMapping → publish)
// with a single applier call:
applier.apply(ref.caseId(), ref.planItemId(), status, workItem);
```

The `applyStatus()` and `applyOutputMapping()` private methods in `WorkItemLifecycleAdapter` can be removed since they're now in `PlanItemCompletionApplier`. The `handleEscalation()` method stays as-is (it's a different code path that doesn't use the applier).

Keep `onWorkItemGroupLifecycle()` as-is — it handles group outcomes and doesn't need the applier.

Also remove these now-unused imports from `WorkItemLifecycleAdapter`:
- `import com.fasterxml.jackson.core.type.TypeReference;`
- `import com.fasterxml.jackson.databind.JsonNode;`
- `import com.fasterxml.jackson.databind.ObjectMapper;`
- `import io.casehub.api.model.evaluator.JQExpressionEvaluator;`
- `import io.casehub.engine.common.internal.jq.JQEvaluator;`
- `import io.casehub.engine.common.internal.jq.ValidationResult;`
- `import io.casehub.engine.common.internal.model.CaseInstance;`
- `import io.casehub.engine.common.spi.CaseInstanceRepository;`

These are now internal to `PlanItemCompletionApplier`.

- [ ] **Step 3: Build and run work-adapter tests**

```bash
mvn --batch-mode install -DskipTests -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml -q
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor: extract PlanItemCompletionApplier — shared status-transition logic between WorkItemLifecycleAdapter and HumanTaskRecoveryService

Refs #398"
```

---

### Task 12: Add WorkItemService.findByCallerRef() to casehub-work

**Files:**
- Modify: `~/claude/casehub/work/runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`

First file a GitHub issue on casehub-work:

- [ ] **Step 1: File casehub-work issue**

```bash
gh issue create -R casehubio/work \
  --title "feat: add WorkItemService.findByCallerRef(String) for HumanTask restart recovery" \
  --body "## Context

casehub-engine#398 requires a way to look up a WorkItem by its callerRef (e.g. \`case:{caseId}/pi:{planItemId}\`) during startup recovery. Currently there is no method for this.

## Required

Add to \`WorkItemService\`:

\`\`\`java
public Optional<WorkItem> findByCallerRef(String callerRef) {
    return workItemStore.scanAll().stream()
        .filter(w -> callerRef.equals(w.callerRef))
        .findFirst();
}
\`\`\`

This is called only during JVM startup recovery — not on the hot path. A full-table scan is acceptable.

Refs casehubio/engine#398"
```

Note the issue number returned (e.g., `casehubio/work#N`).

- [ ] **Step 2: Implement findByCallerRef() in WorkItemService**

Add this method to `WorkItemService.java` (after the `scanAll()` or `scan()` methods, before the closing `}`):

```java
/**
 * Finds a WorkItem by its callerRef. Used only during JVM startup recovery by
 * casehub-engine's HumanTaskRecoveryService — not called on the hot path.
 *
 * @param callerRef the callerRef to match (format: "case:{caseId}/pi:{planItemId}")
 * @return an Optional containing the WorkItem if found
 */
public Optional<WorkItem> findByCallerRef(final String callerRef) {
    return workItemStore.scanAll().stream()
        .filter(w -> callerRef.equals(w.callerRef))
        .findFirst();
}
```

- [ ] **Step 3: Build and install casehub-work locally**

```bash
mvn --batch-mode install -DskipTests -q -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Commit to casehub-work**

```bash
git -C /Users/mdproctor/claude/casehub/work add runtime/
git -C /Users/mdproctor/claude/casehub/work commit -m "feat: add WorkItemService.findByCallerRef(String) for HumanTask restart recovery

Refs casehubio/work#N, casehubio/engine#398"
```

---

### Task 13: Write HumanTaskRecoveryTest (TDD) + implement HumanTaskRecoveryService

**Files:**
- Create: `work-adapter/src/test/java/io/casehub/workadapter/recovery/HumanTaskRecoveryServiceTest.java`
- Create: `work-adapter/src/main/java/io/casehub/workadapter/recovery/HumanTaskRecoveryService.java`

- [ ] **Step 1: Write failing integration test**

Create `work-adapter/src/test/java/io/casehub/workadapter/recovery/HumanTaskRecoveryServiceTest.java`:

```java
package io.casehub.workadapter.recovery;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.persistence.memory.MemoryPlanItemStore;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.testing.InMemoryWorkItemStore;
import io.casehub.workadapter.CallerRef;
import io.quarkus.runtime.StartupEvent;
import io.quarkus.test.junit.QuarkusTest;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicBoolean;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies HumanTaskRecoveryService catches up WorkItems that terminated while the JVM was down.
 * Simulates restart by: seeding PlanItemStore with DELEGATED items + WorkItemStore with terminal
 * WorkItems, then calling onStart() directly.
 */
@QuarkusTest
class HumanTaskRecoveryServiceTest {

  @Inject HumanTaskRecoveryService recoveryService;
  @Inject BlackboardRegistry registry;
  @Inject PlanItemStore planItemStore;
  @Inject WorkItemStore workItemStore;
  @Inject EventBus eventBus;

  private UUID caseId;
  private String planItemId;
  private String callerRef;

  @BeforeEach
  @Transactional
  void setUp() {
    if (planItemStore instanceof MemoryPlanItemStore mem) mem.clear();
    if (workItemStore instanceof InMemoryWorkItemStore mem) mem.clear();
    caseId = UUID.randomUUID();
    planItemId = UUID.randomUUID().toString();
    callerRef = CallerRef.encode(caseId, planItemId);

    // Simulate a DELEGATED PlanItem in the store (persisted before JVM crash)
    planItemStore.save(new PlanItemSaveRequest(
        caseId, planItemId, "review-task",
        PlanItemStatus.DELEGATED, Instant.now(),
        TargetType.HUMAN_TASK, null));
  }

  @AfterEach
  @Transactional
  void tearDown() {
    if (planItemStore instanceof MemoryPlanItemStore mem) mem.clear();
    if (workItemStore instanceof InMemoryWorkItemStore mem) mem.clear();
    registry.evict(caseId);
  }

  @Test
  @Transactional
  void onStart_transitionsPlanItemWhenWorkItemIsCompleted() {
    WorkItem workItem = createWorkItem(callerRef, WorkItemStatus.COMPLETED);

    AtomicBoolean contextChangedFired = new AtomicBoolean(false);
    eventBus.consumer(EventBusAddresses.CONTEXT_CHANGED, msg -> contextChangedFired.set(true));

    recoveryService.onStart(new StartupEvent());

    // PlanItem should be COMPLETED in the registry
    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.COMPLETED);

    await().atMost(Duration.ofSeconds(2)).untilTrue(contextChangedFired);
  }

  @Test
  @Transactional
  void onStart_transitionsPlanItemToRejectedWhenWorkItemIsRejected() {
    createWorkItem(callerRef, WorkItemStatus.REJECTED);

    recoveryService.onStart(new StartupEvent());

    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.REJECTED);
  }

  @Test
  @Transactional
  void onStart_transitionsPlanItemToFaultedWhenWorkItemIsExpired() {
    createWorkItem(callerRef, WorkItemStatus.EXPIRED);

    recoveryService.onStart(new StartupEvent());

    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.FAULTED);
  }

  @Test
  @Transactional
  void onStart_skipsWhenWorkItemIsStillInFlight() {
    createWorkItem(callerRef, WorkItemStatus.IN_PROGRESS);

    recoveryService.onStart(new StartupEvent());

    // Registry should be hydrated (from PlanItemStore) but PlanItem stays DELEGATED
    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.DELEGATED);
  }

  @Test
  @Transactional
  void onStart_skipsWhenNoMatchingWorkItemFound() {
    // No WorkItem in store — WorkItem was cleaned up externally
    recoveryService.onStart(new StartupEvent());

    // Registry hydrated but PlanItem stays DELEGATED (no WorkItem to catch up)
    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.DELEGATED);
  }

  @Test
  @Transactional
  void onStart_isIdempotent_whenPlanItemAlreadyTerminal() {
    createWorkItem(callerRef, WorkItemStatus.COMPLETED);

    // First run — transitions to COMPLETED
    recoveryService.onStart(new StartupEvent());
    // Second run — should not throw, silently skips
    recoveryService.onStart(new StartupEvent());

    PlanItem item = registry.get(caseId)
        .flatMap(plan -> plan.getPlanItem(planItemId))
        .orElse(null);
    assertThat(item).isNotNull();
    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.COMPLETED);
  }

  @Transactional
  private WorkItem createWorkItem(String callerRef, WorkItemStatus status) {
    WorkItem w = new WorkItem();
    w.id = UUID.randomUUID();
    w.callerRef = callerRef;
    w.title = "Test WorkItem";
    w.status = status;
    w.createdAt = Instant.now();
    workItemStore.put(w);
    return w;
  }
}
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -Dtest=HumanTaskRecoveryServiceTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: compilation failure (`HumanTaskRecoveryService` not found).

- [ ] **Step 3: Implement HumanTaskRecoveryService**

Create `work-adapter/src/main/java/io/casehub/workadapter/recovery/HumanTaskRecoveryService.java`:

```java
package io.casehub.workadapter.recovery;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.workadapter.CallerRef;
import io.casehub.workadapter.PlanItemCompletionApplier;
import io.quarkus.runtime.StartupEvent;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import java.util.EnumSet;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import org.jboss.logging.Logger;

/**
 * Catches up WorkItems that completed while the JVM was down — the "offline-completion" scenario
 * in casehubio/engine#398.
 *
 * <p>Runs at @Priority(25) — after Quartz recovery at 20 and after BlackboardRegistry lazy
 * hydration is available (hydration is on-demand, no ordering needed). For each DELEGATED
 * PlanItem in the store, checks the corresponding WorkItem status; if terminal, applies the
 * transition and fires CONTEXT_CHANGED.
 */
@ApplicationScoped
public class HumanTaskRecoveryService {

  private static final Logger LOG = Logger.getLogger(HumanTaskRecoveryService.class);

  private static final Set<WorkItemStatus> TERMINAL_STATUSES =
      EnumSet.of(
          WorkItemStatus.COMPLETED,
          WorkItemStatus.REJECTED,
          WorkItemStatus.CANCELLED,
          WorkItemStatus.EXPIRED);

  @Inject PlanItemStore planItemStore;
  @Inject WorkItemService workItemService;
  @Inject PlanItemCompletionApplier applier;

  void onStart(@Observes @Priority(25) StartupEvent ev) {
    List<PlanItemRecord> delegated = planItemStore.findAllDelegated();
    if (delegated.isEmpty()) {
      LOG.debug("HumanTaskRecoveryService: no DELEGATED PlanItems found — nothing to recover");
      return;
    }
    LOG.infof("HumanTaskRecoveryService: scanning %d DELEGATED PlanItem(s) for offline completions",
        delegated.size());
    int recovered = 0;
    for (PlanItemRecord r : delegated) {
      if (tryRecover(r)) recovered++;
    }
    LOG.infof("HumanTaskRecoveryService: %d PlanItem(s) recovered out of %d scanned",
        recovered, delegated.size());
  }

  private boolean tryRecover(PlanItemRecord r) {
    String callerRef = CallerRef.encode(r.caseId(), r.planItemId());
    Optional<WorkItem> workItemOpt = workItemService.findByCallerRef(callerRef);

    if (workItemOpt.isEmpty()) {
      LOG.debugf("No WorkItem found for callerRef=%s — WorkItem may have been cleaned up; skipping",
          callerRef);
      return false;
    }

    WorkItem workItem = workItemOpt.get();
    if (!TERMINAL_STATUSES.contains(workItem.status)) {
      LOG.debugf("WorkItem %s for callerRef=%s is still in-flight (status=%s) — skipping",
          workItem.id, callerRef, workItem.status);
      return false;
    }

    LOG.infof("Recovering PlanItem %s in case %s — WorkItem %s was %s during downtime",
        r.planItemId(), r.caseId(), workItem.id, workItem.status);
    applier.apply(r.caseId(), r.planItemId(), workItem.status, workItem);
    return true;
  }
}
```

The `@Priority` annotation is `jakarta.annotation.Priority` — already imported above. This matches the pattern in `QuartzWorkerExecutionManager` which uses `@Observes @Priority(20) StartupEvent ev`.

- [ ] **Step 4: Run test to confirm it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -Dtest=HumanTaskRecoveryServiceTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|FAIL|ERROR|BUILD" | tail -10
```

Expected: `Tests run: 6, Failures: 0, Errors: 0`

If the `@Priority` annotation syntax is wrong, consult `QuartzWorkerExecutionManager.onStart()` for the exact Quarkus observer ordering pattern and mirror it.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat: HumanTaskRecoveryService — startup scan catches WorkItems that completed offline during JVM downtime; fires CONTEXT_CHANGED to unblock cases

Closes #398"
```

---

### Task 14: Full test suite run + final build

- [ ] **Step 1: Build everything with tests (blackboard)**

```bash
mvn --batch-mode install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl casehub-blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 2: Run work-adapter tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -15
```

Expected: `BUILD SUCCESS`

- [ ] **Step 3: Run common module tests (contract tests)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl common,persistence-memory -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -10
```

Expected: `BUILD SUCCESS`

- [ ] **Step 4: Push engine branch**

```bash
git -C /Users/mdproctor/claude/casehub/engine push --set-upstream origin issue-274-registry-hydration-recovery
```

- [ ] **Step 5: Push casehub-work changes**

```bash
git -C /Users/mdproctor/claude/casehub/work push
```
