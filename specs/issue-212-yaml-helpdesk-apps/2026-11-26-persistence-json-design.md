# Design: persistence-json — JSON Collection CaseInstanceCache

**Issue:** casehubio/engine#1366
**Date:** 2026-11-26
**Status:** Draft

## Problem

Zero-Java case applications need persistence without Hibernate, Flyway, or PostgreSQL. The case engine's `CaseInstanceCache` SPI has only an in-memory implementation (`CaseInstanceCacheImpl`) that loses all state on restart. Apps deployed on Pages already have SQLite via `casehub-pages-sqlite` — reusing that as the persistence layer gives persistence for free.

## Solution

New Maven module `casehub-engine-persistence-json` implementing `CaseInstanceCache` backed by `JsonCollectionStore` from `casehub-pages-data-json`.

## Architecture

### Module Structure

```
persistence-json/
  pom.xml
  src/main/java/io/casehub/engine/persistence/json/
    JsonCollectionCaseInstanceCache.java    # CaseInstanceCache impl
    CaseInstanceDocument.java              # Serializable snapshot record
    quarkus/
      PersistenceJsonBeans.java            # CDI producer
```

### Dependencies

| Dependency | Purpose |
|---|---|
| `casehub-engine-common-core` | `CaseInstanceCache` SPI, `CaseInstance` model |
| `casehub-pages-data-json` | `JsonCollectionStore` for SQLite-backed JSON collections |
| `casehub-pages-data-backend` | `DataContext` record |
| `jackson-databind` | Serialization (transitive) |

### Serialization: CaseInstanceDocument

A flat record capturing the serializable state of a `CaseInstance`:

```java
record CaseInstanceDocument(
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

**What's stored:** UUID, state, version, case metadata identity (namespace/name/version), context snapshot (the JsonNode already maintained by the engine), parent references, actor, labels, types, timestamps.

**What's NOT stored (transient runtime state):**
- `CaseContext` with lambda evaluators — reconstructed from contextSnapshot + definition
- `ReentrantLock stateLock` — runtime-only
- `PendingActionGate` — in-memory only per existing design (see CaseInstance javadoc)
- `waitingForWorkId` — transient scheduler state
- `PropagationContext` — transient execution state
- `exchangeHeaders` — transient per-execution headers

### JsonCollectionCaseInstanceCache

Implements `CaseInstanceCache` with these mappings:

| SPI method | Implementation |
|---|---|
| `put(CaseInstance)` | Serialize to `CaseInstanceDocument`, upsert into collection `case-instances` with doc ID = UUID |
| `get(UUID)` | Fetch from collection by UUID, deserialize, reconstruct `CaseInstance` |
| `clear()` | Delete all docs in collection (or drop + recreate) |
| `getAll()` | `listAll()` on the collection, deserialize each |

**Collection coordinates:**
- `appId`: `"casehub-engine"` (fixed)
- `collection`: `"case-instances"` (fixed)
- `tenantId`: from `CurrentPrincipal.tenancyId()` (CDI-injected)

**Reconstruction on `get()`:**
1. Deserialize `CaseInstanceDocument` from JSON
2. Create new `CaseInstance`, set all scalar fields
3. Set `contextSnapshot` from stored JsonNode
4. Look up `CaseMetaModel` from `CaseDefinitionRegistry.findByIdentity()` using stored namespace/name/version
5. Leave transient fields (`caseContext`, `stateLock`, `pendingActionGate`) at defaults

**Note on CaseContext reconstruction:** The cache stores the context snapshot. The engine reconstructs the full `CaseContext` (with evaluators) from the snapshot when it needs it — this is the same path used by the existing persistence-hibernate implementation. The cache doesn't need to reconstruct `CaseContext` itself.

### CDI Integration

`PersistenceJsonBeans` produces `JsonCollectionCaseInstanceCache` as an `@Alternative @Priority(200)` bean. This overrides the default `CaseInstanceCacheImpl` when the module is on the classpath. The in-memory default remains available as `@DefaultBean` for tests.

```java
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

### Upsert Strategy

`JsonCollectionStore.create()` throws on duplicate slug. For `put()`:
1. Try `get()` first
2. If exists: `update()` with `MergeStrategy.REPLACE` and no version check
3. If not: `create()` with UUID as doc ID

This is a cache — optimistic concurrency (version checks) is not needed. The engine serializes writes through its own locking.

## Testing

Unit test `JsonCollectionCaseInstanceCacheTest`:
- `put_thenGet_returnsSameData` — round-trip UUID, state, version, context snapshot, metadata coordinates
- `put_existingEntry_updates` — overwrite works
- `get_nonExistent_returnsNull` — returns null for unknown UUID
- `getAll_returnsAllEntries` — multiple entries returned
- `clear_removesAll` — empty after clear
- `reconstruction_setsMetadataFromRegistry` — CaseMetaModel resolved from stored coordinates

Test uses an in-memory SQLite database (`:memory:` path) — no Docker needed.

## Limitations (v1)

1. Transient state (`pendingActionGate`, `waitingForWorkId`, `propagationContext`) is lost on restart — acceptable for simple zero-Java apps
2. No query support beyond get-by-UUID and getAll — the `CaseInstanceCache` SPI doesn't require it
3. No migration path from in-memory to JSON or vice versa

## References

- `common-core/.../CaseInstanceCache.java` — 4-method SPI contract
- `common-core/.../CaseInstance.java` — domain object with contextSnapshot field
- `runtime-core/.../CaseInstanceCacheImpl.java` — in-memory reference implementation
- `persistence-memory/pom.xml` — module structure reference
- `pages/backend/data-json/.../JsonCollectionStore.java` — backing store API
- `pages/backend/data/.../DataContext.java` — tenant context record
