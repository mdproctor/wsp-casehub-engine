# AgentWorkLifecycleListener + AgentWorkState + Role Lifecycle Events

**Issue:** engine#1285
**Parent epic:** engine#1281 (Swarm Mechanical Enforcement Layer)
**Cross-repo parent:** agency#377 (Distributed Per-Agent LLM Coordination)
**Depends on:** engine#1282 (coordination vocabulary types) — done, engine#1284 (SwarmTopologyProvider) — done

## Summary

Extend engine's per-agent lifecycle from 3 states (JOINING → ACTIVE → DEPARTED) to
include work-level states and role lifecycle events. Adds `AgentWorkState` enum,
`AgentWorkLifecycleListener` SPI, and an `AgentWorkStateNotifier` bridge interface so
the worker execution path (common-core) can drive state transitions tracked by
StigmergyCoordinator (runtime-core).

## AgentWorkState

**Package:** `io.casehub.api.model.stigmergy`
**Module:** api

```java
public enum AgentWorkState {
  IDLE,
  ASSIGNED,
  EXECUTING,
  ADAPTING,
  SIGNALING,
  WAITING
}
```

### State Machine

Valid transitions:

| From | To | Trigger |
|------|----|---------|
| IDLE | ASSIGNED | Task accepted by orchestrator |
| ASSIGNED | EXECUTING | Worker function invocation begins |
| ASSIGNED | IDLE | Assignment failed (resolution error before execution) |
| EXECUTING | IDLE | Task completion (success or failure) |
| EXECUTING | ADAPTING | Mid-execution adaptation (future) |
| ADAPTING | EXECUTING | Adaptation complete, resume execution |
| EXECUTING | SIGNALING | Signal deposit during execution (brief) |
| SIGNALING | EXECUTING | Signal deposited, resume |
| EXECUTING | WAITING | Blocked on dependency |
| WAITING | EXECUTING | Dependency resolved |
| any | IDLE | Agent departed (reset on lifecycle departure) |

Invalid transitions are rejected with `IllegalStateException`. The state machine is
enforced in StigmergyCoordinator.

## AgentState Extension

**Existing record** (`api/model/stigmergy/AgentState.java`):

```java
public record AgentState(
    String agentId,
    String bindingName,
    AgentLifecycleState state,
    AgentWorkState workState,          // NEW
    Instant joinedAt,
    @Nullable Instant activatedAt,
    @Nullable Instant departedAt,
    Instant lastActivity) {}
```

`workState` defaults to `IDLE` when an agent joins. On departure, it resets to `IDLE`.
The field is non-nullable — an agent always has a work state once it exists.

### Callers to update

All AgentState construction is in `StigmergyCoordinator`:
- `agentJoined()` — add `AgentWorkState.IDLE`
- `agentActivated()` — preserve current `workState`
- `agentDeparted()` — reset to `AgentWorkState.IDLE`

Tests that construct AgentState directly also need updating.

## AgentWorkLifecycleListener

**Package:** `io.casehub.api.spi.stigmergy`
**Module:** api

```java
public interface AgentWorkLifecycleListener {

  default void onWorkStateChanged(UUID caseId, String agentId,
      AgentWorkState from, AgentWorkState to) {}

  default void onTaskAssigned(UUID caseId, String agentId, String taskId,
      Map<String, Object> context) {}

  default void onTaskAdapted(UUID caseId, String agentId, String taskId,
      String adaptationReason) {}

  default void onRoleAdopted(UUID caseId, String agentId, String groupId,
      String role) {}

  default void onRoleReleased(UUID caseId, String agentId, String groupId,
      String role) {}
}
```

Follows `SwarmCoordinationListener` precedent: multi-instance via CDI `Instance`/Spring
`ObjectProvider`. All methods default no-op. Do not use `@DefaultBean` on implementations.

### Event dispatch

StigmergyCoordinator injects `jakarta.enterprise.inject.Instance<AgentWorkLifecycleListener>`
and iterates all instances on each event. Dispatch is synchronous (same as
SwarmCoordinationListener — the coordinator isn't CDI-managed today, so `@ObservesAsync`
isn't an option).

## AgentWorkStateNotifier — Cross-Module Bridge

**Problem:** WorkerExecutionOrchestrator (common-core) needs to drive state transitions,
but StigmergyCoordinator (runtime-core) implements them. common-core cannot depend on
runtime-core.

**Solution:** A thin api-level interface that the orchestrator calls and
StigmergyCoordinator implements.

**Package:** `io.casehub.api.spi.stigmergy`
**Module:** api

```java
public interface AgentWorkStateNotifier {

  void transitionWorkState(UUID caseId, String agentId, AgentWorkState targetState);

  void taskAssigned(UUID caseId, String agentId, String taskId,
      Map<String, Object> context);
}
```

- `transitionWorkState` — validates and applies the transition, fires
  `onWorkStateChanged` to all listeners
- `taskAssigned` — transitions to ASSIGNED and fires `onTaskAssigned`

### Wiring

- StigmergyCoordinator implements `AgentWorkStateNotifier`
- WorkerExecutionOrchestrator receives `AgentWorkStateNotifier` via constructor injection
- A `@DefaultBean` no-op implementation ensures non-swarm deployments don't break

### Agent identity resolution in the orchestrator

The orchestrator knows `bindingName` from event log metadata. In swarm contexts,
`bindingName` is the agent identity. Non-swarm executions have `bindingName == null` —
the notifier call is skipped (guard: `if (bindingName != null)`).

## Integration Points in WorkerExecutionOrchestrator

All three transitions happen inside `executeWorker()` because `bindingName` (the agent
identity) is only available after event log metadata extraction (~line 188).

### After bindingName extraction — task accepted

```
IDLE → ASSIGNED
```

After `bindingName` is extracted from event log metadata, before worker resolution:
```java
if (bindingName != null) {
    workStateNotifier.taskAssigned(caseId, bindingName, workerId, inputContext);
}
```

### Before workerExecutor.execute() — execution begins

```
ASSIGNED → EXECUTING
```

After worker and capability resolution, before `workerExecutor.execute()`:
```java
if (bindingName != null) {
    workStateNotifier.transitionWorkState(caseId, bindingName, AgentWorkState.EXECUTING);
}
```

### After publishOutcome() — task complete

```
EXECUTING → IDLE
```

After `publishOutcome()` returns. Also in `execute()`'s catch block — but only if
`bindingName` was resolved (the catch must guard against early failures before metadata
extraction):
```java
if (bindingName != null) {
    workStateNotifier.transitionWorkState(caseId, bindingName, AgentWorkState.IDLE);
}
```

### Not integrated in this issue

- ADAPTING — requires adaptation support (future: agency#383)
- SIGNALING — requires signal deposit hooks (signals are deposited by workers via
  WorkerRuntime, not the orchestrator)
- WAITING — requires dependency tracking (future)

These states exist in the enum for forward compatibility. The state machine validates
them, but no code fires those transitions in this issue.

## Role Lifecycle — SPI Surface Only

`onRoleAdopted` and `onRoleReleased` are defined on the listener interface but **no
integration points fire them in this issue**. They exist for:

- Hive topology group formation (future: agency#378+)
- MOISE+ dynamic role management (eidos#216)

The methods carry `groupId` to identify which organisational group the role belongs to.
An agent can hold roles in multiple groups simultaneously. Role lifecycle events are
distinct from `DetectedRole` (emergent behavioral roles in stigmergy) — these are
declared structural roles adopted and released at runtime.

## No-Op Default Implementation

**Package:** `io.casehub.engine.runtime.coordination`
**Module:** runtime-core

```java
public class NoOpAgentWorkStateNotifier implements AgentWorkStateNotifier {
    // all methods empty
}
```

Wired via `SpiDefaultBeans` with `@Produces @DefaultBean`, following the
`NoOpAgentCapabilityGuard` pattern from engine#1283.

## Test Strategy

1. **AgentWorkState transition validation** — unit test the state machine:
   valid transitions succeed, invalid transitions throw
2. **AgentState record** — verify workState field in construction and accessors
3. **StigmergyCoordinator work state tracking** — extend existing
   `StigmergyCoordinatorTest`: transition work states, verify listener dispatch,
   verify departure resets work state to IDLE
4. **WorkerExecutionOrchestrator integration** — verify IDLE → ASSIGNED → EXECUTING → IDLE
   cycle fires through the notifier for swarm agents (bindingName != null) and is
   skipped for non-swarm workers (bindingName == null)
5. **AgentWorkLifecycleListener dispatch** — verify multiple listeners all receive events

## Files Changed

### New files (api module)
- `api/src/main/java/io/casehub/api/model/stigmergy/AgentWorkState.java`
- `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java`
- `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkStateNotifier.java`

### New files (runtime-core module)
- `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentWorkStateNotifier.java`

### Modified files
- `api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java` — add workState field
- `runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinator.java` — implement AgentWorkStateNotifier, track work state, fire listener events
- `runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/SpiDefaultBeans.java` — add @DefaultBean producer for NoOpAgentWorkStateNotifier
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java` — inject AgentWorkStateNotifier, fire transitions
- `runtime-core/src/test/java/.../StigmergyCoordinatorTest.java` — work state tests
- `runtime-core/src/test/java/.../StigmergyIntegrationTest.java` — AgentState constructor update
- `runtime-core/src/test/java/.../SwarmIntegrationTest.java` — AgentState constructor update

## Decisions

See `decisions-1285.md` in this directory. Key choices:
- D1: AgentWorkState in stigmergy package (per-agent runtime state)
- D2: Single listener interface (5 default methods)
- D3: Listener SPI in spi/stigmergy/ (follows state owner)
- D4: StigmergyCoordinator owns work state tracking
- D5: WorkerExecutionOrchestrator fires transitions via bridge interface
- D6: Role lifecycle SPI surface only, integration deferred

## References

- `AgentLifecycleState.java` (api/model/stigmergy/) — existing 3-state enum
- `AgentState.java` (api/model/stigmergy/) — record to extend
- `StigmergyCoordinator.java` (runtime-core/.../stigmergy/) — state machine owner
- `SwarmCoordinationListener.java` (api/spi/stigmergy/) — listener SPI pattern
- `WorkerExecutionOrchestrator.java` (common-core/.../executor/) — integration point
- `NoOpAgentCapabilityGuard.java` (runtime-core/.../coordination/) — no-op default pattern
- `SpiDefaultBeans.java` (runtime-core/.../coordination/) — @DefaultBean producer pattern
- `WorkerStatusListener.java` (api/spi/) — cross-module bridge pattern precedent
- engine#1282 spec — coordination vocabulary types
- engine#1284 spec — SwarmTopologyProvider design
- agency#377 — cross-repo parent epic
- eidos#216 — MOISE+ society model (role lifecycle consumer)
