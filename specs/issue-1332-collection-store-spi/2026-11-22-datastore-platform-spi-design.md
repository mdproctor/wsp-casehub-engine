# JSON Collection Adapter for Case Persistence — Design Spec

**Issue:** casehubio/engine#1332
**Branch:** issue-1332-collection-store-spi
**Date:** 2026-11-22
**Revised:** Round 2 — retargeted from DataStore platform SPI to focused CaseInstanceRepository adapter over JsonCollectionStore

## Overview

A `CaseInstanceRepository` and `CrossTenantCaseInstanceRepository` implementation backed by `JsonCollectionStore` from pages' `data-json` module. This gives YAML-only applications durable case instance persistence without requiring a database or JPA stack.

The in-memory `CaseInstanceCache` layer remains unchanged — it continues to provide shared-reference semantics and O(1) volatile access on top of the now-persistent repository.

## Why CaseInstanceRepository, not CaseInstanceCache

Issue #1332 names `CaseInstanceCache` as the target. Three architectural constraints make `CaseInstanceCache` unsuitable for persistent backing:

**1. Shared-reference aliasing (from R2-03):** `CaseInstanceCacheImpl` stores objects directly in a `ConcurrentHashMap`. Every `get()` returns the **same Java object** — mutations to one reference are visible to all callers. `CaseInstance.stateLock` (a `final ReentrantLock` for `trySetTerminalState()`) relies on this: all code paths share the same lock instance. A serializing cache returns different objects per `get()`, breaking the concurrency guard across 284+ concurrent call sites. `CaseTimeoutEnforcer.scanForTimeouts()` directly mutates `instance.setState(CaseStatus.FAULTED)` on the cached reference — this only works because the cache returns the live object.

**2. Non-serializable fields (from R2-04):** `CaseInstance` has fields that cannot round-trip through JSON:
- `stateLock` — `final ReentrantLock`, field initializer, not serializable
- `pendingActionGate` — documented as "in-memory only in v1" (CaseInstance.java line 161)
- `caseContext` — polymorphic `CaseContext` interface
- `exchangeHeaders` — untyped `Map<String, Object>` with numeric type variance

**3. Semantic mismatch (from R2-05):** `clear()` currently empties a `ConcurrentHashMap` — O(1), no I/O. With durable backing, it would destroy persistent state. `getAll()` returns only currently-active cases; with durable backing, it returns ALL historical cases, changing `CaseTimeoutEnforcer`'s timeout scan behavior (it would scan completed and historical cases, not just active ones).

**Conclusion:** `CaseInstanceRepository` is the right target — it's the actual persistence SPI designed for durable storage. Repository implementations already return copies by convention (JPA does this). The in-memory cache layer continues to serve shared-reference access on top of the persistent repository, exactly as it does today with the in-memory repository and with the JPA repository.

**DECISION_NEEDED:** Issue #1332 names `CaseInstanceCache`. This spec targets `CaseInstanceRepository` because `CaseInstanceCache` is architecturally unsuitable for persistent backing. Should #1332 be retargeted?

## Module: `persistence-json`

New engine module implementing `CaseInstanceRepository` and `CrossTenantCaseInstanceRepository` by delegating to `JsonCollectionStore`.

### JsonCollectionCaseInstanceRepository

```java
@ApplicationScoped
public class JsonCollectionCaseInstanceRepository
    implements CaseInstanceRepository, CrossTenantCaseInstanceRepository {

    private final JsonCollectionStore store;
    private final EventLogRepository eventLogRepository;
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private static final String APP_ID = "engine";
    private static final String COLLECTION = "case_instances";
    private final AtomicLong idSeq = new AtomicLong(0);
    private final String defaultTenantId;
    private CaseContextRecoveryStrategy recoveryStrategy;

    @Inject
    public JsonCollectionCaseInstanceRepository(
            JsonCollectionStore store,
            EventLogRepository eventLogRepository,
            @ConfigProperty(name = "casehub.tenant.default", defaultValue = "default")
            String defaultTenantId) {
        this.store = store;
        this.eventLogRepository = eventLogRepository;
        this.defaultTenantId = defaultTenantId;
        this.idSeq.set(initMaxId());
    }

    public void setRecoveryStrategy(CaseContextRecoveryStrategy strategy) {
        this.recoveryStrategy = strategy;
    }

    @Override
    public CaseInstance save(CaseInstance instance, String tenancyId) {
        rwLock.writeLock().lock();
        try {
            if (instance.id == null) {
                instance.id = idSeq.incrementAndGet();
            }
            if (recoveryStrategy != null && instance.getCaseContext() != null) {
                recoveryStrategy.onContextChanged(instance, instance.getCaseContext());
            }
            instance.tenancyId = tenancyId;
            ObjectNode record = CaseInstanceMapper.toRecord(instance);
            DataContext ctx = new DataContext(tenancyId);
            store.create(ctx, APP_ID, COLLECTION, record, "uuid");
            return instance;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    @Override
    public CaseInstance update(CaseInstance instance, String tenancyId) {
        rwLock.writeLock().lock();
        try {
            String docId = instance.getUuid().toString();
            DataContext ctx = new DataContext(tenancyId);
            Optional<JsonNode> existing = store.get(ctx, APP_ID, COLLECTION, docId);
            if (existing.isEmpty() || !tenancyId.equals(
                    existing.get().path("tenancyId").asText())) {
                throw new IllegalStateException(
                    "CaseInstance not found or wrong tenant: " + instance.getUuid());
            }
            ObjectNode record = CaseInstanceMapper.toRecord(instance);
            int currentVersion = existing.get().get("_version").asInt();
            store.update(ctx, APP_ID, COLLECTION, docId,
                        record, OptionalInt.of(currentVersion),
                        MergeStrategy.SHALLOW);
            return instance;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    @Override
    public Optional<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.get(new DataContext(tenancyId), APP_ID, COLLECTION,
                           uuid.toString())
                .map(CaseInstanceMapper::fromRecord);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    // CrossTenantCaseInstanceRepository — no tenant filter
    @Override
    public Optional<CaseInstance> findByUuid(UUID uuid) {
        rwLock.readLock().lock();
        try {
            return store.get(new DataContext(defaultTenantId), APP_ID, COLLECTION,
                           uuid.toString())
                .map(CaseInstanceMapper::fromRecord);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    @Override
    public void updateStateAndAppendEvent(
            CaseInstance instance, EventLog eventLog, String tenancyId) {
        rwLock.writeLock().lock();
        try {
            if (recoveryStrategy != null && instance.getCaseContext() != null) {
                recoveryStrategy.onContextChanged(instance, instance.getCaseContext());
            }
            instance.tenancyId = tenancyId;
            String docId = instance.getUuid().toString();
            DataContext ctx = new DataContext(tenancyId);
            ObjectNode record = CaseInstanceMapper.toRecord(instance);
            store.update(ctx, APP_ID, COLLECTION, docId,
                        record, OptionalInt.empty(), MergeStrategy.SHALLOW);
            eventLogRepository.append(eventLog, tenancyId);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    @Override
    public List<CaseInstance> findByStatus(CaseStatus status, String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.listAll(tenancyId, APP_ID, COLLECTION).stream()
                .map(CaseInstanceMapper::fromRecord)
                .filter(ci -> ci.getState() == status)
                .toList();
        } finally {
            rwLock.readLock().unlock();
        }
    }

    @Override
    public List<CaseInstance> findAll(String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.listAll(tenancyId, APP_ID, COLLECTION).stream()
                .map(CaseInstanceMapper::fromRecord)
                .toList();
        } finally {
            rwLock.readLock().unlock();
        }
    }

    @Override
    public List<CaseInstance> findByNamespaceAndName(
            String namespace, String name, String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.listAll(tenancyId, APP_ID, COLLECTION).stream()
                .map(CaseInstanceMapper::fromRecord)
                .filter(ci -> ci.getCaseMetaModel() != null
                    && namespace.equals(ci.getCaseMetaModel().getNamespace())
                    && name.equals(ci.getCaseMetaModel().getName()))
                .toList();
        } finally {
            rwLock.readLock().unlock();
        }
    }

    @Override
    public List<CaseInstance> query(CaseInstanceQuery query, String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.listAll(tenancyId, APP_ID, COLLECTION).stream()
                .map(CaseInstanceMapper::fromRecord)
                .filter(i -> query.status() == null || query.status().equals(i.getState()))
                .filter(i -> {
                    if (query.namespace() == null) return true;
                    var meta = i.getCaseMetaModel();
                    return meta != null && query.namespace().equals(meta.getNamespace());
                })
                .filter(i -> {
                    if (query.name() == null) return true;
                    var meta = i.getCaseMetaModel();
                    return meta != null && query.name().equals(meta.getName());
                })
                .skip((long) query.page() * query.size())
                .limit(query.size())
                .toList();
        } finally {
            rwLock.readLock().unlock();
        }
    }

    @Override
    public long count(CaseInstanceQuery query, String tenancyId) {
        rwLock.readLock().lock();
        try {
            return store.listAll(tenancyId, APP_ID, COLLECTION).stream()
                .map(CaseInstanceMapper::fromRecord)
                .filter(i -> query.status() == null || query.status().equals(i.getState()))
                .filter(i -> {
                    if (query.namespace() == null) return true;
                    var meta = i.getCaseMetaModel();
                    return meta != null && query.namespace().equals(meta.getNamespace());
                })
                .filter(i -> {
                    if (query.name() == null) return true;
                    var meta = i.getCaseMetaModel();
                    return meta != null && query.name().equals(meta.getName());
                })
                .count();
        } finally {
            rwLock.readLock().unlock();
        }
    }

    private long initMaxId() {
        return store.listAll(defaultTenantId, APP_ID, COLLECTION).stream()
            .map(CaseInstanceMapper::fromRecord)
            .mapToLong(ci -> ci.id != null ? ci.id : 0)
            .max().orElse(0);
    }
}
```

### Key Design Choices

**Document ID = CaseInstance UUID:** The CaseInstance's UUID (business identity, assigned by the caller before `save()`) is used as the JSON document ID via the slug mechanism. The adapter never assigns or overwrites UUIDs — this preserves the existing identity model where `CaseInstance.uuid` is set before save and `CaseInstance.id` (sequential Long) is assigned by the repository.

**Write lock for atomicity:** `updateStateAndAppendEvent` uses a `ReentrantReadWriteLock` (identical pattern to `InMemoryCaseInstanceRepository`) for cross-operation atomicity. Event log persistence is delegated to the injected `EventLogRepository` (which remains in-memory by default). For YAML-only single-instance deployments, process-level locking is sufficient.

**CrossTenantCaseInstanceRepository in same class:** Matches the `InMemoryCaseInstanceRepository` pattern — both interfaces implemented by one class. For YAML-only (single-tenant), the cross-tenant `findByUuid(UUID)` queries with the configured default tenant.

**Uses domain-typed CaseInstanceQuery directly:** No generic `Filter/Query` abstraction. The repository methods accept `CaseInstanceQuery` with typed `CaseStatus` and string fields — query/filter logic is in the adapter, identical to `InMemoryCaseInstanceRepository`.

**Cache layer unchanged:** The `CaseInstanceCache` (wired as `CaseInstanceCacheImpl` in `RuntimeBeans`) remains an in-memory `ConcurrentHashMap`. It sits in front of this repository, providing shared-reference access with identity semantics. This is the same architecture used with the JPA repository — the cache provides fast L1 access, the repository provides durable persistence.

### CaseInstanceMapper

Bidirectional mapping between `CaseInstance` domain object and `ObjectNode` records.

**Serialized fields:**
- Primitive: `id` (Long), `uuid` (UUID→String), `version` (long), `state` (CaseStatus→String), `actorId`, `createdAt` (Instant→ISO-8601), `tenancyId`
- Complex: `caseMetaModel` (namespace/name/version as nested object), `contextSnapshot` (JsonNode — stored directly, no conversion needed)
- Collections: `labels` (Set→JSON array), `types` (Set→JSON array)

**Not serialized (in-memory-only fields):**
- `stateLock` (ReentrantLock) — `final` field with initializer. Fresh lock created on deserialization. This is correct: each loaded instance gets its own lock, matching JPA behavior where each entity load creates a new Java object. The in-memory cache provides shared-reference locking across concurrent code paths — the repository-layer copies are independent.
- `pendingActionGate` — documented as "in-memory only in v1" per CaseInstance.java line 161: *"If the engine restarts while a gate is pending, the gate is lost from memory."* Not persisted — matching both JPA and in-memory behavior.
- `propagationContext` — runtime-only execution context, not persisted. Set during case execution, not part of durable state.

**Serialization challenges addressed:**
- **`caseContext` (CaseContext interface):** Serialized via `contextSnapshot` — `CaseInstance` already carries a `contextSnapshot: JsonNode` field that captures the context state as raw JSON. The mapper stores `contextSnapshot` and reconstructs `caseContext` through the standard `CaseContext` factory on deserialization. This is the same approach used by JPA persistence (`CaseInstanceEntity` stores the context as a JSONB column).
- **`exchangeHeaders` (Map<String, Object>):** Serialized as a JSON object. On deserialization, values come back as Jackson types (TextNode, IntNode, etc.). This matches the type variance seen in JPA JSONB round-trips — callers that depend on specific Java numeric types already handle this.

### CDI Wiring

```java
@ApplicationScoped
public class PersistenceJsonBeans {

    @Produces
    @Alternative
    @Priority(200)
    @ApplicationScoped
    JsonCollectionCaseInstanceRepository jsonCollectionCaseInstanceRepository(
            JsonCollectionStore store,
            EventLogRepository eventLogRepository,
            Instance<CaseContextRecoveryStrategy> recoveryStrategy,
            @ConfigProperty(name = "casehub.tenant.default", defaultValue = "default")
            String defaultTenantId) {
        JsonCollectionCaseInstanceRepository repo =
            new JsonCollectionCaseInstanceRepository(store, eventLogRepository, defaultTenantId);
        if (recoveryStrategy.isResolvable()) {
            repo.setRecoveryStrategy(recoveryStrategy.get());
        }
        return repo;
    }
}
```

**CDI priority:** `@Alternative @Priority(200)` displaces the in-memory default:
- `InMemoryCaseInstanceRepository` is `@DefaultBean` in `PersistenceMemoryBeans`
- `CaseInstanceCacheImpl` is `@ApplicationScoped` in `RuntimeBeans` — not affected; the cache layer is independent

The CDI wiring mirrors `PersistenceMemoryBeans`: same constructor dependencies (`EventLogRepository`, `CaseContextRecoveryStrategy`), same `setRecoveryStrategy` pattern. The `EventLogRepository` remains in-memory — only case instance persistence moves to JSON storage.

### Tenancy

`CaseInstanceRepository` methods take `tenancyId` as an explicit parameter — no ambient context needed. The `defaultTenantId` is only used for `CrossTenantCaseInstanceRepository.findByUuid(UUID)` (no tenant parameter) and `initMaxId()` on startup. For YAML-only single-tenant deployments, this is a fixed value via `@ConfigProperty`.

## Module Dependencies

```
casehub-engine common-core          (CaseInstanceRepository, CrossTenantCaseInstanceRepository,
                                     EventLogRepository SPIs)
  ↑
casehub-pages data-json             (JsonCollectionStore, MergeStrategy, DataContext)
  ↑
casehub-engine persistence-json     (CaseInstanceRepository adapter)
```

No new platform-api types. No Jackson dependency added to platform-api. The `persistence-json` module depends on pages' `data-json` directly — a one-directional dependency (pages has no dependency on engine). The `data-json` module is effectively a standalone JSON document storage library.

## Testing

- **CaseInstanceMapper unit tests:** Round-trip serialization for all field types. Verify non-persisted fields (`stateLock`, `pendingActionGate`, `propagationContext`) get fresh/null instances on deserialization.
- **CaseInstanceRepositoryContractTest runner:** The existing contract test in `common-core` gains a new concrete runner in `persistence-json` that wires through JsonCollectionStore → CaseInstanceRepository. This validates all 9 repository methods against the adapter.
- **CrossTenantCaseInstanceRepositoryContractTest runner:** Same pattern for cross-tenant lookups.
- **Atomicity test:** Verify `updateStateAndAppendEvent` persists case state and delegates event log append within the write lock boundary.
- **CDI wiring test:** Verify that with `persistence-json` on the classpath, `CaseInstanceRepository` injection resolves to `JsonCollectionCaseInstanceRepository` rather than `InMemoryCaseInstanceRepository`.
- **Restart persistence test:** Create case, simulate restart (new repository instance against same SQLite DB), verify case is recoverable via `findByUuid`.

## Scope Boundaries

**In scope (this issue):**
1. `CaseInstanceRepository` + `CrossTenantCaseInstanceRepository` implementation backed by `JsonCollectionStore`
2. `CaseInstanceMapper` with serialization challenges addressed
3. CDI wiring (`@Alternative @Priority(200)`)
4. Contract test runners
5. Module: `persistence-json` in engine repo

**Out of scope (follow-up — see §Deferred Issues):**
- Platform DataStore SPI (unified collection-based persistence abstraction)
- Companion SPI adapters (EventLogRepository, CaseMetaModelRepository, SubCaseGroupRepository, PlanItemStore)
- Neocortex/qhorus storage consolidation
- Pages `JsonCollectionStore` extraction to shared platform module

## Deferred Issues

GitHub issues to file:

1. **Platform DataStore SPI epic** (casehubio/platform-api): Unified collection-based persistence SPI consolidating pages, engine, neocortex, and qhorus storage. Design concerns to address: naming collision with existing `DataSource` SPI and qhorus `DataStore`, cross-collection atomicity/transaction model, `search()` behavioral specification, `RelationDescriptor` behavioral contract, Jackson dependency on zero-dep platform-api (requires ADR), table-backed column collision on shared table, `DataContext` appId optionality. Extract from `JsonCollectionStore` rather than duplicate.

2. **Companion engine persistence-json adapters** (casehubio/engine): `EventLogRepository`, `CaseMetaModelRepository`, `SubCaseGroupRepository`, `PlanItemStore` adapters over `JsonCollectionStore`. `EventLogRepository` is highest priority — enables durable audit trail. When `EventLogRepository` gets a JSON adapter, the `updateStateAndAppendEvent` atomicity can optionally be strengthened with SQLite-level transactions (shared connection).

3. **Pages data-json extraction** (casehubio/pages): Extract `JsonCollectionStore`, `MergeStrategy`, `DataContext`, and `SqliteConnectionFactory` to a shared platform module for cross-repo reuse without engine→pages coupling.

## References

- Issue #1332: JSON collection SPI adapter for case persistence
- Parent: casehubio/examples#212 (YAML-only helpdesk)
- `CaseInstanceRepository` — engine/common-core/src/main/java/.../spi/CaseInstanceRepository.java
- `CrossTenantCaseInstanceRepository` — engine/common-core/src/main/java/.../spi/CrossTenantCaseInstanceRepository.java
- `InMemoryCaseInstanceRepository` — engine/engine-support-core/src/main/java/.../InMemoryCaseInstanceRepository.java
- `CaseInstanceCache` — engine/common-core/src/main/java/.../spi/cache/CaseInstanceCache.java
- `CaseInstanceCacheImpl` — engine/runtime-core/src/main/java/.../cache/CaseInstanceCacheImpl.java (wired `@ApplicationScoped` in RuntimeBeans line 1083)
- `CaseInstance` — engine/common-core/src/main/java/.../model/CaseInstance.java (stateLock line 43, pendingActionGate line 169)
- `CaseTimeoutEnforcer` — engine/resilience-core/src/main/java/.../timeout/CaseTimeoutEnforcer.java (cache.getAll() line 74)
- `JsonCollectionStore` — pages/backend/data-json/src/main/java/.../json/JsonCollectionStore.java
- `DataContext` — pages/backend/data/src/main/java/.../data/DataContext.java (`record DataContext(String tenantId)`)
- `PersistenceMemoryBeans` — engine/persistence-memory/src/main/java/.../PersistenceMemoryBeans.java
- `CaseRecoveryService` — engine/runtime-core/src/main/java/.../recovery/CaseRecoveryService.java (cache-then-repository pattern)
