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
boundary. The JPA `@PrePersist` listener (`LedgerTraceListener`) calls
`LedgerTraceIdProvider.currentTraceId()` at persist time; since no span is active on the async
thread, it writes `null` into every `CaseLedgerEntry.traceId`.

### Design

Capture the trace ID **synchronously** on the originating thread (where the HTTP span is live),
carry it inside `CaseLifecycleEvent`, and set it explicitly on the entry before `save()`.
`LedgerTraceListener` respects a pre-set `traceId` value — it only writes when the field is null.

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

**All `CaseLifecycleEvent` fire-sites** — inject `LedgerTraceIdProvider`, capture before
`fireAsync()`:
```java
String traceId = traceIdProvider.currentTraceId().orElse(null);
lifecycleEvents.fireAsync(new CaseLifecycleEvent(caseId, cmd, evt, status, actor, role, traceId));
```

**`CaseLedgerEventCapture`** — set on entry before `ledgerRepo.save()`:
```java
entry.traceId = event.traceId();  // null-safe — LedgerTraceListener skips if null
```

### Files Changed
- `common/src/main/java/io/casehub/engine/internal/event/CaseLifecycleEvent.java`
- All `CaseLifecycleEvent` fire-sites (grep for `fireAsync(new CaseLifecycleEvent`)
- `ledger/src/main/java/io/casehub/ledger/service/CaseLedgerEventCapture.java`

### Test
New `@QuarkusTest` in `ledger/src/test/`: activate an OTel span, fire a `CaseLifecycleEvent`
synchronously via the CDI event bus, assert `CaseLedgerEntry.traceId` equals the span's trace ID.
Existing test `CaseLedgerEventCaptureTest` verifies the null path still works (no span → null
traceId).

---

## Fix 2 — engine#331: CapabilityTarget PlanItem stays RUNNING when retries exhausted

### Root Cause

`WorkerRetriesExhaustedEventHandler` (in `runtime/`) faults the **case** but never touches the
**PlanItem** in the blackboard. Two paths both leave the PlanItem in `RUNNING`:

1. Quartz exhausts retries → `QuartzWorkerExecutionJobListener` publishes `WORKER_RETRIES_EXHAUSTED`
2. `WorkerScheduleEventHandler` guard blocks the worker → also publishes `WORKER_RETRIES_EXHAUSTED`

In both cases `hasActivePlanItem()` returns true forever, re-triggering is blocked, and stage
autocomplete never fires.

### Design

New handler **`WorkerRetryExhaustionHandler`** in `blackboard/handler/`, following the exact
pattern of `PlanItemCompletionHandler`:

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
            evaluateStageAutocomplete(event.caseId(), plan, planItemId);
            planItemCompletedEvents.fireAsync(
                new PlanItemCompletedEvent(event.caseId(), planItemId, event.workerId()));
        });

        return Uni.createFrom().voidItem();
    }
}
```

`event.workerId()` equals the worker name, which is the tracking key registered in
`BlackboardRegistry.indexForCompletion()`. No changes to `WorkerRetriesExhaustedEventHandler` in
`runtime/` — the tier boundary between runtime and blackboard is preserved.

`evaluateStageAutocomplete()` is extracted from `PlanItemCompletionHandler` into a shared
package-private helper method on a new `StageAutocompleteEvaluator` `@ApplicationScoped` bean
in `io.casehub.blackboard.handler`. Both `PlanItemCompletionHandler` and
`WorkerRetryExhaustionHandler` inject and delegate to it. This avoids duplicating the terminal-
state logic and ensures both handlers stay in sync on future changes.

### Files Changed
- `blackboard/src/main/java/io/casehub/blackboard/handler/StageAutocompleteEvaluator.java` (new)
- `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandler.java` (new)
- `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java` (delegates to `StageAutocompleteEvaluator`)
- `blackboard/src/test/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandlerTest.java` (new)

---

## Fix 3 — engine#248: JpaSubCaseGroupRepository atomicity

### Root Cause

`incrementCompleted()` and `incrementRejected()` use read-modify-write:
```java
e.completedCount++;   // read entity → mutate → Hibernate dirty write
```
Under PostgreSQL READ COMMITTED, two concurrent transactions can both read `completedCount = N`,
both write `N+1`, and one increment is silently lost. The `markPolicyTriggered()` conditional
UPDATE is already correct — the race is in the counter increments.

### Design

Replace read-modify-write with **atomic JPQL increments** + re-read:

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
                .map(this::toDomain);
        }));
}
```

Same pattern for `incrementRejected()`. The JPQL UPDATE is a single atomic SQL statement,
serialized at the database level — no retry logic, no lock acquisition overhead.

Also add `synchronized` to `incrementCompleted()` and `incrementRejected()` in
`MemorySubCaseGroupRepository` — in-memory tests run on Vert.x IO threads and the current
unsynchronized `HashMap` mutation is a data race.

### Files Changed
- `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepository.java`
- `persistence-memory/src/main/java/io/casehub/persistence/memory/MemorySubCaseGroupRepository.java`
- `persistence-hibernate/src/test/java/io/casehub/persistence/jpa/JpaSubCaseGroupRepositoryTest.java`

---

## Fix 4 — engine#338: WorkItemLifecycleAdapter collapses REJECTED/EXPIRED/ESCALATED → FAULTED

### Root Cause

`applyStatus()` maps three semantically distinct `WorkItemStatus` values to a single
`PlanItem.markFaulted()`:
```java
case REJECTED, EXPIRED -> item.markFaulted();
```
ESCALATED is already correctly excluded. REJECTED (intentional human refusal) and EXPIRED
(deadline missed) are not the same — case definitions cannot distinguish them.

### Design

#### 4a — Add `PlanItemStatus.REJECTED`

```java
public enum PlanItemStatus {
    PENDING, RUNNING, DELEGATED, COMPLETED, FAULTED, REJECTED, CANCELLED
}
```

REJECTED is semantically: an external actor (human or group policy) explicitly refused the work.
It is terminal. Distinct from FAULTED (computation failure or timeout).

#### 4b — Add `PlanItem.markRejected()`

Transitions from `DELEGATED` only — REJECTED arrives via human task refusal (WorkItemStatus.REJECTED)
or M-of-N group policy failure. Capability targets do not reject; they fault or complete.

```java
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

Add `PlanItemStatus.REJECTED` to the existing terminal-state exclusion:
```java
.filter(pi ->
    pi.getStatus() != PlanItemStatus.COMPLETED &&
    pi.getStatus() != PlanItemStatus.FAULTED    &&
    pi.getStatus() != PlanItemStatus.REJECTED   &&  // new
    pi.getStatus() != PlanItemStatus.CANCELLED)
```

#### 4f — Update `evaluateStageAutocomplete()` — terminal-state gate

Currently autocomplete fires only when all required items are `COMPLETED`. Updated to fire when
all required items have reached **any terminal state** (COMPLETED, REJECTED, FAULTED, CANCELLED) —
the stage has concluded regardless of outcome:

```java
private static boolean isTerminal(PlanItemStatus s) {
    return s == COMPLETED || s == REJECTED || s == FAULTED || s == CANCELLED;
}

// In allDone check:
.map(pi -> isTerminal(pi.getStatus()))
```

This change applies to `PlanItemCompletionHandler.evaluateStageAutocomplete()` and to the
same logic in the new `WorkerRetryExhaustionHandler` (Fix 2).

#### Audit: all switch/match on PlanItemStatus

Before implementation: grep for all switches on `PlanItemStatus` and `item.getStatus()` across
the codebase. Any exhaustive switch must be updated to handle `REJECTED`.

### Files Changed
- `common/src/main/java/io/casehub/engine/internal/model/PlanItemStatus.java`
- `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`
- `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java`
- `blackboard/src/main/java/io/casehub/blackboard/subcase/SubCaseCompletionService.java`
- `work-adapter/src/main/java/io/casehub/workadapter/WorkItemLifecycleAdapter.java`
- `work-adapter/src/test/java/io/casehub/workadapter/WorkItemLifecycleAdapterTest.java`

---

## Implementation Order

Fix these in dependency order — each builds on the previous:

1. **#338** — `PlanItemStatus.REJECTED` + `markRejected()` first; everything else references it
2. **#331** — `WorkerRetryExhaustionHandler` uses updated `evaluateStageAutocomplete()` from #338
3. **#342** — independent; `CaseLifecycleEvent` change and ledger fix
4. **#248** — independent; pure persistence layer

Commits: one per issue, each `Closes #N`.
