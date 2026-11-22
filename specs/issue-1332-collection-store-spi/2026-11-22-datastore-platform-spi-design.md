# DataStore Platform SPI — Design Spec

**Issue:** casehubio/engine#1332
**Branch:** issue-1332-collection-store-spi
**Date:** 2026-11-22
**Revised:** Post-review — incorporated naming collision analysis (qhorus), Jackson dependency note, defensive copy contract, extract-from-JsonCollectionStore strategy

## Overview

A platform-level data persistence SPI that any application built on CaseHub can use. The SPI abstracts collection-based record storage with optional schema descriptors, supporting both schema-less document storage (simple apps) and schema-ful typed storage (complex apps) behind the same contract.

The engine's `CaseInstanceRepository` becomes one adapter over this SPI. Neocortex storage modules (memory, mindmap, cognitive-observability) and qhorus' 20+ per-entity stores are future migration targets. End-user applications get a first-class persistence API.

## Motivation

The platform currently has fragmented storage:
- **Engine:** domain-specific SPIs (`CaseInstanceRepository`, `EventLogRepository`, `PlanItemStore`) with per-database implementations (in-memory, JPA, Spring JPA)
- **Pages:** concrete `JsonCollectionStore` class coupled to SQLite — not an SPI
- **Neocortex:** four independent SQLite-backed stores, each with their own connection pooling and migration
- **Qhorus:** 20+ per-entity store SPIs (`ChannelStore`, `MessageStore`, `InstanceStore`, `DataStore`, etc.), each with in-memory, JPA, and Spring implementations

This fragmentation means every new persistence need requires a new SPI, new implementations, and new CDI wiring. A unified DataStore SPI eliminates this — consumers work with collections of records, implementations handle physical storage.

### Why CaseInstanceRepository, not CaseInstanceCache

`CaseInstanceCache` (the SPI named in issue #1332) is a 4-method in-memory L1 cache sitting in front of `CaseInstanceRepository`. The runtime pattern is: writes go through the repository first, then the cache. Backing only the cache with persistent storage doesn't give durable persistence — the repository data remains volatile. For YAML-only apps that need data to survive restarts, the adapter must target `CaseInstanceRepository` (the actual persistence SPI with tenant isolation, versioning, and query support).

## SPI Contract

### Package

`io.casehub.platform.api.data` in `casehub-platform-api`.

Distinct from the existing `io.casehub.platform.api.datasource` package, which is a reactive event stream SPI (subscribe to typed data sources). DataStore is CRUD persistence, not event streaming.

**Naming note:** The name `DataStore` temporarily collides with `io.casehub.qhorus.api.store.DataStore` (a single-entity store for `SharedData`). Qhorus' `DataStore` is a migration target for the platform DataStore SPI — during that migration, the qhorus-specific interface will be removed and its functionality served by a registered collection. The platform-level name takes precedence as the long-term canonical interface.

**Jackson dependency:** Adding `ObjectNode` to platform-api introduces the first external dependency on a module that is currently zero-dep pure Java. Jackson is a universal dependency across the platform (every consumer already has it), and `ObjectNode` is the platform's de facto interchange format. The dependency is accepted as the pragmatic choice over alternatives (`jakarta.json.JsonObject`, custom `Record` type, `Map<String, Object>`) that would add indirection without clear benefit.

### Core Interfaces

```java
package io.casehub.platform.api.data;

public interface DataStore {

    /**
     * Register a collection schema. Optional — collections without
     * registration operate in schema-less mode. Implementations use
     * the descriptor to optimise storage (typed columns, indexes).
     */
    void register(CollectionDescriptor descriptor);

    /**
     * Access a named collection. The collection may or may not have
     * been registered — unregistered collections are schema-less.
     */
    DataCollection collection(String name);
}

public interface DataCollection {

    String insert(ObjectNode record, DataContext ctx);

    Optional<ObjectNode> get(String id, DataContext ctx);

    void update(String id, ObjectNode changes,
                OptionalInt expectedVersion, MergeStrategy mergeStrategy,
                DataContext ctx);

    void delete(String id, DataContext ctx);

    QueryResult query(Query query, DataContext ctx);

    long count(Query query, DataContext ctx);

    List<ObjectNode> search(String text, DataContext ctx);
}
```

**Defensive copy contract:** All `ObjectNode` values returned by `get()`, `query()`, and `search()` are independent copies. Consumers may mutate returned nodes freely without affecting store state. Implementations must not return shared references to internal storage.

### DataContext

```java
public record DataContext(String tenantId, String appId) {}
```

Extends the pages concept (`tenantId` only) with `appId` to scope collections per application. Every operation is tenant-and-app-scoped.

### CollectionDescriptor

```java
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

### FieldDescriptor

```java
public record FieldDescriptor(
    String name,
    FieldType type,
    boolean required,
    Object defaultValue
) {}

public enum FieldType {
    STRING, INTEGER, LONG, DOUBLE, BOOLEAN, TIMESTAMP, JSON
}
```

### IndexDescriptor

```java
public record IndexDescriptor(
    String name,
    List<String> fields,
    IndexType type
) {}

public enum IndexType {
    STANDARD, UNIQUE, FULLTEXT
}
```

### RelationDescriptor

```java
public record RelationDescriptor(
    String name,
    String field,
    String targetCollection,
    String targetField,
    RelationType type
) {}

public enum RelationType {
    MANY_TO_ONE, ONE_TO_MANY
}
```

### StorageHint

```java
public enum StorageHint {
    DOCUMENT,   // store as JSON blob — schema-less, flexible
    TYPED,      // store in typed columns — schema-ful, queryable
    HYBRID      // typed columns for indexed fields, JSON blob for the rest
}
```

Hints are non-binding — a simple implementation can store everything as documents regardless of the hint. A capable implementation uses the hint to optimise.

### MergeStrategy

```java
public enum MergeStrategy {
    REPLACE,    // full replacement of the record
    SHALLOW,    // top-level field merge
    DEEP,       // recursive merge of nested objects
    RFC_7396    // JSON Merge Patch — null values remove fields
}
```

### Query

```java
public record Query(
    List<Filter> filters,
    List<SortField> sort,
    int page,
    int size
) {
    public static Query all() {
        return new Query(List.of(), List.of(), 0, 20);
    }
}

public record Filter(String field, FilterOp op, Object value) {}

public enum FilterOp {
    EQ, NEQ, GT, GTE, LT, LTE, IN, CONTAINS, EXISTS
}

public record SortField(String field, SortDirection direction) {}

public enum SortDirection { ASC, DESC }
```

### QueryResult

```java
public record QueryResult(
    List<ObjectNode> records,
    long totalCount,
    int page,
    int size
) {}
```

### System Fields

Implementations manage these system fields transparently:
- `_id` — document identifier (String, assigned on insert)
- `_version` — optimistic concurrency version (integer, starts at 1, incremented on update)
- `_created_at` — creation timestamp (ISO-8601)
- `_updated_at` — last modification timestamp (ISO-8601)

System fields are injected on read and stripped on write — consumers never set them directly. `_version` is used for optimistic concurrency when `expectedVersion` is provided on update.

### Error Handling

```java
public class DataStoreException extends RuntimeException { ... }
public class VersionConflictException extends DataStoreException { ... }
public class RecordNotFoundException extends DataStoreException { ... }
public class CollectionNotFoundException extends DataStoreException { ... }
```

## Implementations

### Implementation Strategy: Extract from JsonCollectionStore

The document-backed implementation is not built from scratch — it is extracted from the existing `JsonCollectionStore` in pages' `data-json` module. `JsonCollectionStore` already provides: SQLite-backed JSON document storage, tenant isolation, optimistic concurrency (`_version` column), system fields, merge strategies, and collection scoping. The extraction refactors this proven code to implement the new `DataStore` SPI interface.

### Document-Backed (SQLite)

Single `data_records` table:

```sql
CREATE TABLE data_records (
    tenant_id   TEXT NOT NULL,
    app_id      TEXT NOT NULL,
    collection  TEXT NOT NULL,
    doc_id      TEXT NOT NULL,
    data        TEXT NOT NULL,       -- JSON blob
    _version    INTEGER NOT NULL DEFAULT 1,
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL,
    PRIMARY KEY (tenant_id, app_id, collection, doc_id)
);
```

All records stored as JSON blobs. Filtering uses SQLite's `json_extract()` for field-based queries (not read-all-filter-in-Java). Suitable for simple apps with modest data volumes.

FTS support via SQLite FTS5 virtual table (created on-demand when an `IndexType.FULLTEXT` descriptor is registered).

### Table-Backed (SQLite)

Same single table structure, but with additional indexed columns derived from `CollectionDescriptor` fields. When a collection is registered with `StorageHint.TYPED` or `HYBRID`:

- Extracted fields are stored in dedicated indexed columns (added via `ALTER TABLE`)
- The JSON blob column still holds the full record for flexibility
- Queries use the indexed columns for filtering and sorting

This avoids per-collection tables while giving query performance on declared fields.

### In-Memory

`ConcurrentHashMap`-based implementation for tests. No SQLite dependency. Supports all SPI operations including query, search, and version checks.

## Engine Adapter

### Module: `persistence-datastore`

New engine module that implements `CaseInstanceRepository` by delegating to `DataStore`.

```java
@ApplicationScoped
public class DataStoreCaseInstanceRepository implements CaseInstanceRepository {

    private final DataStore dataStore;
    private static final String COLLECTION = "case_instances";

    @Inject
    public DataStoreCaseInstanceRepository(DataStore dataStore) {
        this.dataStore = dataStore;
        dataStore.register(CaseInstanceSchema.descriptor());
    }

    @Override
    public CaseInstance save(CaseInstance instance, String tenancyId) {
        ObjectNode record = CaseInstanceMapper.toRecord(instance);
        String docId = dataStore.collection(COLLECTION)
            .insert(record, new DataContext(tenancyId, "engine"));
        instance.setUuid(UUID.fromString(docId));
        return instance;
    }

    @Override
    public Optional<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
        return dataStore.collection(COLLECTION)
            .get(uuid.toString(), new DataContext(tenancyId, "engine"))
            .map(CaseInstanceMapper::fromRecord);
    }

    // ... remaining CaseInstanceRepository methods delegate similarly
}
```

### CaseInstanceMapper

Bidirectional mapping between `CaseInstance` domain object and `ObjectNode` records. Handles:
- Primitive fields: `uuid`, `version`, `state`, `actorId`, `createdAt`
- Complex fields: `caseMetaModel` (namespace/name/version), `caseContext`, `contextSnapshot`
- Collections: `labels`, `types`, `exchangeHeaders`
- Enum fields: `state` (CaseStatus) stored as string, converted back via `CaseStatus.valueOf()`

### CaseInstanceSchema

Declares the `CollectionDescriptor` for case instances:

```java
public class CaseInstanceSchema {
    public static CollectionDescriptor descriptor() {
        return new CollectionDescriptor(
            "case_instances",
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

### CDI Wiring

```java
@ApplicationScoped
public class PersistenceDataStoreBeans {

    @Produces @DefaultBean @ApplicationScoped
    DataStoreCaseInstanceRepository dataStoreCaseInstanceRepository(DataStore dataStore) {
        return new DataStoreCaseInstanceRepository(dataStore);
    }
}
```

## Contract Tests

### DataStoreContractTest

Abstract base class in a test-support module:

```java
public abstract class DataStoreContractTest {

    protected abstract DataStore dataStore();
    protected abstract DataContext testContext();

    @Test void insert_and_get_roundTrip() { ... }
    @Test void update_incrementsVersion() { ... }
    @Test void update_withExpectedVersion_throwsOnConflict() { ... }
    @Test void delete_removesRecord() { ... }
    @Test void query_filtersByField() { ... }
    @Test void query_pagination() { ... }
    @Test void query_sorting() { ... }
    @Test void search_fullText() { ... }
    @Test void schemaless_collection_worksWithoutRegistration() { ... }
    @Test void registered_collection_honorsFieldTypes() { ... }
    @Test void systemFields_injectedOnRead_strippedOnWrite() { ... }
    @Test void relations_crossCollectionReference() { ... }
    @Test void returnedNodes_areDefensiveCopies() { ... }
}
```

Each implementation module provides a concrete runner.

### CaseInstanceRepository Contract Tests

The existing `CaseInstanceRepositoryContractTest` gains a new concrete runner in `persistence-datastore` that wires through DataStore → CaseInstanceRepository adapter.

## Module Dependencies

```
casehub-platform-api          (SPI interfaces + Jackson dependency)
  ↑
casehub-data-sqlite           (document-backed + table-backed SQLite impls,
                               extracted from pages' JsonCollectionStore)
  ↑
casehub-engine persistence-datastore  (CaseInstanceRepository adapter)
```

`casehub-data-sqlite` is a new standalone module — not part of engine, not part of pages. It implements the platform DataStore SPI using SQLite. Any repo can depend on it.

## Platform Consolidation Roadmap

This SPI is the foundation for consolidating per-entity storage across the platform:

| Repo | Current state | Migration target |
|------|--------------|------------------|
| **Engine** | CaseInstanceRepository, EventLogRepository, PlanItemStore (3+ SPIs, each with in-memory/JPA/Spring impls) | Domain adapters over DataStore |
| **Qhorus** | 20+ per-entity store SPIs (ChannelStore, MessageStore, etc.), each with in-memory/JPA/Spring impls | Registered collections with schema descriptors |
| **Neocortex** | 4 independent SQLite stores (memory, mindmap, observability, pages) | DataStore collections (simple cases); dedicated stores for graph-heavy mindmap |
| **Pages** | Concrete JsonCollectionStore | Extracted as DataStore SPI reference implementation |

This issue delivers the SPI + first adapter. Each row above is a follow-up epic.

## Scope Boundaries

**In scope (this issue):**
1. `DataStore` SPI interfaces in `casehub-platform-api`
2. Document-backed SQLite implementation (extracted from JsonCollectionStore)
3. Table-backed SQLite implementation
4. In-memory implementation (tests)
5. `CaseInstanceRepository` adapter in engine
6. Contract tests for both DataStore and the adapter

**Out of scope (follow-up issues):**
- Qhorus store consolidation (20+ per-entity SPIs → DataStore collections)
- Neocortex storage migration (memory, mindmap, cognitive-observability)
- DataSource SPI abstraction (abstract away SQLite dependency)
- Pages refactor (make pages consume DataStore instead of JsonCollectionStore directly)
- Companion engine SPI adapters (EventLogRepository, CaseMetaModelRepository, PlanItemStore)

## References

- [D1–D5] decisions.md — design decisions with rationale
- Salesforce MT_Data universal table architecture — metadata-driven types
- JCR/Oak NodeStore SPI — storage knows nothing about types
- Keycloak MapStorageProvider — per-entity SPIs → generic area-based SPI
- `CaseInstanceRepository` — engine/common-core/src/main/java/.../spi/CaseInstanceRepository.java
- `CaseInstanceQuery` — engine/common-core/src/main/java/.../spi/query/CaseInstanceQuery.java
- `CaseInstanceRepositoryContractTest` — engine/common-core/src/test/java/.../spi/CaseInstanceRepositoryContractTest.java
- `JsonCollectionStore` — pages/backend/data-json/src/main/java/.../json/JsonCollectionStore.java
- `DataStore` (qhorus) — qhorus/api/src/main/java/.../api/store/DataStore.java (migration target)
- `PersistenceMemoryBeans` — engine/persistence-memory/src/main/java/.../PersistenceMemoryBeans.java
- Neocortex storage modules: memory-sqlite, mindmap-sqlite, cognitive-observability-sqlite
- `io.casehub.platform.api.datasource` — existing reactive data stream SPI (separate concern)
