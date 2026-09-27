# Persistent DLQ Storage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1185 — persistent DLQ storage
**Issue group:** #1182, #732, #1183, #1184, #1185

**Goal:** Make DLQ entries survive restarts by introducing a `DeadLetterEntryStore` SPI with in-memory and JPA implementations, then wiring `DeadLetterQueue` as a facade over the store.

**Architecture:** Define `DeadLetterEntryStore` SPI in `resilience-core` alongside `DeadLetterEntry`. Create `InMemoryDeadLetterEntryStore` (same module, test default). Create `JpaDeadLetterEntryStore` + `DeadLetterEntryEntity` in `persistence-hibernate`. Refactor `DeadLetterQueue` to delegate to the store.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, Jackson (JSONB serialization)

## Global Constraints

- No Flyway/Liquibase — schema managed by Hibernate `drop-and-create`
- Test classes must be `*Test.java`, never `*IT.java`
- `TESTCONTAINERS_RYUK_DISABLED=true` for test runs
- `persistence-hibernate` gains `resilience-core` as a new dependency

---

## Batch 1: SPI + InMemory + DeadLetterQueue facade

### Task 1: Define DeadLetterEntryStore SPI and InMemory implementation

**Files:**
- Create: `resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterEntryStore.java`
- Create: `resilience-core/src/main/java/io/casehub/resilience/deadletter/InMemoryDeadLetterEntryStore.java`
- Create: `resilience-core/src/test/java/io/casehub/resilience/deadletter/InMemoryDeadLetterEntryStoreTest.java`
- Modify: `resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterEntry.java` — widen constructor to public

**Interfaces:**
- Produces: `DeadLetterEntryStore` (SPI), `InMemoryDeadLetterEntryStore` (impl)

- [ ] **Step 1: Write the SPI interface**

```java
package io.casehub.resilience.deadletter;

import java.util.List;

public interface DeadLetterEntryStore {
    DeadLetterEntry save(DeadLetterEntry entry);
    DeadLetterEntry findById(String deadLetterId);
    List<DeadLetterEntry> query(DeadLetterQuery query);
    void updateStatus(String deadLetterId, DeadLetterStatus status);
    void incrementReplayAttempts(String deadLetterId);
    void deleteAll();
}
```

- [ ] **Step 2: Write failing test for InMemoryDeadLetterEntryStore**

```java
package io.casehub.resilience.deadletter;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class InMemoryDeadLetterEntryStoreTest {

    private InMemoryDeadLetterEntryStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryDeadLetterEntryStore();
    }

    @Test
    void save_and_findById() {
        DeadLetterEntry entry = new DeadLetterEntry(
            "dlq-1", UUID.randomUUID(), "worker-a", "hash-1", Map.of("k", "v"), null);
        store.save(entry);
        assertThat(store.findById("dlq-1")).isNotNull();
        assertThat(store.findById("dlq-1").workerId()).isEqualTo("worker-a");
    }

    @Test
    void query_byStatus() {
        DeadLetterEntry entry = new DeadLetterEntry(
            "dlq-2", UUID.randomUUID(), "worker-b", "hash-2", Map.of(), null);
        store.save(entry);
        store.updateStatus("dlq-2", DeadLetterStatus.DISCARDED);
        assertThat(store.query(DeadLetterQuery.withStatus(DeadLetterStatus.DISCARDED))).hasSize(1);
        assertThat(store.query(DeadLetterQuery.withStatus(DeadLetterStatus.PENDING_REVIEW))).isEmpty();
    }

    @Test
    void incrementReplayAttempts_updatesCountAndTimestamp() {
        DeadLetterEntry entry = new DeadLetterEntry(
            "dlq-3", UUID.randomUUID(), "worker-c", "hash-3", Map.of(), null);
        store.save(entry);
        store.incrementReplayAttempts("dlq-3");
        DeadLetterEntry updated = store.findById("dlq-3");
        assertThat(updated.replayAttempts()).isEqualTo(1);
        assertThat(updated.lastReplayAttemptAt()).isNotNull();
    }

    @Test
    void deleteAll_clearsStore() {
        store.save(new DeadLetterEntry(
            "dlq-4", UUID.randomUUID(), "w", "h", Map.of(), null));
        store.deleteAll();
        assertThat(store.query(DeadLetterQuery.all())).isEmpty();
    }
}
```

- [ ] **Step 3: Run test — expect FAIL (class doesn't exist)**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience-core -Dtest=InMemoryDeadLetterEntryStoreTest -q`
Expected: FAIL — compilation error

- [ ] **Step 4: Widen DeadLetterEntry constructor to public**

Change `DeadLetterEntry.java` line 44: `DeadLetterEntry(` → `public DeadLetterEntry(`

- [ ] **Step 5: Implement InMemoryDeadLetterEntryStore**

```java
package io.casehub.resilience.deadletter;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryDeadLetterEntryStore implements DeadLetterEntryStore {

    private final Map<String, DeadLetterEntry> store = new ConcurrentHashMap<>();

    @Override
    public DeadLetterEntry save(DeadLetterEntry entry) {
        store.put(entry.deadLetterId(), entry);
        return entry;
    }

    @Override
    public DeadLetterEntry findById(String deadLetterId) {
        return store.get(deadLetterId);
    }

    @Override
    public List<DeadLetterEntry> query(DeadLetterQuery query) {
        return store.values().stream().filter(query.toPredicate()).toList();
    }

    @Override
    public void updateStatus(String deadLetterId, DeadLetterStatus status) {
        DeadLetterEntry entry = store.get(deadLetterId);
        if (entry != null) {
            entry.setStatus(status);
        }
    }

    @Override
    public void incrementReplayAttempts(String deadLetterId) {
        DeadLetterEntry entry = store.get(deadLetterId);
        if (entry != null) {
            entry.incrementReplayAttempts();
        }
    }

    @Override
    public void deleteAll() {
        store.clear();
    }
}
```

- [ ] **Step 6: Run test — expect PASS**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience-core -Dtest=InMemoryDeadLetterEntryStoreTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterEntryStore.java \
  resilience-core/src/main/java/io/casehub/resilience/deadletter/InMemoryDeadLetterEntryStore.java \
  resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterEntry.java \
  resilience-core/src/test/java/io/casehub/resilience/deadletter/InMemoryDeadLetterEntryStoreTest.java
git commit -m "feat(#1185): add DeadLetterEntryStore SPI and InMemory implementation

Refs casehubio/engine#1185"
```

### Task 2: Refactor DeadLetterQueue to delegate to store

**Files:**
- Modify: `resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterQueue.java`
- Modify: `resilience-core/src/test/java/io/casehub/resilience/deadletter/DeadLetterQueueTest.java`
- Modify: `resilience-core/src/test/java/io/casehub/resilience/deadletter/DeadLetterAutoReplayJobTest.java`
- Modify: `resilience-core/src/test/java/io/casehub/resilience/deadletter/DeadLetterReplayServiceTest.java`
- Modify: `resilience/src/main/java/io/casehub/resilience/quarkus/ResilienceBeans.java:50-54`

**Interfaces:**
- Consumes: `DeadLetterEntryStore`, `InMemoryDeadLetterEntryStore` from Task 1
- Produces: `DeadLetterQueue(DeadLetterEntryStore)` — new constructor

- [ ] **Step 1: Update DeadLetterQueue to take a store parameter**

Replace the `ConcurrentHashMap` with a `DeadLetterEntryStore` field. Delegate all storage operations:

```java
public class DeadLetterQueue {

    private final DeadLetterEntryStore store;

    public DeadLetterQueue(DeadLetterEntryStore store) {
        this.store = store;
    }

    public DeadLetterEntry add(UUID caseId, String workerId, String idempotencyHash,
            Map<String, Object> inputContext, RetryState retryState) {
        String id = UUID.randomUUID().toString();
        DeadLetterEntry entry = new DeadLetterEntry(
            id, caseId, workerId, idempotencyHash, inputContext, retryState);
        return store.save(entry);
    }

    public List<DeadLetterEntry> query(DeadLetterQuery query) {
        return store.query(query);
    }

    public DeadLetterEntry findById(String deadLetterId) {
        return store.findById(deadLetterId);
    }

    public void discard(String deadLetterId) {
        store.updateStatus(deadLetterId, DeadLetterStatus.DISCARDED);
    }

    public void markReplayed(String deadLetterId) {
        store.updateStatus(deadLetterId, DeadLetterStatus.REPLAYED);
    }

    public int size() {
        return store.query(DeadLetterQuery.all()).size();
    }

    public void clear() {
        store.deleteAll();
    }
}
```

- [ ] **Step 2: Update DeadLetterQueueTest to use InMemoryDeadLetterEntryStore**

Change `setUp()`:
```java
queue = new DeadLetterQueue(new InMemoryDeadLetterEntryStore());
```

- [ ] **Step 3: Update DeadLetterAutoReplayJobTest — all `new DeadLetterQueue()` calls**

Replace every `new DeadLetterQueue()` with `new DeadLetterQueue(new InMemoryDeadLetterEntryStore())`.

- [ ] **Step 4: Update DeadLetterReplayServiceTest**

Replace `queue = new DeadLetterQueue()` with `queue = new DeadLetterQueue(new InMemoryDeadLetterEntryStore())`.

- [ ] **Step 5: Update ResilienceBeans producer**

```java
@Produces
@ApplicationScoped
DeadLetterQueue deadLetterQueue(DeadLetterEntryStore store) {
    return new DeadLetterQueue(store);
}

@Produces
@ApplicationScoped
DeadLetterEntryStore deadLetterEntryStore() {
    return new InMemoryDeadLetterEntryStore();
}
```

- [ ] **Step 6: Run all resilience-core tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience-core -q`
Expected: PASS — all existing tests work with the delegating queue.

- [ ] **Step 7: Commit**

```bash
git add resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterQueue.java \
  resilience-core/src/test/java/io/casehub/resilience/deadletter/ \
  resilience/src/main/java/io/casehub/resilience/quarkus/ResilienceBeans.java
git commit -m "feat(#1185): refactor DeadLetterQueue to delegate to DeadLetterEntryStore

DeadLetterQueue is now a facade. Entry creation and API preserved;
storage delegated to the injected store. InMemory default for tests.

Refs casehubio/engine#1185"
```

## Batch 2: JPA implementation + integration

### Task 3: JPA entity and store implementation

**Files:**
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/DeadLetterEntryEntity.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaDeadLetterEntryStore.java`
- Modify: `persistence-hibernate/pom.xml` — add `resilience-core` dependency

**Interfaces:**
- Consumes: `DeadLetterEntryStore` SPI, `DeadLetterEntry`, `DeadLetterQuery`, `DeadLetterStatus` from resilience-core
- Produces: `JpaDeadLetterEntryStore implements DeadLetterEntryStore`

- [ ] **Step 1: Add resilience-core dependency to persistence-hibernate pom.xml**

Add to `persistence-hibernate/pom.xml` `<dependencies>`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-resilience-core</artifactId>
    <version>${version.io.casehub}</version>
</dependency>
```

- [ ] **Step 2: Create DeadLetterEntryEntity**

```java
package io.casehub.persistence.jpa;

import io.casehub.resilience.deadletter.DeadLetterStatus;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Index;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(
    name = "dead_letter_entry",
    indexes = {
        @Index(name = "idx_dle_dead_letter_id", columnList = "deadLetterId", unique = true),
        @Index(name = "idx_dle_case_id", columnList = "caseId")
    })
public class DeadLetterEntryEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public Long id;

    @Column(nullable = false, unique = true)
    public String deadLetterId;

    @Column(nullable = false)
    public UUID caseId;

    @Column(nullable = false)
    public String workerId;

    public String idempotencyHash;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    public String inputContext;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    public String retryState;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    public DeadLetterStatus status;

    @Column(nullable = false)
    public int replayAttempts;

    @Column(nullable = false)
    public Instant arrivedAt;

    public Instant lastReplayAttemptAt;

    @Column(nullable = false)
    public String tenancyId;
}
```

- [ ] **Step 3: Create JpaDeadLetterEntryStore**

Follow the `JpaCaseQueueEntryStore` pattern — `@ApplicationScoped`, `@Transactional`, inject `EntityManager` and `TenantContextManager`:

```java
package io.casehub.persistence.jpa;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.RetryState;
import io.casehub.resilience.deadletter.DeadLetterEntry;
import io.casehub.resilience.deadletter.DeadLetterEntryStore;
import io.casehub.resilience.deadletter.DeadLetterQuery;
import io.casehub.resilience.deadletter.DeadLetterStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;
import java.util.Map;

@ApplicationScoped
public class JpaDeadLetterEntryStore implements DeadLetterEntryStore {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};

    @Inject EntityManager em;
    @Inject TenantContextManager tcm;

    @Override
    @Transactional
    public DeadLetterEntry save(DeadLetterEntry entry) {
        DeadLetterEntryEntity entity = toEntity(entry);
        em.persist(entity);
        em.flush();
        return entry;
    }

    @Override
    @Transactional
    public DeadLetterEntry findById(String deadLetterId) {
        tcm.setCrossTenantContext();
        var results = em.createQuery(
                "SELECT e FROM DeadLetterEntryEntity e WHERE e.deadLetterId = :dlid",
                DeadLetterEntryEntity.class)
            .setParameter("dlid", deadLetterId)
            .getResultList();
        return results.isEmpty() ? null : fromEntity(results.get(0));
    }

    @Override
    @Transactional
    public List<DeadLetterEntry> query(DeadLetterQuery query) {
        tcm.setCrossTenantContext();
        // Load all and filter in-memory (same as in-memory impl)
        // For pre-release scale this is fine; push filtering to JPQL later if needed
        return em.createQuery("SELECT e FROM DeadLetterEntryEntity e", DeadLetterEntryEntity.class)
            .getResultList().stream()
            .map(this::fromEntity)
            .filter(query.toPredicate())
            .toList();
    }

    @Override
    @Transactional
    public void updateStatus(String deadLetterId, DeadLetterStatus status) {
        tcm.setCrossTenantContext();
        em.createQuery("UPDATE DeadLetterEntryEntity e SET e.status = :s WHERE e.deadLetterId = :dlid")
            .setParameter("s", status)
            .setParameter("dlid", deadLetterId)
            .executeUpdate();
    }

    @Override
    @Transactional
    public void incrementReplayAttempts(String deadLetterId) {
        tcm.setCrossTenantContext();
        em.createQuery(
                "UPDATE DeadLetterEntryEntity e SET e.replayAttempts = e.replayAttempts + 1, "
                    + "e.lastReplayAttemptAt = :now WHERE e.deadLetterId = :dlid")
            .setParameter("now", Instant.now())
            .setParameter("dlid", deadLetterId)
            .executeUpdate();
    }

    @Override
    @Transactional
    public void deleteAll() {
        tcm.setCrossTenantContext();
        em.createQuery("DELETE FROM DeadLetterEntryEntity").executeUpdate();
    }

    private DeadLetterEntryEntity toEntity(DeadLetterEntry entry) {
        DeadLetterEntryEntity entity = new DeadLetterEntryEntity();
        entity.deadLetterId = entry.deadLetterId();
        entity.caseId = entry.caseId();
        entity.workerId = entry.workerId();
        entity.idempotencyHash = entry.idempotencyHash();
        try {
            entity.inputContext = MAPPER.writeValueAsString(entry.inputContext());
            entity.retryState = MAPPER.writeValueAsString(entry.retryState());
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize DLQ entry", e);
        }
        entity.status = entry.status();
        entity.replayAttempts = entry.replayAttempts();
        entity.arrivedAt = entry.arrivedAt();
        entity.lastReplayAttemptAt = entry.lastReplayAttemptAt();
        entity.tenancyId = "default"; // DLQ is cross-tenant; tenancyId for DB partitioning
        return entity;
    }

    private DeadLetterEntry fromEntity(DeadLetterEntryEntity entity) {
        Map<String, Object> inputContext;
        RetryState retryState;
        try {
            inputContext = entity.inputContext != null
                ? MAPPER.readValue(entity.inputContext, MAP_TYPE) : Map.of();
            retryState = entity.retryState != null
                ? MAPPER.readValue(entity.retryState, RetryState.class) : RetryState.empty();
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize DLQ entry", e);
        }
        DeadLetterEntry entry = new DeadLetterEntry(
            entity.deadLetterId, entity.caseId, entity.workerId,
            entity.idempotencyHash, inputContext, retryState);
        entry.setStatus(entity.status);
        // replay attempts and timestamps are read-only from entity
        return entry;
    }
}
```

Note: `DeadLetterQuery.toPredicate()` is package-private. It needs to be widened to `public` for the JPA store to use it for in-memory filtering. Alternatively, push filter criteria to JPQL. For pre-release, widen the visibility.

- [ ] **Step 4: Widen DeadLetterQuery.toPredicate() to public**

Change `resilience-core/.../DeadLetterQuery.java` line 62: `Predicate<DeadLetterEntry> toPredicate()` → `public Predicate<DeadLetterEntry> toPredicate()`

- [ ] **Step 5: Compile the persistence-hibernate module**

Run: `mvn compile -pl persistence-hibernate -Dcheckstyle.skip=true -Dspotless.check.skip=true -o -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add persistence-hibernate/pom.xml \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/DeadLetterEntryEntity.java \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaDeadLetterEntryStore.java \
  resilience-core/src/main/java/io/casehub/resilience/deadletter/DeadLetterQuery.java
git commit -m "feat(#1185): JPA DeadLetterEntryStore implementation

Entity + store in persistence-hibernate. Cross-tenant context for
DLQ operations. JSONB for inputContext and retryState.

Refs casehubio/engine#1185"
```

### Task 4: Integration verification

**Files:**
- No new files

- [ ] **Step 1: Install all modules**

Run: `mvn install -DskipTests -q -Dcheckstyle.skip=true -Dspotless.check.skip=true -o`

- [ ] **Step 2: Run resilience-core tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience-core -q -Dcheckstyle.skip=true -Dspotless.check.skip=true`
Expected: PASS

- [ ] **Step 3: Run resilience integration tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience -Dtest=DeadLetterReplayIntegrationTest -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 4: Fix any wiring or compilation failures**

Common issues: CDI ambiguity (two `DeadLetterEntryStore` beans — InMemory from ResilienceBeans + JPA from persistence-hibernate). Fix with `@Alternative @Priority(1)` on the JPA impl, or remove the InMemory producer from ResilienceBeans when the JPA module is on classpath.

- [ ] **Step 5: Commit if any fixes were needed**

---

## References

- [2026-09-27-persistent-dlq-storage-design.md] — design spec
- [DeadLetterQueue.java:33-113] — current in-memory implementation
- [DeadLetterEntry.java:31-109] — entry record
- [DeadLetterQuery.java:26-75] — query filter
- [DeadLetterQueueTest.java] — existing tests
- [ResilienceBeans.java:50-54] — CDI producer
- [CaseQueueEntryStore.java] — similar SPI pattern
- [JpaCaseQueueEntryStore.java] — similar JPA pattern
- [decisions.md D9-D11] — design decisions
- [GitHub casehubio/engine#1185] — focal issue
