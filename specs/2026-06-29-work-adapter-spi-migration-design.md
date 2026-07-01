# Work-Adapter SPI Migration Design

**Issues:** engine#578, engine#579
**Date:** 2026-06-29

## Problem

The engine's `casehub-engine-work-adapter` module depends on `casehub-work` (full runtime) when it should depend on `casehub-work-api` only. This violates the platform's three-tier module convention: adapter modules should depend on API/SPI modules, not runtime implementations.

Seven production files import runtime types: `WorkItemService`, `WorkItemTemplateService`, `WorkItemStore`, `WorkItem`, `WorkItemTemplate`, `WorkItemLifecycleEvent`, and `OutcomeCodecs`. (`WorkAdapterPlanItemEntity` has a javadoc mention but no import.) The `casehub-work-api` SPI surface (`WorkItemCreator`, `WorkItemLifecycle`, `WorkItemEvent`, `WorkItemRef`, `WorkLifecycleEvent`) already covers every operation the adapter performs.

## Scope

Two issues, one branch, two commits:

1. **#579 — Event observation migration:** Replace `event.source()` → `WorkItem` casts with `WorkItemEvent` typed accessors in the observer and all internal helper methods. Method signatures (`apply()` on appliers) keep `WorkItem` parameters in this commit — they change atomically in commit 2.

2. **#578 — Dependency migration:** Replace `WorkItemService`/`WorkItemStore`/`WorkItemTemplateService` injections with `WorkItemCreator`/`WorkItemLifecycle`. Switch CDI observer from `WorkItemLifecycleEvent` (runtime) to `WorkLifecycleEvent` (api). Change `apply()` signatures from `WorkItem` to `WorkItemRef`. Change internal helper method signatures from `WorkItemLifecycleEvent` to `WorkItemEvent`. Change pom dependency from `casehub-work` to `casehub-work-api`.

## Design

### Event Observation (#579)

`WorkItemLifecycleAdapter` currently observes `WorkItemLifecycleEvent` (runtime) and casts `event.source()` to `WorkItem` (runtime entity) to access fields like `callerRef`, `id`, `candidateGroups`, `assigneeId`, `resolution`.

`WorkItemLifecycleEvent` already implements `WorkItemEvent` (api interface), which exposes all these fields as typed accessors: `callerRef()`, `workItemId()`, `candidateGroups()`, `assigneeId()`, `resolution()`, `status()`, `outcome()`.

**Commit 1 observer change — replace `source()` casts with `WorkItemEvent` accessors:**

```java
// Before
public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
    if (!(event.source() instanceof WorkItem workItem)) return;
    CallerRef ref = CallerRef.parse(workItem.callerRef);
}

// After (commit 1: #579 — typed accessors, still runtime event type, still passes WorkItem to apply())
public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
    CallerRef ref = CallerRef.parse(event.callerRef());
}
```

Commit 1 replaces `source()` → `WorkItem` casts with `WorkItemEvent` typed accessors in the main observer and all four internal helper methods (`handleEscalation`, `handleSuspension`, `handlePossibleResume`, `routeGate`). Field mapping for `handleEscalation` (the most complex):
- `workItem.callerRef` → `event.callerRef()`
- `workItem.candidateGroups` → `event.candidateGroups()`
- `workItem.id` → `event.workItemId()`

`apply()` signatures on `PlanItemCompletionApplier` and `ActionGateCompletionApplier` keep `WorkItem` in commit 1. The observer constructs a `WorkItemRef` from event accessors at each call site to pass to `apply()` — this avoids breaking `HumanTaskRecoveryService` which still calls `apply()` with a `WorkItem` from `workItemService.findByCallerRef()` until commit 2 migrates it.

**Commit 2 observer change — api-only types:**

```java
// After (commit 2: #578 — api-only types)
public void onWorkItemLifecycle(@ObservesAsync WorkLifecycleEvent event) {
    if (!(event instanceof WorkItemEvent wie)) return;
    CallerRef ref = CallerRef.parse(wie.callerRef());
}
```

CDI delivers `WorkItemLifecycleEvent` (the concrete class fired at runtime). The observer catches it as `WorkLifecycleEvent` (api abstract class). The `instanceof WorkItemEvent` check succeeds because `WorkItemLifecycleEvent implements WorkItemEvent`. All referenced types are in `casehub-work-api`.

**Commit 2 method signature changes (atomic):**

| Method | Before | After |
|--------|--------|-------|
| `PlanItemCompletionApplier.apply()` | `(UUID, String, WorkItemStatus, WorkItem)` | `(UUID, String, WorkItemStatus, WorkItemRef)` |
| `ActionGateCompletionApplier.apply()` | `(GateCallerRef, WorkItemStatus, WorkItem)` | `(GateCallerRef, WorkItemStatus, WorkItemRef)` |
| `handleEscalation()` | `(WorkItemLifecycleEvent)` | `(WorkItemEvent)` |
| `handleSuspension()` | `(WorkItemLifecycleEvent)` | `(WorkItemEvent)` |
| `handlePossibleResume()` | `(WorkItemLifecycleEvent)` | `(WorkItemEvent)` |
| `routeGate()` | called with `WorkItem` fields | called with `WorkItemEvent` accessors |

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

**tenancyId precedence change (intentional):** The current code preserves a template-provided tenancyId (`if (workItem.tenancyId == null) { workItem.tenancyId = event.tenancyId(); }`). The migrated code always passes `event.tenancyId()` on the request, and `createFromTemplate` uses request-wins semantics — the engine's runtime tenancyId overrides any template default. This is the correct behavior: the engine knows the tenant context at dispatch time, and a template default should not override it. No existing templates carry tenancyId (templates are tenant-agnostic definitions), so this has no practical impact today, but the semantic change is explicit.

**UUID validation retained:** The handler continues to parse `target.templateRef()` as a UUID with a try-catch before setting `templateId` on the request. Invalid UUIDs log a warning and leave the PlanItem PENDING — same as current behavior. This is handler-level input validation, not SPI concern.

**Template not-found handling:** The SPI throws if the template isn't found. The handler wraps the `workItemCreator.create()` call in a try-catch, logs a warning, and leaves the PlanItem PENDING — preserving current error semantics.

**Eliminated types:** `WorkItemTemplate`, `OutcomeCodecs`, `WorkItemStore` (as direct injection), `WorkItemService`, `WorkItemTemplateService` — none appear in production code after migration.

### POM Change

Production dependency changes from `casehub-work` to `casehub-work-api`. Test dependencies stay: `casehub-work-persistence-memory` and `quarkus-jdbc-h2` remain at test scope because tests need the full runtime to fire CDI events and verify end-to-end behavior.

### CLAUDE.md Update

The `casehub-work-adapter` section updates references from runtime types to API types: `WorkItemService` → `WorkItemCreator`, `WorkItemTemplateService` → eliminated, `WorkItem` → `WorkItemRef`, `WorkItemLifecycleEvent` → `WorkLifecycleEvent`/`WorkItemEvent`, `OutcomeCodecs` → eliminated.

## Commit Strategy

1. **Commit 1 (Refs #579):** Replace all `source()` → `WorkItem` casts with `WorkItemEvent` typed accessors in the observer and helper methods. `apply()` signatures keep `WorkItem` — callers construct `WorkItemRef` at call sites where needed. Observer type stays `WorkItemLifecycleEvent`. Compiles and tests independently.

2. **Commit 2 (Closes #578, Closes #579):** Switch observer to `WorkLifecycleEvent`. Change `apply()` signatures from `WorkItem` to `WorkItemRef`. Change helper method signatures from `WorkItemLifecycleEvent` to `WorkItemEvent`. Replace all runtime service/store injections with API SPIs. Rewrite template mode. Change pom. Update tests. Update CLAUDE.md.

## Files Changed

### Commit 1 — #579
- `WorkItemLifecycleAdapter.java` — remove `source()` casts in `onWorkItemLifecycle`, `handleEscalation`, `handleSuspension`, `handlePossibleResume`, `routeGate`; use `WorkItemEvent` typed accessors
- `WorkItemLifecycleAdapterTest.java` — update assertions for typed accessor usage

### Commit 2 — #578
- `WorkItemLifecycleAdapter.java` — observer type `WorkItemLifecycleEvent` → `WorkLifecycleEvent`; helper methods `WorkItemLifecycleEvent` → `WorkItemEvent`
- `PlanItemCompletionApplier.java` — `WorkItem` → `WorkItemRef` parameter
- `ActionGateCompletionApplier.java` — `WorkItem` → `WorkItemRef` parameter
- `HumanTaskScheduleHandler.java` — `WorkItemService`/`WorkItemTemplateService` → `WorkItemCreator`; rewrite template mode with UUID validation retained
- `ActionGateWorkItemHandler.java` — `WorkItemService` → `WorkItemCreator`
- `ActionGateCancelledHandler.java` — `WorkItemStore`/`WorkItemService` → `WorkItemCreator`/`WorkItemLifecycle`
- `HumanTaskRecoveryService.java` — `WorkItemService` → `WorkItemCreator`; `WorkItem` → `WorkItemRef`
- `work-adapter/pom.xml` — `casehub-work` → `casehub-work-api`
- `HumanTaskScheduleHandlerTest.java` — update wiring
- `WorkItemLifecycleAdapterTest.java` — update for `WorkLifecycleEvent` observer type
- `CLAUDE.md` — update work-adapter section

### Not changed
- `WorkAdapterPlanItemEntity.java` — comment-only reference to `WorkItemService` (javadoc, not import)

## Design Review Findings (Round 1)

Adversarial review completed 2026-06-29. Seven findings — all addressed:

| ID | Finding | Resolution |
|----|---------|------------|
| R1-01 | Commit 1 breaks compilation: `apply()` signature change orphans `HumanTaskRecoveryService` | Fixed: `apply()` signatures stay `WorkItem` in commit 1, change atomically in commit 2 |
| R1-02 | Template mode tenancyId precedence inverted by request-wins merge | Accepted as intentional: engine context should override template defaults; documented explicitly |
| R1-03 | Template mode UUID validation and error handling not addressed | Fixed: UUID parse try-catch retained in handler, separate from SPI exception handling |
| R1-04 | Internal helper method signature changes omitted | Fixed: four helper methods enumerated in commit 2 file list |
| R1-05 | handleEscalation field mapping not shown | Fixed: field mapping documented in observer change section |
| R1-06 | File count "eight" incorrect | Fixed: corrected to "seven" |
| R1-07 | Verified claims | All architectural claims confirmed against source code |

Assumption A1 verified: `PlanItemCompletionApplier.apply()` and `ActionGateCompletionApplier.apply()` have no callers outside `casehub-engine-work-adapter`.