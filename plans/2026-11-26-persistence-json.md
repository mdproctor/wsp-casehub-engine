# persistence-json Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1366 — JSON collection SPI persistence adapter
**Issue group:** casehubio/examples#212 (YAML helpdesk apps)

**Goal:** Implement `CaseInstanceCache` backed by `JsonCollectionStore` (SQLite) so zero-Java case applications get persistence without Hibernate or PostgreSQL.

**Architecture:** New `persistence-json` module with `JsonCollectionCaseInstanceCache` implementing the 4-method `CaseInstanceCache` SPI. Serializes `CaseInstance` state to a flat `CaseInstanceDocument` record, stores in `json_documents` table via `JsonCollectionStore`. On retrieval, reconstructs `CaseInstance` from stored fields plus `CaseMetaModel` looked up from `CaseDefinitionRegistry`.

**Tech Stack:** Java 21, Quarkus CDI, Jackson, SQLite (via casehub-pages-sqlite), casehub-pages-data-json

## Global Constraints

- Java 21, Quarkus 3.32.2
- All cross-project deps use `${version.io.casehub}` (0.4-SNAPSHOT)
- Module naming: `casehub-engine-persistence-json`
- Test classes must be `*Test.java` (never `*IT.java`)
- Inject repos by SPI interface, not concrete class

---

## Batch 1: Module scaffold + CaseInstanceDocument + round-trip test

### Task 1: Create persistence-json module with CaseInstanceDocument and JsonCollectionCaseInstanceCache

**Files:**
- Create: `persistence-json/pom.xml`
- Create: `persistence-json/src/main/java/io/casehub/engine/persistence/json/CaseInstanceDocument.java`
- Create: `persistence-json/src/main/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCache.java`
- Create: `persistence-json/src/main/java/io/casehub/engine/persistence/json/quarkus/PersistenceJsonBeans.java`
- Create: `persistence-json/src/test/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCacheTest.java`
- Create: `persistence-json/src/test/resources/application.properties`
- Modify: `pom.xml` (root) — add `<module>persistence-json</module>`

**Interfaces:**
- Consumes: `CaseInstanceCache` (SPI from `common-core`), `JsonCollectionStore` (from `casehub-pages-data-json`), `CaseDefinitionRegistry` (from `common-core`), `CurrentPrincipal` (from `casehub-platform-api`)
- Produces: `JsonCollectionCaseInstanceCache` — CDI `@Alternative @Priority(200)` bean implementing `CaseInstanceCache`

- [ ] **Step 1: Create `persistence-json/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-parent</artifactId>
        <version>0.4-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-engine-persistence-json</artifactId>
    <name>Case Hub :: Persistence :: JSON</name>
    <description>CaseInstanceCache backed by JSON collections (SQLite via casehub-pages-data-json)</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-common-core</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-data-json</artifactId>
            <version>${version.io.casehub}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.logging</groupId>
            <artifactId>jboss-logging</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-pages-sqlite</artifactId>
            <version>${version.io.casehub}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals>
                            <goal>jandex</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                        <maven.home>${maven.home}</maven.home>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>persistence-json</module>` after `<module>persistence-memory</module>` (line 104) in the root `pom.xml`.

- [ ] **Step 3: Create `CaseInstanceDocument.java`**

Path: `persistence-json/src/main/java/io/casehub/engine/persistence/json/CaseInstanceDocument.java`

```java
package io.casehub.engine.persistence.json;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.Set;

public record CaseInstanceDocument(
    String uuid,
    String state,
    long version,
    String metaNamespace,
    String metaName,
    String metaVersion,
    JsonNode contextSnapshot,
    String parentCaseId,
    String parentPlanItemId,
    String actorId,
    Set<String> labels,
    Set<String> types,
    String createdAt
) {}
```

- [ ] **Step 4: Write the failing test**

Path: `persistence-json/src/test/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCacheTest.java`

```java
package io.casehub.engine.persistence.json;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.pages.data.DataContext;
import io.casehub.pages.data.json.JsonCollectionStore;
import io.casehub.pages.sqlite.SqliteConnectionFactory;
import io.casehub.platform.api.identity.CurrentPrincipal;
import java.time.Instant;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class JsonCollectionCaseInstanceCacheTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private static final String TENANT = "test-tenant";

  JsonCollectionCaseInstanceCache cache;
  JsonCollectionStore store;
  CaseDefinitionRegistry registry;
  CurrentPrincipal principal;

  @BeforeEach
  void setUp() {
    SqliteConnectionFactory connFactory = createInMemorySqlite();
    store = new JsonCollectionStore(connFactory);
    registry = mock(CaseDefinitionRegistry.class);
    principal = mock(CurrentPrincipal.class);
    when(principal.tenancyId()).thenReturn(TENANT);
    cache = new JsonCollectionCaseInstanceCache(store, registry, principal);
  }

  @Test
  void put_thenGet_roundTrips() {
    CaseInstance instance = createInstance("test-ns", "test-case", "1.0", CaseStatus.ACTIVE);
    UUID uuid = instance.getUuid();

    cache.put(instance);
    CaseInstance retrieved = cache.get(uuid);

    assertThat(retrieved).isNotNull();
    assertThat(retrieved.getUuid()).isEqualTo(uuid);
    assertThat(retrieved.getState()).isEqualTo(CaseStatus.ACTIVE);
    assertThat(retrieved.getVersion()).isEqualTo(instance.getVersion());
    assertThat(retrieved.getActorId()).isEqualTo(instance.getActorId());
    assertThat(retrieved.getContextSnapshot()).isEqualTo(instance.getContextSnapshot());
  }

  @Test
  void put_existingEntry_updates() {
    CaseInstance instance = createInstance("ns", "case", "1.0", CaseStatus.ACTIVE);
    UUID uuid = instance.getUuid();
    cache.put(instance);

    instance.setState(CaseStatus.COMPLETED);
    instance.setVersion(2L);
    cache.put(instance);

    CaseInstance retrieved = cache.get(uuid);
    assertThat(retrieved.getState()).isEqualTo(CaseStatus.COMPLETED);
    assertThat(retrieved.getVersion()).isEqualTo(2L);
  }

  @Test
  void get_nonExistent_returnsNull() {
    assertThat(cache.get(UUID.randomUUID())).isNull();
  }

  @Test
  void getAll_returnsAllEntries() {
    cache.put(createInstance("ns", "a", "1.0", CaseStatus.ACTIVE));
    cache.put(createInstance("ns", "b", "1.0", CaseStatus.ACTIVE));
    cache.put(createInstance("ns", "c", "1.0", CaseStatus.COMPLETED));

    assertThat(cache.getAll()).hasSize(3);
  }

  @Test
  void clear_removesAll() {
    cache.put(createInstance("ns", "a", "1.0", CaseStatus.ACTIVE));
    cache.put(createInstance("ns", "b", "1.0", CaseStatus.ACTIVE));

    cache.clear();

    assertThat(cache.getAll()).isEmpty();
  }

  @Test
  void reconstruction_setsMetadataFromRegistry() {
    CaseMetaModel meta = new CaseMetaModel();
    meta.setNamespace("reg-ns");
    meta.setName("reg-case");
    meta.setVersion("2.0");
    when(registry.findByIdentity("reg-ns", "reg-case", "2.0"))
        .thenReturn(Optional.of(meta));

    CaseInstance instance = createInstance("reg-ns", "reg-case", "2.0", CaseStatus.ACTIVE);
    cache.put(instance);

    CaseInstance retrieved = cache.get(instance.getUuid());
    assertThat(retrieved.getCaseMetaModel()).isNotNull();
    assertThat(retrieved.getCaseMetaModel().getNamespace()).isEqualTo("reg-ns");
    assertThat(retrieved.getCaseMetaModel().getName()).isEqualTo("reg-case");
  }

  private CaseInstance createInstance(String ns, String name, String version, CaseStatus status) {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setState(status);
    instance.setVersion(1L);
    instance.setActorId("test-actor");
    instance.setLabels(Set.of("label-a"));
    instance.setTypes(Set.of("type-a"));
    instance.setCreatedAt(Instant.now());

    CaseMetaModel meta = new CaseMetaModel();
    meta.setNamespace(ns);
    meta.setName(name);
    meta.setVersion(version);
    instance.setCaseMetaModel(meta);

    ObjectNode snapshot = MAPPER.createObjectNode();
    snapshot.put("key", "value");
    instance.setContextSnapshot(snapshot);

    return instance;
  }

  private static SqliteConnectionFactory createInMemorySqlite() {
    // Direct instantiation with in-memory SQLite for unit testing
    org.sqlite.SQLiteDataSource ds = new org.sqlite.SQLiteDataSource();
    ds.setUrl("jdbc:sqlite::memory:");
    return new SqliteConnectionFactory() {
      @Override
      public java.sql.Connection getConnection() {
        try {
          return ds.getConnection();
        } catch (java.sql.SQLException e) {
          throw new RuntimeException(e);
        }
      }

      @Override
      public javax.sql.DataSource getDataSource() {
        return ds;
      }
    };
  }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn test -pl persistence-json -Dtest="JsonCollectionCaseInstanceCacheTest" -f pom.xml`
Expected: Compilation failure — `JsonCollectionCaseInstanceCache` does not exist yet.

- [ ] **Step 6: Implement `JsonCollectionCaseInstanceCache.java`**

Path: `persistence-json/src/main/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCache.java`

```java
package io.casehub.engine.persistence.json;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.pages.data.DataContext;
import io.casehub.pages.data.json.JsonCollectionStore;
import io.casehub.pages.data.json.MergeStrategy;
import io.casehub.platform.api.identity.CurrentPrincipal;
import java.time.Instant;
import java.util.HashSet;
import java.util.List;
import java.util.Optional;
import java.util.OptionalInt;
import java.util.Set;
import java.util.UUID;
import org.jboss.logging.Logger;

public class JsonCollectionCaseInstanceCache implements CaseInstanceCache {

  private static final Logger LOG = Logger.getLogger(JsonCollectionCaseInstanceCache.class);
  private static final ObjectMapper MAPPER = new ObjectMapper();
  private static final String APP_ID = "casehub-engine";
  private static final String COLLECTION = "case-instances";

  private final JsonCollectionStore store;
  private final CaseDefinitionRegistry registry;
  private final CurrentPrincipal principal;

  public JsonCollectionCaseInstanceCache(
      JsonCollectionStore store,
      CaseDefinitionRegistry registry,
      CurrentPrincipal principal) {
    this.store = store;
    this.registry = registry;
    this.principal = principal;
  }

  @Override
  public void put(CaseInstance instance) {
    CaseInstanceDocument doc = toDocument(instance);
    JsonNode json = MAPPER.valueToTree(doc);
    DataContext ctx = new DataContext(principal.tenancyId());
    String docId = instance.getUuid().toString();

    Optional<JsonNode> existing = store.get(ctx, APP_ID, COLLECTION, docId);
    if (existing.isPresent()) {
      store.update(ctx, APP_ID, COLLECTION, docId, json, OptionalInt.empty(), MergeStrategy.SHALLOW);
    } else {
      store.create(ctx, APP_ID, COLLECTION, json, null);
    }
  }

  @Override
  public CaseInstance get(UUID caseId) {
    DataContext ctx = new DataContext(principal.tenancyId());
    Optional<JsonNode> doc = store.get(ctx, APP_ID, COLLECTION, caseId.toString());
    return doc.map(this::fromDocument).orElse(null);
  }

  @Override
  public void clear() {
    DataContext ctx = new DataContext(principal.tenancyId());
    List<JsonNode> all = store.listAll(principal.tenancyId(), APP_ID, COLLECTION);
    for (JsonNode doc : all) {
      JsonNode idNode = doc.get("uuid");
      if (idNode != null) {
        store.delete(ctx, APP_ID, COLLECTION, idNode.asText());
      }
    }
  }

  @Override
  public List<CaseInstance> getAll() {
    List<JsonNode> docs = store.listAll(principal.tenancyId(), APP_ID, COLLECTION);
    return docs.stream().map(this::fromDocument).toList();
  }

  private CaseInstanceDocument toDocument(CaseInstance instance) {
    CaseMetaModel meta = instance.getCaseMetaModel();
    return new CaseInstanceDocument(
        instance.getUuid().toString(),
        instance.getState() != null ? instance.getState().name() : null,
        instance.getVersion(),
        meta != null ? meta.getNamespace() : null,
        meta != null ? meta.getName() : null,
        meta != null ? meta.getVersion() : null,
        instance.getContextSnapshot(),
        instance.getParentCaseId() != null ? instance.getParentCaseId().toString() : null,
        instance.getParentPlanItemId() != null ? instance.getParentPlanItemId().toString() : null,
        instance.getActorId(),
        instance.getLabels(),
        instance.getTypes(),
        instance.getCreatedAt() != null ? instance.getCreatedAt().toString() : null);
  }

  private CaseInstance fromDocument(JsonNode json) {
    try {
      CaseInstanceDocument doc = MAPPER.treeToValue(json, CaseInstanceDocument.class);
      CaseInstance instance = new CaseInstance();
      instance.setUuid(UUID.fromString(doc.uuid()));
      if (doc.state() != null) {
        instance.setState(io.casehub.api.model.CaseStatus.valueOf(doc.state()));
      }
      instance.setVersion(doc.version());
      instance.setActorId(doc.actorId());
      instance.setContextSnapshot(doc.contextSnapshot());
      instance.setLabels(doc.labels() != null ? new HashSet<>(doc.labels()) : new HashSet<>());
      instance.setTypes(doc.types() != null ? new HashSet<>(doc.types()) : new HashSet<>());
      if (doc.parentCaseId() != null) {
        instance.setParentCaseId(UUID.fromString(doc.parentCaseId()));
      }
      if (doc.parentPlanItemId() != null) {
        instance.setParentPlanItemId(UUID.fromString(doc.parentPlanItemId()));
      }
      if (doc.createdAt() != null) {
        instance.setCreatedAt(Instant.parse(doc.createdAt()));
      }

      if (doc.metaNamespace() != null && doc.metaName() != null && doc.metaVersion() != null) {
        Optional<CaseMetaModel> meta =
            registry.findByIdentity(doc.metaNamespace(), doc.metaName(), doc.metaVersion());
        meta.ifPresent(instance::setCaseMetaModel);
        if (meta.isEmpty()) {
          CaseMetaModel stub = new CaseMetaModel();
          stub.setNamespace(doc.metaNamespace());
          stub.setName(doc.metaName());
          stub.setVersion(doc.metaVersion());
          instance.setCaseMetaModel(stub);
        }
      }
      return instance;
    } catch (Exception e) {
      LOG.errorf(e, "Failed to deserialize CaseInstance from JSON collection");
      return null;
    }
  }
}
```

- [ ] **Step 7: Create `PersistenceJsonBeans.java`**

Path: `persistence-json/src/main/java/io/casehub/engine/persistence/json/quarkus/PersistenceJsonBeans.java`

```java
package io.casehub.engine.persistence.json.quarkus;

import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.persistence.json.JsonCollectionCaseInstanceCache;
import io.casehub.pages.data.json.JsonCollectionStore;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class PersistenceJsonBeans {

  @Produces
  @Alternative
  @Priority(200)
  @ApplicationScoped
  JsonCollectionCaseInstanceCache jsonCollectionCaseInstanceCache(
      JsonCollectionStore store,
      CaseDefinitionRegistry registry,
      CurrentPrincipal principal) {
    return new JsonCollectionCaseInstanceCache(store, registry, principal);
  }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn test -pl persistence-json -Dtest="JsonCollectionCaseInstanceCacheTest" -f pom.xml`
Expected: All 6 tests PASS.

Note: The test bypasses CDI — it directly instantiates `JsonCollectionCaseInstanceCache` with an in-memory SQLite `SqliteConnectionFactory`. The `SqliteConnectionFactory` test helper may need adjustment depending on whether the class has accessible constructors or is CDI-only. If CDI-only, mock the connection factory or create a test subclass.

- [ ] **Step 9: Compile the full project**

Run: `mvn install -DskipTests -q -f pom.xml`
Expected: BUILD SUCCESS — all modules compile including the new one.

- [ ] **Step 10: Commit**

```bash
git add persistence-json/ pom.xml
git commit -m "feat(#1366): add persistence-json module — CaseInstanceCache backed by JSON collections

Implements CaseInstanceCache using JsonCollectionStore (SQLite) for
zero-Java case applications. Serializes CaseInstance to flat document,
reconstructs on retrieval via CaseDefinitionRegistry.

Refs casehubio/engine#1366

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-26-persistence-json-design.md] — design spec this plan implements
- [common-core/.../CaseInstanceCache.java] — 4-method SPI contract
- [common-core/.../CaseInstance.java] — domain object with contextSnapshot field
- [runtime-core/.../CaseInstanceCacheImpl.java] — in-memory reference implementation
- [persistence-memory/pom.xml] — module structure reference
- [pages/backend/data-json/.../JsonCollectionStore.java] — backing store API
- [pages/backend/data/.../DataContext.java] — tenant context record
- [pages/backend/data-json/.../MergeStrategy.java] — merge strategies for update
- [casehubio/engine#1366] — focal issue
