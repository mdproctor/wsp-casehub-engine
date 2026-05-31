# Multi-Tenancy Foundation — Design Spec

**Issue:** casehubio/engine#299  
**Branch:** `issue-299-multi-tenancy-foundation`  
**Date:** 2026-05-31  
**Protocols:** PP-20260520-439daf (unconditional filtering), PP-20260520-e6a5f0 (repository-layer binding)

---

## Context

The platform's `CurrentPrincipal` SPI (shipped in platform#17) now exposes `tenancyId()` and `isCrossTenantAdmin()`. This spec implements the consumer side for `casehub-engine`: every entity gets a `tenancyId` column, every JPA repository filters by it unconditionally, and every domain object carries it as plain data so downstream code — event construction, audit logging, registry keying — can read it without re-injecting `CurrentPrincipal`.

**Design decision:** `tenancyId` is a first-class field on domain objects, not JPA-layer-only. The repository is the single write site (reads from `CurrentPrincipal`, sets on entity and domain object). All downstream code reads from the domain object. This avoids spreading `CurrentPrincipal` injection into event construction and Vert.x handlers.

**No Flyway.** Schema is `drop-and-create` throughout this codebase.

---

## Section 1: Data Model

### Domain objects — `String tenancyId` added to each

| Class | Module |
|---|---|
| `CaseInstance` | `casehub-engine-common` |
| `CaseMetaModel` | `casehub-engine-common` |
| `EventLog` | `casehub-engine-common` |
| `PlanItemRecord` | `casehub-engine-common` |

`tenancyId` is a `public String tenancyId` field on each domain object, matching the `public Long id` pattern already used for the surrogate key. This makes it directly assignable by repository code without a setter, signals that it is infrastructure metadata (not business logic), and is consistent with how the persistence layer already treats `id`. No setter method is added. It is populated by the repository on save (`instance.tenancyId = tenancyId`) and on load (`fromEntity()` maps `entity.tenancyId → domain.tenancyId`). No other layer writes it.

### JPA entities — `tenancyId` column + index

Every entity gets:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

And a corresponding `@Index(name = "idx_<table>_tenancy_id", columnList = "tenancy_id")` on the table annotation.

Affected entities:
1. `CaseInstanceEntity` — `case_instance`
2. `CaseMetaModelEntity` — `case_meta_model`
3. `EventLogEntity` — `event_log`
4. `PlanItemEntity` — `plan_item`
5. `SubCaseGroupEntity` — `sub_case_group`
6. `WorkAdapterPlanItemEntity` — `work_adapter_plan_item`
7. `CaseLedgerEntry` — `case_ledger_entry` (ledger module)
8. `WorkerDecisionEntry` — `worker_decision` (ledger module)

---

## Section 2: Repository Layer

### JPA repositories — pattern

`persistence-hibernate` adds `casehub-platform-api` as a compile dependency.

Every JPA repository implementation follows this exact pattern:

```java
@Inject CurrentPrincipal currentPrincipal;

// save: capture tenancyId before Uni pipeline (RequestScoped not safe on IO thread)
@Override
public Uni<CaseInstance> save(CaseInstance instance) {
    final String tenancyId = currentPrincipal.tenancyId(); // MUST be before Uni
    return withSafeContext(() -> Panache.withTransaction(() ->
        Panache.getSession().chain(session -> {
            CaseInstanceEntity entity = new CaseInstanceEntity();
            entity.tenancyId = tenancyId;         // set on entity
            entity.uuid = instance.getUuid();
            // ... other fields
            return entity.persist().map(v -> {
                instance.id = entity.id;
                instance.tenancyId = tenancyId;   // populate back to domain object
                return instance;
            });
        })
    ));
}

// find: capture tenancyId before Uni, add AND tenancyId = ?N to every query
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

// fromEntity: always map tenancyId
private CaseInstance fromEntity(CaseInstanceEntity entity) {
    CaseInstance instance = new CaseInstance();
    instance.tenancyId = entity.tenancyId;        // always populate
    instance.id = entity.id;
    // ... other fields
    return instance;
}
```

**update() methods:** `tenancyId` is immutable — never update it. The `update()` methods do not touch the column.

Repositories receiving this treatment:
- `JpaCaseInstanceRepository`
- `JpaCaseMetaModelRepository`
- `JpaEventLogRepository` (tenant-scoped methods only — see cross-tenant split below)
- `JpaSubCaseGroupRepository`
- `JpaPlanItemStore`
- `JpaReactivePlanItemStore`
- `JpaPlanItemStore` in `work-adapter`

### Cross-tenant split

Three methods currently in `JpaEventLogRepository` return data with no tenancy filter (called only by recovery services):
- `findByTypes(Collection<CaseHubEventType>)`
- `findByWorkerAndType(String, CaseHubEventType)`
- `findSubmittedWorkWithoutCompletion()`

These move to a new SPI interface `CrossTenantEventLogRepository` in `casehub-engine-common/spi/`. The JPA implementation `JpaCrosstenantEventLogRepository` lives in `persistence-hibernate` with **no** tenancy filter — cross-tenant by design. These methods are removed from `EventLogRepository` and its in-memory implementation.

Recovery services (`DefaultWorkerExecutionRecoveryService`, `DeadLetterReplayService`) switch their injection point from `EventLogRepository` to `CrossTenantEventLogRepository`.

**In-memory implementation:** `InMemoryEventLogRepository` implements both `EventLogRepository` and `CrossTenantEventLogRepository`. The cross-tenant methods return unfiltered in-memory data — correct for single-tenant test contexts where all data belongs to one tenant anyway. No separate in-memory cross-tenant class is needed.

A `@CrossTenant` CDI qualifier and producer (checking `isCrossTenantAdmin()`) are deferred to engine#405, pending a system-actor principal from the platform.

### Memory stores

`persistence-memory` adds `casehub-platform-api` as a compile dependency. Every in-memory store injects `CurrentPrincipal` and filters its internal map by `tenancyId`. The `ConcurrentHashMap` key remains the same (already scoped by domain identity); filtering is applied at every read operation.

`persistence-memory` ships a `@DefaultBean @ApplicationScoped DefaultTestPrincipal` (in main sources, not test sources) so any consumer's test classpath gets a working principal without extra wiring:

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

### BlackboardRegistry — composite key

`BlackboardRegistry` changes its map from `ConcurrentHashMap<UUID, CaseEntry>` to `ConcurrentHashMap<String, CaseEntry>`.

```java
@Inject CurrentPrincipal currentPrincipal;

private String registryKey(UUID caseId) {
    return currentPrincipal.tenancyId() + ':' + caseId;
}
```

All six public methods (`getOrCreate`, `get`, `indexForCompletion`, `getPlanItemId`, `markConfigured`, `evict`) use `registryKey(caseId)` instead of raw `caseId`. Injecting `@RequestScoped CurrentPrincipal` into an `@ApplicationScoped` bean is safe in Quarkus — the CDI proxy resolves per-request.

**Why:** A cross-tenant lookup by caseId UUID produces a cache miss rather than returning another tenant's plan model. Belt-and-suspenders alongside application-level filtering.

### CaseLifecycleEvent — add `tenancyId`

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

`tenancyId` is sourced from `caseInstance.tenancyId` at every fire site. This is a **breaking change** for all `@ObservesAsync CaseLifecycleEvent` observers in other modules. Issues filed: claudony#143, devtown#61, aml#47, clinical#51.

All other internal Vert.x bus messages (`WorkerScheduleEvent`, `WorkerRetriesExhaustedEvent`, etc.) carry `CaseInstance` — they already carry tenancyId via the domain object. No additional changes needed.

---

## Section 4: Module Wiring

| Module | Change |
|---|---|
| `persistence-hibernate` | Add `casehub-platform-api` compile dep |
| `persistence-memory` | Add `casehub-platform-api` compile dep; ship `DefaultTestPrincipal` |
| `common` | No change — domain objects carry `String tenancyId` as plain data |
| `blackboard`, `work-adapter`, `resilience`, `runtime` | Already depend on `casehub-platform`; no pom changes |

---

## Section 5: Tests

### Single-tenant pass-through

Existing `@QuarkusTest` suites require no test-logic changes. `DefaultTestPrincipal` is `@DefaultBean` — it activates wherever no real principal is wired. All existing tests run against `DEFAULT_TENANT_ID` transparently.

### Multi-tenant isolation contract test

One new abstract contract test per repository, following the existing `PlanItemStoreContractTest` pattern. A `MutableTestPrincipal` (in `persistence-memory` test sources) allows per-test principal switching:

```java
public class MutableTestPrincipal implements CurrentPrincipal {
    private String tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
    public void setTenancyId(String id) { this.tenancyId = id; }
    public String tenancyId() { return tenancyId; }
    // ...
}
```

Each contract test:
1. Saves entity as tenant A → verifies found
2. Switches principal to tenant B → verifies not found
3. Switches back to tenant A → verifies found again

### BlackboardRegistry isolation unit test

One unit test: register two plan models with the same UUID but different tenant prefixes, verify `get()` from each tenant's context returns only that tenant's plan model.

### CaseLifecycleEvent fire sites

All tests constructing `CaseLifecycleEvent` get a compiler-enforced break — the record gains a new component. Fixing every construction is the migration.

---

## Deferred Issues

| Issue | Description |
|---|---|
| engine#405 | `@CrossTenant` CDI producer pattern for recovery services |
| engine#406 | DB-level RLS after application-level filtering is stable |
| engine#407 | `WorkerDecisionEvent` tenancyId audit |
| claudony#143, devtown#61, aml#47, clinical#51 | `CaseLifecycleEvent` observer updates in consuming repos |
