# Harden CaseRecoveryService.unfault() — Design Spec

**Issue:** casehubio/engine#1183
**Epic:** casehubio/engine#210 — cancellation, timeout, and error recovery
**Date:** 2026-09-27

## Problem

`CaseRecoveryService.unfault()` transitions FAULTED → RUNNING but has three gaps:

1. **JPA persistence race** — state is set synchronously in memory, then dispatched async via Vert.x event bus (`eventBus.publish()` returns immediately). `DeadLetterReplayService.doReplay()` queries the DB via `CrossTenantCaseInstanceRepository.findByUuid()`, which creates a NEW `CaseInstance` from the entity — it never sees in-memory state. If DLQ replay runs before the async handler persists RUNNING, the DB still has FAULTED → replay rejected. This race is JPA-specific: in-memory tests pass because `InMemoryCaseInstanceRepository` implements both interfaces on the same backing `ConcurrentHashMap`.

2. **Torn-down execution infrastructure** — when the case went FAULTED (terminal), `CaseStatusChangedHandler` evicted the case from all engine registries (observations, signals, rules, activity tracking, convergence, stigmergy) and tore down execution infrastructure (channels closed, grants revoked, triggers cancelled, scoped workers terminated). `unfault()` only re-registers the `CaseCompletionTracker`. The case runs but subsystems are blind and time-based bindings are permanently lost.

3. **CANCELLED policy undocumented** — only FAULTED → RUNNING is supported, but the rejection of CANCELLED cases is implicit (generic "not FAULTED" log). The policy needs explicit documentation.

## Constraint

`unfault()` is in `runtime-core`, not `runtime`. It cannot depend on Vert.x or Quarkus-specific APIs. All new dependencies must be SPIs from `common-core` or `api`.

## Solution

### 1. Synchronous State Persistence (D5)

Add `CaseInstanceRepository` as a dependency. Persist state to DB before dispatching the async event:

```java
instance.setState(CaseStatus.RUNNING);
caseInstanceRepository.update(instance, instance.tenancyId);
caseInstanceCache.put(instance);
```

The `tenancyId` comes from the instance itself (populated by `CrossTenantCaseInstanceRepository.findByUuid()` at load time — `JpaCrossTenantCaseInstanceRepository.java:63`).

The handler's subsequent `updateStateAndAppendEvent` is idempotent — it writes RUNNING again (same value) and appends the EventLog entry. The EventLog remains the handler's responsibility; `unfault()` does not write one.

### 2. Infrastructure Restoration (D7)

Mirror `CaseStartedEventHandler.onCaseStarted()` (lines 101-104). Add `CaseChannelProvider` and `SchedulerService` as dependencies:

```java
caseChannelProvider.openChannel(instance.getUuid(), "coordination");
schedulerService.registerScheduledTriggers(instance);
```

`registerScheduledTriggers` reads bindings from the `CaseDefinition` and re-registers all `ScheduleTrigger`-based bindings with the `JobScheduler`. Idempotent — the scheduler handles duplicate registrations.

`openChannel` re-opens the coordination channel. Channels closed during terminal cleanup are not automatically re-created.

Remaining infrastructure is re-established by the existing async event chain:

1. `CaseStatusChanged` event → handler persists EventLog + dispatches `CaseContextChangedEvent`
2. `CaseContextChangedEvent` → `CaseContextChangedEventHandler.evaluateAndDispatch()` re-evaluates ALL `ContextChangeTrigger` bindings against the current context → dispatches `WorkerScheduleEvent` for matches
3. Workers execute → naturally re-register observations, rules, signals, activity tracking, convergence state

Grants are re-issued during worker dispatch. Data channels are re-created during worker execution. Scoped workers are re-started when their `ScopeActivatedTrigger` bindings re-evaluate.

### 3. CANCELLED Policy (D6)

CANCELLED remains permanently terminal. Add a specific rejection path:

```java
if (instance.getState() == CaseStatus.CANCELLED) {
    LOG.warnf("Cannot recover CANCELLED case %s — cancellation is an administrative "
        + "decision. Start a new instance with the same context to retry.", caseId);
    return Optional.empty();
}
if (instance.getState() != CaseStatus.FAULTED) {
    LOG.warnf("Unfault: case %s is %s, not FAULTED", caseId, instance.getState());
    return Optional.empty();
}
```

Javadoc on `unfault()` documents the exclusion: FAULTED is system-initiated (recoverable), CANCELLED is human-initiated (terminal by design).

## Revised unfault() Flow

```java
public Optional<CaseInstance> unfault(UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        instance = crossTenantRepository.findByUuid(caseId).orElse(null);
    }
    if (instance == null) { /* not found */ return Optional.empty(); }
    if (instance.getState() == CaseStatus.CANCELLED) { /* specific WARN */ return Optional.empty(); }
    if (instance.getState() != CaseStatus.FAULTED) { /* generic WARN */ return Optional.empty(); }

    String oldStatus = instance.getState().name();
    instance.setState(CaseStatus.RUNNING);

    // 1. Persist synchronously — eliminates JPA race
    caseInstanceRepository.update(instance, instance.tenancyId);
    caseInstanceCache.put(instance);

    // 2. Re-register completion tracker
    caseCompletionTracker.remove(caseId);
    caseCompletionTracker.register(caseId);

    // 3. Restore execution infrastructure
    caseChannelProvider.openChannel(caseId, "coordination");
    schedulerService.registerScheduledTriggers(instance);

    // 4. Dispatch for EventLog + binding re-evaluation
    eventDispatcher.dispatch(new CaseStatusChanged(instance, oldStatus, CaseStatus.RUNNING.name()));

    LOG.infof("Case unfaulted: caseId=%s (%s → RUNNING)", caseId, oldStatus);
    return Optional.of(instance);
}
```

## Dependencies Changed

`CaseRecoveryService` constructor grows from 4 to 7 parameters:

| Dependency | Source | Purpose |
|------------|--------|---------|
| `CaseInstanceCache` | existing | Instance lookup/caching |
| `CrossTenantCaseInstanceRepository` | existing | Cross-tenant instance find |
| `CaseCompletionTracker` | existing | Completion tracking |
| `EventDispatcher` | existing | Async event dispatch |
| `CaseInstanceRepository` | **new** | Synchronous state persist |
| `CaseChannelProvider` | **new** | Channel re-opening |
| `SchedulerService` | **new** | Trigger re-registration |

`RuntimeBeans.caseRecoveryService()` producer method updated to wire the three new dependencies.

## Files Changed

| File | Change |
|------|--------|
| `runtime-core/.../recovery/CaseRecoveryService.java` | Add 3 dependencies. Persist state synchronously. Add cache put. Add infrastructure restoration. Add CANCELLED-specific rejection. Update javadoc. |
| `runtime/.../quarkus/RuntimeBeans.java` | Wire 3 new dependencies into `caseRecoveryService()` producer. |
| `runtime-core/src/test/.../CaseRecoveryServiceTest.java` | New: test synchronous persistence, CANCELLED rejection, infrastructure restoration, idempotent handler. |
| `resilience/src/test/.../DeadLetterReplayIntegrationTest.java` | Update: verify unfault + replay succeeds without race (may need JPA-specific variant). |

## What This Does NOT Cover

- **Durable store context re-opening** — `CaseContextStore.close()` is a default no-op. For future durable stores that override `close()`, the context would need re-creation via `CaseContextImpl.loadFromStore()`. Deferred until durable stores are deployed.
- **Worker-specific state** — observations, rules, signals, convergence state are re-populated by workers during re-execution, not by unfault(). History in `ContextHistoryBuffer` is lost (reset from the re-bind point).

## References

- `CaseRecoveryService.java:62` — current unfault() implementation
- `VertxEventDispatcher.java:101` — `eventBus.publish()` is async
- `CaseStatusChangedEventBusAdapter.java:29` — `@ConsumeEvent(blocking=true)` — worker thread
- `JpaCrossTenantCaseInstanceRepository.java:60` — `fromEntity` creates NEW object from DB
- `InMemoryCaseInstanceRepository.java:36` — implements both interfaces (masks race in tests)
- `DeadLetterReplayService.java:121-132` — DB query + terminal check
- `CaseStartedEventHandler.java:101-104` — initial channel + trigger setup
- `CaseStatusChangedHandler.java:195-243` — terminal cleanup (what needs reversing)
- `CaseContextChangedEventHandler.java:248-310` — binding re-evaluation on RUNNING
- `CaseContextStore.java:60` — `close()` default no-op
- `JpaCaseInstanceRepository.java:86` — `update()` method
- `SchedulerService.java:56` — `registerScheduledTriggers()`
- decisions.md — D5-D7
