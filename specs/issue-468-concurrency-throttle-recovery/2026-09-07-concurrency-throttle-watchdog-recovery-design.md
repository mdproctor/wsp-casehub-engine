# Concurrency Throttle and Watchdog→Recovery Bridge

**Epic:** casehubio/parent#468
**Engine issues:** #1043 (concurrency budget), #1044 (watchdog bridge)
**Date:** 2026-09-07

## Overview

Two independent mechanisms that compose through the existing CONTEXT_CHANGED execution loop:

1. **Concurrency budget** (pre-dispatch) — limits how many bindings a case can dispatch simultaneously, and lets external providers (claudony) advertise session-level capacity.
2. **Watchdog→recovery bridge** (post-dispatch) — translates qhorus watchdog alerts for hung workers into synthetic `Expired` outcomes, feeding the existing failure → retry → RecoveryCoordinator pipeline.

Neither mechanism depends on the other. They compose naturally: watchdog-faulted PlanItems reduce active count, freeing budget. Worker completions trigger CONTEXT_CHANGED, which re-evaluates bindings and re-checks budget.

## Part 1: Concurrency Budget (engine#1043)

### Case-Level Cap

Engine-internal. Not an SPI. Configured per case definition.

**CaseDefinition field:**
```java
// Nullable — null means unlimited (backward compat)
Integer maxConcurrentDispatches;
```

**YAML:**
```yaml
spec:
  maxConcurrentDispatches: 5
```

**Integration point:** `CaseContextChangedEventHandler.rules()`, after `loopControl.select()` returns the selected bindings, before dispatch.

**Active count computation:** Count PlanItems in non-terminal, resource-consuming states: `RUNNING`, `DISPATCHING`, `DELEGATED`. Exclude `PENDING` (not consuming resources), `COMPLETED`, `FAULTED`, `CANCELLED`, `OBSOLETE`, `SKIPPED`. Query via `BlackboardRegistry` → `CasePlanModel` for the case.

**Dispatch flow:**
```
selected = loopControl.select(planCtx, eligible)
if (definition.getMaxConcurrentDispatches() != null) {
    int active = countResourceConsumingPlanItems(caseId)
    int caseBudget = definition.getMaxConcurrentDispatches() - active
    if (caseBudget <= 0) return  // all slots occupied
    if (selected.size() > caseBudget) selected = selected.subList(0, caseBudget)
}
```

**No TOCTOU race:** `CaseEvaluationSerializer` guarantees one evaluation per case at a time (per-case `ReentrantLock` with coalescing). The PlanItem count is accurate at check time within a single case.

**Priority when budget-constrained:** The planning strategy's output order determines which bindings fire first. `ChoreographyStrategy` returns bindings in declaration order. `SequentialPlanningStrategy` returns one at a time. No new priority mechanism needed — the strategy already owns ordering semantics.

**Deferred bindings:** Bindings that can't be admitted simply wait. When a dispatched worker completes, it updates context → CONTEXT_CHANGED fires → bindings re-evaluate → budget re-checks. The loop is self-correcting. No explicit queue.

### External Dispatch Budget SPI

For session-level capacity providers (claudony). Lives in `engine-api`.

**SPI:**
```java
package io.casehub.api.spi;

public interface DispatchBudget {
    int availableCapacity(DispatchBudgetQuery query);
}

public record DispatchBudgetQuery(UUID caseId, String tenancyId) {}
```

**Default:** `@DefaultBean @ApplicationScoped` in engine runtime, returns `Integer.MAX_VALUE`.

**Consumer implementation:** Claudony provides `@ApplicationScoped` checking its session pool. Displaces the `@DefaultBean` automatically.

**Integration point:** Same location in `rules()`, after case-level cap:
```
int externalBudget = dispatchBudget.availableCapacity(
    new DispatchBudgetQuery(caseInstance.getUuid(), caseInstance.tenancyId))
int admitted = Math.min(selected.size(), Math.min(caseBudget, externalBudget))
selected = selected.subList(0, admitted)
```

**Advisory semantics:** The capacity query prevents wasted work (routing strategy evaluation, CBR retrieval, EventLog creation, PlanItem persistence for workers that can't submit). Cross-case TOCTOU races can briefly over-dispatch — `WorkerExecutionManager.submit()` is the hard gate. The race window is milliseconds and the provider handles overflow gracefully (queue or block at submit).

### Not in scope

- Per-capability throttling (e.g., max 3 concurrent web-search calls across all cases) — requires cross-case state tracking. Future issue.
- Per-tenancy budget — future concern for multi-tenant deployments.

## Part 2: Watchdog→Recovery Bridge (engine#1044)

### Scope

Worker-hung conditions only. Six of twelve watchdog conditions have clear automated responses:

| Condition | What's detected | Bridge action |
|-----------|---|---|
| AGENT_STALE | Agent instance hasn't reported activity | Synthetic Expired for matched workers |
| BARRIER_STUCK | Barrier channel waiting for missing contributors | Synthetic Expired for missing contributors |
| LOOP_DETECTED | Agent repeating similar messages | Synthetic Expired for looping sender |
| CONVERSATION_STALL | Open commitments with no progress | Synthetic Expired for stalled obligors |
| ECHO_CHAMBER | Multiple agents echoing similar content | Synthetic Expired for participants |
| CIRCULAR_DELEGATION | Delegation chain forms a cycle | Synthetic Expired for cycle participants |

Six case-level conditions (CONTEXT_PRESSURE, CHANNEL_IDLE, QUEUE_DEPTH, OBLIGATION_FAN_OUT, APPROVAL_PENDING, DELIVERY_LAG) have no engine-side automated response in v1. Tracked in engine#1057.

### Alert Delivery

Qhorus's `WatchdogEvaluationService.fireAlert()` already calls `alertEvents.fireAsync(new WatchdogAlertEvent(...))`. The engine bridge is a CDI observer:

```java
@ApplicationScoped
public class WatchdogRecoveryBridge {
    void onWatchdogAlert(@ObservesAsync WatchdogAlertEvent event) { ... }
}
```

No new delivery mechanism. No new SPI. The bridge observes what qhorus already publishes.

### Qhorus-API Change (qhorus#433)

Add `@Nullable UUID caseId` to `WatchdogAlertEvent`:

```java
public record WatchdogAlertEvent(
    UUID watchdogId,
    String targetName,
    String notificationChannel,
    String summary,
    Instant firedAt,
    AlertContext context,
    @Nullable UUID caseId    // NEW — resolved from channel metadata
) { ... }
```

`WatchdogEvaluationService` resolves caseId from the channel's metadata (set by engine's `CaseChannelProvider.openChannel()`). Per-channel alerts carry caseId; cross-channel alerts (target="*") carry null.

Backward-compatible: existing 6-arg constructor delegates with null caseId.

### Identity Resolution

The bridge needs to map watchdog agent identities to engine PlanItems:

| Alert context field | Identity space | Maps to `PlanItem.executorName()`? |
|---|---|---|
| `sender` (LOOP_DETECTED, ECHO_CHAMBER) | Channel message sender | YES — engine sets `from: workerName` in `postToChannel()` |
| `missingContributors` (BARRIER_STUCK) | Channel contributor names | YES — contributor = sender name |
| `cycle` (CIRCULAR_DELEGATION) | Channel senders | YES |
| `correlationIds` (CONVERSATION_STALL) | Commitment correlation | YES — correlationId = eventLogId, resolvable via EventLog |
| `staleInstanceIds` (AGENT_STALE) | Qhorus instance UUIDs | NO — best-effort match against executorName |

**Resolution flow:**
1. Get `caseId` from enriched `WatchdogAlertEvent`
2. Get `affectedAgentIds()` from `AlertContext`
3. Query active PlanItems for the case (via `PlanItemStore` or `BlackboardRegistry`)
4. Match agent IDs against `PlanItem.executorName()`
5. For each matched PlanItem, publish synthetic failure

**Graceful degradation:** No match → log warning, skip that agent. AGENT_STALE (instance IDs) may not match — acceptable for v1.

### Cancellation Mechanism

The bridge does NOT cancel the Quartz/db-scheduler job directly (interrupting a running virtual thread is unreliable). Instead, it publishes a synthetic `WorkflowExecutionCompleted` event:

```java
WorkflowExecutionCompleted event = new WorkflowExecutionCompleted(
    caseId,
    workerName,
    bindingName,
    capabilityName,       // from CaseDefinition binding lookup
    planItem.id(),
    WorkerOutcome.expired("Watchdog: " + conditionType),
    Map.of()              // empty protocolMetadata
);
eventBus.publish(EventBusAddresses.WORKER_EXECUTION_FINISHED, event);
```

This follows the established `QhorusMessageSignalBridge.handleWorkerOutcome()` pattern. The existing `WorkflowExecutionCompletedHandler` handles all side effects:
- PlanItem status transition (RUNNING → routes to `handleSemanticFailure()`)
- Failure classification (`DefaultFailureClassifier` classifies Expired as Transient)
- `_diagnostics` failure state recording
- Agent exclusion and reroute
- Compound completion evaluation
- Settlement tracking
- RecoveryCoordinator escalation (if retries exhaust)

**Idempotency:** When the original Quartz job eventually times out (its own timeout fires), it also publishes `WorkflowExecutionCompleted(Expired)`. The handler sees the PlanItem is already terminal and discards. No double-processing.

### Per-Condition Response Policy

Configurable on `CaseDefinition`:

```java
// Nullable — null means use defaults
Map<WatchdogConditionType, WatchdogAction> watchdogPolicy;
```

```java
public enum WatchdogAction {
    CANCEL_AFFECTED,  // cancel hung workers → failure pipeline
    IGNORE            // no engine action
}
```

**Defaults** (applied when `watchdogPolicy` is null or condition not in map):
- AGENT_STALE, BARRIER_STUCK, LOOP_DETECTED, CONVERSATION_STALL, ECHO_CHAMBER, CIRCULAR_DELEGATION → `CANCEL_AFFECTED`
- All case-level conditions → `IGNORE` (no worker to cancel)

**YAML:**
```yaml
spec:
  watchdogPolicy:
    AGENT_STALE: CANCEL_AFFECTED
    LOOP_DETECTED: IGNORE          # override: let the loop run
```

### Module Placement

The bridge lives in `engine-runtime` (not a separate module). It:
- Depends on `casehub-qhorus-api` (already a dependency — `QhorusMessageSignalBridge` imports from it)
- Observes `WatchdogAlertEvent` via CDI
- Uses existing engine services (`PlanItemStore`, `CaseDefinitionRegistry`, `EventBus`)

### Interaction Between Budget and Watchdog

The two features compose naturally through the existing loop:

1. Budget limits dispatch → fewer concurrent workers → less likely to trigger watchdog
2. Watchdog detects stall → synthetic Expired → PlanItem transitions to terminal → active count decreases → budget frees capacity
3. Worker completion → CONTEXT_CHANGED → budget re-checks → deferred bindings admitted

`OBLIGATION_FAN_OUT` (watchdog condition for too many concurrent obligations) is the watchdog detecting what the budget prevents. When the budget is configured, OBLIGATION_FAN_OUT shouldn't fire. When it does (budget not configured or too generous), the default response is IGNORE — the watchdog observes but doesn't act, because the right fix is configuring the budget, not cancelling workers.

## Audit Trail — Related Issues

| Issue | What | Status |
|---|---|---|
| parent#468 | Epic: concurrency throttle + watchdog bridge | Active |
| engine#1043 | ConcurrencyBudget contract + case-level cap | This spec |
| engine#1044 | Watchdog → recovery bridge | This spec |
| qhorus#433 | Add caseId to WatchdogAlertEvent | Required for engine#1044 |
| engine#1057 | Automated case-level watchdog response (WatchdogTrigger) | Deferred |
| engine#1066 | Migrate watchdog notification to platform notification service | Depends on parent#471 |
| parent#471 | Platform notification service — modular message types + delivery routes | Separate initiative |
| claudony#203 | Session-level spawn throttle (implements DispatchBudget) | Separate repo |
| devtown#185 | Domain-specific error classifier + worker callbacks | Separate repo |

## Testing Strategy

### engine#1043 tests
- `CaseContextChangedEventHandler` unit test: eligible bindings exceed `maxConcurrentDispatches` → only budget-permitted count dispatched
- `CaseContextChangedEventHandler` unit test: active PlanItems at max → zero dispatched
- `CaseContextChangedEventHandler` unit test: `DispatchBudget` returns lower than case budget → external budget wins
- `CaseContextChangedEventHandler` unit test: `maxConcurrentDispatches` null → unlimited (backward compat)
- `CaseContextChangedEventHandler` unit test: worker completes → next CONTEXT_CHANGED admits deferred binding
- `DispatchBudget` contract test: `@DefaultBean` returns `Integer.MAX_VALUE`
- YAML round-trip: `maxConcurrentDispatches` field parsed and serialized

### engine#1044 tests
- `WatchdogRecoveryBridge` unit test: AGENT_STALE alert → synthetic Expired published for matched PlanItem
- `WatchdogRecoveryBridge` unit test: LOOP_DETECTED alert → synthetic Expired for sender
- `WatchdogRecoveryBridge` unit test: alert with null caseId → logged warning, no action
- `WatchdogRecoveryBridge` unit test: no matching PlanItem for agentId → logged warning, no action
- `WatchdogRecoveryBridge` unit test: `watchdogPolicy` overrides default action → IGNORE skips cancellation
- `WatchdogRecoveryBridge` unit test: PlanItem already terminal → no duplicate event published
- Integration test: synthetic Expired flows through failure pipeline → retry → recovery coordinator
- Integration test: synthetic Expired + real timeout → only first processes, second discards (idempotency)

## References

- `CaseContextChangedEventHandler.rules()` (runtime) — dispatch fan-out, no throttle today
- `CaseEvaluationSerializer` (runtime) — per-case ReentrantLock + coalescing
- `RecoveryCoordinator` / `RecoveryContext` (common/spi/recovery/) — worker-failure-scoped recovery
- `WatchdogAlertRouter` / `WatchdogAlertEvent` / `AlertContext` (qhorus-api) — 12 condition types
- `WatchdogEvaluationService` (qhorus runtime) — fires CDI events, containment actions
- `QhorusMessageSignalBridge.handleWorkerOutcome()` (runtime) — established pattern: resolve via EventLog → publish WorkflowExecutionCompleted
- `PathologyCondition` / `handlePathologyAlert()` — existing pathology→context signal (not reused — different concern)
- Platform notification service (`io.casehub.platform.api.notification`) — human inbox model, separate initiative
- `JobScheduler.cancel(JobIdentifier)` — available but not used (thread interruption unreliable)
- `PlanItem.executorName()` — worker identity on PlanItems
- `WorkerExecutionManager.getActiveCaseIds()` / `getActiveWorkCount()` — per-worker tracking
