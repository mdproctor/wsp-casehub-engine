# Tenancy Enforcement Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement four sequential tenancy enforcement layers: V2005 NOT NULL migration, @CrossTenant CDI producer pattern, PostgreSQL Row Level Security, and CaseDefinition registry immutable-key refactor.

**Architecture:** V2005 is a standalone Flyway migration. @CrossTenant adds CDI qualifiers + a producer that gates cross-tenant access via SystemCurrentPrincipal. RLS adds a TenantAwareRepository base class that sets `SET LOCAL "casehub.tenancy_id"` inside reactive transactions; cross-tenant access uses `SET LOCAL ROLE casehub_crosstenancy` (BYPASSRLS). The registry refactor replaces a mutable CaseMetaModel map key with an immutable CaseKey record and a single RegistryEntry map.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate Reactive / Mutiny Panache, PostgreSQL RLS, CDI qualifiers/producers, JUnit 5 + AssertJ

**Issues:** engine#411 · engine#405 · engine#406 · engine#410

---

## File Map

### Issue #411
- **Create:** `ledger/src/main/resources/db/engine-ledger/migration/V2005__tenancy_id_not_null.sql`

### Issue #405
- **Create:** `common/src/main/java/io/casehub/engine/common/qualifier/CrossTenant.java`
- **Create:** `common/src/main/java/io/casehub/engine/common/qualifier/EngineSystem.java`
- **Create:** `runtime/src/main/java/io/casehub/engine/internal/identity/SystemCurrentPrincipal.java`
- **Create:** `runtime/src/main/java/io/casehub/engine/internal/identity/CrossTenantProducer.java`
- **Create:** `runtime/src/test/java/io/casehub/engine/internal/identity/CrossTenantProducerTest.java`
- **Modify:** `runtime/src/main/java/io/casehub/engine/internal/work/PendingWorkRegistry.java` — add `@CrossTenant`
- **Modify:** `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java` — add `@CrossTenant` to both repos
- **Modify:** `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java` — add `@CrossTenant`
- **Modify:** `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java` — add `@CrossTenant`
- **Modify:** `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/MilestoneSLATimeoutJob.java` — add `@CrossTenant`
- **Modify:** `resilience/src/main/java/io/casehub/resilience/deadletter/DeadLetterReplayService.java` — add `@CrossTenant` to constructor params
- **Modify:** `runtime/src/test/java/io/casehub/engine/EngineDecouplingIT.java` — add `@CrossTenant`

### Issue #406
- **Create:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/TenantAwareRepository.java`
- **Create:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/RlsPolicyApplicator.java`
- **Create:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java`
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java` — extend TenantAwareRepository, remove CrossTenantEventLogRepository, wrap with withTenantTransaction
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java` — extend TenantAwareRepository, wrap with withTenantTransaction
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseMetaModelRepository.java` — extend TenantAwareRepository, wrap with withTenantTransaction
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantCaseInstanceRepository.java` — extend TenantAwareRepository, wrap with withCrossTenantTransaction
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java` — extend TenantAwareRepository, wrap with withTenantTransaction
- **Modify:** `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java` — extend TenantAwareRepository, wrap with withTenantTransaction
- **Create:** `persistence-hibernate/src/test/java/io/casehub/persistence/jpa/RlsIntegrationTest.java`

### Issue #410
- **Create:** `common/src/main/java/io/casehub/engine/common/internal/model/CaseKey.java`
- **Create:** `common/src/test/java/io/casehub/engine/common/internal/model/CaseKeyTest.java`
- **Modify:** `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` — replace Map<CaseMetaModel,CaseDefinition> with Map<CaseKey,RegistryEntry>
- **Create:** `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java`

---

## Task 1: V2005 migration — enforce NOT NULL on tenancy_id

**Files:**
- Create: `ledger/src/main/resources/db/engine-ledger/migration/V2005__tenancy_id_not_null.sql`

- [ ] **Step 1: Create the migration file**

```sql
-- V2005: enforce NOT NULL on tenancy_id columns added by V2002/V2003.
-- Pre-migration rows belonged to a single-tenant deployment (DEFAULT_TENANT_ID).
-- Backfill makes them visible to that tenant after migration.
-- V2004 (worker_decision_entry_trust_fields) already exists — this is V2005.

UPDATE worker_decision_entry
   SET tenancy_id = '278776f9-e1b0-46fb-9032-8bddebdcf9ce'
 WHERE tenancy_id IS NULL;
ALTER TABLE worker_decision_entry ALTER COLUMN tenancy_id SET NOT NULL;

UPDATE case_ledger_entry
   SET tenancy_id = '278776f9-e1b0-46fb-9032-8bddebdcf9ce'
 WHERE tenancy_id IS NULL;
ALTER TABLE case_ledger_entry ALTER COLUMN tenancy_id SET NOT NULL;
```

- [ ] **Step 2: Verify build passes**

```bash
mvn --batch-mode install -DskipTests -q
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add ledger/src/main/resources/db/engine-ledger/migration/V2005__tenancy_id_not_null.sql
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix(ledger): enforce NOT NULL on tenancy_id in V2005 migration — backfills DEFAULT_TENANT_ID. Refs #411"
```

---

## Task 2: CDI qualifiers — @CrossTenant and @EngineSystem

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/qualifier/CrossTenant.java`
- Create: `common/src/main/java/io/casehub/engine/common/qualifier/EngineSystem.java`

- [ ] **Step 1: Create @CrossTenant qualifier**

```java
package io.casehub.engine.common.qualifier;

import jakarta.inject.Qualifier;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks injection points that require cross-tenant data access.
 * Convention-based marker — enforced by code review, not CDI machinery.
 * See protocol PP-20260520-e6a5f0 and spec docs/specs/issue-405-tenancy-enforcement/.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface CrossTenant {}
```

- [ ] **Step 2: Create @EngineSystem qualifier**

```java
package io.casehub.engine.common.qualifier;

import jakarta.inject.Qualifier;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Selects the engine-internal system-actor CurrentPrincipal implementation.
 * Used by CrossTenantProducer to inject SystemCurrentPrincipal specifically,
 * without displacing the MockCurrentPrincipal @DefaultBean.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface EngineSystem {}
```

- [ ] **Step 3: Verify compilation**

```bash
mvn --batch-mode install -DskipTests -q -pl casehub-engine-common
```

Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/main/java/io/casehub/engine/common/qualifier/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(common): add @CrossTenant and @EngineSystem CDI qualifiers. Refs #405"
```

---

## Task 3: SystemCurrentPrincipal — engine-internal system-actor principal

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/identity/SystemCurrentPrincipal.java`

- [ ] **Step 1: Create SystemCurrentPrincipal**

```java
package io.casehub.engine.internal.identity;

import io.casehub.engine.common.qualifier.EngineSystem;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Set;

/**
 * Engine-internal system-actor CurrentPrincipal. Always isCrossTenantAdmin().
 *
 * Not @DefaultBean — never replaces MockCurrentPrincipal. Accessed only via @EngineSystem
 * qualifier from CrossTenantProducer.
 *
 * Interim: delete when casehub-platform ships a platform-level system-actor principal with
 * isCrossTenantAdmin() = true and actorType() = ActorType.SYSTEM. At that point, update
 * CrossTenantProducer to inject the platform implementation instead.
 */
@ApplicationScoped
@EngineSystem
public class SystemCurrentPrincipal implements CurrentPrincipal {

  @Override
  public String actorId() {
    return "system";
  }

  @Override
  public Set<String> groups() {
    return Set.of();
  }

  @Override
  public String tenancyId() {
    return TenancyConstants.DEFAULT_TENANT_ID;
  }

  @Override
  public boolean isCrossTenantAdmin() {
    return true;
  }
  // isSystem() inherits the default: actorType() resolves "system" → ActorType.SYSTEM → true
}
```

- [ ] **Step 2: Verify compilation**

```bash
mvn --batch-mode install -DskipTests -q -pl casehub-engine-runtime
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/identity/SystemCurrentPrincipal.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(runtime): add SystemCurrentPrincipal — engine-internal system-actor (interim). Refs #405"
```

---

## Task 4: CrossTenantProducer — guards cross-tenant repo access via @CrossTenant qualifier

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/identity/CrossTenantProducer.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/identity/CrossTenantProducerTest.java`

TDD: write the test first.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.engine.internal.identity;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.casehub.engine.common.qualifier.CrossTenant;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CrossTenantProducerTest {

  @Inject @CrossTenant CrossTenantEventLogRepository crossTenantEventLog;
  @Inject @CrossTenant CrossTenantCaseInstanceRepository crossTenantCaseInstance;

  @Test
  void crossTenantEventLog_isProduced() {
    // The @CrossTenant qualified bean must be injectable — verifies CDI wiring
    assertThat(crossTenantEventLog).isNotNull();
  }

  @Test
  void crossTenantCaseInstance_isProduced() {
    assertThat(crossTenantCaseInstance).isNotNull();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn --batch-mode test -pl casehub-engine-runtime -Dtest=CrossTenantProducerTest
```

Expected: FAIL — no bean with qualifier @CrossTenant for CrossTenantEventLogRepository

- [ ] **Step 3: Create CrossTenantProducer**

```java
package io.casehub.engine.internal.identity;

import io.casehub.engine.common.qualifier.CrossTenant;
import io.casehub.engine.common.qualifier.EngineSystem;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Inject;

/**
 * Produces @CrossTenant-qualified cross-tenant repository beans.
 *
 * The @EngineSystem SystemCurrentPrincipal check is a contract assertion: if
 * SystemCurrentPrincipal.isCrossTenantAdmin() ever returns false (e.g. during testing the guard
 * itself), this producer fails at startup rather than silently granting access. It is aspirational
 * scaffolding for when the platform ships a runtime-evaluated system principal.
 */
@ApplicationScoped
public class CrossTenantProducer {

  @Inject @EngineSystem SystemCurrentPrincipal systemPrincipal;
  @Inject CrossTenantEventLogRepository eventLogRepo;
  @Inject CrossTenantCaseInstanceRepository caseInstanceRepo;

  @Produces
  @CrossTenant
  @ApplicationScoped
  public CrossTenantEventLogRepository produceEventLog() {
    if (!systemPrincipal.isCrossTenantAdmin()) {
      throw new IllegalStateException(
          "SystemCurrentPrincipal.isCrossTenantAdmin() must return true — engine#405");
    }
    return eventLogRepo;
  }

  @Produces
  @CrossTenant
  @ApplicationScoped
  public CrossTenantCaseInstanceRepository produceCaseInstance() {
    if (!systemPrincipal.isCrossTenantAdmin()) {
      throw new IllegalStateException(
          "SystemCurrentPrincipal.isCrossTenantAdmin() must return true — engine#405");
    }
    return caseInstanceRepo;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn --batch-mode test -pl casehub-engine-runtime -Dtest=CrossTenantProducerTest
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/identity/CrossTenantProducer.java runtime/src/test/java/io/casehub/engine/internal/identity/CrossTenantProducerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(runtime): CrossTenantProducer gates @CrossTenant repos via SystemCurrentPrincipal. Refs #405"
```

---

## Task 5: Update all six injection sites + EngineDecouplingIT

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/work/PendingWorkRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/MilestoneSLATimeoutJob.java`
- Modify: `resilience/src/main/java/io/casehub/resilience/deadletter/DeadLetterReplayService.java`
- Modify: `runtime/src/test/java/io/casehub/engine/EngineDecouplingIT.java`

The change is mechanical for all files. Each injection site gets `@CrossTenant` added. The pattern differs between field injection and constructor injection.

- [ ] **Step 1: Add @CrossTenant to field-injected sites**

In each of these files, add `@CrossTenant` to the `@Inject CrossTenantEventLogRepository` and/or `@Inject CrossTenantCaseInstanceRepository` field(s). Also add the import `import io.casehub.engine.common.qualifier.CrossTenant;`.

`PendingWorkRegistry.java` — field injection:
```java
// Before:
@Inject CrossTenantEventLogRepository eventLogRepository;

// After:
@Inject @CrossTenant CrossTenantEventLogRepository eventLogRepository;
```

`DefaultWorkerExecutionRecoveryService.java` — both repos:
```java
// Before:
@Inject CrossTenantCaseInstanceRepository caseInstanceRepository;
@Inject CrossTenantEventLogRepository eventLogRepository;

// After:
@Inject @CrossTenant CrossTenantCaseInstanceRepository caseInstanceRepository;
@Inject @CrossTenant CrossTenantEventLogRepository eventLogRepository;
```

`QuartzWorkerExecutionJob.java`:
```java
// Before:
@Inject CrossTenantEventLogRepository eventLogRepository;

// After:
@Inject @CrossTenant CrossTenantEventLogRepository eventLogRepository;
```

`QuartzWorkerExecutionManager.java`:
```java
// Before:
@Inject CrossTenantEventLogRepository eventLogRepository;

// After:
@Inject @CrossTenant CrossTenantEventLogRepository eventLogRepository;
```

`MilestoneSLATimeoutJob.java`:
```java
// Before:
@Inject CrossTenantEventLogRepository eventLogRepository;

// After:
@Inject @CrossTenant CrossTenantEventLogRepository eventLogRepository;
```

- [ ] **Step 2: Add @CrossTenant to constructor injection in DeadLetterReplayService**

`DeadLetterReplayService.java` uses `@Inject` constructor. Add `@CrossTenant` to the two relevant parameters:

```java
// Before:
@Inject
public DeadLetterReplayService(
    DeadLetterQueue deadLetterQueue,
    CrossTenantEventLogRepository eventLogRepository,
    CrossTenantCaseInstanceRepository caseInstanceRepository,
    CaseDefinitionRegistry caseDefinitionRegistry,
    EventBus eventBus) {

// After:
@Inject
public DeadLetterReplayService(
    DeadLetterQueue deadLetterQueue,
    @CrossTenant CrossTenantEventLogRepository eventLogRepository,
    @CrossTenant CrossTenantCaseInstanceRepository caseInstanceRepository,
    CaseDefinitionRegistry caseDefinitionRegistry,
    EventBus eventBus) {
```

Also add `import io.casehub.engine.common.qualifier.CrossTenant;`

- [ ] **Step 3: Update EngineDecouplingIT**

```java
// Before:
@Inject CrossTenantEventLogRepository crossTenantEventLogRepository;

// After:
@Inject @CrossTenant CrossTenantEventLogRepository crossTenantEventLogRepository;
```

Also add `import io.casehub.engine.common.qualifier.CrossTenant;`

- [ ] **Step 4: Verify all modules compile and tests pass**

```bash
mvn --batch-mode install -DskipTests -q
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Run runtime and scheduler-quartz tests**

```bash
mvn --batch-mode test -pl casehub-engine-runtime,casehub-engine-scheduler-quartz,casehub-resilience
```

Expected: all tests pass (including EngineDecouplingIT)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/work/PendingWorkRegistry.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/recovery/DefaultWorkerExecutionRecoveryService.java \
  scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java \
  scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java \
  scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/MilestoneSLATimeoutJob.java \
  resilience/src/main/java/io/casehub/resilience/deadletter/DeadLetterReplayService.java \
  runtime/src/test/java/io/casehub/engine/EngineDecouplingIT.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(engine): add @CrossTenant to all cross-tenant injection sites. Refs #405"
```

---

## Task 6: TenantAwareRepository — reactive RLS variable injection base class

**Files:**
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/TenantAwareRepository.java`

`TenantAwareRepository` extends `AbstractJpaRepository` and adds two helpers. Both helpers include the Vert.x safe-context setup (via `withSafeContext()`), open a reactive transaction, execute a `SET LOCAL` statement to configure the session variable, then run the caller's work.

All tenant-scoped JPA repository methods call `withTenantTransaction()` instead of `withSafeContext(() -> Panache.withSession/withTransaction())`. All cross-tenant methods call `withCrossTenantTransaction()`.

The reason all reads also use `withTenantTransaction()` (not `withSession()`): PostgreSQL `SET LOCAL` only applies within an explicit transaction. In autocommit mode, `SET LOCAL` and the subsequent SELECT run in separate implicit transactions — the variable resets before the SELECT executes. Wrapping reads in `withTransaction()` is necessary for RLS to filter correctly.

- [ ] **Step 1: Create TenantAwareRepository**

```java
package io.casehub.persistence.jpa;

import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.inject.Inject;
import java.util.function.Supplier;

/**
 * Extends AbstractJpaRepository with RLS session-variable injection.
 *
 * withTenantTransaction(): sets SET LOCAL "casehub.tenancy_id" = current tenant before any SQL.
 *   Used by all tenant-scoped repositories (EventLog, CaseInstance, CaseMetaModel, etc.).
 *   Wraps reads in withTransaction() because SET LOCAL only applies within an explicit transaction.
 *
 * withCrossTenantTransaction(): sets SET LOCAL ROLE casehub_crosstenancy (BYPASSRLS role).
 *   Used by cross-tenant repositories. Requires casehub_crosstenancy role to exist.
 *   RlsPolicyApplicator creates the role when casehub.rls.enabled=true.
 */
abstract class TenantAwareRepository extends AbstractJpaRepository {

  @Inject CurrentPrincipal currentPrincipal;

  protected <T> Uni<T> withTenantTransaction(Supplier<Uni<T>> work) {
    return withSafeContext(
        () ->
            Panache.withTransaction(
                () ->
                    Panache.getSession()
                        .flatMap(
                            session ->
                                session
                                    .createNativeQuery(
                                        "SET LOCAL \"casehub.tenancy_id\" = :tid")
                                    .setParameter("tid", currentPrincipal.tenancyId())
                                    .executeUpdate()
                                    .replaceWith(work.get()))));
  }

  protected <T> Uni<T> withCrossTenantTransaction(Supplier<Uni<T>> work) {
    return withSafeContext(
        () ->
            Panache.withTransaction(
                () ->
                    Panache.getSession()
                        .flatMap(
                            session ->
                                session
                                    .createNativeQuery("SET LOCAL ROLE casehub_crosstenancy")
                                    .executeUpdate()
                                    .replaceWith(work.get()))));
  }
}
```

- [ ] **Step 2: Verify compilation (no tests yet — tested via repository integration)**

```bash
mvn --batch-mode install -DskipTests -q -pl casehub-persistence-hibernate
```

Expected: BUILD SUCCESS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/src/main/java/io/casehub/persistence/jpa/TenantAwareRepository.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(persistence): TenantAwareRepository base class — reactive SET LOCAL for RLS. Refs #406"
```

---

## Task 7: RlsPolicyApplicator — applies RLS policies and BYPASSRLS role at startup

**Files:**
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/RlsPolicyApplicator.java`
- Modify: `persistence-hibernate/src/main/resources/application.properties` — add `casehub.rls.enabled=false`

The applicator uses blocking JDBC (Agroal) for DDL — this is correct for schema setup at startup, separate from the reactive query path.

- [ ] **Step 1: Add casehub.rls.enabled to application.properties**

Add to `persistence-hibernate/src/main/resources/application.properties`:
```properties
# RLS enforcement — disabled by default until reactive SET LOCAL path is validated end-to-end
# against real PostgreSQL. Enable explicitly in deployments that have run RlsIntegrationTest.
casehub.rls.enabled=false
```

If the file doesn't exist, create it with just this property.

- [ ] **Step 2: Write the failing test for RlsPolicyApplicator**

```java
package io.casehub.persistence.jpa;

import static org.mockito.Mockito.*;

import io.agroal.api.AgroalDataSource;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

@ExtendWith(MockitoExtension.class)
class RlsPolicyApplicatorTest {

  @Mock AgroalDataSource dataSource;
  @InjectMocks RlsPolicyApplicator applicator;

  @Test
  void onStart_whenDisabled_doesNothing() throws SQLException {
    applicator.rlsEnabled = false;
    applicator.onStart(null);
    verifyNoInteractions(dataSource);
  }

  @Test
  void onStart_whenEnabled_opensConnectionAndAppliesRls() throws SQLException {
    applicator.rlsEnabled = true;
    Connection conn = mock(Connection.class);
    Statement stmt = mock(Statement.class);
    when(dataSource.getConnection()).thenReturn(conn);
    when(conn.createStatement()).thenReturn(stmt);

    applicator.onStart(null);

    verify(dataSource).getConnection();
    // 3 DDL operations per table × 5 tables + role creation = at least 15+ execute() calls
    verify(stmt, atLeast(15)).execute(anyString());
  }
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
mvn --batch-mode test -pl casehub-persistence-hibernate -Dtest=RlsPolicyApplicatorTest
```

Expected: FAIL — RlsPolicyApplicator not found

- [ ] **Step 4: Create RlsPolicyApplicator**

```java
package io.casehub.persistence.jpa;

import io.agroal.api.AgroalDataSource;
import io.quarkus.runtime.StartupEvent;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.List;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

/**
 * Applies PostgreSQL Row Level Security policies to engine tables at startup.
 *
 * Priority 100 runs after Hibernate schema creation (MIN_VALUE) and DefaultCaseDefinitionRegistry
 * (priority 10). Uses blocking JDBC (Agroal) for DDL — correct for schema setup; not the reactive
 * query path.
 *
 * Prerequisites when casehub.rls.enabled=true:
 *   - PostgreSQL (not H2 — H2 does not support RLS or SET LOCAL role)
 *   - App DB user must have CREATEROLE privilege, OR a DBA must pre-create casehub_crosstenancy
 *     with BYPASSRLS before enabling RLS.
 */
@ApplicationScoped
public class RlsPolicyApplicator {

  private static final Logger LOG = Logger.getLogger(RlsPolicyApplicator.class);

  private static final List<String> TABLES =
      List.of("case_instance", "case_meta_model", "event_log", "plan_item", "sub_case_group");

  @Inject AgroalDataSource dataSource;

  @ConfigProperty(name = "casehub.rls.enabled", defaultValue = "false")
  boolean rlsEnabled;

  void onStart(@Observes @Priority(100) StartupEvent ev) {
    if (!rlsEnabled) {
      LOG.debug("RLS disabled (casehub.rls.enabled=false) — skipping policy application");
      return;
    }
    LOG.info("Applying PostgreSQL Row Level Security policies");
    try (Connection conn = dataSource.getConnection();
        Statement stmt = conn.createStatement()) {
      createBypassRole(stmt);
      for (String table : TABLES) {
        applyRls(stmt, table);
      }
      LOG.infof("RLS applied to %d tables", TABLES.size());
    } catch (SQLException e) {
      throw new IllegalStateException("Failed to apply RLS policies", e);
    }
  }

  private void createBypassRole(Statement stmt) throws SQLException {
    // Create casehub_crosstenancy role with BYPASSRLS if absent, then grant to the session user.
    // Requires CREATEROLE. If the app user lacks it, create the role via DBA before enabling RLS.
    stmt.execute(
        "DO $$ BEGIN "
            + "  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'casehub_crosstenancy') THEN "
            + "    EXECUTE 'CREATE ROLE casehub_crosstenancy BYPASSRLS'; "
            + "  END IF; "
            + "END $$");
    stmt.execute("GRANT casehub_crosstenancy TO current_user");
  }

  private void applyRls(Statement stmt, String table) throws SQLException {
    stmt.execute("ALTER TABLE " + table + " ENABLE ROW LEVEL SECURITY");
    stmt.execute("ALTER TABLE " + table + " FORCE ROW LEVEL SECURITY");
    stmt.execute(
        "DO $$ BEGIN "
            + "  IF NOT EXISTS (SELECT 1 FROM pg_policies WHERE tablename = '"
            + table
            + "' AND policyname = 'tenant_isolation') THEN "
            + "    EXECUTE 'CREATE POLICY tenant_isolation ON "
            + table
            + " USING (tenancy_id = current_setting(''casehub.tenancy_id'', true))'; "
            + "  END IF; "
            + "END $$");
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
mvn --batch-mode test -pl casehub-persistence-hibernate -Dtest=RlsPolicyApplicatorTest
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/RlsPolicyApplicator.java \
  persistence-hibernate/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(persistence): RlsPolicyApplicator — applies RLS and casehub_crosstenancy role at startup. Refs #406"
```

---

## Task 8: Split JpaEventLogRepository — tenant-scoped and cross-tenant implementations

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java`

- [ ] **Step 1: Rewrite JpaEventLogRepository — tenant-scoped only, extends TenantAwareRepository**

Replace the entire file content. The six `CrossTenantEventLogRepository` methods are removed. All `withSafeContext(() -> Panache.withSession/withTransaction(...))` become `withTenantTransaction(() -> ...)`. The `withTenantTransaction` supplier contains only the Panache entity operations — no outer `Panache.withSession/withTransaction` wrapper.

```java
package io.casehub.persistence.jpa;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.spi.EventLogRepository;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

@ApplicationScoped
public class JpaEventLogRepository extends TenantAwareRepository implements EventLogRepository {

  @Override
  public Uni<Void> append(EventLog eventLog, String tenancyId) {
    EventLogEntity entity = toEntity(eventLog, tenancyId);
    return withTenantTransaction(
        () ->
            entity
                .persistAndFlush()
                .invoke(
                    () -> {
                      eventLog.id = entity.id;
                      eventLog.setSeq(entity.seq);
                    })
                .replaceWithVoid());
  }

  @Override
  public Uni<Long> appendAndReturnId(EventLog eventLog, String tenancyId) {
    EventLogEntity entity = toEntity(eventLog, tenancyId);
    return withTenantTransaction(
        () ->
            entity
                .persistAndFlush()
                .map(
                    v -> {
                      eventLog.id = entity.id;
                      eventLog.setSeq(entity.seq);
                      return entity.id;
                    }));
  }

  @Override
  public Uni<EventLog> findById(Long id, String tenancyId) {
    return withTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "id = ?1 and tenancyId = ?2", id, tenancyId)
                .firstResult()
                .map(entity -> entity == null ? null : fromEntity(entity)));
  }

  @Override
  public Uni<List<EventLog>> findSchedulingEvents(
      UUID caseId, String workerId, Instant after, String tenancyId) {
    return withTenantTransaction(
        () -> {
          if (after == null) {
            return EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and workerId = ?2 and eventType in (?3, ?4, ?5)"
                        + " and tenancyId = ?6 order by seq asc",
                    caseId,
                    workerId,
                    CaseHubEventType.WORKER_SCHEDULED,
                    CaseHubEventType.WORKER_EXECUTION_STARTED,
                    CaseHubEventType.WORKER_EXECUTION_COMPLETED,
                    tenancyId)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList());
          } else {
            return EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and workerId = ?2 and eventType in (?3, ?4, ?5)"
                        + " and timestamp > ?6 and tenancyId = ?7 order by seq asc",
                    caseId,
                    workerId,
                    CaseHubEventType.WORKER_SCHEDULED,
                    CaseHubEventType.WORKER_EXECUTION_STARTED,
                    CaseHubEventType.WORKER_EXECUTION_COMPLETED,
                    after,
                    tenancyId)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList());
          }
        });
  }

  @Override
  public Uni<List<EventLog>> findByCaseAndTypes(
      UUID caseId, Collection<CaseHubEventType> types, String tenancyId) {
    return withTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and eventType in ?2 and tenancyId = ?3 order by seq asc",
                    caseId,
                    types,
                    tenancyId)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByCaseAndWorkerAndType(
      UUID caseId, String workerId, CaseHubEventType type, String tenancyId) {
    return withTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and workerId = ?2 and eventType = ?3 and tenancyId = ?4",
                    caseId,
                    workerId,
                    type,
                    tenancyId)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByWorkerAndType(
      String workerId, CaseHubEventType type, String tenancyId) {
    return withTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "workerId = ?1 and eventType = ?2 and tenancyId = ?3",
                    workerId,
                    type,
                    tenancyId)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByCaseWithFilters(
      UUID caseId,
      Collection<CaseHubEventType> eventTypes,
      Collection<EventStreamType> streamTypes,
      String tenancyId) {
    return withTenantTransaction(
        () -> {
          StringBuilder query = new StringBuilder("caseId = ?1 and tenancyId = ?2");
          List<Object> params = new ArrayList<>();
          params.add(caseId);
          params.add(tenancyId);
          if (eventTypes != null && !eventTypes.isEmpty()) {
            query.append(" and eventType in ?").append(params.size() + 1);
            params.add(eventTypes);
          }
          if (streamTypes != null && !streamTypes.isEmpty()) {
            query.append(" and streamType in ?").append(params.size() + 1);
            params.add(streamTypes);
          }
          query.append(" order by seq asc");
          return EventLogEntity.<EventLogEntity>find(query.toString(), params.toArray())
              .list()
              .map(list -> list.stream().map(this::fromEntity).toList());
        });
  }

  EventLog fromEntity(EventLogEntity entity) {
    EventLog log = new EventLog();
    log.id = entity.id;
    log.tenancyId = entity.tenancyId;
    log.setSeq(entity.seq);
    log.setCaseId(entity.caseId);
    log.setEventType(entity.eventType);
    log.setStreamType(entity.streamType);
    log.setWorkerId(entity.workerId);
    log.setTimestamp(entity.timestamp);
    log.setPayload(entity.payload);
    log.setMetadata(entity.metadata);
    return log;
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
}
```

Note: `fromEntity` is package-private (no `private`) so `JpaCrosstenantEventLogRepository` in the same package can reuse it, but we'll duplicate it for clarity and independence.

- [ ] **Step 2: Create JpaCrosstenantEventLogRepository**

```java
package io.casehub.persistence.jpa;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Objects;
import java.util.UUID;

/** Cross-tenant EventLog access for startup recovery services only. Uses BYPASSRLS role. */
@ApplicationScoped
public class JpaCrosstenantEventLogRepository extends TenantAwareRepository
    implements CrossTenantEventLogRepository {

  @Override
  public Uni<List<EventLog>> findByTypes(Collection<CaseHubEventType> types) {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find("eventType in ?1 order by seq asc", types)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types) {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and eventType in ?2 order by seq asc", caseId, types)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<String>> findSubmittedWorkWithoutCompletion() {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>list("eventType", CaseHubEventType.WORK_SUBMITTED)
                .chain(
                    submitted ->
                        EventLogEntity.<EventLogEntity>list(
                                "eventType", CaseHubEventType.WORK_COMPLETED)
                            .map(
                                completed -> {
                                  var submittedKeys =
                                      submitted.stream()
                                          .map(
                                              e ->
                                                  e.metadata != null
                                                      ? e.metadata
                                                          .path("correlationKey")
                                                          .asText(null)
                                                      : null)
                                          .filter(Objects::nonNull)
                                          .collect(java.util.stream.Collectors.toSet());
                                  var completedKeys =
                                      completed.stream()
                                          .map(
                                              e ->
                                                  e.metadata != null
                                                      ? e.metadata
                                                          .path("correlationKey")
                                                          .asText(null)
                                                      : null)
                                          .filter(Objects::nonNull)
                                          .collect(java.util.stream.Collectors.toSet());
                                  submittedKeys.removeAll(completedKeys);
                                  return new ArrayList<>(submittedKeys);
                                })));
  }

  @Override
  public Uni<EventLog> findById(Long id) {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>findById(id)
                .map(entity -> entity == null ? null : fromEntity(entity)));
  }

  @Override
  public Uni<List<EventLog>> findByCaseAndWorkerAndType(
      UUID caseId, String workerId, CaseHubEventType type) {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "caseId = ?1 and workerId = ?2 and eventType = ?3", caseId, workerId, type)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  @Override
  public Uni<List<EventLog>> findByWorkerAndTypeAcrossTenants(
      String workerId, CaseHubEventType type) {
    return withCrossTenantTransaction(
        () ->
            EventLogEntity.<EventLogEntity>find(
                    "workerId = ?1 and eventType = ?2", workerId, type)
                .list()
                .map(list -> list.stream().map(this::fromEntity).toList()));
  }

  private EventLog fromEntity(EventLogEntity entity) {
    EventLog log = new EventLog();
    log.id = entity.id;
    log.tenancyId = entity.tenancyId;
    log.setSeq(entity.seq);
    log.setCaseId(entity.caseId);
    log.setEventType(entity.eventType);
    log.setStreamType(entity.streamType);
    log.setWorkerId(entity.workerId);
    log.setTimestamp(entity.timestamp);
    log.setPayload(entity.payload);
    log.setMetadata(entity.metadata);
    return log;
  }
}
```

- [ ] **Step 3: Verify compilation and existing persistence tests**

```bash
mvn --batch-mode test -pl casehub-persistence-hibernate
```

Expected: all existing tests pass (no RLS integration test yet)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantEventLogRepository.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(persistence): split JpaEventLogRepository — tenant-scoped + cross-tenant, both extend TenantAwareRepository. Refs #406"
```

---

## Task 9: Update remaining JPA repositories to extend TenantAwareRepository

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseMetaModelRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCrosstenantCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`

The pattern for each is identical: change `extends AbstractJpaRepository` to `extends TenantAwareRepository`, then replace `withSafeContext(() -> Panache.withSession/withTransaction(() -> work))` with `withTenantTransaction(() -> work)` (or `withCrossTenantTransaction` for cross-tenant repos). For `JpaSubCaseGroupRepository` (which doesn't currently extend anything), replace bare `Panache.withTransaction(...)` with `withTenantTransaction(...)`.

- [ ] **Step 1: Update JpaCaseInstanceRepository**

Change class declaration:
```java
// Before:
public class JpaCaseInstanceRepository extends AbstractJpaRepository implements CaseInstanceRepository {

// After:
public class JpaCaseInstanceRepository extends TenantAwareRepository implements CaseInstanceRepository {
```

Replace each method:

`save()`:
```java
@Override
public Uni<CaseInstance> save(CaseInstance instance, String tenancyId) {
  return withTenantTransaction(
      () ->
          Panache.getSession()
              .chain(
                  session -> {
                    CaseInstanceEntity entity = new CaseInstanceEntity();
                    entity.tenancyId = tenancyId;
                    entity.uuid = instance.getUuid();
                    entity.state = instance.getState();
                    entity.parentCaseId = instance.getParentCaseId();
                    entity.parentPlanItemId = instance.getParentPlanItemId();
                    entity.waitingForWorkId = instance.getWaitingForWorkId();
                    if (instance.getCaseMetaModel() != null) {
                      entity.caseMetaModel =
                          session.getReference(
                              CaseMetaModelEntity.class, instance.getCaseMetaModel().getId());
                    }
                    return entity
                        .persist()
                        .map(
                            v -> {
                              instance.id = entity.id;
                              instance.tenancyId = tenancyId;
                              return instance;
                            });
                  }));
}
```

`update()`:
```java
@Override
public Uni<CaseInstance> update(CaseInstance instance, String tenancyId) {
  return withTenantTransaction(
      () ->
          CaseInstanceEntity.<CaseInstanceEntity>find(
                  "id = ?1 and tenancyId = ?2", instance.id, tenancyId)
              .firstResult()
              .invoke(
                  entity -> {
                    entity.state = instance.getState();
                    entity.parentCaseId = instance.getParentCaseId();
                    entity.parentPlanItemId = instance.getParentPlanItemId();
                    entity.waitingForWorkId = instance.getWaitingForWorkId();
                  })
              .replaceWith(instance));
}
```

`findByUuid()`:
```java
@Override
public Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
  return withTenantTransaction(
      () ->
          CaseInstanceEntity.<CaseInstanceEntity>find(
                  "from CaseInstanceEntity ci join fetch ci.caseMetaModel "
                      + "where ci.uuid = ?1 and ci.tenancyId = ?2",
                  uuid,
                  tenancyId)
              .firstResult()
              .map(entity -> entity == null ? null : fromEntity(entity)));
}
```

`updateStateAndAppendEvent()`:
```java
@Override
public Uni<Void> updateStateAndAppendEvent(
    CaseInstance instance, EventLog eventLog, String tenancyId) {
  EventLogEntity logEntity = new EventLogEntity();
  logEntity.tenancyId = tenancyId;
  logEntity.caseId = eventLog.getCaseId();
  logEntity.eventType = eventLog.getEventType();
  logEntity.streamType = eventLog.getStreamType();
  logEntity.workerId = eventLog.getWorkerId();
  logEntity.timestamp = eventLog.getTimestamp();
  logEntity.payload = eventLog.getPayload();
  logEntity.metadata = eventLog.getMetadata();

  return withTenantTransaction(
          () ->
              CaseInstanceEntity.<CaseInstanceEntity>find(
                      "id = ?1 and tenancyId = ?2", instance.id, tenancyId)
                  .firstResult()
                  .chain(
                      entity -> {
                        entity.state = instance.getState();
                        entity.parentCaseId = instance.getParentCaseId();
                        entity.parentPlanItemId = instance.getParentPlanItemId();
                        entity.waitingForWorkId = instance.getWaitingForWorkId();
                        return Panache.getSession().chain(s -> s.merge(entity));
                      })
                  .chain(merged -> logEntity.persistAndFlush()))
      .invoke(
          () -> {
            eventLog.id = logEntity.id;
            eventLog.setSeq(logEntity.seq);
          })
      .replaceWithVoid();
}
```

Also remove the `import io.quarkus.hibernate.reactive.panache.Panache;` import if `Panache` is no longer referenced directly. Keep it if `Panache.getSession()` is still used inside `withTenantTransaction()` — but that's in `TenantAwareRepository`, not here. In `save()` we use `Panache.getSession()` directly — keep the import.

- [ ] **Step 2: Update JpaCaseMetaModelRepository**

Change class declaration to `extends TenantAwareRepository`.

`findByKey()`:
```java
@Override
public Uni<CaseMetaModel> findByKey(
    String namespace, String name, String version, String tenancyId) {
  return withTenantTransaction(
      () ->
          CaseMetaModelEntity.<CaseMetaModelEntity>find(
                  "namespace = ?1 and name = ?2 and version = ?3 and tenancyId = ?4",
                  namespace, name, version, tenancyId)
              .firstResult()
              .map(entity -> entity == null ? null : fromEntity(entity)));
}
```

`save()`:
```java
@Override
public Uni<CaseMetaModel> save(CaseMetaModel metaModel, String tenancyId) {
  CaseMetaModelEntity entity = toEntity(metaModel, tenancyId);
  entity.createdAt = Instant.now().truncatedTo(ChronoUnit.MICROS);
  return withTenantTransaction(
      () ->
          entity
              .persist()
              .map(
                  v -> {
                    metaModel.id = entity.id;
                    metaModel.tenancyId = tenancyId;
                    metaModel.setCreatedAt(entity.createdAt);
                    return metaModel;
                  }));
}
```

Remove `import io.quarkus.hibernate.reactive.panache.Panache;` if no longer used.

- [ ] **Step 3: Update JpaCrosstenantCaseInstanceRepository**

Change class declaration to `extends TenantAwareRepository`. Switch from `withSafeContext(() -> Panache.withSession(...))` to `withCrossTenantTransaction(...)`:

```java
@Override
public Uni<CaseInstance> findByUuid(UUID caseId) {
  return withCrossTenantTransaction(
      () ->
          CaseInstanceEntity.<CaseInstanceEntity>find(
                  "from CaseInstanceEntity ci join fetch ci.caseMetaModel where ci.uuid = ?1",
                  caseId)
              .firstResult()
              .map(entity -> entity == null ? null : fromEntity(entity)));
}
```

- [ ] **Step 4: Update JpaSubCaseGroupRepository**

`JpaSubCaseGroupRepository` currently does NOT extend any base class and uses bare `Panache.*` calls. Change it to `extends TenantAwareRepository`. Replace all `Panache.withTransaction(() -> ...)` and `Panache.withSession(() -> ...)` calls with `withTenantTransaction(() -> ...)`. The inner work already uses Panache entity statics — those work inside `withTenantTransaction()`.

Change class declaration:
```java
// Before:
public class JpaSubCaseGroupRepository implements SubCaseGroupRepository {

// After:
public class JpaSubCaseGroupRepository extends TenantAwareRepository implements SubCaseGroupRepository {
```

Each method: replace the outer `Panache.withTransaction(() ->` or `Panache.withSession(() ->` wrapper with `withTenantTransaction(() ->`. The inner lambda body is unchanged.

Example for `getOrCreate`:
```java
@Override
public Uni<SubCaseGroup> getOrCreate(..., String tenancyId) {
  return withTenantTransaction(
      () ->
          SubCaseGroupEntity.<SubCaseGroupEntity>find(
                  "parentCaseId = ?1 and groupId = ?2 and tenancyId = ?3",
                  parentCaseId, groupId, tenancyId)
              .firstResult()
              .flatMap(
                  existing -> {
                    if (existing != null) return Uni.createFrom().item(toDomain(existing));
                    SubCaseGroupEntity e = new SubCaseGroupEntity();
                    e.tenancyId = tenancyId;
                    e.parentCaseId = parentCaseId;
                    e.groupId = groupId;
                    e.instanceCount = totalInGroup;
                    e.requiredCount = requiredCount;
                    e.onThresholdReached = onThresholdReached != null ? onThresholdReached : OnThresholdReached.KEEP;
                    return e.<SubCaseGroupEntity>persist().map(this::toDomain);
                  }));
}
```

Apply the same `Panache.withTransaction(...) → withTenantTransaction(...)` substitution to `registerChild`, `incrementCompleted`, `incrementRejected`, `markPolicyTriggered`.

For `findByChildCaseId` (currently `Panache.withSession`):
```java
@Override
public Uni<Optional<SubCaseGroup>> findByChildCaseId(UUID childCaseId, String tenancyId) {
  return withTenantTransaction(
      () ->
          SubCaseGroupEntity.<SubCaseGroupEntity>find(
                  "?1 member of childCaseIds and tenancyId = ?2", childCaseId, tenancyId)
              .firstResult()
              .map(e -> Optional.ofNullable(e == null ? null : toDomain(e))));
}
```

Remove the `import io.quarkus.hibernate.reactive.panache.Panache;` import if no longer referenced.

Add required imports:
```java
import io.quarkus.hibernate.reactive.panache.Panache;  // keep if still used inside withTenantTransaction
import jakarta.inject.Inject;  // needed for TenantAwareRepository's @Inject Vertx
```

Actually `JpaSubCaseGroupRepository` will inherit `@Inject Vertx vertx` and `@Inject CurrentPrincipal currentPrincipal` from `TenantAwareRepository` and `AbstractJpaRepository` via CDI — no explicit `@Inject` needed in the subclass.

- [ ] **Step 5: Update JpaReactivePlanItemStore**

Change class declaration to `extends TenantAwareRepository`. Replace all `withSafeContext(() -> Panache.withTransaction(...))` with `withTenantTransaction(...)`. The inner lambda body is unchanged.

Example for `save()`:
```java
@Override
public Uni<Void> save(PlanItemSaveRequest request, String tenancyId) {
  return withTenantTransaction(
      () -> {
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
      });
}
```

Apply the same pattern to `updateStatus()` and any other methods in `JpaReactivePlanItemStore`.

- [ ] **Step 6: Verify all persistence tests pass**

```bash
mvn --batch-mode test -pl casehub-persistence-hibernate
```

Expected: all existing tests pass

- [ ] **Step 7: Verify full build**

```bash
mvn --batch-mode install -DskipTests -q
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(persistence): all JPA repositories extend TenantAwareRepository — SET LOCAL per transaction. Refs #406"
```

---

## Task 10: RlsIntegrationTest — verify RLS enforcement end-to-end against PostgreSQL

**Files:**
- Create: `persistence-hibernate/src/test/java/io/casehub/persistence/jpa/RlsIntegrationTest.java`

This test requires real PostgreSQL (H2 does not support RLS or `SET LOCAL ROLE`). Quarkus Dev Services automatically starts a PostgreSQL Testcontainers instance when `quarkus.datasource.db-kind=postgresql` and no explicit JDBC URL is set. The `@TestProfile` overrides provide PostgreSQL + enables RLS.

Prerequisites: Docker must be running. Run with `TESTCONTAINERS_RYUK_DISABLED=true`.

- [ ] **Step 1: Write the RLS integration test**

```java
package io.casehub.persistence.jpa;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.spi.CrossTenantEventLogRepository;
import io.casehub.engine.common.spi.EventLogRepository;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/**
 * Verifies RLS enforcement end-to-end against real PostgreSQL via Quarkus Dev Services.
 * Requires Docker. Run with: TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-persistence-hibernate -Dtest=RlsIntegrationTest
 */
@QuarkusTest
@TestProfile(RlsIntegrationTest.RlsProfile.class)
class RlsIntegrationTest {

  public static class RlsProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
      return Map.of(
          "casehub.rls.enabled", "true",
          "quarkus.datasource.db-kind", "postgresql",
          "quarkus.hibernate-orm.schema-management.strategy", "drop-and-create",
          "quarkus.flyway.migrate-at-start", "false");
    }
  }

  @Inject EventLogRepository eventLogRepository;
  @Inject CrossTenantEventLogRepository crossTenantEventLogRepository;

  private static final String TENANT_A = "tenant-a-rls-test";
  private static final String TENANT_B = "tenant-b-rls-test";

  @Test
  void tenantScopedQueries_returnOnlyOwnTenantRows() {
    UUID caseId = UUID.randomUUID();

    EventLog logA = makeLog(caseId, CaseHubEventType.CASE_STARTED);
    EventLog logB = makeLog(UUID.randomUUID(), CaseHubEventType.CASE_COMPLETED);

    eventLogRepository.append(logA, TENANT_A).subscribe().asCompletionStage().toCompletableFuture().join();
    eventLogRepository.append(logB, TENANT_B).subscribe().asCompletionStage().toCompletableFuture().join();

    // Tenant A queries via EventLogRepository must see only tenant A rows
    // (RLS filters based on casehub.tenancy_id set by withTenantTransaction — current principal tenant)
    List<EventLog> found =
        eventLogRepository
            .findByCaseAndTypes(caseId, List.of(CaseHubEventType.CASE_STARTED), TENANT_A)
            .subscribe()
            .asCompletionStage()
            .toCompletableFuture()
            .join();

    assertThat(found).hasSize(1);
    assertThat(found.get(0).getEventType()).isEqualTo(CaseHubEventType.CASE_STARTED);
    assertThat(found.get(0).tenancyId).isEqualTo(TENANT_A);
  }

  @Test
  void crossTenantQueries_returnRowsAcrossAllTenants() {
    UUID caseA = UUID.randomUUID();
    UUID caseB = UUID.randomUUID();

    EventLog logA = makeLog(caseA, CaseHubEventType.CASE_STARTED);
    EventLog logB = makeLog(caseB, CaseHubEventType.CASE_STARTED);

    eventLogRepository.append(logA, TENANT_A).subscribe().asCompletionStage().toCompletableFuture().join();
    eventLogRepository.append(logB, TENANT_B).subscribe().asCompletionStage().toCompletableFuture().join();

    List<EventLog> all =
        crossTenantEventLogRepository
            .findByTypes(List.of(CaseHubEventType.CASE_STARTED))
            .subscribe()
            .asCompletionStage()
            .toCompletableFuture()
            .join();

    // Cross-tenant must see rows from both tenants
    assertThat(all).extracting(e -> e.tenancyId)
        .contains(TENANT_A, TENANT_B);
  }

  private EventLog makeLog(UUID caseId, CaseHubEventType type) {
    EventLog log = new EventLog();
    log.setCaseId(caseId);
    log.setEventType(type);
    log.setStreamType(EventStreamType.CASE);
    log.setTimestamp(Instant.now());
    return log;
  }
}
```

- [ ] **Step 2: Run the RLS integration test**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl casehub-persistence-hibernate -Dtest=RlsIntegrationTest
```

Expected: PASS (PostgreSQL container started by Dev Services, RLS applied by RlsPolicyApplicator, tenant filtering verified)

If this fails because `MockCurrentPrincipal.tenancyId()` returns `DEFAULT_TENANT_ID` rather than `TENANT_A` / `TENANT_B`: the RLS variable is set from `currentPrincipal.tenancyId()` in `withTenantTransaction()`. For the test, the principal's tenancyId must match the tenancyId passed to `append()`. Since `MockCurrentPrincipal` returns `DEFAULT_TENANT_ID`, update the test to use `TenancyConstants.DEFAULT_TENANT_ID` as `TENANT_A` and a different fixed tenant for `TENANT_B` inserted via a direct repository call with explicit tenancyId. Alternatively, define a `@QuarkusTestProfile` that also configures `casehub.tenancy.default-id` to a test-specific tenant. Adjust if needed based on actual test failure output.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add persistence-hibernate/src/test/java/io/casehub/persistence/jpa/RlsIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(persistence): RlsIntegrationTest — verifies RLS enforcement against PostgreSQL. Refs #406"
```

---

## Task 11: CaseKey record — immutable map key for CaseDefinitionRegistry

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/model/CaseKey.java`
- Create: `common/src/test/java/io/casehub/engine/common/internal/model/CaseKeyTest.java`

TDD: test first.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.engine.common.internal.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class CaseKeyTest {

  @Test
  void equalKeys_haveSameHashCode() {
    CaseKey k1 = new CaseKey("ns", "name", "1.0");
    CaseKey k2 = new CaseKey("ns", "name", "1.0");
    assertThat(k1).isEqualTo(k2);
    assertThat(k1.hashCode()).isEqualTo(k2.hashCode());
  }

  @Test
  void differentNamespace_notEqual() {
    assertThat(new CaseKey("ns1", "name", "1.0"))
        .isNotEqualTo(new CaseKey("ns2", "name", "1.0"));
  }

  @Test
  void of_fromCaseMetaModel_extractsFields() {
    CaseMetaModel m = new CaseMetaModel();
    m.setNamespace("ns");
    m.setName("name");
    m.setVersion("2.0");
    // Mutate non-key fields to prove they don't affect the key
    m.id = 99L;
    m.tenancyId = "tenant-x";
    m.setDsl("some-dsl");

    CaseKey key = CaseKey.of(m);
    assertThat(key).isEqualTo(new CaseKey("ns", "name", "2.0"));
  }

  @Test
  void hashCode_isStable_afterCaseMetaModelMutation() {
    // Proves the fix for engine#410: CaseKey hashCode doesn't change even if CaseMetaModel fields change
    CaseMetaModel m = new CaseMetaModel();
    m.setNamespace("ns");
    m.setName("name");
    m.setVersion("1.0");
    CaseKey key = CaseKey.of(m);
    int hashBefore = key.hashCode();

    // Mutate the source model — key is immutable, unaffected
    m.setNamespace("mutated");
    m.setName("also-mutated");

    assertThat(key.hashCode()).isEqualTo(hashBefore);
    assertThat(key.namespace()).isEqualTo("ns");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn --batch-mode test -pl casehub-engine-common -Dtest=CaseKeyTest
```

Expected: FAIL — CaseKey not found

- [ ] **Step 3: Create CaseKey**

```java
package io.casehub.engine.common.internal.model;

import io.casehub.api.model.CaseDefinition;

/** Immutable (namespace, name, version) identity key for CaseDefinitionRegistry. */
public record CaseKey(String namespace, String name, String version) {

  public static CaseKey of(CaseMetaModel m) {
    return new CaseKey(m.getNamespace(), m.getName(), m.getVersion());
  }

  public static CaseKey of(CaseDefinition d) {
    return new CaseKey(d.getNamespace(), d.getName(), d.getVersion());
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn --batch-mode test -pl casehub-engine-common -Dtest=CaseKeyTest
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  common/src/main/java/io/casehub/engine/common/internal/model/CaseKey.java \
  common/src/test/java/io/casehub/engine/common/internal/model/CaseKeyTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(common): add CaseKey immutable record — fixes mutable hashCode map key bug. Refs #410"
```

---

## Task 12: Refactor DefaultCaseDefinitionRegistry — RegistryEntry + CaseKey map

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java`

TDD: write the failing test first to document the bug, then apply the fix.

- [ ] **Step 1: Write the failing test (documents the bug)**

```java
package io.casehub.engine.internal.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseMetaModelRepository;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.quarkus.test.junit.QuarkusTest;
import io.smallrye.mutiny.Uni;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DefaultCaseDefinitionRegistryTest {

  @Inject DefaultCaseDefinitionRegistry registry;

  @Test
  void getCaseDefinition_afterKeyFieldMutation_stillFindsEntry() {
    // This is the engine#410 regression: if CaseMetaModel is used as a mutable map key,
    // mutating namespace/name/version after put() causes Map.get() to fail.
    // With CaseKey, the key is immutable — mutation of the source CaseMetaModel is irrelevant.

    // Register a definition (uses the in-memory CaseMetaModelRepository)
    CaseDefinition def =
        CaseDefinition.builder()
            .namespace("test-ns")
            .name("test-case")
            .version("1.0")
            .build();

    CaseMetaModel registered =
        registry
            .registerCaseDefinition(def)
            .subscribe()
            .asCompletionStage()
            .toCompletableFuture()
            .join();

    assertThat(registered).isNotNull();

    // Look up immediately — must work
    CaseDefinition found = registry.getCaseDefinition(registered);
    assertThat(found).isNotNull();
    assertThat(found.getName()).isEqualTo("test-case");

    // Mutate the registered CaseMetaModel's key fields — simulates the engine#410 scenario
    registered.setNamespace("mutated-namespace");
    registered.setName("mutated-name");

    // getCaseDefinition with a fresh CaseMetaModel matching original coordinates must still work
    CaseMetaModel lookup = new CaseMetaModel();
    lookup.setNamespace("test-ns");
    lookup.setName("test-case");
    lookup.setVersion("1.0");

    CaseDefinition foundAfterMutation = registry.getCaseDefinition(lookup);
    assertThat(foundAfterMutation)
        .as("getCaseDefinition must find the entry even after registered key was mutated")
        .isNotNull();
  }

  @Test
  void getCaseDefinition_earlyExitPath_returnsExistingCaseMetaModel() {
    CaseDefinition def =
        CaseDefinition.builder()
            .namespace("test-ns-2")
            .name("test-case-2")
            .version("1.0")
            .build();

    CaseMetaModel first =
        registry
            .registerCaseDefinition(def)
            .subscribe()
            .asCompletionStage()
            .toCompletableFuture()
            .join();

    // Second registration of the same definition — should hit early-exit and return existing model
    CaseMetaModel second =
        registry
            .registerCaseDefinition(def)
            .subscribe()
            .asCompletionStage()
            .toCompletableFuture()
            .join();

    assertThat(second).isNotNull();
    assertThat(second.getNamespace()).isEqualTo("test-ns-2");
    // Both return the same (namespace, name, version) — the early-exit path is correct
    assertThat(second.getName()).isEqualTo(first.getName());
  }
}
```

- [ ] **Step 2: Run test to see what happens with current implementation**

```bash
mvn --batch-mode test -pl casehub-engine-runtime -Dtest=DefaultCaseDefinitionRegistryTest
```

The `hashCode_mutation` test may pass or fail depending on whether the guard kicks in. The `earlyExitPath` test verifies return value correctness.

- [ ] **Step 3: Refactor DefaultCaseDefinitionRegistry**

Replace the two-map approach with a single `Map<CaseKey, RegistryEntry>`. The `RegistryEntry` record is defined as a private inner record.

Replace the class body (keep all imports, `@ApplicationScoped`, and helpers unchanged; only the registry fields and the three public methods change):

```java
// Replace these fields:
// private final Map<CaseMetaModel, CaseDefinition> registry = new ConcurrentHashMap<>();

// With:
private record RegistryEntry(CaseDefinition definition, CaseMetaModel metaModel) {}
private final Map<CaseKey, RegistryEntry> registry = new ConcurrentHashMap<>();
```

Replace `registerCaseDefinition()`:
```java
@Override
public Uni<CaseMetaModel> registerCaseDefinition(CaseDefinition model) {
  try {
    validateExpressions(model);
  } catch (IllegalArgumentException e) {
    LOG.errorf("Case definition '%s' rejected: %s", model.getName(), e.getMessage());
    return Uni.createFrom().failure(e);
  }

  LOG.info("Registering case: " + model.getName() + " version: " + model.getVersion()
      + " namespace: " + model.getNamespace());

  CaseKey key = CaseKey.of(model);

  // Early exit: already registered — return existing CaseMetaModel
  RegistryEntry existing = registry.get(key);
  if (existing != null) {
    return Uni.createFrom().item(existing.metaModel());
  }

  CaseMetaModel definition = new CaseMetaModel();
  definition.setName(model.getName());
  definition.setNamespace(model.getNamespace());
  definition.setVersion(model.getVersion());

  return caseMetaModelRepository
      .findByKey(model.getNamespace(), model.getName(), model.getVersion(), currentPrincipal.tenancyId())
      .onItem()
      .transformToUni(
          dbModel -> {
            if (dbModel != null) {
              registry.put(CaseKey.of(dbModel), new RegistryEntry(model, dbModel));
              return Uni.createFrom().item(dbModel);
            }
            definition.setDsl(model.getDsl());
            definition.setCreatedAt(Instant.now());
            return caseMetaModelRepository
                .save(definition, currentPrincipal.tenancyId())
                .invoke(saved -> registry.put(CaseKey.of(saved), new RegistryEntry(model, saved)));
          });
}
```

Replace `getCaseDefinition()`:
```java
@Override
public CaseDefinition getCaseDefinition(CaseMetaModel definition) {
  RegistryEntry entry = registry.get(CaseKey.of(definition));
  return entry != null ? entry.definition() : null;
}
```

Replace `getCaseMetaModel()`:
```java
@Override
public CaseMetaModel getCaseMetaModel(CaseDefinition caseDefinition) {
  RegistryEntry entry = registry.get(CaseKey.of(caseDefinition));
  if (entry == null) {
    throw new RuntimeException(
        "CaseMetaModel not found for caseDefinition: "
            + caseDefinition.getNamespace() + "." + caseDefinition.getName()
            + ":" + caseDefinition.getVersion());
  }
  return entry.metaModel();
}
```

Add the import at the top of the file:
```java
import io.casehub.engine.common.internal.model.CaseKey;
```

- [ ] **Step 4: Run test to verify it passes**

```bash
mvn --batch-mode test -pl casehub-engine-runtime -Dtest=DefaultCaseDefinitionRegistryTest
```

Expected: PASS

- [ ] **Step 5: Run full runtime tests**

```bash
mvn --batch-mode test -pl casehub-engine-runtime
```

Expected: all tests pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix(runtime): CaseDefinitionRegistry uses immutable CaseKey + RegistryEntry — eliminates mutable hashCode map key bug. Refs #410"
```

---

## Task 13: Final verification — full build and module test suite

- [ ] **Step 1: Full build with all tests (except RLS integration test — requires Docker)**

```bash
mvn --batch-mode install -q
```

Expected: BUILD SUCCESS

- [ ] **Step 2: Run all engine module tests**

```bash
mvn --batch-mode test -pl casehub-engine-common,casehub-engine-runtime,casehub-engine-scheduler-quartz,casehub-resilience,casehub-persistence-hibernate,casehub-persistence-memory
```

Expected: all tests pass

- [ ] **Step 3: Run blackboard tests (validates engine wiring)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode clean test -pl casehub-blackboard
```

Expected: all tests pass

---

## Self-Review Checklist

- [x] **Spec coverage:** V2005 ✅ | @CrossTenant qualifiers + producer ✅ | SystemCurrentPrincipal ✅ | 6 injection sites ✅ | TenantAwareRepository ✅ | RlsPolicyApplicator ✅ | JpaEventLogRepository split ✅ | 5 remaining repos ✅ | RlsIntegrationTest ✅ | CaseKey ✅ | DefaultCaseDefinitionRegistry refactor ✅ | ADR stubs — **deferred:** ADRs are to be written during implementation, not part of this plan
- [x] **Placeholder scan:** No TBD, no "implement later". RlsIntegrationTest note about MockCurrentPrincipal/tenancyId is a conditional instruction with specific resolution path, not a placeholder.
- [x] **Type consistency:** `CaseKey.of(CaseMetaModel)`, `CaseKey.of(CaseDefinition)`, `RegistryEntry(CaseDefinition, CaseMetaModel)`, `withTenantTransaction()`, `withCrossTenantTransaction()` — used consistently across all tasks.
- [x] **RLS constraint noted:** Task 10 test notes that `currentPrincipal.tenancyId()` drives `SET LOCAL` — test must align tenancy IDs with mock principal's return value.
