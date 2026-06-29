# Work-Adapter SPI Migration Design

**Issues:** engine#578, engine#579
**Date:** 2026-06-29

## Problem

The engine's `casehub-engine-work-adapter` module depends on `casehub-work` (full runtime) when it should depend on `casehub-work-api` only. This violates the platform's three-tier module convention: adapter modules should depend on API/SPI modules, not runtime implementations.

Eight production files import runtime types: `WorkItemService`, `WorkItemTemplateService`, `WorkItemStore`, `WorkItem`, `WorkItemTemplate`, `WorkItemLifecycleEvent`, and `OutcomeCodecs`. The `casehub-work-api` SPI surface (`WorkItemCreator`, `WorkItemLifecycle`, `WorkItemEvent`, `WorkItemRef`, `WorkLifecycleEvent`) already covers every operation the adapter performs.

## Scope

Two issues, one branch, two commits:

1. **#579 — Event observation migration:** Replace `event.source()` → `WorkItem` casts with `WorkItemEvent` typed accessors. Change internal method signatures from `WorkItem` to `WorkItemRef`.

2. **#578 — Dependency migration:** Replace `WorkItemService`/`WorkItemStore`/`WorkItemTemplateService` injections with `WorkItemCreator`/`WorkItemLifecycle`. Switch CDI observer from `WorkItemLifecycleEvent` (runtime) to `WorkLifecycleEvent` (api). Change pom dependency from `casehub-work` to `casehub-work-api`.

## Design

### Event Observation (#579)

`WorkItemLifecycleAdapter` currently observes `WorkItemLifecycleEvent` (runtime) and casts `event.source()` to `WorkItem` (runtime entity) to access fields like `callerRef`, `id`, `candidateGroups`, `assigneeId`, `resolution`.

`WorkItemLifecycleEvent` already implements `WorkItemEvent` (api interface), which exposes all these fields as typed accessors: `callerRef()`, `workItemId()`, `candidateGroups()`, `assigneeId()`, `resolution()`, `status()`, `outcome()`.

**Observer change:**

```java
// Before
public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
    if (!(event.source() instanceof WorkItem workItem)) return;
    CallerRef ref = CallerRef.parse(workItem.callerRef);
}

// After (commit 1: #579 — still references runtime event type)
public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
    CallerRef ref = CallerRef.parse(event.callerRef());
}

// After (commit 2: #578 — api-only types)
public void onWorkItemLifecycle(@ObservesAsync WorkLifecycleEvent event) {
    if (!(event instanceof WorkItemEvent wie)) return;
    CallerRef ref = CallerRef.parse(wie.callerRef());
}
```

CDI delivers `WorkItemLifecycleEvent` (the concrete class fired at runtime). The observer catches it as `WorkLifecycleEvent` (api abstract class). The `instanceof WorkItemEvent` check succeeds because `WorkItemLifecycleEvent implements WorkItemEvent`. All referenced types are in `casehub-work-api`.

**Method signature changes:**

| Method | Before | After |
|--------|--------|-------|
| `PlanItemCompletionApplier.apply()` | `(UUID, String, WorkItemStatus, WorkItem)` | `(UUID, String, WorkItemStatus, WorkItemRef)` |
| `ActionGateCompletionApplier.apply()` | `(GateCallerRef, WorkItemStatus, WorkItem)` | `(GateCallerRef, WorkItemStatus, WorkItemRef)` |

Fields accessed change from entity field access (`workItem.resolution`) to record accessors (`ref.resolution()`).

### Service and Dependency Migration (#578)

**Injection replacements:**

| File | Before | After |
|------|--------|-------|
| `HumanTaskScheduleHandler` | `WorkItemService`, `WorkItemTemplateService` | `WorkItemCreator` |
| `ActionGateWorkItemHandler` | `WorkItemService` | `WorkItemCreator` |
| `ActionGateCancelledHandler` | `WorkItemStore`, `WorkItemService` | `WorkItemCreator`, `WorkItemLifecycle` |
| `HumanTaskRecoveryService` | `WorkItemService` | `WorkItemCreator` |

**Operation mapping:**

| Operation | Runtime call | API call |
|-----------|-------------|----------|
| Create WorkItem (inline) | `workItemService.create(request)` | `workItemCreator.create(request)` |
| Create WorkItem (template) | `workItemTemplateService.findById()` + `instantiate()` + entity mutation + `persist()` | `workItemCreator.create(request)` with `templateId` set |
| Cancel WorkItem | `workItemService.cancel(id, actor, reason)` | `workItemLifecycle.cancel(id, actor, reason)` |
| Find by callerRef | `workItemService.findByCallerRef(ref)` / `workItemStore.findByCallerRef(ref)` | `workItemCreator.findByCallerRef(ref)` returns `Optional<WorkItemRef>` |

**Template mode rewrite:** The current handler calls `findById()`, `instantiate()`, mutates entity fields (`scope`, `tenancyId`, `candidateGroups`, `candidateUsers`, `payload`, `permittedOutcomes`), then calls `persist()`. The new version builds a single `WorkItemCreateRequest` with all fields set via the builder (including `templateId`) and calls `workItemCreator.create(request)`. The SPI adapter routes `templateId != null` to `WorkItemTemplateService.createFromTemplate()` internally.

Template existence validation: the SPI throws if the template isn't found. The handler catches the exception and leaves the PlanItem PENDING — preserving current error semantics.

**Eliminated types:** `WorkItemTemplate`, `OutcomeCodecs`, `WorkItemStore` (as direct injection), `WorkItemService`, `WorkItemTemplateService` — none appear in production code after migration.

### POM Change

Production dependency changes from `casehub-work` to `casehub-work-api`. Test dependencies stay: `casehub-work-persistence-memory` and `quarkus-jdbc-h2` remain at test scope because tests need the full runtime to fire CDI events and verify end-to-end behavior.

### CLAUDE.md Update

The `casehub-work-adapter` section updates references from runtime types to API types: `WorkItemService` → `WorkItemCreator`, `WorkItemTemplateService` → eliminated, `WorkItem` → `WorkItemRef`, `WorkItemLifecycleEvent` → `WorkLifecycleEvent`/`WorkItemEvent`, `OutcomeCodecs` → eliminated.

## Commit Strategy

1. **Commit 1 (Refs #579):** Migrate `source()` → typed accessors. Change method signatures from `WorkItem` to `WorkItemRef`. Update tests. Observer type stays `WorkItemLifecycleEvent` — compiles against runtime.

2. **Commit 2 (Closes #578, Closes #579):** Switch observer to `WorkLifecycleEvent`. Replace all runtime service/store injections with API SPIs. Rewrite template mode. Change pom. Update tests. Update CLAUDE.md.

## Files Changed

### Commit 1 — #579
- `WorkItemLifecycleAdapter.java` — remove `source()` casts, use `WorkItemEvent` typed accessors
- `PlanItemCompletionApplier.java` — `WorkItem` → `WorkItemRef` parameter
- `ActionGateCompletionApplier.java` — `WorkItem` → `WorkItemRef` parameter
- `WorkItemLifecycleAdapterTest.java` — update assertions

### Commit 2 — #578
- `WorkItemLifecycleAdapter.java` — observer type `WorkItemLifecycleEvent` → `WorkLifecycleEvent`
- `HumanTaskScheduleHandler.java` — `WorkItemService`/`WorkItemTemplateService` → `WorkItemCreator`, rewrite template mode
- `ActionGateWorkItemHandler.java` — `WorkItemService` → `WorkItemCreator`
- `ActionGateCancelledHandler.java` — `WorkItemStore`/`WorkItemService` → `WorkItemCreator`/`WorkItemLifecycle`
- `HumanTaskRecoveryService.java` — `WorkItemService` → `WorkItemCreator`
- `work-adapter/pom.xml` — `casehub-work` → `casehub-work-api`
- `HumanTaskScheduleHandlerTest.java` — update wiring
- `CLAUDE.md` — update work-adapter section

### Not changed
- `WorkAdapterPlanItemEntity.java` — comment-only reference to `WorkItemService` (javadoc, not import)
