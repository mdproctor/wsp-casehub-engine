# Multi-Tenancy Foundation — Design Spec

**Issue:** casehubio/engine#299  
**Branch:** `issue-299-multi-tenancy-foundation`  
**Date:** 2026-05-31  
**Protocols:** PP-20260520-439daf (unconditional filtering), PP-20260520-e6a5f0 (repository-layer binding)

---

## Context

The platform's `CurrentPrincipal` SPI (shipped in platform#17) exposes `tenancyId()` and `isCrossTenantAdmin()`. This spec implements the consumer side for `casehub-engine`: every entity gets a `tenancyId` column, every JPA repository filters by it unconditionally, and every domain object carries it as plain data so downstream code — event construction, audit logging, registry keying — can read it without re-injecting `CurrentPrincipal`.

**Design decision: domain objects carry tenancyId.** The repository is the single write site (reads from `CurrentPrincipal`, sets on entity and domain object). All downstream code reads from the domain object. This avoids spreading `CurrentPrincipal` injection into event constructors and Vert.x handlers.

**Implicit vs explicit tradeoff.** The implicit approach (inject `CurrentPrincipal`, filter silently) is ergonomic for callers — they never think about tenancy. It works correctly for HTTP-triggered synchronous paths where CDI request scope is always active. For async contexts (`@ObservesAsync`, recovery startup), CDI request scope is NOT active and `currentPrincipal.tenancyId()` throws `ContextNotActiveException`. The resolution for async sites is documented in Section 3.

**No Flyway.** Schema is `drop-and-create` throughout this codebase.

---

## Section 1: Data Model

### Domain objects — `public String tenancyId` added to each

| Class | Module |
|---|---|
| `CaseInstance` | `casehub-engine-common` |
| `CaseMetaModel` | `casehub-engine-common` |
| `EventLog` | `casehub-engine-common` |
| `PlanItemRecord` | `casehub-engine-common` |

`tenancyId` is a `public String tenancyId` field, matching the `public Long id` pattern already used for the surrogate key. This makes it directly assignable by repository code without a setter and signals it is infrastructure metadata, not business logic. It is populated by the repository on save (`instance.tenancyId = tenancyId`) and on load (`fromEntity()` maps `entity.tenancyId → domain.tenancyId`). No other layer writes it.

### JPA entities — `tenancyId` column + index

Every entity gets:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

`length = 64`: platform tenant IDs are UUIDs (36 chars; confirmed: `DEFAULT_TENANT_ID = "278776f9-e1b0-46fb-9032-8bddebdcf9ce"`). 64 provides headroom for future alternative formats without a schema migration.

Each entity also gets a `@Index` entry: `@Index(name = "idx_<table>_tenancy_id", columnList = "tenancy_id")`. The naming follows the existing convention in `PlanItemEntity` (`idx_plan_item_case_id`, `idx_plan_item_plan_item_id`).

Affected entities:
1. `CaseInstanceEntity` — `case_instance`
2. `CaseMetaModelEntity` — `case_meta_model`; unique constraint changes from `(namespace, name, version)` to `(tenancy_id, namespace, name, version)` — each tenant owns its own definition namespace
3. `EventLogEntity` — `event_log`
4. `PlanItemEntity` — `plan_item`
5. `SubCaseGroupEntity` — `sub_case_group`
6. `WorkAdapterPlanItemEntity` — `work_adapter_plan_item`
7. `CaseLedgerEntry` — `case_ledger_entry` (ledger module)
8. `WorkerDecisionEntry` — `worker_decision` (ledger module)

---

## Section 2: Repository Layer

### JPA repositories — pattern

`persistence-hibernate` adds `casehub-platform-api` as a compile dependency. `TenancyConstants` lives in `casehub-platform-api` (alongside `CurrentPrincipal`); `DefaultTestPrincipal` references it via this dep.

**Capture tenancyId before any `Uni` pipeline** — CDI `@RequestScoped` context is not available on Vert.x IO threads. Every repository method reads `currentPrincipal.tenancyId()` synchronously before entering any reactive chain:

```java
@Inject CurrentPrincipal currentPrincipal;

@Override
public Uni<CaseInstance> save(CaseInstance instance) {
    final String tenancyId = currentPrincipal.tenancyId(); // BEFORE Uni
    return withSafeContext(() -> Panache.withTransaction(() ->
        Panache.getSession().chain(session -> {
            CaseInstanceEntity entity = new CaseInstanceEntity();
            entity.tenancyId = tenancyId;
            entity.uuid = instance.getUuid();
            // ...
            return entity.persist().map(v -> {
                instance.id = entity.id;
                instance.tenancyId = tenancyId; // populate domain object
                return instance;
            });
        })
    ));
}

@Override
public Uni<CaseInstance> findByUuid(UUID uuid) {
    final String tenancyId = currentPrincipal.tenancyId();
    return withSafeContext(() -> Panache.withSession(() ->
        CaseInstanceEntity.<CaseInstanceEntity>find(
            "from CaseInstanceEntity ci join fetch ci.caseMetaModel " +
            "where ci.uuid = ?1 and ci.tenancyId = ?2", uuid, tenancyId)
            .firstResult())
        .map(e -> e == null ? null : fromEntity(e)));
}

// CaseMetaModel: findByKey pattern
@Override
public Uni<CaseMetaModel> findByKey(String namespace, String name, String version) {
    final String tenancyId = currentPrincipal.tenancyId();
    return withSafeContext(() -> Panache.withSession(() ->
        CaseMetaModelEntity.<CaseMetaModelEntity>find(
            "namespace = ?1 and name = ?2 and version = ?3 and tenancyId = ?4",
            namespace, name, version, tenancyId)
            .firstResult())
        .map(e -> e == null ? null : fromEntity(e)));
}

// fromEntity: always map tenancyId
private CaseInstance fromEntity(CaseInstanceEntity entity) {
    CaseInstance instance = new CaseInstance();
    instance.tenancyId = entity.tenancyId;
    instance.id = entity.id;
    // ...
    return instance;
}
```

`update()` methods do not touch `tenancyId` — it is immutable once set.

Repositories receiving this treatment:
- `JpaCaseInstanceRepository`
- `JpaCaseMetaModelRepository`
- `JpaEventLogRepository` (tenant-scoped methods)
- `JpaSubCaseGroupRepository`
- `JpaPlanItemStore`
- `JpaReactivePlanItemStore`
- `JpaPlanItemStore` in `work-adapter`

### Cross-tenant split

**Methods that cross tenant boundaries** — these currently exist in `EventLogRepository` with no tenant filter and are called by recovery services or from async observers dealing with events from any tenant:

`findByTypes(Collection<CaseHubEventType>)` — called by `DefaultWorkerExecutionRecoveryService.recoverPendingScheduledWorkers()` to scan all WORKER_SCHEDULED events at startup. Genuinely cross-tenant.

`findSubmittedWorkWithoutCompletion()` — recovery use, cross-tenant.

`findByWorkerAndTypeAcrossTenants(String workerId, CaseHubEventType type)` — **new method, recovery-only**. `findByWorkerAndType` stays in `EventLogRepository` **with a tenant filter** because `SubCaseCompletionService` calls it in a same-tenant context (querying by child case UUID for `SUBCASE_STARTED` events belonging to the same parent case's tenant). The recovery-only cross-tenant variant is a separate method.

These three methods move to / are added to `CrossTenantEventLogRepository` (new interface in `casehub-engine-common/spi/`). `JpaCrosstenantEventLogRepository` in `persistence-hibernate` implements it with no tenancy filter. `InMemoryEventLogRepository` implements both `EventLogRepository` and `CrossTenantEventLogRepository` — the cross-tenant methods return unfiltered in-memory data (correct for single-tenant test contexts).

**Recovery also calls `CaseInstanceRepository`** — `loadOrRestoreCaseInstance` calls `caseInstanceRepository.findByUuid(caseId)` and `rebuildStateContext` calls `eventLogRepository.findByCaseAndTypes(caseId, types)`. At startup recovery time there is no active principal. Resolution: add `CrossTenantCaseInstanceRepository` (new SPI in `casehub-engine-common/spi/`) with `findByUuid(UUID)`, and add `findByCaseAndTypes(UUID, Collection)` to `CrossTenantEventLogRepository`. `DefaultWorkerExecutionRecoveryService` injects both cross-tenant interfaces instead of the tenanted ones.

The `@CrossTenant` CDI qualifier and `isCrossTenantAdmin()` producer are deferred to engine#405, pending a system-actor principal from the platform.

### Memory stores

`persistence-memory` adds `casehub-platform-api` as a compile dependency. Every in-memory store injects `CurrentPrincipal` and filters its internal map by `tenancyId`.

`persistence-memory` ships a `@DefaultBean @ApplicationScoped DefaultTestPrincipal` (main sources) so any consumer's test classpath gets a working principal without extra wiring:

```java
@DefaultBean
@ApplicationScoped
public class DefaultTestPrincipal implements CurrentPrincipal {
    public String tenancyId() { return TenancyConstants.DEFAULT_TENANT_ID; }
    public boolean isCrossTenantAdmin() { return false; }
    public String actorId() { return "system"; }
    public Set<String> groups() { return Set.of(); }
}
```

---

## Section 3: In-Process Isolation

### Async CDI observer context — `@ObservesAsync` and `@RequestScoped`

CDI `@ObservesAsync` observers run on a separate CDI executor thread pool. Quarkus does **not** activate CDI request scope for these threads. Any call to `@RequestScoped CurrentPrincipal` in an `@ObservesAsync` observer throws `ContextNotActiveException` at runtime.

`SubCaseCompletionListener.onCaseLifecycle(@ObservesAsync CaseLifecycleEvent)` is the affected site. Resolution:

```java
@ApplicationScoped
public class SubCaseCompletionListener {
    @Inject SubCaseCompletionService subCaseCompletionService;
    @Inject MutableTenantContext tenantContext;  // NEW — engine-internal @RequestScoped bean

    @ActivateRequestContext  // NEW — activates a fresh CDI request scope for this observer
    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        tenantContext.set(event.tenancyId());   // populate from event before delegate
        subCaseCompletionService.handleCompletion(event);
    }
}
```

`MutableTenantContext` is an engine-internal `@RequestScoped` bean (in `casehub-engine-common` or `runtime`) that implements `CurrentPrincipal`, stores the tenant ID in a plain field, and is populated by the observer before delegating. `@ActivateRequestContext` (Quarkus CDI annotation) starts a fresh request scope for the duration of the observer method — `MutableTenantContext` is active within it.

This pattern is self-contained (no platform changes required) and works for any future `@ObservesAsync` site that calls tenanted repositories. Any new `@ObservesAsync` observer that calls a tenanted repository must follow the same `@ActivateRequestContext` + `tenantContext.set(event.tenancyId())` pattern.

**Recovery startup** — `@Observes StartupEvent` has no CDI request scope either. Recovery services inject `CrossTenantCaseInstanceRepository` and `CrossTenantEventLogRepository` (see Section 2) and never call tenanted repositories at startup. No `@ActivateRequestContext` needed there — the cross-tenant interfaces are explicitly unscoped by design.

### BlackboardRegistry — composite key with stored tenancyId

`BlackboardRegistry` changes its map from `ConcurrentHashMap<UUID, CaseEntry>` to `ConcurrentHashMap<String, CaseEntry>`.

`CaseEntry` stores tenancyId at construction time, making eviction robust regardless of what principal is active when a case reaches terminal state:

```java
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
```

`getOrCreate(caseId)` reads `currentPrincipal.tenancyId()` synchronously (called from HTTP-triggered `PlanningStrategyLoopControl.select()` — request scope active), stores it in `CaseEntry`, and uses `entry.tenancyId + ':' + caseId` as the map key.

`evict(caseId)` cannot use `currentPrincipal` — it fires from terminal-state handlers that may not have request scope. Instead: look up the entry, read its stored `tenancyId`, and remove by the stored composite key. If the entry is not found, eviction is a no-op (already evicted).

```java
public void evict(UUID caseId) {
    // Find any entry for this caseId across all tenant prefixes, using stored tenancyId
    entries.values().stream()
        .filter(e -> e.planModel.getCaseId().equals(caseId))
        .findFirst()
        .ifPresent(e -> entries.remove(e.tenancyId + ':' + caseId));
}
```

Alternative: entries carry their own composite key. Either approach is correct; the linear scan in `evict()` is negligible (cases don't accumulate in production — they are evicted on terminal state).

`BlackboardRegistry` injects `CurrentPrincipal` for `getOrCreate` only. All other methods use stored tenancyId.

### `CaseLifecycleEvent` — add `tenancyId`

```java
public record CaseLifecycleEvent(
    UUID caseId,
    String tenancyId,   // NEW
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId) {}
```

`tenancyId` is sourced from `caseInstance.tenancyId` at every fire site. Breaking change for all observers. Issues filed: claudony#143, devtown#61, aml#47, clinical#51.

All internal Vert.x bus messages (`WorkerScheduleEvent`, `WorkerRetriesExhaustedEvent`, etc.) carry `CaseInstance` — they already carry tenancyId via the domain object. No additional changes needed.

---

## Section 4: Module Wiring

| Module | Change |
|---|---|
| `persistence-hibernate` | Add `casehub-platform-api` compile dep |
| `persistence-memory` | Add `casehub-platform-api` compile dep; ship `DefaultTestPrincipal` |
| `common` | No dep change — domain objects carry `String tenancyId` as plain data; `MutableTenantContext` lives here |
| `blackboard`, `work-adapter`, `resilience`, `runtime` | Already depend on `casehub-platform`; no pom changes |

New SPI interfaces in `casehub-engine-common/spi/`:
- `CrossTenantEventLogRepository` — `findByTypes`, `findByCaseAndTypes`, `findSubmittedWorkWithoutCompletion`, `findByWorkerAndTypeAcrossTenants`
- `CrossTenantCaseInstanceRepository` — `findByUuid(UUID)`

New engine-internal class:
- `MutableTenantContext` — `@RequestScoped`, implements `CurrentPrincipal`, plain field for tenancyId, used only by `@ObservesAsync` observers via `@ActivateRequestContext`

---

## Section 5: Tests

### Single-tenant pass-through

Existing `@QuarkusTest` suites require no test-logic changes. `DefaultTestPrincipal` is `@DefaultBean` — activates automatically wherever no real principal is wired. All existing tests run against `DEFAULT_TENANT_ID` transparently.

### Multi-tenant isolation contract test

One new abstract contract test per repository following the `PlanItemStoreContractTest` pattern. `MutableTestPrincipal` ships in `persistence-memory` test sources:

```java
// @ApplicationScoped with ThreadLocal backing — safe for parallel test execution
@ApplicationScoped
@Alternative
@Priority(1)
public class MutableTestPrincipal implements CurrentPrincipal {
    private static final ThreadLocal<String> TENANT = new ThreadLocal<>();

    public static void set(String tenancyId) { TENANT.set(tenancyId); }
    public static void reset() { TENANT.remove(); }

    public String tenancyId() {
        String t = TENANT.get();
        return t != null ? t : TenancyConstants.DEFAULT_TENANT_ID;
    }
    public boolean isCrossTenantAdmin() { return false; }
    // ...
}
```

Each contract test:
1. `MutableTestPrincipal.set("tenant-a")` → save entity → verify found
2. `MutableTestPrincipal.set("tenant-b")` → verify not found
3. `MutableTestPrincipal.set("tenant-a")` → verify found again
4. `MutableTestPrincipal.reset()` in `@AfterEach`

### `SubCaseCompletionListener` — `@ActivateRequestContext` test

`@ObservesAsync` is unreliable in `@QuarkusTest` (per CLAUDE.md). Test `SubCaseCompletionService.handleCompletion()` directly, injecting `MutableTenantContext` and setting the tenant before calling — confirms the service correctly reads tenant context via `MutableTenantContext.get()`.

### BlackboardRegistry isolation unit test

Two plan models registered with the same UUID under different tenant prefixes; `get()` from each tenant context returns only that tenant's plan model. Verifies composite key isolation.

### `CaseLifecycleEvent` fire sites

Record gains a new component — compiler enforces exhaustive construction at all fire sites.

---

## Architectural Note — CaseMetaModel per-tenant

Case definitions are per-tenant: the same `(namespace, name, version)` triple can exist independently for different tenants, who own and customise their own process types. This makes `CaseMetaModelEntity` tenant-scoped (unique constraint includes `tenancyId`). ADR to be filed documenting this decision and the alternative (global/shared definitions).

---

## Deferred Issues

| Issue | Description |
|---|---|
| engine#405 | `@CrossTenant` CDI producer pattern for recovery services |
| engine#406 | DB-level RLS after application-level filtering is stable |
| engine#407 | `WorkerDecisionEvent` tenancyId audit |
| claudony#143, devtown#61, aml#47, clinical#51 | `CaseLifecycleEvent` observer updates in consuming repos |
| ADR (to file) | CaseMetaModel per-tenant vs global decision |
