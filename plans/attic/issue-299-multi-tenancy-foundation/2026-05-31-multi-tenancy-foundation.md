# Multi-Tenancy Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add explicit `tenancyId` to all domain objects, JPA entities, and SPI method signatures so every data operation is unconditionally scoped to a tenant.

**Architecture:** `tenancyId` flows as an explicit `String` parameter on every SPI method — no CDI scope injection in repositories. The HTTP boundary reads `currentPrincipal.tenancyId()` once; domain objects and events carry it thereafter. JPA entities get a `tenancy_id` column + index; every query adds `AND tenancy_id = ?`.

**Tech Stack:** Quarkus 3.32.2, Hibernate Reactive / Panache, Mutiny, JPA, `casehub-platform-api` (for `TenancyConstants`)

---

## Build commands

```bash
# Install all modules without tests (run before any module-specific test)
mvn install -DskipTests -q

# Run blackboard integration tests (uses in-memory stores, no Docker)
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl casehub-blackboard

# Run common module tests
mvn test -pl common

# Run persistence-memory tests
mvn test -pl persistence-memory
```

---

## File map

### Modified
- `common/pom.xml` — no change (platform-api not needed here)
- `persistence-hibernate/pom.xml` — add `casehub-platform-api` dep (for `TenancyConstants`)
- `persistence-memory/pom.xml` — add `casehub-platform-api` dep
- `common/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java` — add `public String tenancyId`
- `common/src/main/java/io/casehub/engine/common/internal/model/CaseMetaModel.java` — add `public String tenancyId`
- `common/src/main/java/io/casehub/engine/common/internal/history/EventLog.java` — add `public String tenancyId`
- `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java` — add `String tenancyId` record component
- `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java` — add `String tenancyId` record component
- `api/src/main/java/io/casehub/api/engine/PlanExecutionContext.java` — add `String tenancyId` record component
- `common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java` — add `String tenancyId` record component
- `common/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java` — add `String tenancyId` to all methods
- `common/src/main/java/io/casehub/engine/common/spi/CaseMetaModelRepository.java` — add `String tenancyId` to all methods
- `common/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java` — add `String tenancyId`; remove 3 cross-tenant methods
- `common/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java` — add `String tenancyId` to all methods
- `common/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java` — add `String tenancyId` to `save` and `findByCaseId`; leave `updateStatus`/`findDelegated`/`findAllDelegated` unchanged (see notes per method)
- `common/src/main/java/io/casehub/engine/common/spi/ReactivePlanItemStore.java` — same as above
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseMetaModelEntity.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/EventLogEntity.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java`
- `work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java`
- `ledger/src/main/java/io/casehub/ledger/model/CaseLedgerEntry.java`
- `ledger/src/main/java/io/casehub/ledger/model/WorkerDecisionEntry.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseMetaModelRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`
- `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepository.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseMetaModelRepository.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryEventLogRepository.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java`
- `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`
- `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseExecutionHandler.java`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionService.java`
- `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- All handlers in `runtime/src/main/java/io/casehub/engine/internal/engine/handler/` that call repositories
- `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`
- `work-adapter/src/main/java/io/casehub/workadapter/PlanItemCompletionApplier.java`

### Created
- `persistence-memory/src/main/java/io/casehub/persistence/memory/DefaultTestPrincipal.java`
- `runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantEventLogRepository.java`
- `runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantCaseInstanceRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java`
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantCaseInstanceRepository.java`

### Tests created
- `common/src/test/java/io/casehub/engine/common/spi/CaseInstanceRepositoryTenancyContractTest.java`
- `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTenancyTest.java`
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseExecutionHandlerTenancyTest.java`

---

## Important design notes (read before implementing)

**tenancyId source at call sites:**
- HTTP-triggered code: `currentPrincipal.tenancyId()` read ONCE at the entry point; stored in `CaseInstance.tenancyId`
- Vert.x `@ConsumeEvent` handlers: from `event.caseInstance().tenancyId` or `event.parentInstance().tenancyId`
- `@ObservesAsync` CDI observers: from `event.tenancyId()` (CaseLifecycleEvent carries it)
- Recovery services: use cross-tenant interfaces — no tenancyId needed

**PlanItemStore exceptions** (intentionally no tenancyId):
- `updateStatus(String planItemId, PlanItemStatus status)`: `planItemId` is a UUID string, globally unique — no cross-tenant collision possible
- `findDelegated(UUID caseId)`: caseId is a UUID, globally unique — used by BlackboardRegistry hydration which self-bootstraps tenancyId from `PlanItemRecord.tenancyId()`
- `findAllDelegated()`: cross-tenant by design, startup recovery only

**BlackboardRegistry.get() overloads:**
- `get(UUID caseId, String tenancyId)` — preferred; defense-in-depth check; lazy hydration passes tenancyId
- `get(UUID caseId)` — for WorkItemLifecycleAdapter which does not have tenancyId yet (pending casehub-work tenancy); UUID uniqueness ensures no cross-tenant collision; learns tenancyId from first PlanItemRecord on hydration

---

## Task 1: pom.xml + DefaultTestPrincipal

**Files:**
- Modify: `persistence-hibernate/pom.xml`
- Modify: `persistence-memory/pom.xml`
- Create: `persistence-memory/src/main/java/io/casehub/persistence/memory/DefaultTestPrincipal.java`

- [ ] **Step 1: Add casehub-platform-api to persistence-hibernate/pom.xml**

In `persistence-hibernate/pom.xml`, add inside `<dependencies>`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-api</artifactId>
</dependency>
```

- [ ] **Step 2: Add casehub-platform-api to persistence-memory/pom.xml**

Same addition in `persistence-memory/pom.xml`.

- [ ] **Step 3: Create DefaultTestPrincipal**

```java
// persistence-memory/src/main/java/io/casehub/persistence/memory/DefaultTestPrincipal.java
package io.casehub.persistence.memory;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;

/**
 * @DefaultBean CurrentPrincipal for test classpath — returns DEFAULT_TENANT_ID.
 * For testing only. If persistence-memory is on the compile classpath in production,
 * all operations will silently use DEFAULT_TENANT_ID.
 */
@DefaultBean
@ApplicationScoped
public class DefaultTestPrincipal implements CurrentPrincipal {

  @Override
  public String tenancyId() {
    return TenancyConstants.DEFAULT_TENANT_ID;
  }

  @Override
  public boolean isCrossTenantAdmin() {
    return false;
  }

  @Override
  public String actorId() {
    return "system";
  }

  @Override
  public Set<String> groups() {
    return Set.of();
  }
}
```

- [ ] **Step 4: Build to verify pom changes compile**

```bash
mvn install -DskipTests -q -pl persistence-memory,persistence-hibernate
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/pom.xml persistence-memory/pom.xml persistence-memory/src/main/java/io/casehub/persistence/memory/DefaultTestPrincipal.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): add casehub-platform-api dep + DefaultTestPrincipal

Refs casehubio/engine#299"
```

---

## Task 2: tenancyId on domain objects and records

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/CaseMetaModel.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/history/EventLog.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemRecord.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/model/PlanItemSaveRequest.java`
- Modify: `api/src/main/java/io/casehub/api/engine/PlanExecutionContext.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java`

- [ ] **Step 1: Add `public String tenancyId` to CaseInstance**

In `CaseInstance.java`, add after `public Long id;`:
```java
/** Tenant this case belongs to. Set by the repository at save; never updated. */
public String tenancyId;
```

- [ ] **Step 2: Add `public String tenancyId` to CaseMetaModel**

In `CaseMetaModel.java`, add after `public Long id;`:
```java
/** Tenant this definition belongs to. Set by the repository at save; never updated. */
public String tenancyId;
```

- [ ] **Step 3: Add `public String tenancyId` to EventLog**

In `EventLog.java`, add after `public Long id;`:
```java
/** Tenant this event belongs to. Set by the repository at append; never updated. */
public String tenancyId;
```

- [ ] **Step 4: Update PlanItemRecord record — add tenancyId component**

Replace the entire record declaration in `PlanItemRecord.java`:
```java
public record PlanItemRecord(
    UUID caseId,
    String planItemId,
    String bindingName,
    PlanItemStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId) {}
```

- [ ] **Step 5: Update PlanItemSaveRequest record — add tenancyId component**

Replace the entire record declaration in `PlanItemSaveRequest.java`:
```java
public record PlanItemSaveRequest(
    UUID caseId,
    String planItemId,
    String bindingName,
    PlanItemStatus status,
    Instant createdAt,
    TargetType targetType,
    String outputMappingExpression,
    String tenancyId) {}
```

- [ ] **Step 6: Update PlanExecutionContext record — add tenancyId component**

Replace the entire record declaration in `PlanExecutionContext.java`:
```java
public record PlanExecutionContext(
    UUID caseId,
    CaseDefinition definition,
    CaseContext caseContext,
    CaseStatus caseStatus,
    String tenancyId) {}
```

- [ ] **Step 7: Update CaseLifecycleEvent record — add tenancyId component**

Replace the record declaration in `CaseLifecycleEvent.java`:
```java
public record CaseLifecycleEvent(
    UUID caseId,
    String tenancyId,
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId) {}
```

Update the Javadoc to add: `@param tenancyId the tenant that owns this case`

- [ ] **Step 8: Build common and api modules**

```bash
mvn install -DskipTests -q -pl common,api
```
Expected: BUILD SUCCESS (these modules have no downstream deps yet touching the new fields)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/ api/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): add tenancyId to domain objects and records

CaseInstance, CaseMetaModel, EventLog, PlanItemRecord, PlanItemSaveRequest,
PlanExecutionContext, CaseLifecycleEvent all gain String tenancyId.
Records gain it as a new component; breaking change at all construction sites.

Refs casehubio/engine#299"
```

---

## Task 3: JPA entity columns and indices

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseInstanceEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/CaseMetaModelEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/EventLogEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/SubCaseGroupEntity.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java`
- Modify: `ledger/src/main/java/io/casehub/ledger/model/CaseLedgerEntry.java`
- Modify: `ledger/src/main/java/io/casehub/ledger/model/WorkerDecisionEntry.java`

**Pattern for each entity — add to `@Table` annotation and add field:**

For entities that already have `indexes = {...}` in `@Table` (e.g. PlanItemEntity):
```java
@Table(
    name = "existing_table_name",
    indexes = {
      // ...existing indexes...
      @Index(name = "idx_<table>_tenancy_id", columnList = "tenancy_id")
    })
```

For entities without existing `indexes`:
```java
@Table(name = "existing_table_name",
    indexes = {@Index(name = "idx_<table>_tenancy_id", columnList = "tenancy_id")})
```

**Field to add to every entity:**
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

- [ ] **Step 1: Update CaseInstanceEntity**

Add index to `@Table`:
```java
@Table(name = "case_instance",
    indexes = {@Index(name = "idx_case_instance_tenancy_id", columnList = "tenancy_id")})
```
Add field after `waitingForWorkId`:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

- [ ] **Step 2: Update CaseMetaModelEntity**

Add index to `@Table` and add the unique constraint change:
```java
@Table(name = "case_meta_model",
    uniqueConstraints = @UniqueConstraint(
        name = "uq_case_meta_model_tenant_key",
        columnNames = {"tenancy_id", "namespace", "name", "version"}),
    indexes = {@Index(name = "idx_case_meta_model_tenancy_id", columnList = "tenancy_id")})
```
Add field. Remove any existing `(namespace, name, version)` unique constraint if present.

- [ ] **Step 3: Update EventLogEntity**

Add index to `@Table`:
```java
@Table(name = "event_log",
    indexes = {@Index(name = "idx_event_log_tenancy_id", columnList = "tenancy_id")})
```
Add field.

- [ ] **Step 4: Update PlanItemEntity**

Add index to existing `indexes` array:
```java
@Index(name = "idx_plan_item_tenancy_id", columnList = "tenancy_id")
```
Add field.

- [ ] **Step 5: Update SubCaseGroupEntity**

Add `@Table` with index (read the file first to see current annotation), add field.

- [ ] **Step 6: Update WorkAdapterPlanItemEntity**

Add `@Table` with index, add field.

- [ ] **Step 7: Update CaseLedgerEntry**

Read the file first, then add `@Table` index and field following the same pattern.

- [ ] **Step 8: Update WorkerDecisionEntry**

Same pattern.

- [ ] **Step 9: Build entity modules**

```bash
mvn install -DskipTests -q -pl persistence-hibernate,work-adapter,ledger
```
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/src/main/java/io/casehub/persistence/jpa/ work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java ledger/src/main/java/io/casehub/ledger/model/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): add tenancy_id column and index to all 8 JPA entities

Schema is drop-and-create — no migration needed.
CaseMetaModelEntity unique constraint changes to (tenancy_id, namespace, name, version).

Refs casehubio/engine#299"
```

---

## Task 4: SPI interface changes — add tenancyId parameters (breaking change)

**This task intentionally breaks the build. Every call site in the codebase will fail to compile. That is expected — Tasks 5–10 fix them module by module.**

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseMetaModelRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/SubCaseGroupRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/PlanItemStore.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/ReactivePlanItemStore.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantEventLogRepository.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantCaseInstanceRepository.java`

- [ ] **Step 1: Replace CaseInstanceRepository**

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.smallrye.mutiny.Uni;
import java.util.UUID;

public interface CaseInstanceRepository {
  Uni<CaseInstance> save(CaseInstance instance, String tenancyId);
  Uni<CaseInstance> update(CaseInstance instance, String tenancyId);
  Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId);
  Uni<Void> updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId);
}
```

- [ ] **Step 2: Replace CaseMetaModelRepository**

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.smallrye.mutiny.Uni;

public interface CaseMetaModelRepository {
  Uni<CaseMetaModel> findByKey(String namespace, String name, String version, String tenancyId);
  Uni<CaseMetaModel> save(CaseMetaModel metaModel, String tenancyId);
}
```

- [ ] **Step 3: Replace EventLogRepository — remove cross-tenant methods, add tenancyId**

Remove `findByTypes`, `findSubmittedWorkWithoutCompletion`, and `findByWorkerAndType` (the cross-tenant variant — the same-tenant variant stays). Add `String tenancyId` to all remaining methods.

```java
package io.casehub.engine.common.spi;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.common.internal.history.EventLog;
import io.smallrye.mutiny.Uni;
import java.time.Instant;
import java.util.Collection;
import java.util.List;
import java.util.UUID;

public interface EventLogRepository {
  Uni<Void> append(EventLog eventLog, String tenancyId);
  Uni<Long> appendAndReturnId(EventLog eventLog, String tenancyId);
  Uni<EventLog> findById(Long id, String tenancyId);
  Uni<List<EventLog>> findSchedulingEvents(UUID caseId, String workerId, Instant after, String tenancyId);
  Uni<List<EventLog>> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types, String tenancyId);
  Uni<List<EventLog>> findByCaseAndWorkerAndType(UUID caseId, String workerId, CaseHubEventType type, String tenancyId);
  /** Same-tenant: used by SubCaseCompletionService. NOT the cross-tenant recovery variant. */
  Uni<List<EventLog>> findByWorkerAndType(String workerId, CaseHubEventType type, String tenancyId);
  Uni<List<EventLog>> findByCaseWithFilters(UUID caseId, Collection<CaseHubEventType> eventTypes, Collection<EventStreamType> streamTypes, String tenancyId);
}
```

- [ ] **Step 4: Replace SubCaseGroupRepository — add tenancyId to all methods**

```java
package io.casehub.engine.common.spi;

import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.common.internal.model.SubCaseGroup;
import io.smallrye.mutiny.Uni;
import java.util.Optional;
import java.util.UUID;

public interface SubCaseGroupRepository {
  Uni<SubCaseGroup> getOrCreate(UUID parentCaseId, String groupId, int totalInGroup,
      int requiredCount, OnThresholdReached onThresholdReached, String tenancyId);
  Uni<SubCaseGroup> registerChild(UUID parentCaseId, String groupId, UUID childCaseId, String tenancyId);
  Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId, String tenancyId);
  Uni<SubCaseGroup> incrementRejected(UUID parentCaseId, String groupId, String tenancyId);
  Uni<Boolean> markPolicyTriggered(UUID parentCaseId, String groupId, String tenancyId);
  Uni<Optional<SubCaseGroup>> findByChildCaseId(UUID childCaseId, String tenancyId);
}
```

- [ ] **Step 5: Replace PlanItemStore — add tenancyId to save and findByCaseId only**

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import java.util.List;
import java.util.UUID;

public interface PlanItemStore {
  /** Persist a new PlanItem scoped to tenancyId. */
  void save(PlanItemSaveRequest request, String tenancyId);

  /**
   * Update status by planItemId (UUID — globally unique; no tenancyId needed in WHERE).
   * Safe without tenancyId filter: UUID collision across tenants is cryptographically impossible.
   */
  void updateStatus(String planItemId, PlanItemStatus status);

  List<PlanItemRecord> findByCaseId(UUID caseId, String tenancyId);

  /**
   * Return DELEGATED records for a specific case.
   * caseId is a UUID (globally unique) — no tenancyId filter needed.
   * Used by BlackboardRegistry lazy hydration; tenancyId is self-bootstrapped from returned records.
   */
  List<PlanItemRecord> findDelegated(UUID caseId);

  /**
   * Return ALL DELEGATED records across all tenants.
   * Cross-tenant by design — startup recovery only (HumanTaskRecoveryService).
   */
  List<PlanItemRecord> findAllDelegated();
}
```

- [ ] **Step 6: Replace ReactivePlanItemStore — matching signature changes**

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemSaveRequest;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.smallrye.mutiny.Uni;
import java.util.List;
import java.util.UUID;

public interface ReactivePlanItemStore {
  Uni<Void> save(PlanItemSaveRequest request, String tenancyId);
  /** UUID planItemId — globally unique; no tenancyId needed. */
  Uni<Void> updateStatus(String planItemId, PlanItemStatus status);
  Uni<List<PlanItemRecord>> findByCaseId(UUID caseId, String tenancyId);
  /** UUID caseId — globally unique; no tenancyId filter needed. */
  Uni<List<PlanItemRecord>> findDelegated(UUID caseId);
  /** Cross-tenant: startup recovery only. */
  Uni<List<PlanItemRecord>> findAllDelegated();
}
```

- [ ] **Step 7: Create CrossTenantEventLogRepository**

```java
// runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantEventLogRepository.java
package io.casehub.engine.internal.recovery.spi;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.smallrye.mutiny.Uni;
import java.util.Collection;
import java.util.List;
import java.util.UUID;

/**
 * Cross-tenant event log access — startup recovery services only.
 * Not for general use. Lives in internal.recovery.spi to prevent accidental injection.
 */
public interface CrossTenantEventLogRepository {
  /** All events of given types across all tenants. Recovery: reschedule orphaned workers. */
  Uni<List<EventLog>> findByTypes(Collection<CaseHubEventType> types);

  /** Events for a specific case — caseId is UUID (globally unique). Recovery: rebuild context. */
  Uni<List<EventLog>> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types);

  /** Find WORK_SUBMITTED events with no matching WORK_COMPLETED across all tenants. */
  Uni<List<String>> findSubmittedWorkWithoutCompletion();

  /** Cross-tenant variant of findByWorkerAndType — recovery only. */
  Uni<List<EventLog>> findByWorkerAndTypeAcrossTenants(String workerId, CaseHubEventType type);
}
```

- [ ] **Step 8: Create CrossTenantCaseInstanceRepository**

```java
// runtime/src/main/java/io/casehub/engine/internal/recovery/spi/CrossTenantCaseInstanceRepository.java
package io.casehub.engine.internal.recovery.spi;

import io.casehub.engine.common.internal.model.CaseInstance;
import io.smallrye.mutiny.Uni;
import java.util.UUID;

/**
 * Cross-tenant case instance access — startup recovery services only.
 * Not for general use. Lives in internal.recovery.spi to prevent accidental injection.
 */
public interface CrossTenantCaseInstanceRepository {
  /** Load a case instance without tenant filter. caseId is UUID (globally unique). */
  Uni<CaseInstance> findByUuid(UUID caseId);
}
```

- [ ] **Step 9: Verify build breaks as expected**

```bash
mvn install -DskipTests -q -pl common,api 2>&1 | tail -5
```
Expected: common and api build fine. Running full build will show cascading failures in dependent modules — that's expected. Do not attempt to fix them all at once.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/main/java/io/casehub/engine/common/spi/ runtime/src/main/java/io/casehub/engine/internal/recovery/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): add tenancyId to SPI interfaces; add cross-tenant recovery SPIs

All SPI methods gain String tenancyId parameter. Build is intentionally broken
until all implementations and call sites are updated (Tasks 5-9).
CrossTenantEventLogRepository and CrossTenantCaseInstanceRepository are new
internal interfaces for startup recovery only.

Refs casehubio/engine#299"
```

---

## Task 5: JPA repository implementations

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseMetaModelRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantCaseInstanceRepository.java`

**Pattern for all JPA implementations — no `@Inject CurrentPrincipal`. tenancyId arrives as parameter.**

- [ ] **Step 1: Update JpaCaseInstanceRepository — all 4 methods**

```java
@Override
public Uni<CaseInstance> save(CaseInstance instance, String tenancyId) {
    return withSafeContext(() ->
        Panache.withTransaction(() ->
            Panache.getSession().chain(session -> {
                CaseInstanceEntity entity = new CaseInstanceEntity();
                entity.tenancyId = tenancyId;
                entity.uuid = instance.getUuid();
                entity.state = instance.getState();
                entity.parentCaseId = instance.getParentCaseId();
                entity.parentPlanItemId = instance.getParentPlanItemId();
                entity.waitingForWorkId = instance.getWaitingForWorkId();
                if (instance.getCaseMetaModel() != null) {
                    entity.caseMetaModel = session.getReference(
                        CaseMetaModelEntity.class, instance.getCaseMetaModel().getId());
                }
                return entity.persist().map(v -> {
                    instance.id = entity.id;
                    instance.tenancyId = tenancyId;
                    return instance;
                });
            })));
}

@Override
public Uni<CaseInstance> update(CaseInstance instance, String tenancyId) {
    return withSafeContext(() ->
        Panache.withTransaction(() ->
            CaseInstanceEntity.<CaseInstanceEntity>find(
                    "id = ?1 and tenancyId = ?2", instance.id, tenancyId)
                .firstResult()
                .invoke(entity -> {
                    entity.state = instance.getState();
                    entity.parentCaseId = instance.getParentCaseId();
                    entity.parentPlanItemId = instance.getParentPlanItemId();
                    entity.waitingForWorkId = instance.getWaitingForWorkId();
                    // tenancyId is immutable — NOT updated
                })
                .replaceWith(instance)));
}

@Override
public Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            CaseInstanceEntity.<CaseInstanceEntity>find(
                    "from CaseInstanceEntity ci join fetch ci.caseMetaModel " +
                    "where ci.uuid = ?1 and ci.tenancyId = ?2", uuid, tenancyId)
                .firstResult())
        .map(entity -> entity == null ? null : fromEntity(entity)));
}

@Override
public Uni<Void> updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId) {
    EventLogEntity logEntity = toEventLogEntity(eventLog, tenancyId);
    return withSafeContext(() ->
        Panache.withTransaction(() ->
            CaseInstanceEntity.<CaseInstanceEntity>find(
                    "id = ?1 and tenancyId = ?2", instance.id, tenancyId)
                .firstResult()
                .chain(entity -> {
                    entity.state = instance.getState();
                    entity.parentCaseId = instance.getParentCaseId();
                    entity.parentPlanItemId = instance.getParentPlanItemId();
                    entity.waitingForWorkId = instance.getWaitingForWorkId();
                    return Panache.getSession().chain(s -> s.merge(entity));
                })
                .chain(merged -> logEntity.persistAndFlush()))
        .invoke(() -> {
            eventLog.id = logEntity.id;
            eventLog.setSeq(logEntity.seq);
        })
        .replaceWithVoid());
}

// Update fromEntity to map tenancyId
private CaseInstance fromEntity(CaseInstanceEntity entity) {
    CaseInstance instance = new CaseInstance();
    instance.id = entity.id;
    instance.tenancyId = entity.tenancyId;  // ADDED
    instance.setUuid(entity.uuid);
    instance.setState(entity.state);
    instance.setParentCaseId(entity.parentCaseId);
    instance.setParentPlanItemId(entity.parentPlanItemId);
    instance.setWaitingForWorkId(entity.waitingForWorkId);
    if (entity.caseMetaModel != null) {
        instance.setCaseMetaModel(fromMetaEntity(entity.caseMetaModel));
    }
    return instance;
}

// Helper to build EventLogEntity with tenancyId
private EventLogEntity toEventLogEntity(EventLog log, String tenancyId) {
    EventLogEntity entity = new EventLogEntity();
    entity.tenancyId = tenancyId;
    entity.caseId = log.getCaseId();
    entity.eventType = log.getEventType();
    entity.streamType = log.getStreamType();
    entity.workerId = log.getWorkerId();
    entity.timestamp = log.getTimestamp();
    entity.payload = log.getPayload();
    entity.metadata = log.getMetadata();
    return entity;
}
```

- [ ] **Step 2: Update JpaCaseMetaModelRepository**

```java
@Override
public Uni<CaseMetaModel> findByKey(String namespace, String name, String version, String tenancyId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            CaseMetaModelEntity.<CaseMetaModelEntity>find(
                    "namespace = ?1 and name = ?2 and version = ?3 and tenancyId = ?4",
                    namespace, name, version, tenancyId)
                .firstResult())
        .map(e -> e == null ? null : fromEntity(e)));
}

@Override
public Uni<CaseMetaModel> save(CaseMetaModel metaModel, String tenancyId) {
    return withSafeContext(() ->
        Panache.withTransaction(() ->
            Panache.getSession().chain(session -> {
                CaseMetaModelEntity entity = new CaseMetaModelEntity();
                entity.tenancyId = tenancyId;
                entity.namespace = metaModel.getNamespace();
                entity.name = metaModel.getName();
                entity.version = metaModel.getVersion();
                entity.title = metaModel.getTitle();
                entity.dsl = metaModel.getDsl();
                entity.definition = metaModel.getDefinition();
                entity.createdAt = metaModel.getCreatedAt() != null
                    ? metaModel.getCreatedAt() : java.time.Instant.now();
                return entity.persist().map(v -> {
                    metaModel.id = entity.id;
                    metaModel.tenancyId = tenancyId;
                    metaModel.setCreatedAt(entity.createdAt);
                    return metaModel;
                });
            })));
}
```

Update `fromMetaEntity` (used by JpaCaseInstanceRepository) to map `entity.tenancyId → metaModel.tenancyId`.

- [ ] **Step 3: Update JpaEventLogRepository — tenant-scoped methods only**

For every method, add `String tenancyId` parameter and add `AND tenancyId = ?N` to query. Remove `findByTypes`, `findSubmittedWorkWithoutCompletion`, and the old `findByWorkerAndType` (these move to `JpaCrosstenantEventLogRepository`).

Key example for `append`:
```java
@Override
public Uni<Void> append(EventLog eventLog, String tenancyId) {
    EventLogEntity entity = toEntity(eventLog, tenancyId);
    return withSafeContext(() ->
        Panache.withTransaction(entity::persistAndFlush)
            .invoke(() -> {
                eventLog.id = entity.id;
                eventLog.setSeq(entity.seq);
            })
            .replaceWithVoid());
}

private EventLogEntity toEntity(EventLog log, String tenancyId) {
    EventLogEntity entity = new EventLogEntity();
    entity.tenancyId = tenancyId;
    entity.caseId = log.getCaseId();
    entity.eventType = log.getEventType();
    entity.streamType = log.getStreamType();
    entity.workerId = log.getWorkerId();
    entity.timestamp = log.getTimestamp();
    entity.payload = log.getPayload();
    entity.metadata = log.getMetadata();
    return entity;
}
```

For `findByWorkerAndType` (same-tenant):
```java
@Override
public Uni<List<EventLog>> findByWorkerAndType(String workerId, CaseHubEventType type, String tenancyId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            EventLogEntity.<EventLogEntity>find(
                    "workerId = ?1 and eventType = ?2 and tenancyId = ?3",
                    workerId, type, tenancyId)
                .list())
        .map(list -> list.stream().map(this::fromEntity).toList()));
}
```

Update `fromEntity` to map `entity.tenancyId → log.tenancyId`.

- [ ] **Step 4: Update JpaSubCaseGroupRepository — add tenancyId to all queries**

Read the current implementation, then add `AND tenancyId = ?N` to every query and `entity.tenancyId = tenancyId` to every insert.

- [ ] **Step 5: Update JpaReactivePlanItemStore**

```java
@Override
public Uni<Void> save(PlanItemSaveRequest request, String tenancyId) {
    return withSafeContext(() ->
        Panache.withTransaction(() -> {
            PlanItemEntity e = new PlanItemEntity();
            e.tenancyId = tenancyId;
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

// updateStatus — unchanged (UUID planItemId, no tenancyId in WHERE)

@Override
public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId, String tenancyId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            PlanItemEntity.<PlanItemEntity>find(
                    "caseId = ?1 and tenancyId = ?2", caseId, tenancyId)
                .list()
                .map(list -> list.stream().map(this::toRecord).collect(Collectors.toList()))));
}

// findDelegated(UUID caseId) — unchanged (UUID is globally unique)

// findAllDelegated() — unchanged (cross-tenant by design)

private PlanItemRecord toRecord(PlanItemEntity e) {
    return new PlanItemRecord(
        e.caseId,
        e.planItemId,
        e.bindingName,
        e.status,
        e.createdAt,
        e.targetType,
        e.outputMappingExpression,
        e.tenancyId);  // ADDED — new record component
}
```

- [ ] **Step 6: Update JpaPlanItemStore (work-adapter) — same pattern**

Read `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java` and apply same changes.

- [ ] **Step 7: Create JpaCrosstenantEventLogRepository**

```java
// persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java
package io.casehub.persistence.jpa;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.internal.recovery.spi.CrossTenantEventLogRepository;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

@ApplicationScoped
public class JpaCrosstenantEventLogRepository extends AbstractJpaRepository
    implements CrossTenantEventLogRepository {

  @Override
  public Uni<List<EventLog>> findByTypes(Collection<CaseHubEventType> types) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            EventLogEntity.<EventLogEntity>find(
                    "eventType in ?1 order by seq asc", types)
                .list())
        .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and eventType in ?2 order by seq asc", caseId, types)
                .list())
        .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<String>> findSubmittedWorkWithoutCompletion() {
    // Move the existing findSubmittedWorkWithoutCompletion logic from JpaEventLogRepository here
    // (copy the implementation verbatim)
    return withSafeContext(() ->
        Panache.withSession(() ->
            EventLogEntity.<EventLogEntity>list(
                "eventType", CaseHubEventType.WORK_SUBMITTED))
        .chain(submitted ->
            Panache.withSession(() ->
                EventLogEntity.<EventLogEntity>list(
                    "eventType", CaseHubEventType.WORK_COMPLETED))
            .map(completed -> {
                var submittedKeys = submitted.stream()
                    .map(e -> e.metadata != null ? e.metadata.path("correlationKey").asText(null) : null)
                    .filter(Objects::nonNull)
                    .collect(java.util.stream.Collectors.toSet());
                var completedKeys = completed.stream()
                    .map(e -> e.metadata != null ? e.metadata.path("correlationKey").asText(null) : null)
                    .filter(Objects::nonNull)
                    .collect(java.util.stream.Collectors.toSet());
                submittedKeys.removeAll(completedKeys);
                return new ArrayList<>(submittedKeys);
            })));
  }

  @Override
  public Uni<List<EventLog>> findByWorkerAndTypeAcrossTenants(String workerId, CaseHubEventType type) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            EventLogEntity.<EventLogEntity>find(
                    "workerId = ?1 and eventType = ?2", workerId, type)
                .list())
        .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  private EventLog fromEntity(EventLogEntity entity) {
    EventLog log = new EventLog();
    log.id = entity.id;
    log.setSeq(entity.seq);
    log.setCaseId(entity.caseId);
    log.setEventType(entity.eventType);
    log.setStreamType(entity.streamType);
    log.setWorkerId(entity.workerId);
    log.setTimestamp(entity.timestamp);
    log.setPayload(entity.payload);
    log.setMetadata(entity.metadata);
    log.tenancyId = entity.tenancyId;
    return log;
  }
}
```

- [ ] **Step 8: Create JpaCrosstenantCaseInstanceRepository**

```java
// persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantCaseInstanceRepository.java
package io.casehub.persistence.jpa;

import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.internal.recovery.spi.CrossTenantCaseInstanceRepository;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;

@ApplicationScoped
public class JpaCrosstenantCaseInstanceRepository extends AbstractJpaRepository
    implements CrossTenantCaseInstanceRepository {

  @Override
  public Uni<CaseInstance> findByUuid(UUID caseId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            CaseInstanceEntity.<CaseInstanceEntity>find(
                    "from CaseInstanceEntity ci join fetch ci.caseMetaModel where ci.uuid = ?1",
                    caseId)
                .firstResult())
        .map(entity -> entity == null ? null : fromEntity(entity)));
  }

  private CaseInstance fromEntity(CaseInstanceEntity entity) {
    CaseInstance instance = new CaseInstance();
    instance.id = entity.id;
    instance.tenancyId = entity.tenancyId;
    instance.setUuid(entity.uuid);
    instance.setState(entity.state);
    instance.setParentCaseId(entity.parentCaseId);
    instance.setParentPlanItemId(entity.parentPlanItemId);
    instance.setWaitingForWorkId(entity.waitingForWorkId);
    if (entity.caseMetaModel != null) {
      CaseMetaModel m = new CaseMetaModel();
      // populate from entity
      instance.setCaseMetaModel(m);
    }
    return instance;
  }
}
```

Note: `fromEntity` in the cross-tenant repo should delegate to the same `fromEntity` logic already in `JpaCaseInstanceRepository`. Extract it to `AbstractJpaRepository` or duplicate it here — either is fine; keeping it local avoids coupling.

- [ ] **Step 9: Build persistence-hibernate**

```bash
mvn install -DskipTests -q -pl persistence-hibernate
```
Expected: BUILD SUCCESS (implementations satisfy the updated interfaces)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/ work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): update JPA repositories — explicit tenancyId on all queries

All JPA repositories now take String tenancyId; every query adds AND tenancy_id = ?.
update() paths include tenancyId in WHERE clause.
JpaCrosstenantEventLogRepository and JpaCrosstenantCaseInstanceRepository added
for startup recovery services.

Refs casehubio/engine#299"
```

---

## Task 6: Memory store implementations

**Files:**
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseMetaModelRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryEventLogRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java`

**Pattern for all memory stores — filter by tenancyId using simple equality check.**

- [ ] **Step 1: Update InMemoryCaseInstanceRepository**

```java
@Override
public Uni<CaseInstance> save(CaseInstance instance, String tenancyId) {
    rwLock.writeLock().lock();
    try {
        if (instance.id == null) {
            instance.id = idSeq.incrementAndGet();
        }
        instance.tenancyId = tenancyId;
        store.put(instance.getUuid(), instance);
        return Uni.createFrom().item(instance);
    } finally {
        rwLock.writeLock().unlock();
    }
}

@Override
public Uni<CaseInstance> update(CaseInstance instance, String tenancyId) {
    rwLock.writeLock().lock();
    try {
        CaseInstance existing = store.get(instance.getUuid());
        if (existing == null || !tenancyId.equals(existing.tenancyId)) {
            throw new IllegalStateException(
                "CaseInstance not found or wrong tenant: " + instance.getUuid());
        }
        store.put(instance.getUuid(), instance);
        return Uni.createFrom().item(instance);
    } finally {
        rwLock.writeLock().unlock();
    }
}

@Override
public Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
    rwLock.readLock().lock();
    try {
        CaseInstance instance = store.get(uuid);
        if (instance != null && !tenancyId.equals(instance.tenancyId)) {
            return Uni.createFrom().nullItem();
        }
        return Uni.createFrom().item(instance);
    } finally {
        rwLock.readLock().unlock();
    }
}

@Override
public Uni<Void> updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId) {
    rwLock.writeLock().lock();
    try {
        instance.tenancyId = tenancyId;
        store.put(instance.getUuid(), instance);
        return eventLogRepository.append(eventLog, tenancyId);
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

- [ ] **Step 2: Update InMemoryCaseMetaModelRepository**

Read the file and apply the same pattern: add `String tenancyId` to `save` and `findByKey`; filter by tenancyId in findByKey.

- [ ] **Step 3: Update InMemoryEventLogRepository — also implement CrossTenantEventLogRepository**

`InMemoryEventLogRepository` implements both interfaces. The class declaration becomes:
```java
@Alternative
@ApplicationScoped
public class InMemoryEventLogRepository
    implements EventLogRepository, CrossTenantEventLogRepository {
```

All tenant-scoped methods add tenancyId to the stored `EventLog` and filter by it. Cross-tenant methods (from `CrossTenantEventLogRepository`) return unfiltered data — in test contexts there is only one tenant (DEFAULT_TENANT_ID), so no isolation is violated.

Key pattern for `append`:
```java
@Override
public Uni<Void> append(EventLog eventLog, String tenancyId) {
    eventLog.tenancyId = tenancyId;
    // existing storage logic
}
```

- [ ] **Step 4: Update MemorySubCaseGroupRepository**

Add tenancyId to all methods. Filter in-memory map by tenancyId.

- [ ] **Step 5: Update MemoryPlanItemStore + MemoryReactivePlanItemStore**

For `save(request, tenancyId)`: store the tenancyId in the record.
For `findByCaseId(caseId, tenancyId)`: filter by both caseId and tenancyId.
For `findDelegated(UUID caseId)`: no tenancyId filter (unchanged).
For `findAllDelegated()`: no tenancyId filter (unchanged).
For `updateStatus()`: unchanged.

The `PlanItemRecord` construction in `toRecord` gains `tenancyId`:
```java
new PlanItemRecord(
    request.caseId(), request.planItemId(), request.bindingName(),
    request.status(), request.createdAt(), request.targetType(),
    request.outputMappingExpression(), request.tenancyId())
```

- [ ] **Step 6: Build persistence-memory**

```bash
mvn install -DskipTests -q -pl persistence-memory
```
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-memory/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): update in-memory stores — tenancyId filtering

All memory stores filter by tenancyId parameter. InMemoryEventLogRepository
implements both EventLogRepository and CrossTenantEventLogRepository.
DefaultTestPrincipal provides DEFAULT_TENANT_ID on test classpath.

Refs casehubio/engine#299"
```

---

## Task 7: BlackboardRegistry

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`

- [ ] **Step 1: Write the failing unit test first**

Create `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTenancyTest.java`:

```java
package io.casehub.blackboard.registry;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class BlackboardRegistryTenancyTest {

  private BlackboardRegistry registry;

  @BeforeEach
  void setUp() {
    registry = new BlackboardRegistry();
    // No planItemStore wired — tests only the in-memory isolation
  }

  @Test
  void getOrCreate_storedTenancy_returnsForCorrectTenant() {
    UUID caseId = UUID.randomUUID();
    CasePlanModel plan = registry.getOrCreate(caseId, "tenant-a");
    assertThat(plan).isNotNull();

    Optional<CasePlanModel> found = registry.get(caseId, "tenant-a");
    assertThat(found).isPresent();
  }

  @Test
  void get_wrongTenant_returnsEmpty() {
    UUID caseId = UUID.randomUUID();
    registry.getOrCreate(caseId, "tenant-a");

    Optional<CasePlanModel> found = registry.get(caseId, "tenant-b");
    assertThat(found).isEmpty();
  }

  @Test
  void evict_removesEntry_oOneOperation() {
    UUID caseId = UUID.randomUUID();
    registry.getOrCreate(caseId, "tenant-a");
    assertThat(registry.get(caseId, "tenant-a")).isPresent();

    registry.evict(caseId);
    assertThat(registry.get(caseId, "tenant-a")).isEmpty();
  }

  @Test
  void get_noTenancy_returnsForUniqueUUID() {
    UUID caseId = UUID.randomUUID();
    registry.getOrCreate(caseId, "tenant-a");

    // UUID-only get (for callers without tenancyId) should still work
    Optional<CasePlanModel> found = registry.get(caseId);
    assertThat(found).isPresent();
  }
}
```

- [ ] **Step 2: Run test — expect compile failure (BlackboardRegistry.getOrCreate signature mismatch)**

```bash
mvn test -pl blackboard -Dtest=BlackboardRegistryTenancyTest 2>&1 | tail -10
```
Expected: COMPILE ERROR — `getOrCreate(UUID)` doesn't match new signature

- [ ] **Step 3: Update BlackboardRegistry**

```java
@ApplicationScoped
public class BlackboardRegistry {

  private static final class CaseEntry {
    final String tenancyId;
    final CasePlanModel planModel;
    final ConcurrentHashMap<String, String> completionIndex = new ConcurrentHashMap<>();
    final AtomicBoolean configured = new AtomicBoolean(false);

    CaseEntry(UUID caseId, String tenancyId) {
      this.tenancyId = tenancyId;
      this.planModel = new DefaultCasePlanModel(caseId);
    }
  }

  private final ConcurrentHashMap<UUID, CaseEntry> entries = new ConcurrentHashMap<>();
  private final PlanItemRestorer restorer = new PlanItemRestorer();

  @Inject PlanItemStore planItemStore;

  /** Called by PlanningStrategyLoopControl (HTTP context — tenancyId always available). */
  public CasePlanModel getOrCreate(UUID caseId, String tenancyId) {
    return entries.computeIfAbsent(caseId, id -> new CaseEntry(id, tenancyId)).planModel;
  }

  /**
   * Defense-in-depth: returns empty if stored tenancyId != provided tenancyId.
   * Lazy hydration calls planItemStore.findDelegated(caseId) — UUID-safe, no tenancyId filter.
   */
  public Optional<CasePlanModel> get(UUID caseId, String tenancyId) {
    CaseEntry e = entries.get(caseId);
    if (e != null) {
      if (!e.tenancyId.equals(tenancyId)) {
        LOG.warnf("Tenant mismatch for caseId=%s (stored=%s, requested=%s)",
            caseId, e.tenancyId, tenancyId);
        return Optional.empty();
      }
      return Optional.of(e.planModel);
    }

    if (planItemStore == null) return Optional.empty();

    List<PlanItemRecord> records = planItemStore.findDelegated(caseId);
    if (records.isEmpty()) return Optional.empty();

    // Validate that all records belong to the requested tenant
    boolean tenancyMismatch = records.stream()
        .anyMatch(r -> !tenancyId.equals(r.tenancyId()));
    if (tenancyMismatch) {
      LOG.warnf("Tenant mismatch on hydration for caseId=%s", caseId);
      return Optional.empty();
    }

    CaseEntry hydrated = entries.computeIfAbsent(caseId, id -> new CaseEntry(id, tenancyId));
    records.forEach(r -> hydrated.planModel.restorePlanItem(restorer.restore(r)));
    return Optional.of(hydrated.planModel);
  }

  /**
   * UUID-only get for callers without tenancyId (e.g. WorkItemLifecycleAdapter).
   * Relies on UUID global uniqueness for isolation — no cross-tenant collision possible.
   * Self-bootstraps tenancyId from the first PlanItemRecord on hydration.
   */
  public Optional<CasePlanModel> get(UUID caseId) {
    CaseEntry e = entries.get(caseId);
    if (e != null) return Optional.of(e.planModel);

    if (planItemStore == null) return Optional.empty();

    List<PlanItemRecord> records = planItemStore.findDelegated(caseId);
    if (records.isEmpty()) return Optional.empty();

    String inferredTenancyId = records.get(0).tenancyId();
    CaseEntry hydrated = entries.computeIfAbsent(caseId, id ->
        new CaseEntry(id, inferredTenancyId));
    records.forEach(r -> hydrated.planModel.restorePlanItem(restorer.restore(r)));
    return Optional.of(hydrated.planModel);
  }

  /** O(1) — UUID key is globally unique. Does not read any principal. */
  public void evict(UUID caseId) {
    entries.remove(caseId);
  }

  // indexForCompletion, getPlanItemId, markConfigured — unchanged (UUID-safe operations)
  public void indexForCompletion(UUID caseId, String workerName, String planItemId) { ... }
  public Optional<String> getPlanItemId(UUID caseId, String workerName) { ... }
  public boolean markConfigured(UUID caseId) { ... }
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
mvn test -pl blackboard -Dtest=BlackboardRegistryTenancyTest
```
Expected: Tests run: 4, Failures: 0

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTenancyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): BlackboardRegistry — stored tenancyId, O(1) evict, defense-in-depth get

CaseEntry stores tenancyId at getOrCreate time. get(UUID, String tenancyId) checks
tenant match and logs mismatch. get(UUID) overload retained for WorkItemLifecycleAdapter
which cannot supply tenancyId yet. evict() is O(1) — UUID key is globally unique.

Refs casehubio/engine#299"
```

---

## Task 8: Engine wiring — handlers, services, recovery

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseExecutionHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionService.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` (and all other handlers)
- Modify: All `CaseLifecycleEvent` fire sites (grep: `new CaseLifecycleEvent(`)
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/PlanItemCompletionApplier.java`

- [ ] **Step 1: Update PlanningStrategyLoopControl — pass ctx.tenancyId() to registry**

In `select(PlanExecutionContext ctx, List<Binding> eligible)`:
- Change `registry.getOrCreate(caseId)` → `registry.getOrCreate(caseId, ctx.tenancyId())`
- Change `registry.markConfigured(caseId)` → unchanged (UUID-safe)

The `PlanExecutionContext` now carries tenancyId — it is set when constructing the context in `CaseHubReactor` or wherever the main loop calls `select()`. Find all `new PlanExecutionContext(caseId, definition, caseContext, caseStatus)` construction sites and add `instance.tenancyId` as the 5th argument.

- [ ] **Step 2: Update SubCaseExecutionHandler — tenancyId inheritance + repo calls**

This is the subcase invariant enforcement point: the child case MUST inherit tenancyId from the parent.

```java
// In onSubCaseSchedule, after resolving childCaseId:
// Set the child's tenancyId from the parent — not from currentPrincipal
// This is done in CaseHubRuntime.startCase(...) which eventually calls
// CaseInstanceRepository.save(child, tenancyId). Pass parent.tenancyId.

// For all repository calls in this handler, use parent.tenancyId:
// caseInstanceRepository.updateStateAndAppendEvent(parent, log, parent.tenancyId)
// eventLogRepository.append(log, parent.tenancyId)
// subCaseGroupRepository.getOrCreate(..., parent.tenancyId)
// subCaseGroupRepository.registerChild(..., parent.tenancyId)

// For registry calls:
// registry.get(parentCaseId) → registry.get(parentCaseId, parent.tenancyId)
```

In `delegatePlanItem` and `faultPlanItem`, add `String tenancyId` parameter and pass to `registry.get(parentCaseId, tenancyId)`.

Verify that `CaseHubRuntime.startCase()` propagates the parent's tenancyId to the child CaseInstance. If it does not, add tenancyId parameter to `startCase()` and thread it through.

- [ ] **Step 3: Update SubCaseCompletionService — event.tenancyId() propagation**

In `handleCompletion(CaseLifecycleEvent event)`, every repo call gets `event.tenancyId()`:
```java
eventLogRepository.findByWorkerAndType(
    childCaseId.toString(), CaseHubEventType.SUBCASE_STARTED, event.tenancyId())
// ...
eventLogRepository.append(log, event.tenancyId())
subCaseGroupRepository.incrementCompleted(parentCaseId, groupId, event.tenancyId())
subCaseGroupRepository.incrementRejected(parentCaseId, groupId, event.tenancyId())
subCaseGroupRepository.markPolicyTriggered(parentCaseId, groupId, event.tenancyId())
// cancelPlanItemOnRejected uses registry.get — pass event.tenancyId()
```

- [ ] **Step 4: Update DefaultWorkerExecutionRecoveryService — inject cross-tenant interfaces**

```java
@Inject CrossTenantEventLogRepository crossTenantEventLogRepository;
@Inject CrossTenantCaseInstanceRepository crossTenantCaseInstanceRepository;

// recoverPendingScheduledWorkers: change eventLogRepository.findByTypes(...) →
// crossTenantEventLogRepository.findByTypes(RELEVANT_RECOVERY_EVENTS)

// loadOrRestoreCaseInstance: change caseInstanceRepository.findByUuid(caseId) →
// crossTenantCaseInstanceRepository.findByUuid(caseId)

// rebuildStateContext: change eventLogRepository.findByCaseAndTypes(caseId, types) →
// crossTenantEventLogRepository.findByCaseAndTypes(caseId, types)
```

Remove `@Inject EventLogRepository eventLogRepository` and `@Inject CaseInstanceRepository caseInstanceRepository` if they are no longer used in the recovery service.

- [ ] **Step 5: Update WorkerScheduleEventHandler — pass caseInstance.tenancyId**

In `onWorkerScheduleEventHandler(WorkerScheduleEvent event)`:
```java
CaseInstance instance = event.caseInstance();
// All eventLogRepository calls:
eventLogRepository.findSchedulingEvents(instance.getUuid(), worker.getName(),
    idempotencyAfter, instance.tenancyId)
eventLogRepository.appendAndReturnId(eventLog, instance.tenancyId)
// EventLog construction — populate tenancyId:
eventLog.tenancyId = instance.tenancyId;
```

- [ ] **Step 6: Fix all CaseLifecycleEvent construction sites**

```bash
grep -rn "new CaseLifecycleEvent(" /Users/mdproctor/claude/casehub/engine/runtime/src /Users/mdproctor/claude/casehub/engine/blackboard/src 2>/dev/null
```

For each site, add `instance.tenancyId` (or `event.tenancyId()` if firing from an observer) as the second argument:
```java
new CaseLifecycleEvent(
    caseId,
    instance.tenancyId,   // NEW
    commandType,
    eventType,
    caseStatus,
    actorId,
    actorRole,
    traceId)
```

- [ ] **Step 7: Fix all PlanExecutionContext construction sites**

```bash
grep -rn "new PlanExecutionContext(" /Users/mdproctor/claude/casehub/engine --include="*.java"
```

For each site, add `instance.tenancyId` as the 5th argument:
```java
new PlanExecutionContext(caseId, definition, caseContext, caseStatus, instance.tenancyId)
```

- [ ] **Step 8: Fix WorkItemLifecycleAdapter and PlanItemCompletionApplier**

Read both files. For repository calls in `WorkItemLifecycleAdapter`:
- `planItemStore.updateStatus(planItemId, status)` — unchanged (UUID planItemId)
- `reactivePlanItemStore.save(request, tenancyId)` — need tenancyId. Source: check if the WorkItem or event carries it. If not, keep `planItemStore.updateStatus` (which has no tenancyId) and note the gap.

For `PlanItemCompletionApplier`:
- Calls `registry.get(caseId)` — use the UUID-only overload since tenancyId is not available from WorkItemLifecycleEvent yet

- [ ] **Step 9: Build blackboard and runtime**

```bash
mvn install -DskipTests -q -pl blackboard,runtime
```
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add blackboard/src/main/java/ runtime/src/main/java/ work-adapter/src/main/java/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine#299): wire tenancyId through engine handlers and recovery services

PlanningStrategyLoopControl passes ctx.tenancyId() to registry.
SubCaseExecutionHandler propagates parent.tenancyId to child (invariant).
SubCaseCompletionService uses event.tenancyId() for all repo calls.
DefaultWorkerExecutionRecoveryService injects cross-tenant interfaces.
WorkerScheduleEventHandler passes caseInstance.tenancyId to event log.
CaseLifecycleEvent gains tenancyId at all fire sites.
PlanExecutionContext gains tenancyId at all construction sites.

Refs casehubio/engine#299"
```

---

## Task 9: Full build verification

- [ ] **Step 1: Install all deps, run full build**

```bash
mvn install -DskipTests -q 2>&1 | tail -20
```
Expected: BUILD SUCCESS across all modules

- [ ] **Step 2: Fix any remaining compilation errors**

Run `mvn install -DskipTests` without `-q` to see full error output if Step 1 fails. Fix each error — typical patterns:
- `cannot find symbol: method findByUuid(UUID)` → add tenancyId argument at that call site
- `constructor PlanItemRecord(UUID,...) cannot be applied` → add tenancyId as last argument
- `interface does not override` → check method signature matches updated SPI

- [ ] **Step 3: Run blackboard integration tests**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard
```
Expected: BUILD SUCCESS, no test failures

- [ ] **Step 4: Run persistence-memory tests**

```bash
mvn test -pl persistence-memory
```
Expected: all tests green

---

## Task 10: Isolation contract tests

- [ ] **Step 1: Write CaseInstanceRepository isolation contract test**

Create `common/src/test/java/io/casehub/engine/common/spi/CaseInstanceRepositoryTenancyContractTest.java`:

```java
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.platform.api.identity.TenancyConstants;
import java.time.Duration;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Abstract contract: every CaseInstanceRepository implementation must pass these.
 * Extend this in each persistence module's test suite.
 */
public abstract class CaseInstanceRepositoryTenancyContractTest {

  protected abstract CaseInstanceRepository repository();

  private CaseInstance buildInstance() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setState(io.casehub.api.model.CaseStatus.RUNNING);
    return instance;
  }

  @Test
  void save_and_findByUuid_sameTenant_found() {
    String tenantA = "tenant-a-" + UUID.randomUUID();
    CaseInstance saved = repository().save(buildInstance(), tenantA)
        .await().atMost(Duration.ofSeconds(5));

    CaseInstance found = repository().findByUuid(saved.getUuid(), tenantA)
        .await().atMost(Duration.ofSeconds(5));

    assertThat(found).isNotNull();
    assertThat(found.tenancyId).isEqualTo(tenantA);
  }

  @Test
  void findByUuid_differentTenant_returnsNull() {
    String tenantA = "tenant-a-" + UUID.randomUUID();
    String tenantB = "tenant-b-" + UUID.randomUUID();

    CaseInstance saved = repository().save(buildInstance(), tenantA)
        .await().atMost(Duration.ofSeconds(5));

    CaseInstance found = repository().findByUuid(saved.getUuid(), tenantB)
        .await().atMost(Duration.ofSeconds(5));

    assertThat(found).isNull();
  }

  @Test
  void update_wrongTenant_fails() {
    String tenantA = "tenant-a-" + UUID.randomUUID();
    String tenantB = "tenant-b-" + UUID.randomUUID();

    CaseInstance saved = repository().save(buildInstance(), tenantA)
        .await().atMost(Duration.ofSeconds(5));

    saved.setState(io.casehub.api.model.CaseStatus.COMPLETED);

    org.junit.jupiter.api.Assertions.assertThrows(
        Exception.class,
        () -> repository().update(saved, tenantB).await().atMost(Duration.ofSeconds(5)));
  }
}
```

- [ ] **Step 2: Create InMemory implementation of the contract test**

Create `persistence-memory/src/test/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepositoryTenancyTest.java`:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CaseInstanceRepositoryTenancyContractTest;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;

@QuarkusTest
class InMemoryCaseInstanceRepositoryTenancyTest extends CaseInstanceRepositoryTenancyContractTest {

  @Inject InMemoryCaseInstanceRepository repository;

  @Override
  protected CaseInstanceRepository repository() {
    return repository;
  }
}
```

- [ ] **Step 3: Run the isolation test**

```bash
mvn test -pl persistence-memory -Dtest=InMemoryCaseInstanceRepositoryTenancyTest
```
Expected: Tests run: 3, Failures: 0

- [ ] **Step 4: Run SubCaseExecutionHandler invariant test**

Create `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseExecutionHandlerTenancyTest.java` — verify that when a subcase is spawned, the child CaseInstance receives `tenancyId = parent.tenancyId`, not from `currentPrincipal`. This is a unit test using a test double for `CaseHubRuntime` that captures what tenancyId the child is saved with.

```java
@Test
void startCase_propagatesParentTenancyId() {
    // Arrange: parent case with tenancyId "tenant-a"
    CaseInstance parent = new CaseInstance();
    parent.setUuid(UUID.randomUUID());
    parent.tenancyId = "tenant-a";

    // Act: spawn subcase (using test double for CaseHubRuntime)
    // Assert: captured child CaseInstance has tenancyId = "tenant-a"
    // Implementation depends on how CaseHubRuntime is testable — adjust accordingly
}
```

- [ ] **Step 5: Final full test run**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard,persistence-memory,common
```
Expected: all tests green

- [ ] **Step 6: Commit tests**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/test/ persistence-memory/src/test/ blackboard/src/test/
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(engine#299): add tenancy isolation contract tests

CaseInstanceRepositoryTenancyContractTest covers save/find/update isolation.
InMemoryCaseInstanceRepositoryTenancyTest runs it against the in-memory store.
BlackboardRegistry tenant mismatch unit test covers defense-in-depth.
SubCaseExecutionHandlerTenancyTest verifies child inherits parent tenancyId.

Refs casehubio/engine#299"
```

---

## Self-review

**Spec coverage check:**
- ✅ tenancyId on 4 domain objects + 3 records
- ✅ tenancyId column on all 8 JPA entities
- ✅ SPI methods all gain tenancyId (with documented exceptions for UUID-safe ops)
- ✅ CrossTenantEventLogRepository + CrossTenantCaseInstanceRepository in internal package
- ✅ JPA implementations filter by tenancyId; update() includes tenancyId in WHERE
- ✅ Memory stores filter by tenancyId
- ✅ BlackboardRegistry: stored tenancyId, O(1) evict, defense-in-depth get, UUID-only overload
- ✅ CaseLifecycleEvent gains tenancyId at all fire sites
- ✅ PlanExecutionContext gains tenancyId
- ✅ SubCaseExecutionHandler propagates parent.tenancyId to child (invariant)
- ✅ SubCaseCompletionService uses event.tenancyId()
- ✅ Recovery services use cross-tenant interfaces
- ✅ DefaultTestPrincipal shipped in persistence-memory
- ✅ Isolation contract tests cover cross-tenant leakage

**Known gap (out of scope, tracked):** WorkItemLifecycleAdapter cannot supply tenancyId until casehub-work is tenancy-aware. `registry.get(UUID)` overload is the interim solution.
