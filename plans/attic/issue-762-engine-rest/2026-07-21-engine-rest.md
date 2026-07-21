# Engine REST Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #762 — feat: extract engine-rest module from scaffold
**Issue group:** #762

**Goal:** Move scaffold REST surface into a new `casehub-engine-rest`
module with virtual-thread-based resources, SPI-only persistence,
and paginated query objects.

**Architecture:** Move-and-refactor from scaffold. Resources use
`@RunOnVirtualThread` with blocking SPIs. No Panache, no Uni, no ACL
in REST layer. Thin `CaseService` for multi-step flows. Pagination via
query objects on blocking SPIs following work's `AuditQuery` pattern.

**Tech Stack:** Java 21, Quarkus 3.32.2, JAX-RS (quarkus-rest-jackson),
Bean Validation, MicroProfile OpenAPI, virtual threads

## Global Constraints

- Java 21 required (`maven-enforcer-plugin`)
- Quarkus `3.32.2` (`version.quarkus.platform` in root pom)
- Apache License 2.0 header on every file (Spotless enforced)
- Google Java Format 1.17.0 (Spotless enforced)
- No `@author` tags, no wildcard imports (Checkstyle)
- `jandex-maven-plugin` on all library modules (PP-20260601-37179a)
- SPI evolution: new methods added as `default` with no-op returns
- Test classes named `*Test.java` (surefire), never `*IT.java`
- `TESTCONTAINERS_RYUK_DISABLED=true` for all test runs
- `mvn install -DskipTests -q` before module-specific tests
- All query objects follow `AuditQuery` pattern: immutable, builder,
  `all()` factory, zero-based page, size capped at max
- REST pagination: 1-based client page numbers, 0-based internal index

---

### Task 1: SPI pagination — query objects, SPI methods, in-memory implementations

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/query/CaseDefinitionQuery.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/query/CaseInstanceQuery.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/query/EventLogQuery.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseMetaModelRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java`
- Modify: `common/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseMetaModelRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/persistence/memory/InMemoryEventLogRepository.java`
- Create: `common/src/test/java/io/casehub/engine/common/spi/CaseMetaModelRepositoryContractTest.java`
- Create: `common/src/test/java/io/casehub/engine/common/spi/CaseInstanceRepositoryContractTest.java`
- Create: `common/src/test/java/io/casehub/engine/common/spi/EventLogRepositoryContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/persistence/memory/InMemoryCaseMetaModelRepositoryContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/persistence/memory/InMemoryCaseInstanceRepositoryContractTest.java`
- Create: `persistence-memory/src/test/java/io/casehub/persistence/memory/InMemoryEventLogRepositoryContractTest.java`

**Interfaces:**
- Produces: `CaseDefinitionQuery`, `CaseInstanceQuery`, `EventLogQuery` (used by Task 5+)
- Produces: `query()` and `count()` methods on all three blocking SPIs (used by Tasks 5-7)

- [ ] **Step 1: Write CaseDefinitionQuery**

Follow `AuditQuery` pattern exactly: immutable, builder, `all()` factory,
zero-based page, size capped at 100.

```java
package io.casehub.engine.common.spi.query;

public final class CaseDefinitionQuery {
    private static final int DEFAULT_SIZE = 20;
    private static final int MAX_SIZE = 100;

    private final String namespace;
    private final String name;
    private final int page;
    private final int size;

    private CaseDefinitionQuery(Builder b) {
        this.namespace = b.namespace;
        this.name = b.name;
        this.page = Math.max(0, b.page);
        this.size = Math.min(MAX_SIZE, Math.max(1, b.size));
    }

    public static CaseDefinitionQuery all() { return builder().build(); }
    public static Builder builder() { return new Builder(); }

    public String namespace() { return namespace; }
    public String name() { return name; }
    public int page() { return page; }
    public int size() { return size; }

    public static final class Builder {
        private String namespace;
        private String name;
        private int page = 0;
        private int size = DEFAULT_SIZE;
        private Builder() {}
        public Builder namespace(String namespace) { this.namespace = namespace; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder page(int page) { this.page = page; return this; }
        public Builder size(int size) { this.size = size; return this; }
        public CaseDefinitionQuery build() { return new CaseDefinitionQuery(this); }
    }
}
```

Use `ide_create_file` for this file.

- [ ] **Step 2: Write CaseInstanceQuery**

Same pattern. Additional filter: `CaseStatus status`.

```java
package io.casehub.engine.common.spi.query;

import io.casehub.api.model.CaseStatus;

public final class CaseInstanceQuery {
    private static final int DEFAULT_SIZE = 20;
    private static final int MAX_SIZE = 100;

    private final String namespace;
    private final String name;
    private final CaseStatus status;
    private final int page;
    private final int size;

    private CaseInstanceQuery(Builder b) {
        this.namespace = b.namespace;
        this.name = b.name;
        this.status = b.status;
        this.page = Math.max(0, b.page);
        this.size = Math.min(MAX_SIZE, Math.max(1, b.size));
    }

    public static CaseInstanceQuery all() { return builder().build(); }
    public static Builder builder() { return new Builder(); }

    public String namespace() { return namespace; }
    public String name() { return name; }
    public CaseStatus status() { return status; }
    public int page() { return page; }
    public int size() { return size; }

    public static final class Builder {
        private String namespace;
        private String name;
        private CaseStatus status;
        private int page = 0;
        private int size = DEFAULT_SIZE;
        private Builder() {}
        public Builder namespace(String namespace) { this.namespace = namespace; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder status(CaseStatus status) { this.status = status; return this; }
        public Builder page(int page) { this.page = page; return this; }
        public Builder size(int size) { this.size = size; return this; }
        public CaseInstanceQuery build() { return new CaseInstanceQuery(this); }
    }
}
```

- [ ] **Step 3: Write EventLogQuery**

`caseId` is required (no `all()` factory — event logs are always per-case).

```java
package io.casehub.engine.common.spi.query;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import java.util.Collection;
import java.util.Objects;
import java.util.UUID;

public final class EventLogQuery {
    private static final int DEFAULT_SIZE = 50;
    private static final int MAX_SIZE = 1000;

    private final UUID caseId;
    private final Collection<CaseHubEventType> eventTypes;
    private final Collection<EventStreamType> streamTypes;
    private final int page;
    private final int size;

    private EventLogQuery(Builder b) {
        this.caseId = Objects.requireNonNull(b.caseId, "caseId is required");
        this.eventTypes = b.eventTypes;
        this.streamTypes = b.streamTypes;
        this.page = Math.max(0, b.page);
        this.size = Math.min(MAX_SIZE, Math.max(1, b.size));
    }

    public static Builder builder(UUID caseId) { return new Builder(caseId); }

    public UUID caseId() { return caseId; }
    public Collection<CaseHubEventType> eventTypes() { return eventTypes; }
    public Collection<EventStreamType> streamTypes() { return streamTypes; }
    public int page() { return page; }
    public int size() { return size; }

    public static final class Builder {
        private final UUID caseId;
        private Collection<CaseHubEventType> eventTypes;
        private Collection<EventStreamType> streamTypes;
        private int page = 0;
        private int size = DEFAULT_SIZE;
        Builder(UUID caseId) { this.caseId = caseId; }
        public Builder eventTypes(Collection<CaseHubEventType> eventTypes) { this.eventTypes = eventTypes; return this; }
        public Builder streamTypes(Collection<EventStreamType> streamTypes) { this.streamTypes = streamTypes; return this; }
        public Builder page(int page) { this.page = page; return this; }
        public Builder size(int size) { this.size = size; return this; }
        public EventLogQuery build() { return new EventLogQuery(this); }
    }
}
```

- [ ] **Step 4: Add default methods to blocking SPIs**

Use `ide_insert_member` to add to each interface.

`CaseMetaModelRepository`:
```java
default List<CaseMetaModel> query(CaseDefinitionQuery query, String tenancyId) {
    return List.of();
}

default long count(CaseDefinitionQuery query, String tenancyId) {
    return 0;
}
```

`CaseInstanceRepository`:
```java
default List<CaseInstance> query(CaseInstanceQuery query, String tenancyId) {
    return List.of();
}

default long count(CaseInstanceQuery query, String tenancyId) {
    return 0;
}
```

`EventLogRepository`:
```java
default List<EventLog> query(EventLogQuery query, String tenancyId) {
    return List.of();
}

default long count(EventLogQuery query, String tenancyId) {
    return 0;
}
```

Add required imports for query classes and `List`.

- [ ] **Step 5: Write abstract contract tests**

Create in `common/src/test/java/io/casehub/engine/common/spi/`.
Pattern: abstract class with `protected abstract` repo accessor, tests
verify query/count behavior. Extends nothing (plain JUnit 5).

`CaseMetaModelRepositoryContractTest`:
```java
package io.casehub.engine.common.spi;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.query.CaseDefinitionQuery;
import org.junit.jupiter.api.Test;

public abstract class CaseMetaModelRepositoryContractTest {

    protected abstract CaseMetaModelRepository repository();
    protected abstract String tenancyId();

    protected CaseMetaModel createMetaModel(String namespace, String name, String version) {
        CaseMetaModel m = new CaseMetaModel();
        m.setNamespace(namespace);
        m.setName(name);
        m.setVersion(version);
        return repository().save(m, tenancyId());
    }

    @Test
    void queryAll_returnsAllForTenant() {
        createMetaModel("ns", "case-a", "1.0");
        createMetaModel("ns", "case-b", "1.0");
        var result = repository().query(CaseDefinitionQuery.all(), tenancyId());
        assertThat(result).hasSize(2);
    }

    @Test
    void queryByNamespace_filtersCorrectly() {
        createMetaModel("ns1", "case-a", "1.0");
        createMetaModel("ns2", "case-b", "1.0");
        var query = CaseDefinitionQuery.builder().namespace("ns1").build();
        assertThat(repository().query(query, tenancyId())).hasSize(1);
    }

    @Test
    void queryPagination_respectsPageAndSize() {
        for (int i = 0; i < 5; i++) {
            createMetaModel("ns", "case-" + i, "1.0");
        }
        var page0 = CaseDefinitionQuery.builder().size(2).page(0).build();
        var page1 = CaseDefinitionQuery.builder().size(2).page(1).build();
        assertThat(repository().query(page0, tenancyId())).hasSize(2);
        assertThat(repository().query(page1, tenancyId())).hasSize(2);
    }

    @Test
    void count_matchesQueryFilters() {
        createMetaModel("ns1", "case-a", "1.0");
        createMetaModel("ns2", "case-b", "1.0");
        assertThat(repository().count(CaseDefinitionQuery.all(), tenancyId())).isEqualTo(2);
        var filtered = CaseDefinitionQuery.builder().namespace("ns1").build();
        assertThat(repository().count(filtered, tenancyId())).isEqualTo(1);
    }
}
```

Similar contract tests for `CaseInstanceRepository` (filter by status,
namespace, name) and `EventLogRepository` (filter by eventType,
streamType, caseId required).

- [ ] **Step 6: Implement query/count in InMemoryCaseMetaModelRepository**

Use `ide_insert_member` to add methods. Pattern: read-lock, stream,
filter by non-null query fields, skip/limit for pagination.

```java
@Override
public List<CaseMetaModel> query(CaseDefinitionQuery query, String tenancyId) {
    rwLock.readLock().lock();
    try {
        return store.values().stream()
            .filter(m -> m.tenancyId != null && m.tenancyId.equals(tenancyId))
            .filter(m -> query.namespace() == null || query.namespace().equals(m.getNamespace()))
            .filter(m -> query.name() == null || query.name().equals(m.getName()))
            .skip((long) query.page() * query.size())
            .limit(query.size())
            .toList();
    } finally {
        rwLock.readLock().unlock();
    }
}

@Override
public long count(CaseDefinitionQuery query, String tenancyId) {
    rwLock.readLock().lock();
    try {
        return store.values().stream()
            .filter(m -> m.tenancyId != null && m.tenancyId.equals(tenancyId))
            .filter(m -> query.namespace() == null || query.namespace().equals(m.getNamespace()))
            .filter(m -> query.name() == null || query.name().equals(m.getName()))
            .count();
    } finally {
        rwLock.readLock().unlock();
    }
}
```

Repeat pattern for `InMemoryCaseInstanceRepository` (filter by status,
namespace via `getCaseMetaModel().getNamespace()`, name via
`getCaseMetaModel().getName()`).

Repeat pattern for `InMemoryEventLogRepository` (filter by caseId,
eventTypes, streamTypes).

- [ ] **Step 7: Create concrete contract test classes in persistence-memory**

```java
package io.casehub.persistence.memory;

import io.casehub.engine.common.spi.CaseMetaModelRepository;
import io.casehub.engine.common.spi.CaseMetaModelRepositoryContractTest;

class InMemoryCaseMetaModelRepositoryContractTest
    extends CaseMetaModelRepositoryContractTest {

    private final InMemoryCaseMetaModelRepository repo =
        new InMemoryCaseMetaModelRepository();

    @Override protected CaseMetaModelRepository repository() { return repo; }
    @Override protected String tenancyId() { return "test-tenant"; }
}
```

Same pattern for the other two repositories.

- [ ] **Step 8: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common -Dtest="*ContractTest" -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl persistence-memory -Dtest="*ContractTest" -q
```

- [ ] **Step 9: Commit**

```bash
git add common/src persistence-memory/src
git commit -m "feat(#762): add pagination query objects and SPI methods

Refs #762"
```

---

### Task 2: JPA pagination implementations

**Files:**
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactiveCaseMetaModelRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseMetaModelRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactiveCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaCaseInstanceRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactiveEventLogRepository.java`
- Modify: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaEventLogRepository.java`

**Interfaces:**
- Consumes: query objects from Task 1
- Produces: JPA implementations of query/count (used by consumers with persistence-hibernate)

- [ ] **Step 1: Add reactive query/count to JpaReactiveCaseMetaModelRepository**

The blocking JPA repos delegate to reactive. Add the reactive method
first, then have the blocking one delegate.

```java
public Uni<List<CaseMetaModel>> query(CaseDefinitionQuery query, String tenancyId) {
    return withTenantTransaction(tenancyId, (session) -> {
        StringBuilder jpql = new StringBuilder("FROM CaseMetaModelEntity e WHERE 1=1");
        if (query.namespace() != null) jpql.append(" AND e.namespace = :namespace");
        if (query.name() != null) jpql.append(" AND e.name = :name");

        var q = session.createQuery(jpql.toString(), CaseMetaModelEntity.class);
        if (query.namespace() != null) q.setParameter("namespace", query.namespace());
        if (query.name() != null) q.setParameter("name", query.name());
        q.setFirstResult(query.page() * query.size());
        q.setMaxResults(query.size());

        return q.getResultList().map(entities ->
            entities.stream().map(this::toModel).toList());
    });
}

public Uni<Long> count(CaseDefinitionQuery query, String tenancyId) {
    return withTenantTransaction(tenancyId, (session) -> {
        StringBuilder jpql = new StringBuilder("SELECT COUNT(e) FROM CaseMetaModelEntity e WHERE 1=1");
        if (query.namespace() != null) jpql.append(" AND e.namespace = :namespace");
        if (query.name() != null) jpql.append(" AND e.name = :name");

        var q = session.createQuery(jpql.toString(), Long.class);
        if (query.namespace() != null) q.setParameter("namespace", query.namespace());
        if (query.name() != null) q.setParameter("name", query.name());

        return q.getSingleResult();
    });
}
```

Note: `toModel()` maps `CaseMetaModelEntity` → `CaseMetaModel`. Check
if this method already exists on the class — if not, extract from the
scaffold's `CaseDefinitionService.toCaseMetaModel()`.

- [ ] **Step 2: Add blocking delegation to JpaCaseMetaModelRepository**

```java
@Override
public List<CaseMetaModel> query(CaseDefinitionQuery query, String tenancyId) {
    return delegate.query(query, tenancyId).await().indefinitely();
}

@Override
public long count(CaseDefinitionQuery query, String tenancyId) {
    return delegate.count(query, tenancyId).await().indefinitely();
}
```

- [ ] **Step 3: Repeat for CaseInstance**

Same JPQL pattern. Filter by `e.state` for status, join
`e.caseMetaModel` for namespace/name:

```sql
FROM CaseInstanceEntity e JOIN FETCH e.caseMetaModel m WHERE 1=1
  AND (:status IS NULL OR e.state = :status)
  AND (:namespace IS NULL OR m.namespace = :namespace)
  AND (:name IS NULL OR m.name = :name)
```

- [ ] **Step 4: Repeat for EventLog**

Filter by caseId (required), eventTypes, streamTypes:

```sql
FROM EventLogEntity e WHERE e.caseId = :caseId
  AND (:eventTypes IS NULL OR e.eventType IN :eventTypes)
  AND (:streamTypes IS NULL OR e.streamType IN :streamTypes)
```

- [ ] **Step 5: Run persistence-hibernate tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl persistence-hibernate -q
```

- [ ] **Step 6: Commit**

```bash
git add persistence-hibernate/src
git commit -m "feat(#762): JPA query/count implementations for pagination

Refs #762"
```

---

### Task 3: REST module scaffold — pom, DTOs, exceptions

**Files:**
- Create: `rest/pom.xml`
- Modify: `pom.xml` (root — add `rest` to modules)
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/StartCaseRequest.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/CaseInstanceResponse.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/CaseControlRequest.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/CaseControlResponse.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/SendSignalRequest.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/SignalResponse.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/EventLogEntryResponse.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/PagedResponse.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/dto/ProblemDetail.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/EntityNotFoundException.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/ConstraintViolationExceptionMapper.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/AccessDeniedExceptionMapper.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/EntityNotFoundExceptionMapper.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/IllegalStateExceptionMapper.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/exception/CatchAllExceptionMapper.java`

**Interfaces:**
- Produces: all DTOs and exception types (used by Tasks 4-5)

- [ ] **Step 1: Create pom.xml**

Write `rest/pom.xml` with all dependencies from the spec. Include
`jandex-maven-plugin`, `quarkus-maven-plugin`, `maven-failsafe-plugin`.
Parent: `casehub-engine-parent`. ArtifactId: `casehub-engine-rest`.

- [ ] **Step 2: Add `rest` to root pom modules**

Use `Edit` on `pom.xml` to add `<module>rest</module>` to the `<modules>`
section, after `flow`.

- [ ] **Step 3: Create test application.properties**

Create `rest/src/test/resources/application.properties`:

```properties
quarkus.http.test-port=0
quarkus.index-dependency.persistence-memory.group-id=io.casehub
quarkus.index-dependency.persistence-memory.artifact-id=casehub-persistence-memory
quarkus.index-dependency.engine-common.group-id=io.casehub
quarkus.index-dependency.engine-common.artifact-id=casehub-engine-common
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository,\
  io.casehub.persistence.memory.InMemoryReactiveCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryReactiveCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryReactiveEventLogRepository,\
  io.casehub.persistence.memory.InMemoryReactiveSubCaseGroupRepository,\
  io.casehub.persistence.memory.InMemoryReactivePlanItemStore,\
  io.casehub.persistence.memory.InMemoryPlanItemStore,\
  io.casehub.persistence.memory.InMemorySubCaseGroupRepository
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.username=sa
quarkus.datasource.password=
quarkus.hibernate-orm.schema-management.strategy=drop-and-create
quarkus.flyway.migrate-at-start=false
```

Note: check if additional ledger/blackboard alternatives are needed at
test time by running `mvn test` and observing unsatisfied dependency
errors. Add `NoOpLedgerEntryRepository` if `casehub-engine-ledger` is
transitively present (see CLAUDE.md pattern).

- [ ] **Step 4: Move DTOs from scaffold**

Copy each DTO from `scaffold/src/main/java/io/casehub/flow/rest/dto/`
to `rest/src/main/java/io/casehub/engine/rest/dto/`, changing the
package declaration from `io.casehub.flow.rest.dto` to
`io.casehub.engine.rest.dto`.

Use `ide_create_file` for each. Source content from the scaffold files
read earlier in this session.

Changes from scaffold originals:
- `CaseInstanceResponse`: drop `updatedAt` field. Add static
  `from(CaseInstance)` factory:
  ```java
  public static CaseInstanceResponse from(CaseInstance instance) {
      CaseMetaModel meta = instance.getCaseMetaModel();
      return new CaseInstanceResponse(
          instance.getUuid(), instance.getState(),
          meta.getNamespace(), meta.getName(), meta.getVersion(),
          meta.getCreatedAt());
  }
  ```
- `CaseControlResponse`: remove `message` field — 3-arg constructor
  `(UUID caseId, String operation, String status)`. Update `@Schema`
  status code annotations from `202` to `200`.
- `PagedResponse`: rename `content` → `items` per spec.
- All others: package change only.

- [ ] **Step 5: Create EntityNotFoundException**

```java
package io.casehub.engine.rest.exception;

public class EntityNotFoundException extends RuntimeException {
    public EntityNotFoundException(String message) {
        super(message);
    }
}
```

- [ ] **Step 6: Create exception mappers**

Five `@Provider` classes in `io.casehub.engine.rest.exception`. All
produce `ProblemDetail`. Move `ConstraintViolationExceptionMapper` and
`AccessDeniedExceptionMapper` from scaffold, change package and imports.
Create `EntityNotFoundExceptionMapper` (404),
`IllegalStateExceptionMapper` (409), `CatchAllExceptionMapper` (500
with logged exception, no stack trace in response).

- [ ] **Step 7: Verify compilation**

```bash
mvn install -DskipTests -q
mvn compile -pl rest -q
```

- [ ] **Step 8: Commit**

```bash
git add rest/ pom.xml
git commit -m "feat(#762): engine-rest module scaffold — pom, DTOs, exception mappers

Refs #762"
```

---

### Task 4: CaseService and CaseDefinitionResource

**Files:**
- Create: `rest/src/main/java/io/casehub/engine/rest/service/CaseService.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/CaseDefinitionResource.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/CaseDefinitionResourceTest.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/service/CaseServiceTest.java`

**Interfaces:**
- Consumes: `CaseDefinitionRegistry`, `CaseHubRuntime`,
  `CaseInstanceRepository`, `CaseMetaModelRepository`,
  `CurrentPrincipal`, `CaseDefinitionQuery`, all DTOs
- Produces: `CaseService` (used by Tasks 5), `CaseDefinitionResource`

- [ ] **Step 1: Write CaseService test**

```java
package io.casehub.engine.rest.service;

import static org.assertj.core.api.Assertions.*;
import io.casehub.engine.rest.exception.EntityNotFoundException;
// ... test startCase happy path, startCase definition-not-found,
// requireCase happy path, requireCase not-found
```

Unit test with mock/stub SPIs (no Quarkus, plain JUnit).

- [ ] **Step 2: Write CaseService**

Thin service per spec: `startCase(namespace, name, version, context,
tenancyId)` and `requireCase(caseId, tenancyId)`. Injects
`CaseDefinitionRegistry`, `CaseHubRuntime`, `CaseInstanceRepository`.
Throws `EntityNotFoundException` for not-found cases.

- [ ] **Step 3: Write CaseDefinitionResource test**

`@QuarkusTest` with REST Assured:
- `GET /api/v1/case-definitions` — returns 200 with paginated list
- `GET /api/v1/case-definitions/{ns}/{name}` — returns 200 with matches
- `GET /api/v1/case-definitions/{ns}/{name}/{version}` — returns 200 or 404
- Pagination: page/size query params, default values

Requires a test `CaseHub` subclass that registers a definition at
startup.

- [ ] **Step 4: Write CaseDefinitionResource**

Move from scaffold's `CaseDefinitionResource`, refactor:
- Package: `io.casehub.engine.rest`
- Strip ACL checks
- Strip Uni chains → imperative with `@RunOnVirtualThread`
- Replace `CaseDefinitionService` injection with
  `CaseMetaModelRepository` + `CaseDefinitionRegistry`
- Keep OpenAPI annotations, update `@APIResponse` status codes

- [ ] **Step 5: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q
```

- [ ] **Step 6: Commit**

```bash
git add rest/src
git commit -m "feat(#762): CaseService and CaseDefinitionResource

Refs #762"
```

---

### Task 5: CaseInstanceResource

**Files:**
- Create: `rest/src/main/java/io/casehub/engine/rest/CaseInstanceResource.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/CaseInstanceResourceTest.java`

**Interfaces:**
- Consumes: `CaseService`, `CaseInstanceRepository`,
  `CaseHubRuntime`, `CurrentPrincipal`, `CaseInstanceQuery`, DTOs
- Produces: `CaseInstanceResource`

- [ ] **Step 1: Write CaseInstanceResource test**

`@QuarkusTest` with REST Assured:
- `GET /api/v1/cases` — 200 with paginated list, filter by status/namespace/name
- `POST /api/v1/cases` — 201 with started case, 404 for unknown definition, 400 for invalid request
- `GET /api/v1/cases/{id}` — 200 with case, 404 for unknown
- `GET /api/v1/cases/{id}/context` — 200 with context map
- `GET /api/v1/cases/{id}/context/{path}` — 200 with value at path

- [ ] **Step 2: Write CaseInstanceResource**

Move from scaffold's `CaseInstanceResource` + `CaseControlResource`
context endpoints. Refactor: strip Uni, strip ACL, inject
`CaseService` + `CaseInstanceRepository`. Add `listCases` endpoint
(new — not in scaffold). Context queries pre-verify via
`caseService.requireCase()` then call `runtime.query()`.

```java
@GET
@RunOnVirtualThread
@Operation(summary = "List case instances")
public PagedResponse<CaseInstanceResponse> listCases(
        @QueryParam("page") @DefaultValue("1") @Min(1) int page,
        @QueryParam("size") @DefaultValue("20") @Min(1) @Max(100) int size,
        @QueryParam("status") CaseStatus status,
        @QueryParam("namespace") String namespace,
        @QueryParam("name") String name) {
    var query = CaseInstanceQuery.builder()
        .status(status).namespace(namespace).name(name)
        .page(page - 1).size(size).build();
    String tenancyId = currentPrincipal.tenancyId();
    var items = instanceRepository.query(query, tenancyId)
        .stream().map(CaseInstanceResponse::from).toList();
    long total = instanceRepository.count(query, tenancyId);
    int totalPages = (int) Math.ceil((double) total / size);
    return new PagedResponse<>(items, page, size, total, totalPages);
}
```

- [ ] **Step 3: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q
```

- [ ] **Step 4: Commit**

```bash
git add rest/src
git commit -m "feat(#762): CaseInstanceResource — list, start, get, context

Refs #762"
```

---

### Task 6: CaseControlResource, SignalResource, EventLogResource

**Files:**
- Create: `rest/src/main/java/io/casehub/engine/rest/CaseControlResource.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/SignalResource.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/EventLogResource.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/CaseControlResourceTest.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/SignalResourceTest.java`
- Create: `rest/src/test/java/io/casehub/engine/rest/EventLogResourceTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime`, `EventLogRepository`, `CaseService`,
  `CurrentPrincipal`, `EventLogQuery`, DTOs
- Produces: final three resources

- [ ] **Step 1: Write CaseControlResource test**

`@QuarkusTest`:
- `POST /api/v1/cases/{id}/suspend` — 200 for running case, 404 for unknown, 409 for invalid state
- `POST /api/v1/cases/{id}/resume` — 200 for suspended case
- `POST /api/v1/cases/{id}/cancel` — 200 for active case

- [ ] **Step 2: Write CaseControlResource**

Move from scaffold. Strip Uni chains, ACL, `Uni.createFrom().emitter()`.
Replace with imperative `try { runtime.suspendCase(caseId); } catch
(IllegalArgumentException e) { throw new EntityNotFoundException(...); }`.
`@RunOnVirtualThread`. Return `CaseControlResponse`, not `Response`.
HTTP 200, not 202.

- [ ] **Step 3: Write SignalResource test**

`@QuarkusTest`:
- `POST /api/v1/cases/{id}/signals` — 200 with valid signal, 400 for
  invalid request, 404 for unknown case

- [ ] **Step 4: Write SignalResource**

Move from scaffold. Strip Uni, ACL. Pre-verify case exists via
`caseService.requireCase()`. Call `runtime.signal(caseId, path, value)
.toCompletableFuture().join()`. Return `SignalResponse`. HTTP 200.

- [ ] **Step 5: Write EventLogResource test**

`@QuarkusTest`:
- `GET /api/v1/cases/{id}/events` — 200 with paginated events
- Filter by eventType, streamType query params
- 404 for unknown case

- [ ] **Step 6: Write EventLogResource**

Move from scaffold. Strip Uni, ACL, scaffold's `EventLogService`.
Inject `EventLogRepository` directly. Use `EventLogQuery` for
pagination. Pre-verify case via `caseService.requireCase()`. Map
`EventLog` → `EventLogEntryResponse` inline.

Note: scaffold's `EventLogService` used `CaseHubRuntime.eventLog()`
which returns `CaseEventLogRecord`. Engine-rest uses
`EventLogRepository.query()` which returns `EventLog` (internal model).
Map fields: `eventLog.getEventType()`, `eventLog.getStreamType()`,
`eventLog.getTimestamp()`, `eventLog.getPayload()`,
`eventLog.getMetadata()`.

- [ ] **Step 7: Run all rest tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q
```

- [ ] **Step 8: Commit**

```bash
git add rest/src
git commit -m "feat(#762): CaseControlResource, SignalResource, EventLogResource

Refs #762"
```

---

### Task 7: Full build verification and cleanup

**Files:**
- Possibly modify: various files for compilation fixes discovered during
  full build

- [ ] **Step 1: Full build**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest -q
```

- [ ] **Step 2: Run full engine test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -q
```

Verify no regressions in existing modules.

- [ ] **Step 3: Verify diagnostics**

Use `ide_diagnostics` on each new file in the rest module to confirm
no warnings or errors.

- [ ] **Step 4: Verify OpenAPI**

Start the rest module in dev mode (or via a test) and verify the
OpenAPI spec is generated correctly. Check that all endpoints appear
with correct paths, methods, request/response schemas.

- [ ] **Step 5: Commit any fixes**

```bash
git add -A
git commit -m "fix(#762): build verification fixes

Refs #762"
```

Only if fixes were needed — skip if build was clean.
