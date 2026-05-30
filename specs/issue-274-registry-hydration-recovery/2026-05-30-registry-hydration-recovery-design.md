# Design: BlackboardRegistry Hydration + HumanTask Recovery on Restart

**Issues:** casehubio/engine#274 (registry hydration) · casehubio/engine#398 (HumanTask catch-up)
**Branch:** issue-274-registry-hydration-recovery
**Date:** 2026-05-30 (revised after review)

---

## Problem

On JVM restart, `BlackboardRegistry` is empty. Two failure modes result:

**In-flight scenario (#274):** A WorkItem completes after restart. `WorkItemLifecycleAdapter.onWorkItemLifecycle()` calls `registry.get(caseId)` → returns empty → logs DEBUG → discards event. The case never progresses.

**Offline-completion scenario (#398):** A WorkItem completed while the JVM was down. The `WorkItemLifecycleEvent` already fired and is gone. No one re-fires it. The case is permanently stuck with a DELEGATED PlanItem that will never transition.

### Scope constraint — Quartz-only cases

`PlanItemStore.save()` is called only from `HumanTaskScheduleHandler`, always with `DELEGATED` status. RUNNING items (Quartz workers) are never persisted to PlanItemStore. As a result, this PR restores the registry only for cases that have at least one in-flight HumanTask. Cases that exclusively use Quartz workers remain unhydrated. The Quartz completion handlers (`PlanItemCompletionHandler`, `PlanItemFaultHandler`, `WorkerRetryExhaustionHandler`) additionally rely on `completionIndex` (`workerName → planItemId`) which is never persisted and cannot be rebuilt without `workerName` in the store. Full Quartz recovery is a separate future concern requiring `workerName` persistence; tracked as a follow-on issue.

---

## Design

### 1. Data Model Changes

#### `PlanItemRecord` — two new fields

```java
public record PlanItemRecord(
    UUID caseId, String planItemId, String bindingName,
    PlanItemStatus status, Instant createdAt,
    TargetType targetType,              // enum: HUMAN_TASK | CAPABILITY | SUB_CASE | EXTENSION | null
    String outputMappingExpression)     // nullable JQ expression; non-null only for HUMAN_TASK with outputMapping
```

`outputMappingExpression` lives in `PlanItemRecord` (and not only in a separate entity) because the in-flight recovery path (#274) requires it at completion time: `WorkItemLifecycleAdapter.applyOutputMapping()` needs the expression when the hydrated `PlanItem` carries a null outputMapping. Any approach that defers the expression lookup to completion time adds a DB call on every post-restart completion event; carrying it through the hydration path avoids that.

In reactive-path deployments (using `persistence-hibernate`, no `work-adapter`), no HumanTask events are scheduled, so `outputMappingExpression` is always `null` — no functional impact.

#### `TargetType` enum — new, in `casehub-engine-common`

```java
public enum TargetType { HUMAN_TASK, CAPABILITY, SUB_CASE, EXTENSION }
```

Consistent with existing codebase pattern (`PlanItemStatus` enum, sealed target class hierarchy).

#### `PlanItemSaveRequest` value object — new, in `casehub-engine-common`

Replaces the growing parameter list on `save()`:

```java
public record PlanItemSaveRequest(
    UUID caseId, String planItemId, String bindingName,
    PlanItemStatus status, Instant createdAt,
    TargetType targetType, String outputMappingExpression) {}
```

#### `PlanItemStore.save()` — updated to take `PlanItemSaveRequest`

```java
void save(PlanItemSaveRequest request);
```

Updated in: `JpaPlanItemStore`, `JpaReactivePlanItemStore`, `MemoryPlanItemStore`, `MemoryReactivePlanItemStore`, `NoOpPlanItemStore`, `NoOpReactivePlanItemStore`.

Call sites: `HumanTaskScheduleHandler` (inline mode, template mode). At call time, `HumanTaskTarget` is in scope — extract `TargetType.HUMAN_TASK` and the JQ expression from `HumanTaskTarget.outputMapping()` if it is a `JQExpressionEvaluator`.

#### New SPI method — `PlanItemStore` and `ReactivePlanItemStore`

```java
// Returns all DELEGATED records for the given case
List<PlanItemRecord> findDelegated(UUID caseId);
Uni<List<PlanItemRecord>> findDelegated(UUID caseId);  // reactive mirror
```

Named `findDelegated` (not `findNonTerminal`) because only DELEGATED items are persisted. Returns records with all fields including `targetType` and `outputMappingExpression`.

Updated in: `JpaPlanItemStore`, `JpaReactivePlanItemStore`, `NoOpPlanItemStore` (returns empty), `MemoryPlanItemStore`.
Contract test added in `PlanItemStoreContractTest` and `ReactivePlanItemStoreContractTest`.

#### JPA entity updates

Both `WorkAdapterPlanItemEntity` (in `work-adapter`) and `PlanItemEntity` (in `persistence-hibernate`) map the same `plan_item` table and must be kept in sync:

```java
@Enumerated(EnumType.STRING)
@Column(name = "target_type", length = 20)
public TargetType targetType;  // nullable

@Column(name = "output_mapping_expression", length = 1000)
public String outputMappingExpression;  // nullable
```

Both entities are updated. `outputMappingExpression` is always null in `PlanItemEntity`-based deployments (no work-adapter, no HumanTask scheduling), which is correct.

#### `DefaultCasePlanModel.restorePlanItem(PlanItem item)` — new method

```java
public void restorePlanItem(PlanItem item) {
    itemsById.put(item.getPlanItemId(), item);
    activeByBinding.put(item.getBindingName(), item);
    // NOT added to agenda — restored items are not pending dispatch
}
```

Contrast with `addPlanItem()` which writes all three data structures. Restored items are findable by completion handlers without appearing on the dispatch queue.

#### `PlanItem.restore()` — new static factory with status validation

```java
public static PlanItem restore(String planItemId, String bindingName,
                               BindingTarget target, PlanItemStatus status) {
    if (status != PlanItemStatus.RUNNING && status != PlanItemStatus.DELEGATED) {
        throw new IllegalArgumentException(
            "restore() only valid for RUNNING or DELEGATED status, got: " + status);
    }
    // ... construct PlanItem with given status
}
```

Only RUNNING and DELEGATED items may be restored. PENDING items are re-created by evaluation; terminal items must not re-enter the live plan.

#### `WorkItemLifecycleAdapter.applyOutputMapping()` — null-guard

```java
if (item.getTarget() == null) {
    LOG.warnf("PlanItem %s has no target (recovered without target info) — outputMapping skipped",
              item.getPlanItemId());
    return;
}
```

Defensive guard for the case where a PlanItem was restored with a null target. In practice this should not occur for valid DELEGATED HumanTask items — the expression is stored at scheduling time. The guard exists as a safety net.

---

### 2. #274 — Lazy Hydration in `BlackboardRegistry.get()`

Rather than an eager startup service, `BlackboardRegistry.get(caseId)` hydrates lazily when a case is absent:

```java
@Inject PlanItemStore planItemStore;

public Optional<CasePlanModel> get(UUID caseId) {
    CaseEntry e = entries.get(caseId);
    if (e != null) return Optional.of(e.planModel);

    // Lazy hydration: query store on first miss
    List<PlanItemRecord> records = planItemStore.findDelegated(caseId);
    if (records.isEmpty()) return Optional.empty();

    CaseEntry hydrated = entries.computeIfAbsent(caseId, CaseEntry::new);
    for (PlanItemRecord r : records) {
        hydrated.planModel.restorePlanItem(buildPlanItem(r));
    }
    return Optional.of(hydrated.planModel);
}
```

**Why lazy instead of eager:**
- Eliminates the startup priority ordering invariant entirely for #274
- Handles any restart scenario including partial, rolling restarts
- Handles late-arriving cases (created after restart) without re-hydration
- `BlackboardRegistryHydrationService` and its `@Priority` contract disappear
- `HumanTaskRecoveryService` still uses an explicit startup scan — lazy loading cannot help when the completion event is already gone (#398)

**Concurrency:** Two concurrent misses for the same case each call `findDelegated()` independently (the DB query is not inside `computeIfAbsent`). Both then race on `computeIfAbsent`; the loser finds the entry already created and iterates `restorePlanItem()` on the pre-populated model. Since `restorePlanItem()` is idempotent (same `planItemId`, same data), the double-write is harmless. The cost is at most one duplicate DB call per concurrent miss, acceptable in the post-restart recovery window.

**`BlackboardPlanConfigurer` interaction:** `get()` does NOT call `markConfigured()`. When CONTEXT_CHANGED fires after restart for a hydrated case, `PlanningStrategyLoopControl` calls `getOrCreate()` → the case is already in the map → returns existing model → `markConfigured()` returns true → configurers run once (as designed). `addPlanItemIfAbsent()` correctly rejects re-dispatch of DELEGATED items already in `activeByBinding`. This relies on configurers being idempotent with respect to pre-populated plan items: `addPlanItemIfAbsent()` for a DELEGATED binding returns false (active item present, rejected). Any `BlackboardPlanConfigurer` implementation must honour this contract. **This contract is now explicit and required.**

**Stage state gap:** Stages are populated by configurers on first `CONTEXT_CHANGED` after restart. Between restart and first CONTEXT_CHANGED, `plan.getActiveStages()` returns empty for a hydrated case. No startup path accesses stages before CONTEXT_CHANGED fires — confirmed by reviewing startup observers. The window is real but harmless.

**Fail-fast on store unavailable:** If `planItemStore.findDelegated()` throws (e.g., database unavailable), the exception propagates to the caller and the case entry is not populated. The caller logs and returns, dropping the event — same behavior as before this PR. Partial hydration is not worse than no hydration; fail-fast is intentional.

**Blocking thread requirement — four `@ConsumeEvent` handlers must add `blocking = true`:**

`BlackboardRegistry.get()` now makes a blocking JDBC call via `JpaPlanItemStore.findDelegated()` on the first miss after restart. Four `@ConsumeEvent` handlers call `registry.get()` from non-blocking Vert.x IO threads:

| Handler | Event | Current return type | Change |
|---------|-------|---------------------|--------|
| `PlanItemCompletionHandler.onWorkerFinished()` | `WORKER_EXECUTION_FINISHED` | `Uni<Void>` | `blocking = true`, `void` |
| `PlanItemCompletionHandler.onSubCaseFinished()` | `SUBCASE_EXECUTION_COMPLETED` | `Uni<Void>` | `blocking = true`, `void` |
| `WorkerRetryExhaustionHandler.onWorkerRetriesExhausted()` | `WORKER_RETRIES_EXHAUSTED` | `Uni<Void>` | `blocking = true`, `void` |
| `PlanItemFaultHandler.onWorkerRetriesExhausted()` | `WORKER_RETRIES_EXHAUSTED` | `Uni<Void>` | `blocking = true`, `void` |

All four handler bodies are already synchronous — they return `Uni.createFrom().voidItem()` without chaining async operations. Converting to `void` + `blocking = true` is a clean mechanical change. The `planItemCompletedEvents.fireAsync()` call in `PlanItemCompletionHandler` remains fire-and-forget and works correctly from a worker thread.

These handlers are in `blackboard`. This change is a consequence of lazy hydration living in `BlackboardRegistry` — the registry now requires a blocking thread for post-restart misses. The `@ObservesAsync` handlers in `work-adapter` (`WorkItemLifecycleAdapter`) run on a CDI managed executor (not the Vert.x IO thread) and are already safe.

**Target reconstruction — `PlanItemRestorer` (package-private in `blackboard`):**

`BlackboardRegistry` must not import `HumanTaskTarget` or `JQExpressionEvaluator` — it is a pure key/value store. Extract reconstruction to `PlanItemRestorer`:

```java
// package-private — io.casehub.blackboard.recovery.PlanItemRestorer
class PlanItemRestorer {
    PlanItem restore(PlanItemRecord r) {
        BindingTarget target = r.targetType() == TargetType.HUMAN_TASK
            ? buildHumanTaskTarget(r.outputMappingExpression())
            : null;
        return PlanItem.restore(r.planItemId(), r.bindingName(), target, r.status());
    }

    private HumanTaskTarget buildHumanTaskTarget(String expr) {
        ExpressionEvaluator mapping = expr != null ? new JQExpressionEvaluator(expr) : null;
        return HumanTaskTarget.builder().outputMapping(mapping).build();
    }
}
```

`BlackboardRegistry` injects `PlanItemRestorer` and calls `restorer.restore(r)` in place of `buildPlanItem(r)`. The registry itself imports only `PlanItemRecord` and `PlanItem`.

**`BlackboardRegistry` Javadoc update:** Remove stale "rebuilt from EventLog on engine recovery." Replace with: "DELEGATED plan items are lazily restored from PlanItemStore on the first get() miss after restart. RUNNING items and completionIndex are not persisted; Quartz-only case recovery is a separate concern."

---

### 3. #398 — `HumanTaskRecoveryService` (Startup Scan)

**Location:** `work-adapter/src/main/java/io/casehub/workadapter/recovery/HumanTaskRecoveryService.java`

**Startup priority:** `@Priority(25)` — after Quartz at 20; lazy hydration from #274 has already populated the registry for any case touched between restart and this service running (which is none, since Quartz jobs haven't executed yet at startup time).

**Cross-repo dependency:** `WorkItemService.findByCallerRef(String callerRef)` does not currently exist in `casehub-work`. File `casehub-work#NEW` before starting implementation. Implement locally (source available) in the same session.

**Shared logic — `PlanItemCompletionApplier`:**

`HumanTaskRecoveryService` and `WorkItemLifecycleAdapter` both: apply a status transition, apply outputMapping, load CaseInstance, publish CONTEXT_CHANGED. Extract this into a package-private `PlanItemCompletionApplier` in `work-adapter` and inject into both. Future changes to the completion path (new statuses, new events) are made once.

```java
// package-private — io.casehub.workadapter
@ApplicationScoped
class PlanItemCompletionApplier {
    @Transactional
    void apply(UUID caseId, String planItemId, WorkItemStatus status, WorkItem workItem);
}
```

`@Transactional` uses REQUIRED semantics — propagates from `HumanTaskRecoveryService` when called during startup, and opens a new transaction when called from `WorkItemLifecycleAdapter` (which currently has no `@Transactional`). This is intentional: the applier owns the transaction boundary in both contexts.

**Transaction scope:** Steps (apply status, apply outputMapping, publish CONTEXT_CHANGED) run inside `@Transactional` via `PlanItemCompletionApplier`. The recovery service is a `StartupEvent` observer (not a `@ConsumeEvent` handler), so `blocking = true` is not required — `@Transactional` on the applier method is sufficient. The `@Transactional` boundary covers the DB writes (status transition, `planItemStore.updateStatus()`). `eventBus.publish()` is async fire-and-forget and cannot be rolled back — if a synchronous failure occurs before the publish, the transaction rolls back and CONTEXT_CHANGED is not queued. If failure occurs after the publish, CONTEXT_CHANGED has been queued but the DB state reverts on rollback. Recovery re-applies on next restart in both cases.

**Idempotency:** Before calling `markCompleted()` (or equivalent), check `item.getStatus()`. If already terminal, log DEBUG and skip — do not throw. The recovery service may re-run if the JVM crashes mid-recovery.

```
Flow per DELEGATED record from planItemStore.findDelegated(caseId) [scanned globally]:
  (Use a new global findAllDelegated() SPI method, or iterate all known case IDs —
   see implementation note below)
  1. callerRef = CallerRef.encode(caseId, planItemId)
  2. workItem = workItemService.findByCallerRef(callerRef)
  3. workItem absent              → log DEBUG (WorkItem cleaned up); skip
  4. workItem status non-terminal → still in flight; skip
  5. workItem status terminal:
     @Transactional {
       a. planItem = registry.get(caseId).flatMap(p -> p.getPlanItem(planItemId))
                    (lazy hydration fires here if case not yet in registry)
       b. if planItem.getStatus() is terminal → already handled; skip
       c. apply transition via PlanItemCompletionApplier
     }
```

**Global scan SPI method:** The recovery service needs to find DELEGATED items across ALL cases, not per-case. Add `List<PlanItemRecord> findAllDelegated()` to `PlanItemStore` (complements `findDelegated(UUID caseId)`). `NoOpPlanItemStore` returns empty list. `MemoryPlanItemStore` scans its backing store. JPA: indexed query on `status = DELEGATED`.

---

### 4. Module Boundaries

| Component | Module | New dependencies |
|-----------|--------|-----------------|
| `TargetType` enum | `common` | — |
| `PlanItemSaveRequest` | `common` | — |
| `PlanItemStore.findDelegated(UUID)` | `common` SPI | — |
| `PlanItemStore.findAllDelegated()` | `common` SPI | — |
| `PlanItemRecord` (updated) | `common` | — |
| `BlackboardRegistry` (lazy get) | `blackboard` | `PlanItemStore` via `common` (already a dependency) |
| `DefaultCasePlanModel.restorePlanItem` | `blackboard` | — |
| `PlanItem.restore()` | `blackboard` | — |
| `PlanItemCompletionApplier` | `work-adapter` (pkg-private) | — |
| `HumanTaskRecoveryService` | `work-adapter` | `WorkItemService.findByCallerRef` (new, casehub-work) |
| `WorkAdapterPlanItemEntity` (updated) | `work-adapter` | — |
| `PlanItemEntity` (updated) | `persistence-hibernate` | — |

No new inter-module dependencies.

---

### 5. Test Approach

#### `BlackboardRegistryHydrationTest` in `blackboard` module

`@QuarkusTest` using `casehub-persistence-memory`.

- **Lazy get hydration:** seed memory store with DELEGATED HumanTask PlanItem (with JQ expression); call `registry.get(caseId)`; assert non-empty result; assert `getPlanItem(planItemId)` returns item with status DELEGATED; assert `item.getTarget()` is `HumanTaskTarget` with expression set.
- **Empty store → empty result:** `MemoryPlanItemStore` empty; `registry.get(caseId)` returns empty.
- **RUNNING items not hydrated:** store has only RUNNING items (for future RUNNING store support); `registry.get(caseId)` still returns empty (RUNNING items out of scope for `findDelegated()`).
- **Concurrency:** two concurrent `registry.get(caseId)` calls on empty registry; assert that both complete with a populated model; assert that the double `restorePlanItem()` is a no-op (idempotent).

#### `HumanTaskRecoveryTest` in `work-adapter` module

`@QuarkusTest` using `casehub-persistence-memory` + `casehub-work-testing`.

- **Catch-up COMPLETED:** seed DELEGATED PlanItem; WorkItem COMPLETED; call `onStart()`; assert CONTEXT_CHANGED published; PlanItem transitions to COMPLETED.
- **Catch-up REJECTED:** WorkItem REJECTED → PlanItem REJECTED.
- **Catch-up EXPIRED:** WorkItem EXPIRED → PlanItem FAULTED.
- **In-flight skip:** WorkItem DELEGATED → no transition, no CONTEXT_CHANGED.
- **Absent WorkItem:** not found → DEBUG log, no crash.
- **Idempotency:** PlanItem already COMPLETED before recovery runs → no `IllegalStateException`; silent skip.
- **Transaction rollback on event bus failure:** event bus publish throws → assert PlanItem status NOT changed (transaction rolled back).

#### Contract tests in `casehub-engine-common`

- `PlanItemStoreContractTest.findDelegated_returnsOnlyDelegatedForCase()` — saves PENDING, DELEGATED (two cases), COMPLETED; asserts `findDelegated(caseId)` returns only the DELEGATED record for that case.
- `PlanItemStoreContractTest.findAllDelegated_acrossAllCases()` — saves DELEGATED items across three cases; asserts `findAllDelegated()` returns all three.
- `PlanItemStoreContractTest.save_storesTargetTypeAndExpression()` — saves with TargetType.HUMAN_TASK + expression; finds by caseId; asserts both fields match.
- Mirror tests in `ReactivePlanItemStoreContractTest`.

---

### 6. Deferred / Out of Scope

**Quartz worker recovery (RUNNING items + completionIndex):** Adding RUNNING items to PlanItemStore requires `workerName` in `PlanItemRecord` so `completionIndex` can be rebuilt on hydration. `PlanItemCompletionHandler`, `PlanItemFaultHandler`, and `WorkerRetryExhaustionHandler` all depend on `completionIndex` — without it, Stage autocomplete and fault handling remain broken after restart for Quartz-only cases. File as a follow-on issue with the `workerName` schema addition and completionIndex rebuild path.

**PENDING item re-dispatch after restart:** PENDING items are not persisted. After restart they are re-created by normal engine evaluation on CONTEXT_CHANGED. Existing behavior.

**outputMapping for non-JQExpression evaluators:** Lambda-mode (`inputTransformer`) mappings cannot be serialized. Lambda-mode is test-only; production definitions use JQ. Acceptable.

**BlackboardPlanConfigurer idempotency enforcement:** The contract that configurers must tolerate pre-populated `activeByBinding` entries is now explicit. Downstream configurer implementations must honour it. No enforcement mechanism is added in this PR — rely on contract documentation and contract tests.

---

### 7. Platform Coherence Review

- **Dual-trail audit protocol:** No new lifecycle transitions. Existing `markCompleted()` etc. already fire CDI events. ✅
- **`@DefaultBean` / no-op pattern:** `NoOpPlanItemStore.findDelegated()` and `findAllDelegated()` return empty — no hydration in deployments without a real store. ✅
- **Module tier rule:** all changes within Orchestration tier (engine, blackboard, work-adapter). Depend inward only. ✅
- **No migration tooling:** schema managed by Hibernate `drop-and-create`. ✅
- **`casehub-work` boundary:** no orchestration logic added to `casehub-work`; recovery logic is engine-side. ✅
- **`WorkItemService.findByCallerRef()` addition:** this is a narrowly scoped new method on the existing service. It does not cross the work/engine boundary — it's a query method that the engine-side recovery service calls. File casehub-work issue before starting #398 implementation.
