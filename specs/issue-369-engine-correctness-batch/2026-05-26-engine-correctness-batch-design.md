# Design: Engine Correctness Batch — #342 #331 #248 #338

**Branch:** `issue-369-engine-correctness-batch`  
**Date:** 2026-05-26  
**Issues:** engine#342, engine#331, engine#248, engine#338  
**Umbrella:** engine#369

---

## Fix 1 — engine#342: CaseLedgerEntry.traceId always null

### Root Cause

`CaseLedgerEventCapture.onCaseLifecycleEvent()` is `@ObservesAsync` — it runs on a managed
executor thread. OTel context is `ThreadLocal` and is not propagated across the async CDI event
boundary. At persist time, the `LedgerTraceListener` (@EntityListeners on `LedgerEntry`, in
casehub-ledger) delegates to `LedgerEnricherPipeline`, which runs `TraceIdEnricher.enrich()`.
That enricher calls `LedgerTraceIdProvider.currentTraceId()` — returning empty because no span
is active on the async thread — and writes nothing. `CaseLedgerEntry.traceId` stays null on
every row.

`TraceIdEnricher.enrich()` is idempotent: it guards with `if (entry.traceId != null) return;`.
Pre-setting `entry.traceId` before `save()` is therefore sufficient — the enricher will not
overwrite a non-null value.

### Design

Capture the trace ID **synchronously** on the originating thread (where the HTTP span is live),
carry it inside `CaseLifecycleEvent`, and set it explicitly on the entry before `save()`.

**`CaseLifecycleEvent` record** (`common/`) — add `traceId` component:
```java
public record CaseLifecycleEvent(
    UUID caseId,
    String commandType,
    String eventType,
    String caseStatus,
    String actorId,
    String actorRole,
    String traceId)   // nullable — null when no active span at fire-site
{}
```

**All `CaseLifecycleEvent` instantiation sites** — inject `LedgerTraceIdProvider`, capture
before `fireAsync()`. Grep for `new CaseLifecycleEvent(` (covers both `fire()` and `fireAsync()`
call sites — all must be updated for compilation regardless):
```java
String traceId = traceIdProvider.currentTraceId().orElse(null);
lifecycleEvents.fireAsync(new CaseLifecycleEvent(caseId, cmd, evt, status, actor, role, traceId));
```

**`CaseLedgerEventCapture`** — set on entry before `ledgerRepo.save()`:
```java
entry.traceId = event.traceId();  // null-safe — TraceIdEnricher skips if non-null already set
```

### Test

New `@QuarkusTest` in `ledger/src/test/`: fire a `CaseLifecycleEvent` via **`fireAsync()`** (not
`fire()`) and call `.toCompletableFuture().get()` to wait for the `@ObservesAsync` observer to
complete. Assert `CaseLedgerEntry.traceId` equals the expected trace ID. Existing test
`CaseLedgerEventCaptureTest` verifies the null path (no traceId on event → null in entry).

### Files Changed
- `common/src/main/java/io/casehub/engine/internal/event/CaseLifecycleEvent.java`
- All `CaseLifecycleEvent` instantiation sites (grep: `new CaseLifecycleEvent(`)
- `ledger/src/main/java/io/casehub/ledger/service/CaseLedgerEventCapture.java`

---

## Fix 2 — engine#331: CapabilityTarget PlanItem stays RUNNING when retries exhausted

### Root Cause

`WorkerRetriesExhaustedEventHandler` (in `runtime/`) faults the **case** but never touches the
**PlanItem** in the blackboard. Two paths both leave the PlanItem in `RUNNING`:

1. Quartz exhausts retries → `QuartzWorkerExecutionJobListener` publishes `WORKER_RETRIES_EXHAUSTED`
2. `WorkerScheduleEventHandler` guard blocks the worker → also publishes `WORKER_RETRIES_EXHAUSTED`

In both cases `hasActivePlanItem()` returns true forever, re-triggering is blocked, and stage
autocomplete never fires.

### Key Invariant

`WorkerRetriesExhaustedEvent.workerId()` equals `worker.getName()`, which equals the tracking
key stored by `BlackboardRegistry.indexForCompletion()`. Traced:
- `QuartzWorkerExecutionManager.scheduleQuartzJob()` stores `worker.getName()` as `"workerId"`
  in the Quartz job data map
- `QuartzWorkerExecutionJobListener` reads `"workerId"` from the job data map and publishes
  `WorkerRetriesExhaustedEvent(caseId, workerId=worker.getName(), ...)`
- `WorkerScheduleEventHandler` guard path publishes `WorkerRetriesExhaustedEvent(uuid,
  worker.getName(), ...)`
- `BlackboardRegistry.indexForCompletion(caseId, worker.getName(), planItemId)` uses the same key

The lookup `registry.getPlanItemId(event.caseId(), event.workerId())` will find the PlanItem.

### Design

New handler **`WorkerRetryExhaustionHandler`** in `blackboard/handler/`, following the pattern
of `PlanItemCompletionHandler`. Delegates stage autocomplete to `StageAutocompleteEvaluator`
(see below). Does **not** fire `PlanItemCompletedEvent` — the fault is already signalled via
`CASE_STATUS_CHANGED`; firing a "completed" event after `markFaulted()` would mislead consumers.

```java
@ApplicationScoped
public class WorkerRetryExhaustionHandler {

    @ConsumeEvent(EventBusAddresses.WORKER_RETRIES_EXHAUSTED)
    public Uni<Void> onWorkerRetriesExhausted(WorkerRetriesExhaustedEvent event) {
        CasePlanModel plan = registry.get(event.caseId()).orElse(null);
        if (plan == null) return Uni.createFrom().voidItem();

        String planItemId = registry.getPlanItemId(event.caseId(), event.workerId()).orElse(null);
        if (planItemId == null) return Uni.createFrom().voidItem();

        plan.getPlanItem(planItemId).ifPresent(item -> {
            if (item.getStatus() != PlanItemStatus.RUNNING) return;
            item.markFaulted();
            stageAutocompleteEvaluator.evaluate(event.caseId(), plan, planItemId);
        });

        return Uni.createFrom().voidItem();
    }
}
```

**`StageAutocompleteEvaluator`** — new `@ApplicationScoped` bean in `io.casehub.blackboard.handler`.
Extracted from `PlanItemCompletionHandler` so both handlers share the logic. The `evaluate()`
method is **public** (not package-private) to avoid fragility with CDI proxy subclassing.
Uses the updated terminal-state gate (see Fix 4f).

`PlanItemCompletionHandler` refactored to inject and delegate to `StageAutocompleteEvaluator`.

### Files Changed
- `blackboard/src/main/java/io/casehub/blackboard/handler/StageAutocompleteEvaluator.java` (new)
- `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandler.java` (new)
- `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java` (delegates to `StageAutocompleteEvaluator`)
- `blackboard/src/test/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandlerTest.java` (new)
- `blackboard/src/test/java/io/casehub/blackboard/handler/StageAutocompleteEvaluatorTest.java` (new)

---

## Fix 3 — engine#248: JpaSubCaseGroupRepository atomicity

### Root Cause

`incrementCompleted()` and `incrementRejected()` use read-modify-write (`e.completedCount++`).
Under PostgreSQL READ COMMITTED, two concurrent transactions can both read `completedCount = N`,
both write `N+1`, and one increment is silently lost. The `markPolicyTriggered()` conditional
UPDATE is already correct — the race is in the counter increments.

`MemorySubCaseGroupRepository` already uses `synchronized(g)` on both increment methods —
no changes needed there.

### Design

Replace read-modify-write with **atomic JPQL increments** + null-safe re-read:

```java
@Override
public Uni<SubCaseGroup> incrementCompleted(UUID parentCaseId, String groupId) {
    return Panache.withTransaction(() ->
        SubCaseGroupEntity.update(
            "completedCount = completedCount + 1 WHERE parentCaseId = ?1 AND groupId = ?2",
            parentCaseId, groupId)
        .chain(count -> {
            if (count == 0)
                return Uni.createFrom().failure(
                    new IllegalStateException("Group not found: " + parentCaseId + ":" + groupId));
            return SubCaseGroupEntity
                .<SubCaseGroupEntity>find("parentCaseId = ?1 and groupId = ?2", parentCaseId, groupId)
                .firstResult()
                .onItem().ifNotNull().transform(this::toDomain)
                .onItem().ifNull().failWith(() ->
                    new IllegalStateException("Group vanished after increment: " + parentCaseId));
        }));
}
```

Same pattern for `incrementRejected()`.

**Re-read semantics note for callers:** The returned `SubCaseGroup` reflects the database state
at the moment of the SELECT, which may include increments from other concurrent transactions.
Callers must not assume the returned count equals "their" increment plus the previous value —
they may see a higher count if concurrent increments committed between the UPDATE and SELECT.
`SubCaseGroupPolicy.evaluate()` already handles this correctly: it reads `completedCount` and
`requiredCount` and compares them, so a higher-than-expected count is safe.

### Files Changed
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- `persistence-hibernate/src/test/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepositoryTest.java`

---

## Fix 4 — engine#338: WorkItemLifecycleAdapter collapses REJECTED/EXPIRED/ESCALATED → FAULTED

### Root Cause

`applyStatus()` maps two semantically distinct `WorkItemStatus` values to `markFaulted()`:
```java
case REJECTED, EXPIRED -> item.markFaulted();
```
REJECTED (intentional human refusal) and EXPIRED (deadline missed) are not the same — case
definitions cannot distinguish them.

ESCALATED is correctly excluded at the filter before `applyStatus()` is ever called (lines 76–80
of `WorkItemLifecycleAdapter`). The WorkItem re-enters PENDING with new candidate groups; the
PlanItem stays in its current state until the WorkItem reaches a true terminal status. No state
transition occurs, and no PlanItem-level audit entry is written — this is intentional. ESCALATED
is not a terminal state; the engine does not close the plan item.

### Design

#### 4a — Add `PlanItemStatus.REJECTED`

```java
public enum PlanItemStatus {
    PENDING, RUNNING, DELEGATED, COMPLETED, FAULTED, REJECTED, CANCELLED
}
```

Stored as `@Enumerated(EnumType.STRING)` in `PlanItemEntity` — confirmed. No ordinal migration
required.

REJECTED is semantically distinct from FAULTED: an external actor (human or group policy)
explicitly refused the work. FAULTED means computation failure or timeout.

#### 4b — Add `PlanItem.markRejected()`

Transitions from `DELEGATED` only. REJECTED arrives via human task refusal
(`WorkItemStatus.REJECTED`) or M-of-N group policy failure (`GroupStatus.REJECTED`). Both of
these dispatch through `HumanTaskScheduleHandler` which calls `item.markDelegated()` before
the WorkItem is created — so the PlanItem is always in DELEGATED state when a rejection arrives.
CapabilityTarget PlanItems (RUNNING state) do not participate in SpawnGroups and have no
REJECTED path; they fault via retries exhaustion.

```java
/**
 * Transitions DELEGATED → REJECTED.
 *
 * DELEGATED-only guard rationale: only human task refusals (WorkItemStatus.REJECTED) and
 * M-of-N group threshold failures (GroupStatus.REJECTED on human task SpawnGroups) reach
 * this path. CapabilityTarget PlanItems are always in RUNNING — they fault via retries
 * exhaustion, never via rejection. If a group-of-capability-targets path is ever added,
 * this guard must be revisited to allow RUNNING → REJECTED.
 */
public void markRejected() {
    if (status != PlanItemStatus.DELEGATED) {
        throw new IllegalStateException(
            "Cannot reject from " + status + " (planItemId=" + planItemId + ")");
    }
    status = PlanItemStatus.REJECTED;
}
```

#### 4c — Update `WorkItemLifecycleAdapter.applyStatus()`

```java
case COMPLETED -> item.markCompleted();
case REJECTED  -> item.markRejected();   // intentional refusal — was markFaulted()
case EXPIRED   -> item.markFaulted();    // deadline missed — unchanged
case CANCELLED -> item.markCancelled();
```

#### 4d — Update `WorkItemLifecycleAdapter.applyGroupStatus()`

```java
case COMPLETED -> item.markCompleted();
case REJECTED  -> item.markRejected();   // group threshold unreachable — was markFaulted()
```

#### 4e — Update `SubCaseCompletionService.cancelPlanItemOnRejected()` terminal guard

Add `PlanItemStatus.REJECTED` to the terminal-state exclusion:
```java
.filter(pi ->
    pi.getStatus() != PlanItemStatus.COMPLETED &&
    pi.getStatus() != PlanItemStatus.FAULTED    &&
    pi.getStatus() != PlanItemStatus.REJECTED   &&  // new
    pi.getStatus() != PlanItemStatus.CANCELLED)
```

#### 4f — Update `StageAutocompleteEvaluator` — terminal-state gate

Currently autocomplete fires only when all required items are `COMPLETED`. Updated to fire when
all required items have reached **any terminal state** (COMPLETED, REJECTED, FAULTED, CANCELLED).
This is a deliberate architectural change: a stage has concluded when all its work is settled,
regardless of outcome. Case logic downstream inspects context to determine what happened.

```java
public static boolean isTerminal(PlanItemStatus s) {
    return s == COMPLETED || s == REJECTED || s == FAULTED || s == CANCELLED;
}
```

#### 4g — Audit all switches on PlanItemStatus

Before implementation, grep `getStatus\(\)\|PlanItemStatus\.` across all source files. Known
sites that must handle REJECTED:
- `PlanItemCompletionHandler.COMPLETABLE` — REJECTED must NOT be added (it's terminal, not
  completable-from)
- `SubCaseCompletionService.cancelPlanItemOnRejected()` — covered by 4e
- Any persistence or serialization code switching on status — covered by STRING enum

### Test Coverage

- `PlanItem.markRejected()` unit test: assert DELEGATED→REJECTED succeeds; assert PENDING,
  RUNNING, COMPLETED, FAULTED, CANCELLED all throw `IllegalStateException`
- `StageAutocompleteEvaluator` test: all-REJECTED, all-FAULTED, all-CANCELLED, and mixed
  terminal states all trigger autocomplete; any non-terminal item blocks it
- `SubCaseCompletionService.cancelPlanItemOnRejected()` test: REJECTED item is excluded
  from the cancel sweep
- `WorkItemLifecycleAdapterTest`: REJECTED→markRejected, EXPIRED→markFaulted, CANCELLED→markCancelled

### Files Changed
- `common/src/main/java/io/casehub/engine/internal/model/PlanItemStatus.java`
- `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`
- `blackboard/src/main/java/io/casehub/blackboard/handler/StageAutocompleteEvaluator.java` (created in Fix 2)
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionService.java`
- `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`
- `work-adapter/src/test/java/io/casehub/workadapter/WorkItemLifecycleAdapterTest.java`
- `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java` (markRejected tests)
- `blackboard/src/test/java/io/casehub/blackboard/subcase/SubCaseCompletionServiceTest.java`

---

## Implementation Order

Fix in dependency order — each builds on the previous:

1. **#338** — `PlanItemStatus.REJECTED` + `markRejected()` + `StageAutocompleteEvaluator` (with
   updated `isTerminal()` logic) + update `applyStatus()`, `applyGroupStatus()`,
   `cancelPlanItemOnRejected()`, and `PlanItemCompletionHandler` (delegates to evaluator)
2. **#331** — `WorkerRetryExhaustionHandler` injects the `StageAutocompleteEvaluator` from step 1.
   Both handlers share the same evaluator — Fix 2 adds the handler; Fix 4 already updated the
   evaluator logic. The extraction and logic update happen together in step 1.
3. **#342** — independent; `CaseLifecycleEvent` record change and ledger capture fix
4. **#248** — independent; pure persistence layer

Commits: one per issue, each `Closes #N`.
