# DataStore Platform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1332 — JSON collection SPI adapter for case persistence
**Issue group:** #1332

**Goal:** Deliver a platform-level DataStore SPI with document-backed and table-backed SQLite implementations, plus a CaseInstanceRepository adapter in the engine.

**Architecture:** Three new engine modules: `data-api` (SPI interfaces, package `io.casehub.platform.api.data` — to be promoted to `casehub-platform-api` on cross-repo adoption), `data-sqlite` (SQLite implementations extracted from JsonCollectionStore patterns), and `persistence-datastore` (CaseInstanceRepository adapter). In-memory DataStore implementation lives in `data-api` test-jar for use by any module's tests.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson (ObjectNode), SQLite (via JDBC), Maven multi-module

## Global Constraints

- Quarkus version: `3.32.2` (from `version.quarkus.platform` in root pom.xml)
- Cross-project version: `0.2-SNAPSHOT` (`version.io.casehub`)
- Parent: `casehub-engine-parent:0.4-SNAPSHOT`
- Package prefix for SPI: `io.casehub.platform.api.data` (matches eventual platform-api home)
- CDI convention: `@DefaultBean` for fallback, `@Alternative @Priority(N)` for production overrides
- All returned `ObjectNode` values are defensive copies
- System fields (`_id`, `_version`, `_created_at`, `_updated_at`) stripped on write, injected on read
- Test classes must be `*Test.java`, never `*IT.java`
- `TESTCONTAINERS_RYUK_DISABLED=true` for test runs
- Every commit references issue: `Refs #1332`

---

## Batch 1: DataStore SPI + In-Memory Implementation + Contract Tests

### Task 1: DataStore SPI Interfaces

**Files:**
- Create: `data-api/pom.xml`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/DataStore.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/DataCollection.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/DataContext.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/CollectionDescriptor.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/FieldDescriptor.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/FieldType.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/IndexDescriptor.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/IndexType.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/RelationDescriptor.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/RelationType.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/StorageHint.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/MergeStrategy.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/Query.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/Filter.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/FilterOp.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/SortField.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/SortDirection.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/QueryResult.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/DataStoreException.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/VersionConflictException.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/RecordNotFoundException.java`
- Modify: `pom.xml` (root — add `data-api` to modules list after line 107)

**Interfaces:**
- Produces: `DataStore`, `DataCollection`, `DataContext`, `CollectionDescriptor`, `FieldDescriptor`, `FieldType`, `IndexDescriptor`, `IndexType`, `RelationDescriptor`, `RelationType`, `StorageHint`, `MergeStrategy`, `Query`, `Filter`, `FilterOp`, `SortField`, `SortDirection`, `QueryResult`, `DataStoreException`, `VersionConflictException`, `RecordNotFoundException`

- [ ] **Step 1: Create `data-api/pom.xml`**

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

    <artifactId>casehub-data-api</artifactId>
    <name>CaseHub Data API</name>

    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add `data-api` to root pom.xml modules**

Add `<module>data-api</module>` after `persistence-spring-jpa` in root pom.xml modules list (around line 108).

- [ ] **Step 3: Create all SPI types**

Create the following files in `data-api/src/main/java/io/casehub/platform/api/data/`:

**DataStore.java:**
```java
package io.casehub.platform.api.data;

public interface DataStore {
    void register(CollectionDescriptor descriptor);
    DataCollection collection(String name);
}
```

**DataCollection.java:**
```java
package io.casehub.platform.api.data;

import com.fasterxml.jackson.databind.node.ObjectNode;
import java.util.List;
import java.util.Optional;
import java.util.OptionalInt;

public interface DataCollection {
    String insert(ObjectNode record, DataContext ctx);
    Optional<ObjectNode> get(String id, DataContext ctx);
    void update(String id, ObjectNode changes, OptionalInt expectedVersion,
                MergeStrategy mergeStrategy, DataContext ctx);
    void delete(String id, DataContext ctx);
    QueryResult query(Query query, DataContext ctx);
    long count(Query query, DataContext ctx);
    List<ObjectNode> search(String text, DataContext ctx);
}
```

**DataContext.java:**
```java
package io.casehub.platform.api.data;

public record DataContext(String tenantId, String appId) {}
```

**CollectionDescriptor.java:**
```java
package io.casehub.platform.api.data;

import java.util.List;

public record CollectionDescriptor(
    String name,
    List<FieldDescriptor> fields,
    List<IndexDescriptor> indexes,
    List<RelationDescriptor> relations,
    StorageHint storageHint
) {
    public static CollectionDescriptor schemaless(String name) {
        return new CollectionDescriptor(name, List.of(), List.of(),
                                        List.of(), StorageHint.DOCUMENT);
    }
}
```

**FieldDescriptor.java:**
```java
package io.casehub.platform.api.data;

public record FieldDescriptor(String name, FieldType type, boolean required, Object defaultValue) {}
```

**FieldType.java:**
```java
package io.casehub.platform.api.data;

public enum FieldType { STRING, INTEGER, LONG, DOUBLE, BOOLEAN, TIMESTAMP, JSON }
```

**IndexDescriptor.java:**
```java
package io.casehub.platform.api.data;

import java.util.List;

public record IndexDescriptor(String name, List<String> fields, IndexType type) {}
```

**IndexType.java:**
```java
package io.casehub.platform.api.data;

public enum IndexType { STANDARD, UNIQUE, FULLTEXT }
```

**RelationDescriptor.java:**
```java
package io.casehub.platform.api.data;

public record RelationDescriptor(String name, String field, String targetCollection,
                                  String targetField, RelationType type) {}
```

**RelationType.java:**
```java
package io.casehub.platform.api.data;

public enum RelationType { MANY_TO_ONE, ONE_TO_MANY }
```

**StorageHint.java:**
```java
package io.casehub.platform.api.data;

public enum StorageHint { DOCUMENT, TYPED, HYBRID }
```

**MergeStrategy.java:**
```java
package io.casehub.platform.api.data;

public enum MergeStrategy { REPLACE, SHALLOW, DEEP, RFC_7396 }
```

**Query.java:**
```java
package io.casehub.platform.api.data;

import java.util.List;

public record Query(List<Filter> filters, List<SortField> sort, int page, int size) {
    public static Query all() {
        return new Query(List.of(), List.of(), 0, 20);
    }
}
```

**Filter.java:**
```java
package io.casehub.platform.api.data;

public record Filter(String field, FilterOp op, Object value) {}
```

**FilterOp.java:**
```java
package io.casehub.platform.api.data;

public enum FilterOp { EQ, NEQ, GT, GTE, LT, LTE, IN, CONTAINS, EXISTS }
```

**SortField.java:**
```java
package io.casehub.platform.api.data;

public record SortField(String field, SortDirection direction) {}
```

**SortDirection.java:**
```java
package io.casehub.platform.api.data;

public enum SortDirection { ASC, DESC }
```

**QueryResult.java:**
```java
package io.casehub.platform.api.data;

import com.fasterxml.jackson.databind.node.ObjectNode;
import java.util.List;

public record QueryResult(List<ObjectNode> records, long totalCount, int page, int size) {}
```

**DataStoreException.java:**
```java
package io.casehub.platform.api.data;

public class DataStoreException extends RuntimeException {
    public DataStoreException(String message) { super(message); }
    public DataStoreException(String message, Throwable cause) { super(message, cause); }
}
```

**VersionConflictException.java:**
```java
package io.casehub.platform.api.data;

public class VersionConflictException extends DataStoreException {
    public VersionConflictException(String collection, String id, int expected, int actual) {
        super("Version conflict on %s/%s: expected %d, actual %d".formatted(collection, id, expected, actual));
    }
}
```

**RecordNotFoundException.java:**
```java
package io.casehub.platform.api.data;

public class RecordNotFoundException extends DataStoreException {
    public RecordNotFoundException(String collection, String id) {
        super("Record not found: %s/%s".formatted(collection, id));
    }
}
```

- [ ] **Step 4: Compile**

Run: `mvn install -DskipTests -q -pl data-api`
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add data-api/ pom.xml
git commit -m "feat(#1332): add DataStore platform SPI interfaces

Defines the DataStore SPI contract in io.casehub.platform.api.data:
DataStore, DataCollection, DataContext, CollectionDescriptor, Query,
and supporting types. Package will promote to casehub-platform-api
on cross-repo adoption.

Refs #1332"
```

---

### Task 2: In-Memory DataStore + Contract Tests

**Files:**
- Create: `data-api/src/main/java/io/casehub/platform/api/data/memory/InMemoryDataStore.java`
- Create: `data-api/src/main/java/io/casehub/platform/api/data/memory/InMemoryDataCollection.java`
- Create: `data-api/src/test/java/io/casehub/platform/api/data/DataStoreContractTest.java`
- Create: `data-api/src/test/java/io/casehub/platform/api/data/memory/InMemoryDataStoreContractTest.java`
- Modify: `data-api/pom.xml` (add test dependencies)

**Interfaces:**
- Consumes: `DataStore`, `DataCollection`, `DataContext`, `CollectionDescriptor`, `Query`, `Filter`, `FilterOp`, `QueryResult`, `MergeStrategy`, `VersionConflictException`, `RecordNotFoundException`
- Produces: `InMemoryDataStore` (implements `DataStore`), `InMemoryDataCollection` (implements `DataCollection`), `DataStoreContractTest` (abstract base for all implementations)

- [ ] **Step 1: Add test dependencies to data-api pom.xml**

Add to `data-api/pom.xml`:
```xml
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

Also add `maven-jar-plugin` config to produce test-jar:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <executions>
                <execution>
                    <goals><goal>test-jar</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

- [ ] **Step 2: Write the contract test base class**

Create `data-api/src/test/java/io/casehub/platform/api/data/DataStoreContractTest.java`:

```java
package io.casehub.platform.api.data;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.OptionalInt;

import static org.assertj.core.api.Assertions.*;

public abstract class DataStoreContractTest {

    protected static final ObjectMapper MAPPER = new ObjectMapper();
    protected abstract DataStore dataStore();
    protected abstract DataContext testContext();

    @BeforeEach
    void setUp() {
        // Subclasses may override for cleanup
    }

    protected ObjectNode record(String... kvPairs) {
        ObjectNode node = MAPPER.createObjectNode();
        for (int i = 0; i < kvPairs.length; i += 2) {
            node.put(kvPairs[i], kvPairs[i + 1]);
        }
        return node;
    }

    @Test
    void insert_and_get_roundTrip() {
        DataCollection col = dataStore().collection("test");
        ObjectNode rec = record("name", "Alice", "age", "30");
        String id = col.insert(rec, testContext());

        assertThat(id).isNotNull();
        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("name").asText()).isEqualTo("Alice");
        assertThat(result.get("age").asText()).isEqualTo("30");
        assertThat(result.get("_id").asText()).isEqualTo(id);
        assertThat(result.get("_version").asInt()).isEqualTo(1);
        assertThat(result.has("_created_at")).isTrue();
        assertThat(result.has("_updated_at")).isTrue();
    }

    @Test
    void get_nonExistent_returnsEmpty() {
        DataCollection col = dataStore().collection("test");
        assertThat(col.get("nonexistent", testContext())).isEmpty();
    }

    @Test
    void update_incrementsVersion() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice"), testContext());

        col.update(id, record("name", "Bob"), OptionalInt.empty(),
                   MergeStrategy.REPLACE, testContext());

        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("name").asText()).isEqualTo("Bob");
        assertThat(result.get("_version").asInt()).isEqualTo(2);
    }

    @Test
    void update_withExpectedVersion_throwsOnConflict() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice"), testContext());

        assertThatThrownBy(() ->
            col.update(id, record("name", "Bob"), OptionalInt.of(99),
                       MergeStrategy.REPLACE, testContext())
        ).isInstanceOf(VersionConflictException.class);
    }

    @Test
    void update_withCorrectExpectedVersion_succeeds() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice"), testContext());

        col.update(id, record("name", "Bob"), OptionalInt.of(1),
                   MergeStrategy.REPLACE, testContext());

        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("name").asText()).isEqualTo("Bob");
    }

    @Test
    void update_nonExistent_throws() {
        DataCollection col = dataStore().collection("test");
        assertThatThrownBy(() ->
            col.update("nonexistent", record("name", "Bob"), OptionalInt.empty(),
                       MergeStrategy.REPLACE, testContext())
        ).isInstanceOf(RecordNotFoundException.class);
    }

    @Test
    void delete_removesRecord() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice"), testContext());

        col.delete(id, testContext());
        assertThat(col.get(id, testContext())).isEmpty();
    }

    @Test
    void query_filtersByField() {
        DataCollection col = dataStore().collection("test");
        col.insert(record("status", "ACTIVE"), testContext());
        col.insert(record("status", "CLOSED"), testContext());
        col.insert(record("status", "ACTIVE"), testContext());

        Query q = new Query(
            List.of(new Filter("status", FilterOp.EQ, "ACTIVE")),
            List.of(), 0, 20
        );
        QueryResult result = col.query(q, testContext());
        assertThat(result.records()).hasSize(2);
        assertThat(result.totalCount()).isEqualTo(2);
    }

    @Test
    void query_pagination() {
        DataCollection col = dataStore().collection("test");
        for (int i = 0; i < 5; i++) {
            col.insert(record("idx", String.valueOf(i)), testContext());
        }

        Query q = new Query(List.of(), List.of(), 0, 2);
        QueryResult page1 = col.query(q, testContext());
        assertThat(page1.records()).hasSize(2);
        assertThat(page1.totalCount()).isEqualTo(5);

        Query q2 = new Query(List.of(), List.of(), 1, 2);
        QueryResult page2 = col.query(q2, testContext());
        assertThat(page2.records()).hasSize(2);
    }

    @Test
    void query_sorting() {
        DataCollection col = dataStore().collection("test");
        col.insert(record("name", "Charlie"), testContext());
        col.insert(record("name", "Alice"), testContext());
        col.insert(record("name", "Bob"), testContext());

        Query q = new Query(List.of(),
            List.of(new SortField("name", SortDirection.ASC)), 0, 20);
        QueryResult result = col.query(q, testContext());
        assertThat(result.records().get(0).get("name").asText()).isEqualTo("Alice");
        assertThat(result.records().get(1).get("name").asText()).isEqualTo("Bob");
        assertThat(result.records().get(2).get("name").asText()).isEqualTo("Charlie");
    }

    @Test
    void count_returnsTotal() {
        DataCollection col = dataStore().collection("test");
        col.insert(record("x", "1"), testContext());
        col.insert(record("x", "2"), testContext());
        col.insert(record("x", "3"), testContext());

        assertThat(col.count(Query.all(), testContext())).isEqualTo(3);
    }

    @Test
    void schemaless_collection_worksWithoutRegistration() {
        DataCollection col = dataStore().collection("unregistered");
        String id = col.insert(record("anything", "goes"), testContext());
        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("anything").asText()).isEqualTo("goes");
    }

    @Test
    void systemFields_injectedOnRead_strippedOnWrite() {
        DataCollection col = dataStore().collection("test");
        ObjectNode rec = record("name", "Alice");
        rec.put("_id", "should-be-ignored");
        rec.put("_version", 999);

        String id = col.insert(rec, testContext());
        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("_version").asInt()).isEqualTo(1);
        assertThat(result.get("_id").asText()).isEqualTo(id);
    }

    @Test
    void returnedNodes_areDefensiveCopies() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice"), testContext());

        ObjectNode result1 = col.get(id, testContext()).orElseThrow();
        result1.put("name", "MUTATED");

        ObjectNode result2 = col.get(id, testContext()).orElseThrow();
        assertThat(result2.get("name").asText()).isEqualTo("Alice");
    }

    @Test
    void tenant_isolation() {
        DataCollection col = dataStore().collection("test");
        DataContext tenant1 = new DataContext("tenant1", testContext().appId());
        DataContext tenant2 = new DataContext("tenant2", testContext().appId());

        col.insert(record("name", "Alice"), tenant1);
        col.insert(record("name", "Bob"), tenant2);

        assertThat(col.count(Query.all(), tenant1)).isEqualTo(1);
        assertThat(col.count(Query.all(), tenant2)).isEqualTo(1);
    }

    @Test
    void merge_shallow() {
        DataCollection col = dataStore().collection("test");
        String id = col.insert(record("name", "Alice", "age", "30"), testContext());

        col.update(id, record("name", "Bob"), OptionalInt.empty(),
                   MergeStrategy.SHALLOW, testContext());

        ObjectNode result = col.get(id, testContext()).orElseThrow();
        assertThat(result.get("name").asText()).isEqualTo("Bob");
        assertThat(result.get("age").asText()).isEqualTo("30");
    }
}
```

- [ ] **Step 3: Run test to verify it fails (no implementation)**

Run: `mvn test -pl data-api -q`
Expected: Compilation succeeds (abstract class), no concrete tests yet

- [ ] **Step 4: Implement InMemoryDataStore and InMemoryDataCollection**

Create `data-api/src/main/java/io/casehub/platform/api/data/memory/InMemoryDataStore.java`:

```java
package io.casehub.platform.api.data.memory;

import io.casehub.platform.api.data.*;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class InMemoryDataStore implements DataStore {

    private final Map<String, CollectionDescriptor> descriptors = new ConcurrentHashMap<>();
    private final Map<String, InMemoryDataCollection> collections = new ConcurrentHashMap<>();

    @Override
    public void register(CollectionDescriptor descriptor) {
        descriptors.put(descriptor.name(), descriptor);
    }

    @Override
    public DataCollection collection(String name) {
        return collections.computeIfAbsent(name, n -> new InMemoryDataCollection());
    }

    public void clear() {
        collections.values().forEach(InMemoryDataCollection::clearAll);
    }
}
```

Create `data-api/src/main/java/io/casehub/platform/api/data/memory/InMemoryDataCollection.java`:

```java
package io.casehub.platform.api.data.memory;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.platform.api.data.*;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class InMemoryDataCollection implements DataCollection {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    // Key: tenantId::appId::docId
    private final Map<String, ObjectNode> store = new ConcurrentHashMap<>();
    private final Map<String, AtomicInteger> versions = new ConcurrentHashMap<>();

    private String key(String id, DataContext ctx) {
        return ctx.tenantId() + "::" + ctx.appId() + "::" + id;
    }

    @Override
    public String insert(ObjectNode record, DataContext ctx) {
        ObjectNode clean = stripSystemFields(record);
        String id = UUID.randomUUID().toString();
        String key = key(id, ctx);
        String now = Instant.now().toString();

        clean.put("_id", id);
        clean.put("_version", 1);
        clean.put("_created_at", now);
        clean.put("_updated_at", now);

        store.put(key, clean);
        versions.put(key, new AtomicInteger(1));
        return id;
    }

    @Override
    public Optional<ObjectNode> get(String id, DataContext ctx) {
        ObjectNode node = store.get(key(id, ctx));
        return Optional.ofNullable(node).map(this::defensiveCopy);
    }

    @Override
    public void update(String id, ObjectNode changes, OptionalInt expectedVersion,
                       MergeStrategy mergeStrategy, DataContext ctx) {
        String key = key(id, ctx);
        ObjectNode existing = store.get(key);
        if (existing == null) {
            throw new RecordNotFoundException("collection", id);
        }

        AtomicInteger ver = versions.get(key);
        if (expectedVersion.isPresent() && ver.get() != expectedVersion.getAsInt()) {
            throw new VersionConflictException("collection", id,
                    expectedVersion.getAsInt(), ver.get());
        }

        ObjectNode merged = applyMerge(existing, stripSystemFields(changes), mergeStrategy);
        int newVersion = ver.incrementAndGet();
        merged.put("_id", id);
        merged.put("_version", newVersion);
        merged.put("_created_at", existing.get("_created_at").asText());
        merged.put("_updated_at", Instant.now().toString());

        store.put(key, merged);
    }

    @Override
    public void delete(String id, DataContext ctx) {
        String key = key(id, ctx);
        if (store.remove(key) == null) {
            throw new RecordNotFoundException("collection", id);
        }
        versions.remove(key);
    }

    @Override
    public QueryResult query(Query query, DataContext ctx) {
        List<ObjectNode> all = allForContext(ctx);
        Stream<ObjectNode> stream = all.stream();

        for (Filter filter : query.filters()) {
            stream = stream.filter(node -> matchesFilter(node, filter));
        }

        List<ObjectNode> filtered = stream.collect(Collectors.toList());

        if (!query.sort().isEmpty()) {
            filtered.sort((a, b) -> {
                for (SortField sf : query.sort()) {
                    String va = a.has(sf.field()) ? a.get(sf.field()).asText() : "";
                    String vb = b.has(sf.field()) ? b.get(sf.field()).asText() : "";
                    int cmp = va.compareTo(vb);
                    if (sf.direction() == SortDirection.DESC) cmp = -cmp;
                    if (cmp != 0) return cmp;
                }
                return 0;
            });
        }

        long total = filtered.size();
        int skip = query.page() * query.size();
        List<ObjectNode> page = filtered.stream()
                .skip(skip).limit(query.size())
                .map(this::defensiveCopy)
                .toList();

        return new QueryResult(page, total, query.page(), query.size());
    }

    @Override
    public long count(Query query, DataContext ctx) {
        Stream<ObjectNode> stream = allForContext(ctx).stream();
        for (Filter filter : query.filters()) {
            stream = stream.filter(node -> matchesFilter(node, filter));
        }
        return stream.count();
    }

    @Override
    public List<ObjectNode> search(String text, DataContext ctx) {
        String lower = text.toLowerCase();
        return allForContext(ctx).stream()
                .filter(node -> {
                    Iterator<JsonNode> it = node.elements();
                    while (it.hasNext()) {
                        JsonNode val = it.next();
                        if (val.isTextual() && val.asText().toLowerCase().contains(lower)) {
                            return true;
                        }
                    }
                    return false;
                })
                .map(this::defensiveCopy)
                .toList();
    }

    void clearAll() {
        store.clear();
        versions.clear();
    }

    private List<ObjectNode> allForContext(DataContext ctx) {
        String prefix = ctx.tenantId() + "::" + ctx.appId() + "::";
        return store.entrySet().stream()
                .filter(e -> e.getKey().startsWith(prefix))
                .map(Map.Entry::getValue)
                .toList();
    }

    private boolean matchesFilter(ObjectNode node, Filter filter) {
        JsonNode fieldValue = node.get(filter.field());
        if (filter.op() == FilterOp.EXISTS) {
            return fieldValue != null && !fieldValue.isNull();
        }
        if (fieldValue == null || fieldValue.isNull()) return false;

        String actual = fieldValue.asText();
        return switch (filter.op()) {
            case EQ -> actual.equals(filter.value().toString());
            case NEQ -> !actual.equals(filter.value().toString());
            case CONTAINS -> actual.contains(filter.value().toString());
            case IN -> {
                if (filter.value() instanceof Collection<?> c) {
                    yield c.stream().anyMatch(v -> actual.equals(v.toString()));
                }
                yield false;
            }
            default -> true;
        };
    }

    private ObjectNode stripSystemFields(ObjectNode node) {
        ObjectNode copy = node.deepCopy();
        copy.remove("_id");
        copy.remove("_version");
        copy.remove("_created_at");
        copy.remove("_updated_at");
        return copy;
    }

    private ObjectNode defensiveCopy(ObjectNode node) {
        return node.deepCopy();
    }

    private ObjectNode applyMerge(ObjectNode existing, ObjectNode changes,
                                   MergeStrategy strategy) {
        return switch (strategy) {
            case REPLACE -> changes.deepCopy();
            case SHALLOW -> {
                ObjectNode merged = existing.deepCopy();
                changes.fields().forEachRemaining(f -> merged.set(f.getKey(), f.getValue()));
                yield merged;
            }
            case DEEP, RFC_7396 -> {
                ObjectNode merged = existing.deepCopy();
                deepMerge(merged, changes);
                yield merged;
            }
        };
    }

    private void deepMerge(ObjectNode target, ObjectNode source) {
        source.fields().forEachRemaining(f -> {
            if (f.getValue().isNull()) {
                target.remove(f.getKey());
            } else if (f.getValue().isObject() && target.has(f.getKey())
                       && target.get(f.getKey()).isObject()) {
                deepMerge((ObjectNode) target.get(f.getKey()),
                          (ObjectNode) f.getValue());
            } else {
                target.set(f.getKey(), f.getValue());
            }
        });
    }
}
```

- [ ] **Step 5: Create the contract test runner**

Create `data-api/src/test/java/io/casehub/platform/api/data/memory/InMemoryDataStoreContractTest.java`:

```java
package io.casehub.platform.api.data.memory;

import io.casehub.platform.api.data.*;
import org.junit.jupiter.api.BeforeEach;

public class InMemoryDataStoreContractTest extends DataStoreContractTest {

    private InMemoryDataStore store;

    @BeforeEach
    void init() {
        store = new InMemoryDataStore();
        store.clear();
    }

    @Override
    protected DataStore dataStore() {
        return store;
    }

    @Override
    protected DataContext testContext() {
        return new DataContext("test-tenant", "test-app");
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl data-api -q`
Expected: All 14 tests PASS

- [ ] **Step 7: Commit**

```bash
git add data-api/
git commit -m "feat(#1332): add in-memory DataStore implementation and contract tests

InMemoryDataStore provides a ConcurrentHashMap-backed implementation
for testing. DataStoreContractTest defines the SPI contract as an
abstract test class — all implementations must pass it.

Refs #1332"
```

---

## Batch 2: SQLite Implementations

### Task 3: Document-Backed SQLite DataStore

**Files:**
- Create: `data-sqlite/pom.xml`
- Create: `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDocumentDataStore.java`
- Create: `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDocumentDataCollection.java`
- Create: `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDataStoreFactory.java`
- Create: `data-sqlite/src/main/resources/db/data-store/migration/V1__create_data_records.sql`
- Create: `data-sqlite/src/test/java/io/casehub/data/sqlite/SqliteDocumentDataStoreContractTest.java`
- Modify: `pom.xml` (root — add `data-sqlite` to modules list)

**Interfaces:**
- Consumes: `DataStore`, `DataCollection`, `DataContext`, `Query`, `Filter`, `FilterOp`, `QueryResult`, `MergeStrategy`, `VersionConflictException`, `RecordNotFoundException` (from `data-api`)
- Produces: `SqliteDocumentDataStore` (implements `DataStore`), `SqliteDataStoreFactory` (creates DataStore instances from JDBC DataSource)

- [ ] **Step 1: Create `data-sqlite/pom.xml`**

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

    <artifactId>casehub-data-sqlite</artifactId>
    <name>CaseHub Data SQLite</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-data-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
            <version>3.46.0.0</version>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-data-api</artifactId>
            <version>${project.version}</version>
            <type>test-jar</type>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Add `<module>data-sqlite</module>` to root pom.xml after `data-api`.

- [ ] **Step 2: Create the migration SQL**

Create `data-sqlite/src/main/resources/db/data-store/migration/V1__create_data_records.sql`:

```sql
CREATE TABLE IF NOT EXISTS data_records (
    tenant_id   TEXT NOT NULL,
    app_id      TEXT NOT NULL,
    collection  TEXT NOT NULL,
    doc_id      TEXT NOT NULL,
    data        TEXT NOT NULL,
    _version    INTEGER NOT NULL DEFAULT 1,
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL,
    PRIMARY KEY (tenant_id, app_id, collection, doc_id)
);

CREATE INDEX IF NOT EXISTS idx_data_records_collection
    ON data_records(tenant_id, app_id, collection);
```

- [ ] **Step 3: Implement SqliteDataStoreFactory**

Create `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDataStoreFactory.java`:

```java
package io.casehub.data.sqlite;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;

public class SqliteDataStoreFactory {

    public static SqliteDocumentDataStore createInMemory() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:sqlite::memory:");
        config.setMaximumPoolSize(1);
        HikariDataSource ds = new HikariDataSource(config);
        migrate(ds);
        return new SqliteDocumentDataStore(ds);
    }

    public static SqliteDocumentDataStore create(String dbPath) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:sqlite:" + dbPath);
        config.setMaximumPoolSize(1);
        config.addDataSourceProperty("journal_mode", "WAL");
        config.addDataSourceProperty("busy_timeout", "5000");
        HikariDataSource ds = new HikariDataSource(config);
        migrate(ds);
        return new SqliteDocumentDataStore(ds);
    }

    private static void migrate(DataSource ds) {
        try (Connection conn = ds.getConnection();
             Statement stmt = conn.createStatement()) {
            String sql = new String(
                SqliteDataStoreFactory.class.getResourceAsStream(
                    "/db/data-store/migration/V1__create_data_records.sql"
                ).readAllBytes(), StandardCharsets.UTF_8);
            stmt.executeUpdate(sql);
        } catch (SQLException | IOException e) {
            throw new RuntimeException("Failed to migrate data store", e);
        }
    }
}
```

- [ ] **Step 4: Implement SqliteDocumentDataStore and SqliteDocumentDataCollection**

Create `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDocumentDataStore.java`:

```java
package io.casehub.data.sqlite;

import io.casehub.platform.api.data.*;

import javax.sql.DataSource;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class SqliteDocumentDataStore implements DataStore {

    private final DataSource dataSource;
    private final Map<String, CollectionDescriptor> descriptors = new ConcurrentHashMap<>();

    public SqliteDocumentDataStore(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void register(CollectionDescriptor descriptor) {
        descriptors.put(descriptor.name(), descriptor);
    }

    @Override
    public DataCollection collection(String name) {
        return new SqliteDocumentDataCollection(dataSource, name);
    }
}
```

Create `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteDocumentDataCollection.java`:

```java
package io.casehub.data.sqlite;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.platform.api.data.*;

import javax.sql.DataSource;
import java.sql.*;
import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class SqliteDocumentDataCollection implements DataCollection {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private final DataSource dataSource;
    private final String collectionName;

    SqliteDocumentDataCollection(DataSource dataSource, String collectionName) {
        this.dataSource = dataSource;
        this.collectionName = collectionName;
    }

    @Override
    public String insert(ObjectNode record, DataContext ctx) {
        ObjectNode clean = stripSystemFields(record);
        String id = UUID.randomUUID().toString();
        String now = Instant.now().toString();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "INSERT INTO data_records (tenant_id, app_id, collection, doc_id, data, _version, created_at, updated_at) VALUES (?, ?, ?, ?, ?, 1, ?, ?)")) {
            ps.setString(1, ctx.tenantId());
            ps.setString(2, ctx.appId());
            ps.setString(3, collectionName);
            ps.setString(4, id);
            ps.setString(5, MAPPER.writeValueAsString(clean));
            ps.setString(6, now);
            ps.setString(7, now);
            ps.executeUpdate();
        } catch (SQLException | JsonProcessingException e) {
            throw new DataStoreException("Insert failed", e);
        }
        return id;
    }

    @Override
    public Optional<ObjectNode> get(String id, DataContext ctx) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT data, doc_id, _version, created_at, updated_at FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ? AND doc_id = ?")) {
            ps.setString(1, ctx.tenantId());
            ps.setString(2, ctx.appId());
            ps.setString(3, collectionName);
            ps.setString(4, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(toObjectNode(rs));
                }
            }
        } catch (SQLException | JsonProcessingException e) {
            throw new DataStoreException("Get failed", e);
        }
        return Optional.empty();
    }

    @Override
    public void update(String id, ObjectNode changes, OptionalInt expectedVersion,
                       MergeStrategy mergeStrategy, DataContext ctx) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try {
                ObjectNode existing = readForUpdate(conn, id, ctx);
                if (existing == null) {
                    throw new RecordNotFoundException(collectionName, id);
                }

                int currentVersion = existing.get("_version").asInt();
                if (expectedVersion.isPresent() && currentVersion != expectedVersion.getAsInt()) {
                    throw new VersionConflictException(collectionName, id,
                            expectedVersion.getAsInt(), currentVersion);
                }

                ObjectNode merged = applyMerge(existing, stripSystemFields(changes), mergeStrategy);
                int newVersion = currentVersion + 1;

                try (PreparedStatement ps = conn.prepareStatement(
                        "UPDATE data_records SET data = ?, _version = ?, updated_at = ? WHERE tenant_id = ? AND app_id = ? AND collection = ? AND doc_id = ?")) {
                    ps.setString(1, MAPPER.writeValueAsString(stripSystemFields(merged)));
                    ps.setInt(2, newVersion);
                    ps.setString(3, Instant.now().toString());
                    ps.setString(4, ctx.tenantId());
                    ps.setString(5, ctx.appId());
                    ps.setString(6, collectionName);
                    ps.setString(7, id);
                    ps.executeUpdate();
                }
                conn.commit();
            } catch (Exception e) {
                conn.rollback();
                throw e;
            }
        } catch (DataStoreException e) {
            throw e;
        } catch (Exception e) {
            throw new DataStoreException("Update failed", e);
        }
    }

    @Override
    public void delete(String id, DataContext ctx) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "DELETE FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ? AND doc_id = ?")) {
            ps.setString(1, ctx.tenantId());
            ps.setString(2, ctx.appId());
            ps.setString(3, collectionName);
            ps.setString(4, id);
            int rows = ps.executeUpdate();
            if (rows == 0) {
                throw new RecordNotFoundException(collectionName, id);
            }
        } catch (DataStoreException e) {
            throw e;
        } catch (SQLException e) {
            throw new DataStoreException("Delete failed", e);
        }
    }

    @Override
    public QueryResult query(Query query, DataContext ctx) {
        List<ObjectNode> all = listAll(ctx);
        Stream<ObjectNode> stream = all.stream();

        for (Filter filter : query.filters()) {
            stream = stream.filter(node -> matchesFilter(node, filter));
        }

        List<ObjectNode> filtered = stream.collect(Collectors.toList());

        if (!query.sort().isEmpty()) {
            filtered.sort((a, b) -> {
                for (SortField sf : query.sort()) {
                    String va = a.has(sf.field()) ? a.get(sf.field()).asText() : "";
                    String vb = b.has(sf.field()) ? b.get(sf.field()).asText() : "";
                    int cmp = va.compareTo(vb);
                    if (sf.direction() == SortDirection.DESC) cmp = -cmp;
                    if (cmp != 0) return cmp;
                }
                return 0;
            });
        }

        long total = filtered.size();
        int skip = query.page() * query.size();
        List<ObjectNode> page = filtered.stream()
                .skip(skip).limit(query.size()).toList();

        return new QueryResult(page, total, query.page(), query.size());
    }

    @Override
    public long count(Query query, DataContext ctx) {
        if (query.filters().isEmpty()) {
            try (Connection conn = dataSource.getConnection();
                 PreparedStatement ps = conn.prepareStatement(
                     "SELECT COUNT(*) FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ?")) {
                ps.setString(1, ctx.tenantId());
                ps.setString(2, ctx.appId());
                ps.setString(3, collectionName);
                try (ResultSet rs = ps.executeQuery()) {
                    rs.next();
                    return rs.getLong(1);
                }
            } catch (SQLException e) {
                throw new DataStoreException("Count failed", e);
            }
        }
        return query(query, ctx).totalCount();
    }

    @Override
    public List<ObjectNode> search(String text, DataContext ctx) {
        String lower = text.toLowerCase();
        return listAll(ctx).stream()
                .filter(node -> {
                    Iterator<JsonNode> it = node.elements();
                    while (it.hasNext()) {
                        JsonNode val = it.next();
                        if (val.isTextual() && val.asText().toLowerCase().contains(lower)) {
                            return true;
                        }
                    }
                    return false;
                })
                .toList();
    }

    private List<ObjectNode> listAll(DataContext ctx) {
        List<ObjectNode> results = new ArrayList<>();
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT data, doc_id, _version, created_at, updated_at FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ? ORDER BY created_at")) {
            ps.setString(1, ctx.tenantId());
            ps.setString(2, ctx.appId());
            ps.setString(3, collectionName);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    results.add(toObjectNode(rs));
                }
            }
        } catch (SQLException | JsonProcessingException e) {
            throw new DataStoreException("List failed", e);
        }
        return results;
    }

    private ObjectNode readForUpdate(Connection conn, String id, DataContext ctx)
            throws SQLException, JsonProcessingException {
        try (PreparedStatement ps = conn.prepareStatement(
                "SELECT data, doc_id, _version, created_at, updated_at FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ? AND doc_id = ?")) {
            ps.setString(1, ctx.tenantId());
            ps.setString(2, ctx.appId());
            ps.setString(3, collectionName);
            ps.setString(4, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) return toObjectNode(rs);
            }
        }
        return null;
    }

    private ObjectNode toObjectNode(ResultSet rs) throws SQLException, JsonProcessingException {
        ObjectNode node = (ObjectNode) MAPPER.readTree(rs.getString("data"));
        node.put("_id", rs.getString("doc_id"));
        node.put("_version", rs.getInt("_version"));
        node.put("_created_at", rs.getString("created_at"));
        node.put("_updated_at", rs.getString("updated_at"));
        return node;
    }

    private ObjectNode stripSystemFields(ObjectNode node) {
        ObjectNode copy = node.deepCopy();
        copy.remove("_id");
        copy.remove("_version");
        copy.remove("_created_at");
        copy.remove("_updated_at");
        return copy;
    }

    private boolean matchesFilter(ObjectNode node, Filter filter) {
        JsonNode fieldValue = node.get(filter.field());
        if (filter.op() == FilterOp.EXISTS) {
            return fieldValue != null && !fieldValue.isNull();
        }
        if (fieldValue == null || fieldValue.isNull()) return false;
        String actual = fieldValue.asText();
        return switch (filter.op()) {
            case EQ -> actual.equals(filter.value().toString());
            case NEQ -> !actual.equals(filter.value().toString());
            case CONTAINS -> actual.contains(filter.value().toString());
            default -> true;
        };
    }

    private ObjectNode applyMerge(ObjectNode existing, ObjectNode changes, MergeStrategy strategy) {
        ObjectNode clean = stripSystemFields(existing);
        return switch (strategy) {
            case REPLACE -> changes.deepCopy();
            case SHALLOW -> {
                ObjectNode merged = clean.deepCopy();
                changes.fields().forEachRemaining(f -> merged.set(f.getKey(), f.getValue()));
                yield merged;
            }
            case DEEP, RFC_7396 -> {
                ObjectNode merged = clean.deepCopy();
                deepMerge(merged, changes);
                yield merged;
            }
        };
    }

    private void deepMerge(ObjectNode target, ObjectNode source) {
        source.fields().forEachRemaining(f -> {
            if (f.getValue().isNull()) {
                target.remove(f.getKey());
            } else if (f.getValue().isObject() && target.has(f.getKey())
                       && target.get(f.getKey()).isObject()) {
                deepMerge((ObjectNode) target.get(f.getKey()), (ObjectNode) f.getValue());
            } else {
                target.set(f.getKey(), f.getValue());
            }
        });
    }
}
```

- [ ] **Step 5: Create contract test runner**

Create `data-sqlite/src/test/java/io/casehub/data/sqlite/SqliteDocumentDataStoreContractTest.java`:

```java
package io.casehub.data.sqlite;

import io.casehub.platform.api.data.*;
import org.junit.jupiter.api.BeforeEach;

public class SqliteDocumentDataStoreContractTest extends DataStoreContractTest {

    private SqliteDocumentDataStore store;

    @BeforeEach
    void init() {
        store = SqliteDataStoreFactory.createInMemory();
    }

    @Override
    protected DataStore dataStore() {
        return store;
    }

    @Override
    protected DataContext testContext() {
        return new DataContext("test-tenant", "test-app");
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn test -pl data-sqlite -q`
Expected: All 14 contract tests PASS

- [ ] **Step 7: Commit**

```bash
git add data-sqlite/ pom.xml
git commit -m "feat(#1332): add document-backed SQLite DataStore implementation

Single data_records table stores all records as JSON blobs with
(tenant_id, app_id, collection, doc_id) composite key. Supports
optimistic concurrency via _version column, merge strategies,
and tenant isolation. Passes full DataStoreContractTest suite.

Refs #1332"
```

---

### Task 4: Table-Backed SQLite DataStore

**Files:**
- Create: `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteTypedDataStore.java`
- Create: `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteTypedDataCollection.java`
- Create: `data-sqlite/src/test/java/io/casehub/data/sqlite/SqliteTypedDataStoreContractTest.java`

**Interfaces:**
- Consumes: `DataStore`, `DataCollection`, `CollectionDescriptor`, `StorageHint`, `FieldDescriptor`, `FieldType` (from `data-api`), `SqliteDataStoreFactory` (from Task 3)
- Produces: `SqliteTypedDataStore` (implements `DataStore` with typed column extraction)

- [ ] **Step 1: Write a typed-columns-specific test**

Create `data-sqlite/src/test/java/io/casehub/data/sqlite/SqliteTypedDataStoreContractTest.java`:

```java
package io.casehub.data.sqlite;

import io.casehub.platform.api.data.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

public class SqliteTypedDataStoreContractTest extends DataStoreContractTest {

    private SqliteTypedDataStore store;

    @BeforeEach
    void init() {
        store = SqliteDataStoreFactory.createTypedInMemory();
    }

    @Override
    protected DataStore dataStore() {
        return store;
    }

    @Override
    protected DataContext testContext() {
        return new DataContext("test-tenant", "test-app");
    }

    @Test
    void registered_collection_usesIndexedColumns() {
        store.register(new CollectionDescriptor(
            "typed_col",
            List.of(
                new FieldDescriptor("status", FieldType.STRING, true, null),
                new FieldDescriptor("priority", FieldType.INTEGER, false, null)
            ),
            List.of(new IndexDescriptor("idx_status", List.of("status"), IndexType.STANDARD)),
            List.of(),
            StorageHint.TYPED
        ));

        DataCollection col = store.collection("typed_col");
        col.insert(record("status", "OPEN", "priority", "1"), testContext());
        col.insert(record("status", "CLOSED", "priority", "2"), testContext());

        Query q = new Query(
            List.of(new Filter("status", FilterOp.EQ, "OPEN")),
            List.of(), 0, 20
        );
        QueryResult result = col.query(q, testContext());
        assertThat(result.records()).hasSize(1);
        assertThat(result.records().get(0).get("status").asText()).isEqualTo("OPEN");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl data-sqlite -Dtest=SqliteTypedDataStoreContractTest -q`
Expected: FAIL — `SqliteTypedDataStore` class not found

- [ ] **Step 3: Implement SqliteTypedDataStore**

The table-backed implementation extends the document-backed approach: it stores the full JSON blob AND extracts declared fields into indexed columns. Queries on declared fields use `json_extract()` from the data column (the indexed columns are for future optimization — the core behavior is correct with json_extract).

Create `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteTypedDataStore.java`:

```java
package io.casehub.data.sqlite;

import io.casehub.platform.api.data.*;

import javax.sql.DataSource;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class SqliteTypedDataStore implements DataStore {

    private final DataSource dataSource;
    private final Map<String, CollectionDescriptor> descriptors = new ConcurrentHashMap<>();

    public SqliteTypedDataStore(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void register(CollectionDescriptor descriptor) {
        descriptors.put(descriptor.name(), descriptor);
    }

    @Override
    public DataCollection collection(String name) {
        CollectionDescriptor desc = descriptors.get(name);
        return new SqliteTypedDataCollection(dataSource, name, desc);
    }
}
```

Create `data-sqlite/src/main/java/io/casehub/data/sqlite/SqliteTypedDataCollection.java`:

```java
package io.casehub.data.sqlite;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.platform.api.data.*;

import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class SqliteTypedDataCollection extends SqliteDocumentDataCollection {

    private final CollectionDescriptor descriptor;

    SqliteTypedDataCollection(DataSource dataSource, String collectionName,
                               CollectionDescriptor descriptor) {
        super(dataSource, collectionName);
        this.descriptor = descriptor;
    }

    @Override
    public QueryResult query(Query query, DataContext ctx) {
        if (descriptor == null || descriptor.fields().isEmpty()) {
            return super.query(query, ctx);
        }
        return queryWithJsonExtract(query, ctx);
    }

    @Override
    public long count(Query query, DataContext ctx) {
        if (descriptor == null || query.filters().isEmpty()) {
            return super.count(query, ctx);
        }
        return countWithJsonExtract(query, ctx);
    }

    private QueryResult queryWithJsonExtract(Query query, DataContext ctx) {
        StringBuilder sql = new StringBuilder(
            "SELECT data, doc_id, _version, created_at, updated_at FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ?");
        List<Object> params = new ArrayList<>();
        params.add(ctx.tenantId());
        params.add(ctx.appId());
        params.add(getCollectionName());

        for (Filter f : query.filters()) {
            sql.append(" AND json_extract(data, '$.").append(f.field()).append("')");
            switch (f.op()) {
                case EQ -> { sql.append(" = ?"); params.add(f.value().toString()); }
                case NEQ -> { sql.append(" != ?"); params.add(f.value().toString()); }
                case GT -> { sql.append(" > ?"); params.add(f.value().toString()); }
                case GTE -> { sql.append(" >= ?"); params.add(f.value().toString()); }
                case LT -> { sql.append(" < ?"); params.add(f.value().toString()); }
                case LTE -> { sql.append(" <= ?"); params.add(f.value().toString()); }
                case CONTAINS -> { sql.append(" LIKE ?"); params.add("%" + f.value() + "%"); }
                default -> {}
            }
        }

        long total = countWithSql(sql.toString().replace(
            "SELECT data, doc_id, _version, created_at, updated_at",
            "SELECT COUNT(*)"), params);

        if (!query.sort().isEmpty()) {
            sql.append(" ORDER BY ");
            sql.append(query.sort().stream()
                .map(sf -> "json_extract(data, '$." + sf.field() + "') " + sf.direction().name())
                .collect(Collectors.joining(", ")));
        }

        sql.append(" LIMIT ? OFFSET ?");
        params.add(query.size());
        params.add(query.page() * query.size());

        List<ObjectNode> records = executeQuery(sql.toString(), params);
        return new QueryResult(records, total, query.page(), query.size());
    }

    private long countWithJsonExtract(Query query, DataContext ctx) {
        StringBuilder sql = new StringBuilder(
            "SELECT COUNT(*) FROM data_records WHERE tenant_id = ? AND app_id = ? AND collection = ?");
        List<Object> params = new ArrayList<>();
        params.add(ctx.tenantId());
        params.add(ctx.appId());
        params.add(getCollectionName());

        for (Filter f : query.filters()) {
            sql.append(" AND json_extract(data, '$.").append(f.field()).append("')");
            switch (f.op()) {
                case EQ -> { sql.append(" = ?"); params.add(f.value().toString()); }
                case NEQ -> { sql.append(" != ?"); params.add(f.value().toString()); }
                default -> {}
            }
        }

        return countWithSql(sql.toString(), params);
    }

    private long countWithSql(String sql, List<Object> params) {
        try (Connection conn = getDataSource().getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.size(); i++) {
                ps.setObject(i + 1, params.get(i));
            }
            try (ResultSet rs = ps.executeQuery()) {
                rs.next();
                return rs.getLong(1);
            }
        } catch (SQLException e) {
            throw new DataStoreException("Count query failed", e);
        }
    }

    private List<ObjectNode> executeQuery(String sql, List<Object> params) {
        List<ObjectNode> results = new ArrayList<>();
        try (Connection conn = getDataSource().getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < params.size(); i++) {
                ps.setObject(i + 1, params.get(i));
            }
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    results.add(toObjectNode(rs));
                }
            }
        } catch (Exception e) {
            throw new DataStoreException("Query failed", e);
        }
        return results;
    }
}
```

This requires making `SqliteDocumentDataCollection` extensible: change `private` fields/methods to `protected` (`dataSource`, `collectionName`, `toObjectNode`, `getDataSource()`, `getCollectionName()`).

- [ ] **Step 4: Add `createTypedInMemory()` to SqliteDataStoreFactory**

```java
public static SqliteTypedDataStore createTypedInMemory() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:sqlite::memory:");
    config.setMaximumPoolSize(1);
    HikariDataSource ds = new HikariDataSource(config);
    migrate(ds);
    return new SqliteTypedDataStore(ds);
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl data-sqlite -q`
Expected: All tests PASS (both document and typed contract tests + typed-specific test)

- [ ] **Step 6: Commit**

```bash
git add data-sqlite/
git commit -m "feat(#1332): add table-backed SQLite DataStore with json_extract queries

SqliteTypedDataStore pushes filter and sort operations down to SQLite
using json_extract() on declared fields. Falls back to in-Java filtering
for unregistered collections. Passes full DataStoreContractTest suite.

Refs #1332"
```

---

## Batch 3: CaseInstanceRepository Adapter

### Task 5: CaseInstanceMapper

**Files:**
- Create: `persistence-datastore/pom.xml`
- Create: `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/CaseInstanceMapper.java`
- Create: `persistence-datastore/src/test/java/io/casehub/engine/persistence/datastore/CaseInstanceMapperTest.java`
- Modify: `pom.xml` (root — add `persistence-datastore` to modules list)

**Interfaces:**
- Consumes: `CaseInstance` (`common-core/src/main/java/.../model/CaseInstance.java`), `CaseMetaModel` (`common-core/src/main/java/.../model/CaseMetaModel.java`), `CaseStatus` (`api/src/main/java/.../model/CaseStatus.java`), `CaseContext` (`api/src/main/java/.../context/CaseContext.java`)
- Produces: `CaseInstanceMapper.toRecord(CaseInstance): ObjectNode`, `CaseInstanceMapper.fromRecord(ObjectNode): CaseInstance`

- [ ] **Step 1: Create `persistence-datastore/pom.xml`**

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

    <artifactId>casehub-engine-persistence-datastore</artifactId>
    <name>CaseHub Engine Persistence DataStore</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-common</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-support-core</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-data-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-common-core</artifactId>
            <type>test-jar</type>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-data-api</artifactId>
            <version>${project.version}</version>
            <type>test-jar</type>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

Add `<module>persistence-datastore</module>` to root pom.xml after `persistence-spring-jpa`.

- [ ] **Step 2: Write the mapper test**

Create `persistence-datastore/src/test/java/io/casehub/engine/persistence/datastore/CaseInstanceMapperTest.java`:

```java
package io.casehub.engine.persistence.datastore;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Set;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class CaseInstanceMapperTest {

    @Test
    void roundTrip_preservesAllFields() {
        CaseInstance original = createTestInstance();

        ObjectNode record = CaseInstanceMapper.toRecord(original);
        CaseInstance restored = CaseInstanceMapper.fromRecord(record);

        assertThat(restored.getUuid()).isEqualTo(original.getUuid());
        assertThat(restored.getState()).isEqualTo(original.getState());
        assertThat(restored.getActorId()).isEqualTo(original.getActorId());
        assertThat(restored.getCreatedAt()).isEqualTo(original.getCreatedAt());
        assertThat(restored.getCaseMetaModel().getNamespace())
            .isEqualTo(original.getCaseMetaModel().getNamespace());
        assertThat(restored.getCaseMetaModel().getName())
            .isEqualTo(original.getCaseMetaModel().getName());
        assertThat(restored.getCaseMetaModel().getVersion())
            .isEqualTo(original.getCaseMetaModel().getVersion());
        assertThat(restored.getLabels()).isEqualTo(original.getLabels());
        assertThat(restored.getTypes()).isEqualTo(original.getTypes());
    }

    @Test
    void roundTrip_nullActorId() {
        CaseInstance original = createTestInstance();
        original.setActorId(null);

        ObjectNode record = CaseInstanceMapper.toRecord(original);
        CaseInstance restored = CaseInstanceMapper.fromRecord(record);

        assertThat(restored.getActorId()).isNull();
    }

    @Test
    void roundTrip_emptyLabelsAndTypes() {
        CaseInstance original = createTestInstance();
        original.getLabels().clear();
        original.getTypes().clear();

        ObjectNode record = CaseInstanceMapper.toRecord(original);
        CaseInstance restored = CaseInstanceMapper.fromRecord(record);

        assertThat(restored.getLabels()).isEmpty();
        assertThat(restored.getTypes()).isEmpty();
    }

    @Test
    void toRecord_includesQueryableFields() {
        CaseInstance instance = createTestInstance();
        ObjectNode record = CaseInstanceMapper.toRecord(instance);

        assertThat(record.get("uuid").asText()).isEqualTo(instance.getUuid().toString());
        assertThat(record.get("state").asText()).isEqualTo("RUNNING");
        assertThat(record.get("namespace").asText()).isEqualTo("test-ns");
        assertThat(record.get("name").asText()).isEqualTo("test-case");
    }

    private CaseInstance createTestInstance() {
        CaseMetaModel meta = new CaseMetaModel();
        meta.setNamespace("test-ns");
        meta.setName("test-case");
        meta.setVersion("1.0");

        CaseInstance instance = new CaseInstance();
        instance.setCaseMetaModel(meta);
        instance.setUuid(UUID.randomUUID());
        instance.setState(CaseStatus.RUNNING);
        instance.setActorId("actor-1");
        instance.setCreatedAt(Instant.now());
        instance.getLabels().addAll(Set.of("urgent", "customer"));
        instance.getTypes().addAll(Set.of("helpdesk", "ticket"));
        return instance;
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -pl persistence-datastore -Dtest=CaseInstanceMapperTest -q`
Expected: FAIL — `CaseInstanceMapper` class not found

- [ ] **Step 4: Implement CaseInstanceMapper**

Create `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/CaseInstanceMapper.java`:

```java
package io.casehub.engine.persistence.datastore;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;

import java.time.Instant;
import java.util.*;

public class CaseInstanceMapper {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static ObjectNode toRecord(CaseInstance instance) {
        ObjectNode node = MAPPER.createObjectNode();

        node.put("uuid", instance.getUuid().toString());
        node.put("version", instance.getVersion());
        node.put("state", instance.getState().name());
        if (instance.getActorId() != null) {
            node.put("actorId", instance.getActorId());
        }
        if (instance.getCreatedAt() != null) {
            node.put("createdAt", instance.getCreatedAt().toString());
        }

        // CaseMetaModel — flatten to top-level queryable fields
        CaseMetaModel meta = instance.getCaseMetaModel();
        if (meta != null) {
            node.put("namespace", meta.getNamespace());
            node.put("name", meta.getName());
            node.put("metaVersion", meta.getVersion());
            if (meta.getTitle() != null) {
                node.put("metaTitle", meta.getTitle());
            }
        }

        // Context snapshot
        if (instance.getContextSnapshot() != null) {
            node.set("contextSnapshot", instance.getContextSnapshot());
        }

        // Collections
        ArrayNode labelsNode = MAPPER.createArrayNode();
        instance.getLabels().forEach(labelsNode::add);
        node.set("labels", labelsNode);

        ArrayNode typesNode = MAPPER.createArrayNode();
        instance.getTypes().forEach(typesNode::add);
        node.set("types", typesNode);

        // Exchange headers as JSON object
        if (!instance.getExchangeHeaders().isEmpty()) {
            node.set("exchangeHeaders",
                MAPPER.valueToTree(instance.getExchangeHeaders()));
        }

        // Parent references
        if (instance.getParentCaseId() != null) {
            node.put("parentCaseId", instance.getParentCaseId().toString());
        }
        if (instance.getParentPlanItemId() != null) {
            node.put("parentPlanItemId", instance.getParentPlanItemId().toString());
        }
        if (instance.getWaitingForWorkId() != null) {
            node.put("waitingForWorkId", instance.getWaitingForWorkId());
        }

        return node;
    }

    public static CaseInstance fromRecord(ObjectNode node) {
        CaseInstance instance = new CaseInstance();

        instance.setUuid(UUID.fromString(node.get("uuid").asText()));
        instance.setVersion(node.has("version") ? node.get("version").asLong() : 0);
        instance.setState(CaseStatus.valueOf(node.get("state").asText()));

        if (node.has("actorId") && !node.get("actorId").isNull()) {
            instance.setActorId(node.get("actorId").asText());
        }
        if (node.has("createdAt") && !node.get("createdAt").isNull()) {
            instance.setCreatedAt(Instant.parse(node.get("createdAt").asText()));
        }

        // CaseMetaModel
        CaseMetaModel meta = new CaseMetaModel();
        if (node.has("namespace")) meta.setNamespace(node.get("namespace").asText());
        if (node.has("name")) meta.setName(node.get("name").asText());
        if (node.has("metaVersion")) meta.setVersion(node.get("metaVersion").asText());
        if (node.has("metaTitle") && !node.get("metaTitle").isNull()) {
            meta.setTitle(node.get("metaTitle").asText());
        }
        instance.setCaseMetaModel(meta);

        // Context snapshot
        if (node.has("contextSnapshot")) {
            instance.setContextSnapshot(node.get("contextSnapshot"));
        }

        // Collections
        if (node.has("labels")) {
            node.get("labels").forEach(n -> instance.getLabels().add(n.asText()));
        }
        if (node.has("types")) {
            node.get("types").forEach(n -> instance.getTypes().add(n.asText()));
        }

        // Exchange headers
        if (node.has("exchangeHeaders")) {
            ObjectNode headers = (ObjectNode) node.get("exchangeHeaders");
            headers.fields().forEachRemaining(f ->
                instance.getExchangeHeaders().put(f.getKey(),
                    f.getValue().isTextual() ? f.getValue().asText() : f.getValue()));
        }

        // Parent references
        if (node.has("parentCaseId") && !node.get("parentCaseId").isNull()) {
            instance.setParentCaseId(UUID.fromString(node.get("parentCaseId").asText()));
        }
        if (node.has("parentPlanItemId") && !node.get("parentPlanItemId").isNull()) {
            instance.setParentPlanItemId(UUID.fromString(node.get("parentPlanItemId").asText()));
        }
        if (node.has("waitingForWorkId") && !node.get("waitingForWorkId").isNull()) {
            instance.setWaitingForWorkId(node.get("waitingForWorkId").asText());
        }

        return instance;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn test -pl persistence-datastore -Dtest=CaseInstanceMapperTest -q`
Expected: All 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git add persistence-datastore/ pom.xml
git commit -m "feat(#1332): add CaseInstanceMapper for domain ↔ record conversion

Bidirectional mapping between CaseInstance and ObjectNode. Flattens
CaseMetaModel fields (namespace, name, version) for queryability.
Handles nullable fields, collections, and parent references.

Refs #1332"
```

---

### Task 6: DataStoreCaseInstanceRepository + CDI Wiring

**Files:**
- Create: `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/DataStoreCaseInstanceRepository.java`
- Create: `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/CaseInstanceSchema.java`
- Create: `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/quarkus/PersistenceDataStoreBeans.java`
- Create: `persistence-datastore/src/test/java/io/casehub/engine/persistence/datastore/DataStoreCaseInstanceRepositoryContractTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository` (`common-core/.../spi/CaseInstanceRepository.java`), `DataStore`, `DataCollection`, `DataContext`, `Query`, `Filter`, `FilterOp`, `QueryResult`, `CollectionDescriptor` (from `data-api`), `CaseInstanceMapper` (from Task 5)
- Produces: `DataStoreCaseInstanceRepository` (implements `CaseInstanceRepository`), `PersistenceDataStoreBeans` (CDI producer)

- [ ] **Step 1: Write the contract test runner**

Create `persistence-datastore/src/test/java/io/casehub/engine/persistence/datastore/DataStoreCaseInstanceRepositoryContractTest.java`:

```java
package io.casehub.engine.persistence.datastore;

import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CaseInstanceRepositoryContractTest;
import io.casehub.platform.api.data.memory.InMemoryDataStore;
import org.junit.jupiter.api.BeforeEach;

class DataStoreCaseInstanceRepositoryContractTest extends CaseInstanceRepositoryContractTest {

    private DataStoreCaseInstanceRepository repo;

    @BeforeEach
    void init() {
        InMemoryDataStore store = new InMemoryDataStore();
        store.clear();
        repo = new DataStoreCaseInstanceRepository(store);
    }

    @Override
    protected CaseInstanceRepository repository() {
        return repo;
    }

    @Override
    protected String tenancyId() {
        return "test-tenant";
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl persistence-datastore -Dtest=DataStoreCaseInstanceRepositoryContractTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement CaseInstanceSchema**

Create `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/CaseInstanceSchema.java`:

```java
package io.casehub.engine.persistence.datastore;

import io.casehub.platform.api.data.*;

import java.util.List;

public class CaseInstanceSchema {

    public static final String COLLECTION = "case_instances";

    public static CollectionDescriptor descriptor() {
        return new CollectionDescriptor(
            COLLECTION,
            List.of(
                new FieldDescriptor("uuid", FieldType.STRING, true, null),
                new FieldDescriptor("state", FieldType.STRING, true, null),
                new FieldDescriptor("namespace", FieldType.STRING, true, null),
                new FieldDescriptor("name", FieldType.STRING, true, null),
                new FieldDescriptor("actorId", FieldType.STRING, false, null),
                new FieldDescriptor("createdAt", FieldType.TIMESTAMP, true, null)
            ),
            List.of(
                new IndexDescriptor("idx_state", List.of("state"), IndexType.STANDARD),
                new IndexDescriptor("idx_ns_name", List.of("namespace", "name"), IndexType.STANDARD)
            ),
            List.of(),
            StorageHint.HYBRID
        );
    }
}
```

- [ ] **Step 4: Implement DataStoreCaseInstanceRepository**

Create `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/DataStoreCaseInstanceRepository.java`:

```java
package io.casehub.engine.persistence.datastore;

import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.EventLog;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.query.CaseInstanceQuery;
import io.casehub.platform.api.data.*;

import java.util.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class DataStoreCaseInstanceRepository implements CaseInstanceRepository {

    private static final String APP_ID = "engine";
    private final DataStore dataStore;
    private final DataCollection collection;
    private final AtomicLong idSeq = new AtomicLong(1);

    public DataStoreCaseInstanceRepository(DataStore dataStore) {
        this.dataStore = dataStore;
        this.dataStore.register(CaseInstanceSchema.descriptor());
        this.collection = dataStore.collection(CaseInstanceSchema.COLLECTION);
    }

    @Override
    public CaseInstance save(CaseInstance instance, String tenancyId) {
        instance.id = idSeq.getAndIncrement();
        instance.tenancyId = tenancyId;
        ObjectNode record = CaseInstanceMapper.toRecord(instance);
        record.put("_slug", instance.getUuid().toString());
        collection.insert(record, ctx(tenancyId));
        return instance;
    }

    @Override
    public CaseInstance update(CaseInstance instance, String tenancyId) {
        ObjectNode record = CaseInstanceMapper.toRecord(instance);
        collection.update(instance.getUuid().toString(), record,
                OptionalInt.empty(), MergeStrategy.REPLACE, ctx(tenancyId));
        return instance;
    }

    @Override
    public Optional<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
        return collection.get(uuid.toString(), ctx(tenancyId))
                .map(node -> {
                    CaseInstance ci = CaseInstanceMapper.fromRecord(node);
                    ci.tenancyId = tenancyId;
                    return ci;
                });
    }

    @Override
    public void updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId) {
        update(instance, tenancyId);
    }

    @Override
    public List<CaseInstance> findByStatus(CaseStatus status, String tenancyId) {
        Query q = new Query(
            List.of(new Filter("state", FilterOp.EQ, status.name())),
            List.of(), 0, Integer.MAX_VALUE
        );
        return collection.query(q, ctx(tenancyId)).records().stream()
                .map(node -> {
                    CaseInstance ci = CaseInstanceMapper.fromRecord(node);
                    ci.tenancyId = tenancyId;
                    return ci;
                })
                .toList();
    }

    @Override
    public List<CaseInstance> findAll(String tenancyId) {
        return collection.query(Query.all(), ctx(tenancyId)).records().stream()
                .map(node -> {
                    CaseInstance ci = CaseInstanceMapper.fromRecord(node);
                    ci.tenancyId = tenancyId;
                    return ci;
                })
                .toList();
    }

    @Override
    public List<CaseInstance> findByNamespaceAndName(String namespace, String name, String tenancyId) {
        Query q = new Query(
            List.of(
                new Filter("namespace", FilterOp.EQ, namespace),
                new Filter("name", FilterOp.EQ, name)
            ),
            List.of(), 0, Integer.MAX_VALUE
        );
        return collection.query(q, ctx(tenancyId)).records().stream()
                .map(node -> {
                    CaseInstance ci = CaseInstanceMapper.fromRecord(node);
                    ci.tenancyId = tenancyId;
                    return ci;
                })
                .toList();
    }

    @Override
    public List<CaseInstance> query(CaseInstanceQuery query, String tenancyId) {
        List<Filter> filters = new ArrayList<>();
        if (query.namespace() != null) {
            filters.add(new Filter("namespace", FilterOp.EQ, query.namespace()));
        }
        if (query.name() != null) {
            filters.add(new Filter("name", FilterOp.EQ, query.name()));
        }
        if (query.status() != null) {
            filters.add(new Filter("state", FilterOp.EQ, query.status().name()));
        }

        Query q = new Query(filters, List.of(), query.page(), query.size());
        return collection.query(q, ctx(tenancyId)).records().stream()
                .map(node -> {
                    CaseInstance ci = CaseInstanceMapper.fromRecord(node);
                    ci.tenancyId = tenancyId;
                    return ci;
                })
                .toList();
    }

    @Override
    public long count(CaseInstanceQuery query, String tenancyId) {
        List<Filter> filters = new ArrayList<>();
        if (query.namespace() != null) {
            filters.add(new Filter("namespace", FilterOp.EQ, query.namespace()));
        }
        if (query.name() != null) {
            filters.add(new Filter("name", FilterOp.EQ, query.name()));
        }
        if (query.status() != null) {
            filters.add(new Filter("state", FilterOp.EQ, query.status().name()));
        }

        Query q = new Query(filters, List.of(), 0, 1);
        return collection.count(q, ctx(tenancyId));
    }

    private DataContext ctx(String tenancyId) {
        return new DataContext(tenancyId, APP_ID);
    }
}
```

Note: The `insert()` method uses UUID as the doc_id. The `InMemoryDataCollection.insert()` generates its own UUID. We need to handle this — the insert should use the CaseInstance's UUID as the document ID, not generate a new one. This requires either: (a) adding an `insertWithId(String id, ObjectNode record, DataContext ctx)` method to the SPI, or (b) using the UUID as a field and looking up by field. Option (a) is cleaner. Add to `DataCollection`:

```java
String insert(String id, ObjectNode record, DataContext ctx);
```

And update both implementations to accept an explicit ID when provided. If `id` is null, generate one.

- [ ] **Step 5: Create CDI wiring**

Create `persistence-datastore/src/main/java/io/casehub/engine/persistence/datastore/quarkus/PersistenceDataStoreBeans.java`:

```java
package io.casehub.engine.persistence.datastore.quarkus;

import io.casehub.engine.persistence.datastore.DataStoreCaseInstanceRepository;
import io.casehub.platform.api.data.DataStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class PersistenceDataStoreBeans {

    @Produces
    @DefaultBean
    @ApplicationScoped
    DataStoreCaseInstanceRepository dataStoreCaseInstanceRepository(DataStore dataStore) {
        return new DataStoreCaseInstanceRepository(dataStore);
    }
}
```

- [ ] **Step 6: Run contract tests**

Run: `mvn test -pl persistence-datastore -q`
Expected: All CaseInstanceRepositoryContractTest cases PASS (queryAll, queryByStatus, queryByNamespace, queryPagination, count, actorId roundtrip, createdAt roundtrip)

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl data-api,data-sqlite,persistence-datastore -q`
Expected: All tests PASS across all three modules

- [ ] **Step 8: Commit**

```bash
git add persistence-datastore/ pom.xml
git commit -m "feat(#1332): add DataStoreCaseInstanceRepository adapter

Implements CaseInstanceRepository by delegating to the DataStore SPI.
CaseInstanceMapper handles domain ↔ record conversion. CDI wiring
via @DefaultBean producer. Passes full CaseInstanceRepositoryContractTest.

Refs #1332"
```

---

## References

- [2026-11-22-datastore-platform-spi-design.md] — design spec this plan implements
- [decisions.md] — D1-D6 design decisions
- [CaseInstanceRepository] — common-core/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java
- [CaseInstance] — common-core/src/main/java/io/casehub/engine/common/internal/model/CaseInstance.java
- [CaseMetaModel] — common-core/src/main/java/io/casehub/engine/common/internal/model/CaseMetaModel.java
- [CaseStatus] — api/src/main/java/io/casehub/api/model/CaseStatus.java
- [CaseInstanceQuery] — common-core/src/main/java/io/casehub/engine/common/spi/query/CaseInstanceQuery.java
- [CaseInstanceRepositoryContractTest] — common-core/src/test/java/io/casehub/engine/common/spi/CaseInstanceRepositoryContractTest.java
- [InMemoryCaseInstanceRepository] — engine-support-core/src/main/java/io/casehub/engine/persistence/memory/InMemoryCaseInstanceRepository.java
- [PersistenceMemoryBeans] — persistence-memory/src/main/java/io/casehub/engine/persistence/memory/quarkus/PersistenceMemoryBeans.java
- [JsonCollectionStore] — pages/backend/data-json/src/main/java/.../json/JsonCollectionStore.java
- [GitHub #1332] — JSON collection SPI adapter for case persistence
