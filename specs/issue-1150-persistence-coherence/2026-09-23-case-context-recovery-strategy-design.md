# CaseContextRecoveryStrategy SPI — Design Spec

**Issue:** casehubio/engine#1166
**Epic:** casehubio/engine#1150 — persistence layer coherence
**Date:** 2026-09-23

## Problem

On cache miss, `DefaultWorkerExecutionRecoveryService.rebuildStateContext()` replays the event log to reconstruct `CaseContext`. This is O(N) in event count and fragile — every context-mutating event type must have a replay handler, and nothing enforces this. Two known bugs prove it breaks silently: #1151 (`SCOPED_WORKER_OUTPUT` not replayed) and #1152 (`CONTEXT_SIGNAL_APPLIED` doesn't store values for replay).

## Solution

Introduce a `CaseContextRecoveryStrategy` SPI with two implementations: database snapshot (default, O(1) recovery) and event-log replay (experimental, current behavior extracted).

## SPI

```java
package io.casehub.engine.common.spi.recovery;

public interface CaseContextRecoveryStrategy {
    CaseContext recover(UUID caseId, String tenancyId);
    void onContextChanged(UUID caseId, String tenancyId, CaseContext context);
}
```

**Placement:** `common-core/spi/recovery/` — alongside `WorkerExecutionRecoveryService`.

### `recover(UUID caseId, String tenancyId)`

Called by `DefaultWorkerExecutionRecoveryService.loadOrRestoreCaseInstance()` on cache miss. Replaces the inline `rebuildStateContext()` call. Returns a fully reconstructed `CaseContext`.

### `onContextChanged(UUID caseId, String tenancyId, CaseContext context)`

Called synchronously at each `CaseContextChangedEvent` dispatch site, before the async Vert.x event bus publish. Part of the blackboard propagation architecture — the recovery strategy participates in the context-change cycle alongside binding evaluation, worker dispatch, and milestone tracking.

The snapshot strategy persists here. The event-log strategy is a no-op (the event append already happened). Future strategies could stream changes, replicate state, or maintain materialized views.

## Implementations

### SnapshotRecoveryStrategy

**Module:** `runtime-core` (framework-neutral)
**Annotation:** `@DefaultBean @ApplicationScoped`
**Configuration key:** `casehub.context.recovery-strategy=snapshot`

**recover:** Loads the CaseInstance via `CrossTenantCaseInstanceRepository.findByUuid(caseId)`, reads the `contextSnapshot` field (a `JsonNode`), deserializes via `CaseContextImpl.fromLayerDocument(snapshot)`. Falls back to event-log replay if no snapshot exists (first-run migration path).

**onContextChanged:** Serializes `context.asJsonNode()` and stores on the CaseInstance's `contextSnapshot` field. The snapshot is persisted in the next `updateStateAndAppendEvent` transaction — no additional DB round-trip.

### EventLogReplayRecoveryStrategy

**Module:** `runtime` (Quarkus-specific, depends on CDI wiring for `@CrossTenant` repos)
**Annotation:** `@Experimental @ApplicationScoped`
**Configuration key:** `casehub.context.recovery-strategy=event-log`

**recover:** Current `DefaultWorkerExecutionRecoveryService.rebuildStateContext()` logic, extracted verbatim. Queries event log for `CASE_STARTED`, `WORKER_EXECUTION_COMPLETED`, `SUBCASE_COMPLETED`, `SIGNAL_RECEIVED`, `MILESTONE_ACTIVATED`, `MILESTONE_COMPLETED`, `MILESTONE_SLA_VIOLATED` events and replays them in sequence order.

**onContextChanged:** No-op. Events are already appended by the mutation path.

Known gaps in this path (#1151, #1152, #1154) become improvements to the experimental strategy, not blockers for the platform.

## Storage

### CaseInstance (domain object)

Add field:
```java
private JsonNode contextSnapshot;
```

With getter/setter. This field is set by the snapshot strategy's `onContextChanged` and read by its `recover`.

### CaseInstanceEntity (JPA)

Add column:
```java
@Column(name = "context_snapshot", columnDefinition = "jsonb")
@JdbcTypeCode(SqlTypes.JSON)
public JsonNode contextSnapshot;
```

Nullable — null until the first context change is persisted (or when using event-log strategy).

### Repository mappings

- `JpaCaseInstanceRepository`: map `contextSnapshot` in `toEntity`/`fromEntity`
- `SpringJpaCaseInstanceRepository`: same mapping
- `InMemoryCaseInstanceRepository`: no mapping needed — stores `CaseInstance` directly

## Configuration

```properties
# default — durable, no replay gaps
casehub.context.recovery-strategy=snapshot

# opt-in — fast writes, experimental
casehub.context.recovery-strategy=event-log
```

CDI producer reads the config value and selects the appropriate `CaseContextRecoveryStrategy` bean. When unset, defaults to `snapshot` via `@DefaultBean`.

## Integration

### Recovery path (loadOrRestoreCaseInstance)

Current:
```java
CaseContext stateContext = rebuildStateContext(caseId);
```

After:
```java
CaseContext stateContext = recoveryStrategy.recover(caseId, instance.tenancyId);
```

The `rebuildStateContext` method is removed from `DefaultWorkerExecutionRecoveryService` and extracted into `EventLogReplayRecoveryStrategy`.

### Propagation path (CaseContextChangedEvent dispatch)

Each site that constructs and dispatches a `CaseContextChangedEvent` also calls `recoveryStrategy.onContextChanged()` synchronously before the `eventDispatcher.dispatch()` call. The strategy call is within the caller's thread, sharing the same transaction boundary as the preceding event log append.

Dispatch sites (~15) are in: `WorkflowExecutionCompletedHandler`, `CaseStartedEventHandler`, `SignalReceivedEventHandler`, `MilestoneCompletedEventHandler`, `MilestoneSLAViolatedEventHandler`, `MilestoneActivatedEventHandler`, `CaseStatusChangedHandler`, `ScopedWorkerOutputHandler`, `ActionGateRejectedHandler`, `ActionGateExpiredHandler`, `JudgmentCompletedHandler`, `JudgmentExpiredHandler`, `ContextSignalEventHandler`, `PlanItemCompletionApplier`, `WorkerOutcomeResolvedHandler`, `PlanItemCompletionHandler`, `DefaultRecoveryCoordinator`, `CaseContextChangedEventHandler` (local rules re-dispatch).

### Worker rescheduling

Unchanged. `recoverPendingScheduledWorkers()` is independent of context recovery — it queries the event log for pending `WORKER_SCHEDULED` events regardless of strategy.

## Future: Delta-Based Replay

The SPI supports a future third strategy that captures state changes as function + input parameters (the Drools command-based persistence pattern). Each mutation is recorded as input values to a known function; replay re-applies those values. More space-efficient than full snapshots for high-frequency mutation workloads. The `onContextChanged` signature can evolve to carry structured deltas when this is implemented. Pre-release platform, so breaking the signature costs nothing.

## Files Changed

| File | Change |
|------|--------|
| `common-core/.../spi/recovery/CaseContextRecoveryStrategy.java` | New SPI interface |
| `runtime-core/.../recovery/SnapshotRecoveryStrategy.java` | New — `@DefaultBean` snapshot implementation |
| `runtime/.../recovery/EventLogReplayRecoveryStrategy.java` | New — `@Experimental` extracted from `rebuildStateContext()` |
| `runtime/.../recovery/DefaultWorkerExecutionRecoveryService.java` | Inject strategy, delegate `recover()`, remove `rebuildStateContext()` |
| `common-core/.../internal/model/CaseInstance.java` | Add `contextSnapshot` field |
| `persistence-jpa-common/.../CaseInstanceEntity.java` | Add `context_snapshot` column |
| `persistence-hibernate/.../JpaCaseInstanceRepository.java` | Map `contextSnapshot` in toEntity/fromEntity |
| `persistence-spring-jpa/.../SpringJpaCaseInstanceRepository.java` | Map `contextSnapshot` in toEntity/fromEntity |
| ~15 event handler files | Add `strategy.onContextChanged()` before `eventDispatcher.dispatch()` |

## References

- `DefaultWorkerExecutionRecoveryService.java:121` — current `rebuildStateContext()` method
- `CaseContextChangedEvent.java:32` — blackboard propagation event
- `CaseInstanceEntity.java:47` — JPA entity for snapshot column
- `VertxEventDispatcher.java` — async event bus dispatch
- `WorkflowExecutionCompletedHandler.java:388` — representative dispatch site
- Issue #1166 — feature specification
- Issue #1150 — parent epic
- Issues #1151, #1152, #1154 — known event-log replay gaps
