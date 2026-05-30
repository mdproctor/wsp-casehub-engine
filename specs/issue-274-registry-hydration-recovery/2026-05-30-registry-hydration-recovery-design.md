# Design: BlackboardRegistry Hydration + HumanTask Recovery on Restart

**Issues:** casehubio/engine#274 (registry hydration) · casehubio/engine#398 (HumanTask catch-up)
**Branch:** issue-274-registry-hydration-recovery
**Date:** 2026-05-30

---

## Problem

On JVM restart, `BlackboardRegistry` is empty. Two failure modes result:

**In-flight scenario (#274):** A WorkItem completes after restart. `WorkItemLifecycleAdapter.onWorkItemLifecycle()` calls `registry.get(caseId)` → returns empty → logs DEBUG → discards event. The case never progresses.

**Offline-completion scenario (#398):** A WorkItem completed while the JVM was down. The `WorkItemLifecycleEvent` already fired and is gone. No one re-fires it. The case is permanently stuck with a DELEGATED PlanItem that will never transition.

---

## Design

### 1. Data Model Changes

**`PlanItemRecord` — two new fields:**
```java
public record PlanItemRecord(
    UUID caseId, String planItemId, String bindingName,
    PlanItemStatus status, Instant createdAt,
    String targetType,              // "humanTask" | "capability" | "subCase" | "extension" | null
    String outputMappingExpression) // nullable JQ expression; non-null only for humanTask with outputMapping
```

**`PlanItemStore.save()` — updated signature (blocking + reactive mirror):**
```java
void save(UUID caseId, String planItemId, String bindingName,
          PlanItemStatus status, Instant createdAt,
          String targetType, String outputMappingExpression);
```
Call sites: `HumanTaskScheduleHandler` (blocking) and its reactive counterpart. Both already have the `HumanTaskTarget` in scope at save time — extract `targetType = "humanTask"` and the JQ expression from `HumanTaskTarget.outputMapping()` if it is a `JQExpressionEvaluator`.

**New SPI method — both `PlanItemStore` and `ReactivePlanItemStore`:**
```java
// Returns all records with status RUNNING or DELEGATED across all cases
List<PlanItemRecord> findNonTerminal();
Uni<List<PlanItemRecord>> findNonTerminal();  // reactive mirror
```
Updated in: `JpaPlanItemStore`, `JpaReactivePlanItemStore`, `NoOpPlanItemStore` (returns empty list), `MemoryPlanItemStore`. Contract test added in `PlanItemStoreContractTest` and `ReactivePlanItemStoreContractTest`.

**`DefaultCasePlanModel.restorePlanItem(PlanItem item)` — new method:**
```java
public void restorePlanItem(PlanItem item) {
    itemsById.put(item.getPlanItemId(), item);
    activeByBinding.put(item.getBindingName(), item);
    // NOT added to agenda — restored items are not pending dispatch
}
```
Contrast with `addPlanItem()` which writes all three data structures. Restored items are findable by completion handlers; they do not appear on the dispatch queue.

**`PlanItem.restore()` — new static factory:**
```java
public static PlanItem restore(String planItemId, String bindingName,
                               Binding target, PlanItemStatus status)
```
Bypasses the normal PENDING-only creation path. `target` may be null — see null-guard below.

**`WorkItemLifecycleAdapter.applyOutputMapping()` — null-guard:**
```java
if (item.getTarget() == null) {
    LOG.warnf("PlanItem %s has no target (recovered without target info) — outputMapping skipped",
              item.getPlanItemId());
    return;
}
```
Guards against NPE on the sealed switch. Handles both non-HumanTask items and the rare case where `outputMappingExpression` was null at save time.

---

### 2. Services

#### `BlackboardRegistryHydrationService` — engine#274

**Location:** `blackboard/src/main/java/io/casehub/blackboard/recovery/BlackboardRegistryHydrationService.java`

**Startup priority:** `@Priority(10)` — before Quartz at 20

```java
@ApplicationScoped
public class BlackboardRegistryHydrationService {

    @Inject PlanItemStore store;
    @Inject BlackboardRegistry registry;

    void onStart(@Observes @Priority(10) StartupEvent ev) {
        store.findNonTerminal()
             .stream()
             .collect(groupingBy(PlanItemRecord::caseId))
             .forEach((caseId, records) -> {
                 CasePlanModel plan = registry.getOrCreate(caseId);
                 records.forEach(r -> plan.restorePlanItem(buildPlanItem(r)));
             });
        LOG.infof("BlackboardRegistry hydrated: %d cases", /* count */ 0);
    }

    private PlanItem buildPlanItem(PlanItemRecord r) {
        Binding target = "humanTask".equals(r.targetType())
            ? buildHumanTaskTarget(r.outputMappingExpression())
            : null;
        return PlanItem.restore(r.planItemId(), r.bindingName(), target, r.status());
    }

    private HumanTaskTarget buildHumanTaskTarget(String expr) {
        ExpressionEvaluator mapping = expr != null
            ? new JQExpressionEvaluator(expr)
            : null;
        return HumanTaskTarget.builder().outputMapping(mapping).build();
    }
}
```

**Module:** `blackboard` — already depends on `common` (for `PlanItemStore`, `PlanItemStatus`). No new module dependencies.

#### `HumanTaskRecoveryService` — engine#398

**Location:** `work-adapter/src/main/java/io/casehub/workadapter/recovery/HumanTaskRecoveryService.java`

**Startup priority:** `@Priority(25)` — after Quartz at 20, registry already hydrated at 10

**Cross-repo dependency:** requires `WorkItemService.findByCallerRef(String callerRef)` in `casehub-work`. This method does not currently exist. A casehub-work issue is filed before implementing this service (see §4 Deferred).

```
Flow per DELEGATED PlanItemRecord from planItemStore.findNonTerminal():
  1. callerRef = CallerRef.encode(caseId, planItemId)
  2. workItem = workItemService.findByCallerRef(callerRef)
  3. workItem absent              → log DEBUG (WorkItem cleaned up); skip
  4. workItem status non-terminal → still in flight; skip
  5. workItem status terminal:
       a. planItem = registry.get(caseId).flatMap(p -> p.getPlanItem(planItemId))
       b. Apply status transition (markCompleted / markRejected / markFaulted / markCancelled)
       c. Apply outputMapping against workItem.resolution (via JQEvaluator, if target present)
       d. Load CaseInstance; publish CONTEXT_CHANGED to event bus
```

**Module:** `work-adapter` — already depends on `blackboard`, `common`, `casehub-work-core`. No new module dependencies.

---

### 3. Startup Priority Contract

```
@Priority(10)  BlackboardRegistryHydrationService   — registry populated from PlanItemStore
@Priority(20)  QuartzWorkerExecutionManager          — reschedules in-flight Quartz workers
                                                        (jobs execute; registry.get() now succeeds)
@Priority(25)  HumanTaskRecoveryService              — catches up offline-completed WorkItems
                                                        (fires CONTEXT_CHANGED; case progresses)
```

The 10→20 gap is the critical invariant for #274: registry must be populated before Quartz job execution begins.

---

### 4. Test Approach

#### `BlackboardRegistryHydrationTest` in `blackboard` module

`@QuarkusTest` using `casehub-persistence-memory`.

- **Hydration smoke test:** seed memory store with DELEGATED (humanTask, with JQ expression) + RUNNING PlanItem across two cases; call `onStart()`; assert `registry.get(caseId)` is non-empty; assert each PlanItem findable by ID with correct status; assert DELEGATED item's target is `HumanTaskTarget` with the stored expression.
- **End-to-end integration:** after hydration, deliver a simulated `WorkItemLifecycleEvent` (COMPLETED) for the DELEGATED PlanItem; assert CONTEXT_CHANGED fires.
- **No-op store path:** with `MemoryPlanItemStore` empty (default), `onStart()` is a no-op; registry remains empty.

#### `HumanTaskRecoveryTest` in `work-adapter` module

`@QuarkusTest` using `casehub-persistence-memory` + `casehub-work-testing`.

- **Catch-up test:** seed DELEGATED PlanItem in store; seed `InMemoryWorkItemStore` with COMPLETED WorkItem (callerRef matches); call `onStart()`; assert CONTEXT_CHANGED published; assert PlanItem transitions to COMPLETED.
- **Offline REJECTED:** same flow, WorkItem REJECTED → PlanItem REJECTED.
- **Offline EXPIRED:** WorkItem EXPIRED → PlanItem FAULTED.
- **In-flight skip:** WorkItem still DELEGATED in work → no CONTEXT_CHANGED, no transition.
- **Absent WorkItem:** WorkItem not found → DEBUG log, no crash.

#### Contract test additions in `casehub-engine-common`

- `PlanItemStoreContractTest.findNonTerminal_returnsOnlyRunningAndDelegated()` — saves PENDING, RUNNING, DELEGATED, COMPLETED, FAULTED; asserts `findNonTerminal()` returns exactly RUNNING + DELEGATED records.
- Mirror in `ReactivePlanItemStoreContractTest`.

---

### 5. Deferred / Out of Scope

**`casehub-work` issue — `WorkItemService.findByCallerRef()`:** File before starting #398 implementation. `HumanTaskRecoveryService` is blocked on this method. Implement the casehub-work change locally (source available) and include in same session.

**PENDING item re-dispatch after restart:** PENDING items are not stored in PlanItemStore (only DELEGATED items are written at `HumanTaskScheduleHandler` time). After restart, PENDING items are re-created by normal engine evaluation when CONTEXT_CHANGED fires. This is existing behavior and not addressed here.

**RUNNING item zombie detection:** Issue #274 mentions "cases with RUNNING PlanItems but no matching WorkItem should be flagged or auto-recovered." RUNNING items (Quartz workers) are recovered by Quartz itself — the Quartz RAM store is rebuilt from the EventLog by `QuartzWorkerExecutionManager`. Zombie detection (RUNNING with no Quartz job) is a follow-on concern; file as a separate issue if needed after observing production behavior.

**`outputMapping` for non-JQExpression evaluators:** `buildHumanTaskTarget()` only reconstructs `JQExpressionEvaluator` mappings. Lambda-mode (`inputTransformer`) mappings cannot be serialized. This is acceptable — lambda-mode is test-only; production definitions use JQ expressions.

---

### 6. Platform Coherence Review

- **Dual-trail audit protocol:** No new lifecycle transitions introduced; existing transitions (`markCompleted()` etc.) already fire CDI lifecycle events. ✅
- **`@DefaultBean` pattern:** `NoOpPlanItemStore.findNonTerminal()` returns empty list — deployments without the persistence module see no hydration, no crash. ✅
- **Module tier rule:** hydration service in `blackboard` (Orchestration tier); recovery service in `work-adapter` (Orchestration tier). Both depend inward only. ✅
- **No migration tooling:** No Flyway migrations; schema managed by Hibernate `drop-and-create`. ✅
- **`casehub-work` boundary:** no orchestration logic added to `casehub-work`; catch-up logic lives in `work-adapter` (engine side). ✅
