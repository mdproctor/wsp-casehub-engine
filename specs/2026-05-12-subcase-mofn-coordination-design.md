# Sub-case M-of-N Coordination — Design Spec

**Issue:** casehubio/engine#112  
**Date:** 2026-05-12  
**Status:** Approved for implementation

---

## Context

Engine#195 delivered basic sub-case execution wiring: a parent case can spawn a single child
`CaseInstance`, transition to WAITING, and resume when the child reaches a terminal state. All
142 blackboard tests pass.

Engine#112 extends this to:
1. **Parallel sub-cases with M-of-N coordination** — a parent spawns N child cases in a named
   group; the parent resumes when M of them complete.
2. **PropagationContext propagation** — child cases inherit the parent's `traceId` via
   `PropagationContext.createChild()`.
3. **Structural parent/child reference** — `CaseInstance` gains `parentCaseId` so children know
   their parent without querying EventLog.

The native M-of-N implementation mirrors `MultiInstanceGroupPolicy` from casehub-work: it is
justified as a hidden implementation detail (simple counter arithmetic). Complex orchestration
(conditional sequencing, branching) is out of scope here and belongs to a future quarkus-flow
worker path — the two paths are orthogonal.

---

## Out of Scope

- quarkus-flow worker integration (separate issue)
- `SubCasePlanningStrategy` — quarkus-flow handles complex coordination; not needed here
- `getChildCaseIds()` on `CaseInstance` — derivable from EventLog; deferred
- Telemetry / audit hooks on group lifecycle — additive later via `SubCaseGroupLifecycleEvent`
  consumers

---

## 1. Data Model

### 1.1 `CaseInstance` — add `parentCaseId`

**Module:** `casehub-engine-common`

```
UUID parentCaseId   // null for root cases; set by SubCaseExecutionHandler on spawn
```

Allows child cases to know their parent without an EventLog query. No change to existing
callers — field is null by default.

### 1.2 `SubCase` — add `groupId` and `requiredCount`

**Module:** `api`

```
String groupId        // null = ungrouped (existing single-child behaviour unchanged)
int totalInGroup      // total children that will be spawned; required when groupId is set
int requiredCount     // how many must complete; default = totalInGroup (allOf)
OnThresholdReached onThresholdReached  // KEEP (default) | CANCEL
```

`totalInGroup` is declared at design time so the `SubCaseGroup` entity is created with the
final `instanceCount` upfront. This eliminates the race between sibling spawning and early
child completion: `SubCaseGroupPolicy` always knows the total before it evaluates.

`OnThresholdReached`:
- `KEEP` — remaining running children continue to completion (default)
- `CANCEL` — remaining children are cancelled via `CaseHubRuntime` when threshold is reached

`requiredCount=1` gives anyOf semantics. `requiredCount=N` gives allOf. `requiredCount=M`
(1 < M < N) gives M-of-N.

### 1.3 `SubCaseGroup` POJO + `SubCaseGroupRepository` SPI

**Module:** `casehub-engine-common`

```java
public class SubCaseGroup {
    UUID parentCaseId;
    String groupId;
    int instanceCount;
    int requiredCount;
    int completedCount;
    int rejectedCount;
    boolean policyTriggered;
    OnThresholdReached onThresholdReached;
    Set<UUID> childCaseIds;   // child→group lookup; populated on spawn
}
```

SPI methods:
```java
SubCaseGroup getOrCreate(UUID parentCaseId, String groupId, int totalInGroup,
                         int requiredCount, OnThresholdReached onThresholdReached);
SubCaseGroup incrementCompleted(UUID parentCaseId, String groupId, UUID childCaseId);
SubCaseGroup incrementRejected(UUID parentCaseId, String groupId, UUID childCaseId);
Optional<SubCaseGroup> findByChildCaseId(UUID childCaseId);
```

`findByChildCaseId` uses the `childCaseIds` set stored on the group — no separate lookup
table needed.

### 1.4 Waiting semantics

| Path | `waitingForWorkId` value |
|------|-------------------------|
| Ungrouped sub-case (existing) | `childCaseId.toString()` |
| Grouped sub-case (new) | `groupId` |

Parent resumes only when `SubCaseGroupLifecycleEvent.COMPLETED` fires for the matching group.

---

## 2. Coordination Policy

**Module:** `casehub-engine-blackboard`

### 2.1 `SubCaseGroupLifecycleEvent`

CDI record:
```java
public record SubCaseGroupLifecycleEvent(
    UUID parentCaseId, String groupId,
    int instanceCount, int requiredCount,
    int completedCount, int rejectedCount,
    GroupStatus groupStatus   // IN_PROGRESS | COMPLETED | REJECTED
) {}
```

Fired **after** the repository write returns — never inside a transaction. Prevents spurious
events on failure/retry, consistent with casehub-work's discipline.

### 2.2 `SubCaseGroupPolicy`

Called by `SubCaseCompletionListener` on every terminal child event for grouped sub-cases.

Algorithm (mirrors `MultiInstanceGroupPolicy`):
1. Load group via `findByChildCaseId(childCaseId)`
2. If `policyTriggered` → return null (idempotency guard)
3. Increment `completedCount` or `rejectedCount` based on child's terminal status
4. Evaluate:
   - `completedCount >= requiredCount` → COMPLETED: set `policyTriggered`, apply
     `OnThresholdReached`, return COMPLETED event
   - `instanceCount - completedCount - rejectedCount < requiredCount - completedCount`
     (remaining < needed) → REJECTED: set `policyTriggered`, return REJECTED event
   - Otherwise → IN_PROGRESS: return IN_PROGRESS event
5. Caller fires the returned event after method returns

`OnThresholdReached.CANCEL`: on COMPLETED resolution, cancel remaining non-terminal children
via `CaseHubRuntime.cancel(childCaseId)`.

---

## 3. Execution Wiring

### 3.1 `CaseHubRuntime` — new `startCase` overload

**Module:** `api`

```java
CompletionStage<UUID> startCase(
    CaseDefinition definition,
    Map<String, Object> inputData,
    UUID parentCaseId,              // null for root cases
    PropagationContext propagationContext  // null → new root context created
);
```

Existing 2-arg overload delegates with `null, null` — no callers break. Implementation sets
`parentCaseId` and uses `propagationContext.createChild()` on the new `CaseInstance`.

### 3.2 `SubCaseExecutionHandler` — additions

On `SubCaseScheduleEvent`:

1. **PropagationContext**: call the new `startCase` overload, passing
   `parent.getUuid()` and `parent.getPropagationContext().createChild()`

2. **Grouped path** (`subCase.groupId()` non-null):
   - `subCaseGroupRepository.getOrCreate(parent.getUuid(), groupId, totalInGroup, requiredCount, onThresholdReached)`
     — `getOrCreate` uses `totalInGroup` as the fixed `instanceCount`; concurrent calls for
     the same group return the existing entity without modifying `instanceCount`
   - Register `childCaseId` against the group
   - Write `groupId` into `SUBCASE_STARTED` EventLog metadata — required so
     `SubCaseCompletionListener` can route the child's terminal event to the grouped path
   - Set `parent.waitingForWorkId = groupId` only if not already set (idempotent across
     sibling spawns — all siblings share the same `waitingForWorkId`)
   - Transition parent to WAITING and persist (only on first spawn for the group)

3. **Ungrouped path** (`groupId` null): existing logic unchanged

### 3.3 `SubCaseCompletionListener` — split paths

On terminal `CaseLifecycleEvent`:

1. Look up `SUBCASE_STARTED` EventLog entry for the child
2. If `groupId` present in entry metadata → **grouped path**:
   - Call `SubCaseGroupPolicy.process(childCaseId, childStatus)`
   - Fire the returned `SubCaseGroupLifecycleEvent`
   - If `groupStatus == COMPLETED` → call `CaseResumptionService.resumeIfWaiting(parent, groupId, ...)`
   - If `groupStatus == REJECTED` → fault the parent case
3. If no `groupId` → **ungrouped path**: existing `CaseResumptionService.resumeIfWaiting` call

`CaseResumptionService` is untouched.

---

## 4. Persistence

### 4.1 `casehub-engine-persistence-hibernate`

New `SubCaseGroupEntity` — Hibernate creates the table via `drop-and-create` (no migration
needed). `policyTriggered` column provides the idempotency guard. `childCaseIds` stored as
`@ElementCollection`.

`JpaSubCaseGroupRepository` implements `SubCaseGroupRepository` SPI.

### 4.2 `casehub-engine-persistence-memory`

`MemorySubCaseGroupRepository` — `ConcurrentHashMap<String, SubCaseGroup>` keyed by
`parentCaseId + ":" + groupId`. `getOrCreate` uses `computeIfAbsent`. Counter increments use
`synchronized` on the group object (consistent with `DefaultCasePlanModel`).

---

## 5. Testing

### 5.1 Unit tests (pure Java, no Quarkus)

**`SubCaseGroupPolicyTest`:**
- allOf: all N complete → COMPLETED
- anyOf (requiredCount=1): first completes → COMPLETED
- M-of-N: M complete → COMPLETED before remaining finish
- Rejection: rejectedCount makes threshold unreachable → REJECTED
- Idempotency: second call after `policyTriggered=true` → no-op
- `OnThresholdReached.CANCEL`: cancellation requested on remaining children

**`SubCaseGroupTest`:** POJO field validation, `requiredCount` defaults.

### 5.2 Integration tests (`@QuarkusTest`, `blackboard` module)

**`SubCaseParallelIntegrationTest`:** parent spawns 3 grouped children (`requiredCount=3`),
all complete → parent resumes COMPLETED.

**`SubCaseMofNIntegrationTest`:** parent spawns 3, `requiredCount=2`: first two complete →
parent resumes; third runs to completion uninterrupted (KEEP policy).

**`SubCasePropagationContextTest`:** child `CaseInstance.propagationContext.traceId` matches
parent's; `parentCaseId` on child is set correctly.

**Existing `SubCaseIntegrationTest`:** ungrouped single-child path — must stay green.

No Testcontainers — `casehub-engine-persistence-memory` provides in-memory SPI for all
blackboard integration tests.

---

## Module Impact Summary

| Module | Change |
|--------|--------|
| `api` | `CaseHubRuntime` new overload; `SubCase` gains `groupId`/`requiredCount`/`onThresholdReached`; `OnThresholdReached` enum |
| `casehub-engine-common` | `CaseInstance.parentCaseId`; `SubCaseGroup` POJO; `SubCaseGroupRepository` SPI |
| `casehub-engine-blackboard` | `SubCaseGroupPolicy`; `SubCaseGroupLifecycleEvent`; `SubCaseExecutionHandler` wiring; `SubCaseCompletionListener` split |
| `casehub-engine-persistence-hibernate` | `SubCaseGroupEntity`; `JpaSubCaseGroupRepository` |
| `casehub-engine-persistence-memory` | `MemorySubCaseGroupRepository` |

No changes to: `runtime`, `casehub-engine-scheduler-quartz`, `casehub-engine-work-adapter`,
`casehub-engine-ledger`, `casehub-engine-resilience`.
