# Multi-Tenancy Foundation — Design Spec

**Issue:** casehubio/engine#299  
**Branch:** `issue-299-multi-tenancy-foundation`  
**Date:** 2026-05-31  
**Protocols:** PP-20260520-439daf (unconditional filtering), PP-20260520-e6a5f0 (repository-layer binding)

---

## Context

The platform's `CurrentPrincipal` SPI (shipped in platform#17) exposes `tenancyId()` and `isCrossTenantAdmin()`. This spec implements the consumer side for `casehub-engine`.

**Design decision: tenancyId as explicit parameter on every SPI method.**

Repositories do not inject `CurrentPrincipal`. Instead, every SPI method takes `String tenancyId` as an explicit parameter. The caller is responsible for supplying it:

- **HTTP boundary** (`CaseHubReactor`, REST endpoints): `currentPrincipal.tenancyId()` — the one place where CDI request scope is reliably active
- **Any code that has a `CaseInstance`**: `instance.tenancyId` — domain objects carry it as plain data
- **`@ObservesAsync` observers**: `event.tenancyId()` — `CaseLifecycleEvent` carries it
- **Vert.x `@ConsumeEvent` handlers**: `event.caseInstance().tenancyId` or the Vert.x event payload equivalent
- **Recovery at startup**: cross-tenant interfaces (no tenancyId needed — see Section 2)

This approach is immune to CDI scope issues (no `@RequestScoped` CurrentPrincipal in repositories, no ThreadLocal, no `@ActivateRequestContext`), works identically in every execution context, and makes the tenancy requirement visible at every call site.

**Why not ThreadLocal-backed CurrentPrincipal (Option A):** The engine is an embeddable library. It does not own the HTTP boundary — claudony/devtown do. Option A requires the consuming runtime to call a static setter before each request (a hidden convention, not enforced) and still breaks if the consuming runtime's `CurrentPrincipal` is CDI `@RequestScoped` (their impl overrides any `@DefaultBean` the engine provides, and async contexts fail in production). Option B is platform-agnostic.

**Domain objects carry tenancyId.** The repository sets it on the domain object at save time and maps it back from the entity on load. Downstream code reads it from the domain object — no repeat reads from any principal.

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

`tenancyId` is a `public String tenancyId` field, matching the `public Long id` pattern for the surrogate key. Set by the repository at save (`entity.tenancyId = tenancyId; instance.tenancyId = tenancyId`) and mapped back on load (`fromEntity()` maps `entity.tenancyId → domain.tenancyId`). No other layer writes it.

`PlanItemRecord` is a read model assembled from `PlanItemEntity` projections; the JPA query result maps `entity.tenancyId → record.tenancyId`.

### JPA entities — `tenancyId` column + index

Every entity gets:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

`length = 64`: platform tenant IDs are UUIDs (36 chars; `DEFAULT_TENANT_ID = "278776f9-e1b0-46fb-9032-8bddebdcf9ce"`). 64 provides headroom for future alternative formats without a schema migration.

Each entity also gets a `@Index` entry following the existing convention (`idx_<table>_tenancy_id`, consistent with `idx_plan_item_case_id` etc. in `PlanItemEntity`).

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

### SPI method signatures — explicit `tenancyId` parameter

Every SPI method that reads or writes tenant-scoped data gains `String tenancyId` as an explicit parameter. No repository implementation injects `CurrentPrincipal`.

```java
// CaseInstanceRepository
Uni<CaseInstance> save(CaseInstance instance, String tenancyId);
Uni<CaseInstance> update(CaseInstance instance, String tenancyId);
Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId);
Uni<Void> updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId);

// CaseMetaModelRepository
Uni<CaseMetaModel> save(CaseMetaModel model, String tenancyId);
Uni<CaseMetaModel> findByKey(String namespace, String name, String version, String tenancyId);

// EventLogRepository (tenant-scoped methods)
Uni<Void> append(EventLog eventLog, String tenancyId);
Uni<Long> appendAndReturnId(EventLog eventLog, String tenancyId);
Uni<EventLog> findById(Long id, String tenancyId);
Uni<List<EventLog>> findSchedulingEvents(UUID caseId, String workerId, Instant after, String tenancyId);
Uni<List<EventLog>> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types, String tenancyId);
Uni<List<EventLog>> findByCaseAndWorkerAndType(UUID caseId, String workerId, CaseHubEventType type, String tenancyId);
Uni<List<EventLog>> findByWorkerAndType(String workerId, CaseHubEventType type, String tenancyId);
Uni<List<EventLog>> findByCaseWithFilters(UUID caseId, Collection<CaseHubEventType> eventTypes, Collection<EventStreamType> streamTypes, String tenancyId);

// SubCaseGroupRepository
Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId, String tenancyId);
Uni<SubCaseGroup> incrementRejected(UUID parentCaseId, String groupId, String tenancyId);
// ... all methods

// PlanItemStore / ReactivePlanItemStore
Uni<PlanItemRecord> save(PlanItemSaveRequest request, String tenancyId);
List<PlanItemRecord> findDelegated(UUID caseId, String tenancyId);
// ... all methods
```

### JPA repository implementation pattern

```java
// No @Inject CurrentPrincipal — tenancyId arrives as explicit parameter

@Override
public Uni<CaseInstance> save(CaseInstance instance, String tenancyId) {
    return withSafeContext(() -> Panache.withTransaction(() ->
        Panache.getSession().chain(session -> {
            CaseInstanceEntity entity = new CaseInstanceEntity();
            entity.tenancyId = tenancyId;         // set from parameter
            entity.uuid = instance.getUuid();
            // ... other fields
            return entity.persist().map(v -> {
                instance.id = entity.id;
                instance.tenancyId = tenancyId;   // populate domain object
                return instance;
            });
        })
    ));
}

@Override
public Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
    return withSafeContext(() -> Panache.withSession(() ->
        CaseInstanceEntity.<CaseInstanceEntity>find(
            "from CaseInstanceEntity ci join fetch ci.caseMetaModel " +
            "where ci.uuid = ?1 and ci.tenancyId = ?2", uuid, tenancyId)
            .firstResult())
        .map(e -> e == null ? null : fromEntity(e)));
}

@Override
public Uni<CaseInstance> update(CaseInstance instance, String tenancyId) {
    // tenancyId in WHERE — guards against cross-tenant writes via guessable surrogate id
    return withSafeContext(() -> Panache.withTransaction(() ->
        CaseInstanceEntity.<CaseInstanceEntity>find(
            "id = ?1 and tenancyId = ?2", instance.id, tenancyId)
            .firstResult()
            .invoke(entity -> {
                entity.state = instance.getState();
                entity.parentCaseId = instance.getParentCaseId();
                entity.parentPlanItemId = instance.getParentPlanItemId();
                entity.waitingForWorkId = instance.getWaitingForWorkId();
                // tenancyId NOT updated — immutable
            })
            .replaceWith(instance)));
}

// CaseMetaModel: findByKey pattern
@Override
public Uni<CaseMetaModel> findByKey(String namespace, String name, String version, String tenancyId) {
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
    // ... other fields
    return instance;
}
```

`update()` includes `tenancyId` in the WHERE clause on all write paths. `tenancyId` is never written on update — it is immutable once set.

### Cross-tenant split — internal package

Cross-tenant interfaces live in `io.casehub.engine.internal.recovery.spi` (not the public `casehub-engine-common/spi/`). Package placement is the first line of defence against accidental injection outside recovery services.

**`CrossTenantEventLogRepository`** (internal package):
- `findByTypes(Collection<CaseHubEventType>)` — recovery: all tenant WORKER_SCHEDULED events at startup
- `findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType>)` — recovery: rebuild case state context (caseId is known; no tenant filter needed since recovery runs before any principal context exists)
- `findSubmittedWorkWithoutCompletion()` — recovery
- `findByWorkerAndTypeAcrossTenants(String workerId, CaseHubEventType type)` — recovery-only cross-tenant variant

`findByWorkerAndType(String, CaseHubEventType, String tenancyId)` **stays in `EventLogRepository`** (tenant-scoped): `SubCaseCompletionService` calls it in a same-tenant context (querying for a child case's `SUBCASE_STARTED` event, which lives on the parent — always the same tenant).

**`CrossTenantCaseInstanceRepository`** (internal package):
- `findByUuid(UUID caseId)` — recovery: load case instance without tenant filter

`DefaultWorkerExecutionRecoveryService` injects `CrossTenantEventLogRepository` and `CrossTenantCaseInstanceRepository` instead of the tenanted interfaces.

`JpaCrosstenantEventLogRepository` and `JpaCrosstenantCaseInstanceRepository` live in `persistence-hibernate`. `InMemoryEventLogRepository` implements both `EventLogRepository` and `CrossTenantEventLogRepository` (cross-tenant methods return unfiltered data — correct for single-tenant test contexts).

### Memory stores

`persistence-memory` adds `casehub-platform-api` as a compile dependency (for `TenancyConstants`, not `CurrentPrincipal`). Every in-memory store filters its internal map by the explicit `tenancyId` parameter.

`persistence-memory` ships a `@DefaultBean @ApplicationScoped DefaultTestPrincipal` (main sources) so any consumer's test classpath has a working `CurrentPrincipal` at the HTTP boundary without extra wiring. Add `@SuppressWarnings("deprecation")` and a Javadoc warning: *"For testing only. If persistence-memory is on the compile classpath in production, all operations silently use DEFAULT_TENANT_ID."*

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

### Tenancy propagation — no CDI magic

There is no `MutableTenantContext`, no `@ActivateRequestContext`, no `ThreadLocal`. Tenancy flows as plain data:

**HTTP-triggered paths** (case start, signal, REST): `currentPrincipal.tenancyId()` is read once at the entry point and passed to all repository calls. The loaded `CaseInstance` carries `tenancyId` for all subsequent calls.

**`@ObservesAsync` observers** (`SubCaseCompletionListener`): `event.tenancyId()` supplies the value. No context activation needed:

```java
@ApplicationScoped
public class SubCaseCompletionListener {
    @Inject SubCaseCompletionService subCaseCompletionService;

    public void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        // tenancyId flows from event — no CDI scope activation
        subCaseCompletionService.handleCompletion(event);
    }
}
```

`SubCaseCompletionService.handleCompletion(CaseLifecycleEvent event)` passes `event.tenancyId()` to every repository call: `findByWorkerAndType(childCaseId.toString(), SUBCASE_STARTED, event.tenancyId())`, `append(log, event.tenancyId())`, etc.

**`@ConsumeEvent` handlers**: the event payload carries `CaseInstance` (which has `tenancyId`) or another domain object. The handler extracts and passes it.

**`PlanExecutionContext`**: add `String tenancyId()` sourced from the loaded `CaseInstance.tenancyId`. `PlanningStrategyLoopControl.select()` uses `ctx.tenancyId()` for registry calls.

### Subcase tenancyId inheritance — enforced invariant

Subcases always inherit `tenancyId` from the parent case, never from `currentPrincipal`. `SubCaseExecutionHandler` must propagate `parentInstance.tenancyId` when creating the child `CaseInstance`:

```java
// SubCaseExecutionHandler — when spawning a child case
CaseInstance child = new CaseInstance();
child.tenancyId = parentInstance.tenancyId;  // inherit, not from principal
// ...
caseInstanceRepository.save(child, child.tenancyId);
```

This invariant makes `findByWorkerAndType(childCaseId.toString(), SUBCASE_STARTED, tenancyId)` correct: the `SUBCASE_STARTED` event is on the parent case (same tenancyId as the child). Violating this invariant would cause the lookup to silently return null.

### BlackboardRegistry — stored tenancyId + O(1) evict

**Map key stays `UUID caseId`** — UUIDs are globally unique; composite keys are unnecessary for correctness. `CaseEntry` stores `tenancyId` at creation for defense-in-depth in `get()` and robustness of `evict()`.

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

private final ConcurrentHashMap<UUID, CaseEntry> entries = new ConcurrentHashMap<>();
```

`getOrCreate(UUID caseId, String tenancyId)` — explicit, stores tenancyId in `CaseEntry`.

`get(UUID caseId, String tenancyId)` — explicit. Defense-in-depth: if the stored `entry.tenancyId` does not match the caller's `tenancyId`, log a warning and return `Optional.empty()`. Lazy hydration calls `planItemStore.findDelegated(caseId, tenancyId)` — explicit parameter, no CDI scope needed.

`evict(UUID caseId)` — O(1) `entries.remove(caseId)`. UUID uniqueness guarantees no cross-tenant collision. Does not read `currentPrincipal` at all.

`PlanningStrategyLoopControl` passes `ctx.tenancyId()` to `getOrCreate(caseId, tenancyId)`.

`PlanItemCompletionHandler` (and other completion handlers) pass the tenancyId from their event to `get(caseId, tenancyId)`.

**No `@Inject CurrentPrincipal` in `BlackboardRegistry`.**

### `CaseLifecycleEvent` — add `tenancyId`

```java
public record CaseLifecycleEvent(
    UUID caseId,
    String tenancyId,   // NEW — sourced from CaseInstance.tenancyId at fire sites
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId) {}
```

Breaking change for all observers. Issues filed: claudony#143, devtown#61, aml#47, clinical#51.

All internal Vert.x bus messages (`WorkerScheduleEvent` etc.) carry `CaseInstance` — tenancyId is already present via the domain object. No structural changes needed.

### `update()` WHERE clause — tenancyId in all write queries

All `update()` operations filter by `tenancyId` in the WHERE clause, not just read queries. Surrogate IDs are sequential and guessable; without this filter, a bug obtaining a different tenant's surrogate id could mutate that tenant's record. Every `UPDATE ... WHERE id = ? AND tenancy_id = ?`.

---

## Section 4: Module Wiring

| Module | Change |
|---|---|
| `persistence-hibernate` | Add `casehub-platform-api` compile dep (for `TenancyConstants`) |
| `persistence-memory` | Add `casehub-platform-api` compile dep; ship `DefaultTestPrincipal` |
| `common` | No dep change; `PlanExecutionContext` gains `tenancyId()` method |
| `blackboard`, `work-adapter`, `resilience`, `runtime` | No pom changes; no CurrentPrincipal injection in repositories |

New interfaces in `io.casehub.engine.internal.recovery.spi` (internal, not exported via `casehub-engine-common/spi/`):
- `CrossTenantEventLogRepository`
- `CrossTenantCaseInstanceRepository`

**No `MutableTenantContext` class.** Removed entirely.

`PlanExecutionContext` gains `String tenancyId()` — implement in the concrete class; source from the loaded `CaseInstance.tenancyId` passed into `PlanExecutionContext` at construction.

---

## Section 5: Tests

### Single-tenant pass-through

Existing `@QuarkusTest` suites change mechanically: every repository call site adds a `tenancyId` argument. The value is `TenancyConstants.DEFAULT_TENANT_ID` everywhere (sourced from `DefaultTestPrincipal.tenancyId()` at the HTTP boundary, which propagates through). No logic changes — compiler flags every call site.

### Multi-tenant isolation contract test

One abstract contract test per repository. `MutableTestPrincipal` is not needed — tests pass tenancyId strings directly:

```java
@Test
void tenantIsolation() {
    String tenantA = "tenant-a";
    String tenantB = "tenant-b";

    // Save as tenant A
    CaseInstance a = buildInstance();
    repository.save(a, tenantA).await().atMost(Duration.ofSeconds(5));

    // Query as tenant B — must not find it
    CaseInstance found = repository.findByUuid(a.getUuid(), tenantB).await().atMost(Duration.ofSeconds(5));
    assertThat(found).isNull();

    // Query as tenant A — must find it
    found = repository.findByUuid(a.getUuid(), tenantA).await().atMost(Duration.ofSeconds(5));
    assertThat(found).isNotNull();
}
```

No CDI mock beans. No ThreadLocal setup. Just strings.

### `BlackboardRegistry` isolation unit test

`getOrCreate(caseId, "tenant-a")` then `get(caseId, "tenant-b")` returns `Optional.empty()` — tenant mismatch defense-in-depth. `get(caseId, "tenant-a")` returns the plan model. `evict(caseId)` removes it; subsequent `get(caseId, "tenant-a")` returns empty.

### `SubCaseExecutionHandler` invariant test

Creates a parent case with `tenancyId = "tenant-a"`. Spawns a subcase. Asserts child `CaseInstance.tenancyId == "tenant-a"` — inheritance enforced, not read from principal.

### `CaseLifecycleEvent` fire sites

Record gains a new component — compiler enforces exhaustive construction at all fire sites.

---

## Architectural Note — CaseMetaModel per-tenant

Case definitions are per-tenant: each tenant owns its own `(namespace, name, version)` namespace. The unique constraint includes `tenancyId`. ADR to file: this forecloses a shared "template catalog" model unless a sentinel `tenancyId` (e.g. `PLATFORM_TENANT_ID`) is introduced for global definitions. This option should be noted in the ADR before it is implemented — the constraint shape `(tenancy_id, namespace, name, version)` already supports it.

---

## Deferred Issues

| Issue | Description |
|---|---|
| engine#405 | `@CrossTenant` CDI producer pattern for recovery services |
| engine#406 | DB-level RLS after application-level filtering is stable |
| engine#407 | `WorkerDecisionEvent` tenancyId audit |
| claudony#143, devtown#61, aml#47, clinical#51 | `CaseLifecycleEvent` observer updates in consuming repos |
| ADR (to file) | CaseMetaModel per-tenant vs global (with sentinel tenancyId consideration) |
