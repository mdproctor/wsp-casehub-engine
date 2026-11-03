# AgentWorkLifecycleListener + AgentWorkState Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1285 — feat: AgentWorkLifecycleListener + AgentWorkState + role lifecycle events
**Issue group:** casehubio/engine#1285 (part of agency#377 epic, engine#1281 sub-epic)

**Goal:** Extend per-agent lifecycle with work-level states and role lifecycle events so consumers can bind to what an agent is *doing* and what *role* it holds.

**Architecture:** New enum `AgentWorkState` (6 states) and SPI `AgentWorkLifecycleListener` (5 default methods) in api module. `StigmergyCoordinator` extends to track work state and dispatch listener events. `WorkerExecutionOrchestrator` fires state transitions via a bridge interface `AgentWorkStateNotifier` (api-level) since it lives in common-core while the coordinator lives in runtime-core.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (jakarta.enterprise), JUnit 5

## Global Constraints

- All new types use Apache 2.0 license header (copy from any existing file)
- Agent identity is `String` (consistent with AgentCapabilityGuard from #1283)
- Role lifecycle events (`onRoleAdopted`/`onRoleReleased`) — SPI surface only, no integration points in this issue
- ADAPTING, SIGNALING, WAITING states — enum values defined, state machine validates, but no code fires these transitions in this issue

---

## Batch 1: API Surface

### Task 1: AgentWorkState enum + AgentState extension

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/AgentWorkState.java`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java:21-28`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinatorTest.java`

**Interfaces:**
- Produces: `AgentWorkState` enum with values IDLE, ASSIGNED, EXECUTING, ADAPTING, SIGNALING, WAITING
- Produces: `AgentState` record with added `workState` field (4th position, after `state`)

- [ ] **Step 1: Create AgentWorkState enum**

Create `api/src/main/java/io/casehub/api/model/stigmergy/AgentWorkState.java`:

```java
package io.casehub.api.model.stigmergy;

public enum AgentWorkState {
  IDLE,
  ASSIGNED,
  EXECUTING,
  ADAPTING,
  SIGNALING,
  WAITING
}
```

- [ ] **Step 2: Add workState field to AgentState record**

Modify `api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java`. Change the record to:

```java
public record AgentState(
    String agentId,
    String bindingName,
    AgentLifecycleState state,
    AgentWorkState workState,
    Instant joinedAt,
    @Nullable Instant activatedAt,
    @Nullable Instant departedAt,
    Instant lastActivity) {}
```

- [ ] **Step 3: Update StigmergyCoordinator — agentJoined**

In `runtime-core/src/main/java/.../stigmergy/StigmergyCoordinator.java`, update `agentJoined()` (line 119-121). Change:

```java
state.agents.put(
    agentId,
    new AgentState(agentId, bindingName, AgentLifecycleState.JOINING, now, null, null, now));
```

To:

```java
state.agents.put(
    agentId,
    new AgentState(
        agentId, bindingName, AgentLifecycleState.JOINING, AgentWorkState.IDLE,
        now, null, null, now));
```

- [ ] **Step 4: Update StigmergyCoordinator — agentActivated**

In the same file, update `agentActivated()` (lines 134-143). Change:

```java
state.agents.put(
    agentId,
    new AgentState(
        agentId,
        current.bindingName(),
        AgentLifecycleState.ACTIVE,
        current.joinedAt(),
        now,
        null,
        now));
```

To:

```java
state.agents.put(
    agentId,
    new AgentState(
        agentId,
        current.bindingName(),
        AgentLifecycleState.ACTIVE,
        current.workState(),
        current.joinedAt(),
        now,
        null,
        now));
```

- [ ] **Step 5: Update StigmergyCoordinator — agentDeparted**

In the same file, update `agentDeparted()` (lines 163-172). Change:

```java
state.agents.put(
    agentId,
    new AgentState(
        agentId,
        current.bindingName(),
        AgentLifecycleState.DEPARTED,
        current.joinedAt(),
        current.activatedAt(),
        now,
        now));
```

To:

```java
state.agents.put(
    agentId,
    new AgentState(
        agentId,
        current.bindingName(),
        AgentLifecycleState.DEPARTED,
        AgentWorkState.IDLE,
        current.joinedAt(),
        current.activatedAt(),
        now,
        now));
```

- [ ] **Step 6: Run tests to verify existing behavior preserved**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: All existing tests PASS (AgentState construction updated, record compatible)

- [ ] **Step 7: Add test for workState defaults**

Add to `StigmergyCoordinatorTest.java`:

```java
@Test
void agentJoinedSetsWorkStateToIdle() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    var agent = coordinator.getAgent(caseId, "agent-1");
    assertEquals(AgentWorkState.IDLE, agent.workState());
}

@Test
void agentDepartedResetsWorkStateToIdle() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");
    coordinator.agentDeparted(caseId, "agent-1");
    var agent = coordinator.getAgent(caseId, "agent-1");
    assertEquals(AgentWorkState.IDLE, agent.workState());
}
```

Add import: `import io.casehub.api.model.stigmergy.AgentWorkState;`

- [ ] **Step 8: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: All tests PASS including new workState tests

- [ ] **Step 9: Compile full project**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS — all modules compile with updated AgentState record

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/AgentWorkState.java api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinator.java runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinatorTest.java
git commit -m "feat: add AgentWorkState enum + extend AgentState record

Refs casehubio/engine#1285"
```

### Task 2: AgentWorkLifecycleListener + AgentWorkStateNotifier SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java`
- Create: `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkStateNotifier.java`

**Interfaces:**
- Produces: `AgentWorkLifecycleListener` — 5 default methods (onWorkStateChanged, onTaskAssigned, onTaskAdapted, onRoleAdopted, onRoleReleased)
- Produces: `AgentWorkStateNotifier` — 2 methods (transitionWorkState, taskAssigned) — bridge for cross-module integration

- [ ] **Step 1: Create AgentWorkLifecycleListener SPI**

Create `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java`:

```java
package io.casehub.api.spi.stigmergy;

import io.casehub.api.model.stigmergy.AgentWorkState;
import java.util.Map;
import java.util.UUID;

/**
 * Receives per-agent work lifecycle events from the engine runtime.
 *
 * <p>Implementations are multi-instance: multiple listeners coexist via CDI {@code Instance} /
 * Spring {@code ObjectProvider}. Do not use {@code @DefaultBean} on implementations.
 */
public interface AgentWorkLifecycleListener {

  default void onWorkStateChanged(
      UUID caseId, String agentId, AgentWorkState from, AgentWorkState to) {}

  default void onTaskAssigned(
      UUID caseId, String agentId, String taskId, Map<String, Object> context) {}

  default void onTaskAdapted(
      UUID caseId, String agentId, String taskId, String adaptationReason) {}

  default void onRoleAdopted(UUID caseId, String agentId, String groupId, String role) {}

  default void onRoleReleased(UUID caseId, String agentId, String groupId, String role) {}
}
```

- [ ] **Step 2: Create AgentWorkStateNotifier SPI**

Create `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkStateNotifier.java`:

```java
package io.casehub.api.spi.stigmergy;

import io.casehub.api.model.stigmergy.AgentWorkState;
import java.util.Map;
import java.util.UUID;

/**
 * Cross-module bridge for work state transitions. {@code WorkerExecutionOrchestrator} (common-core)
 * calls this interface; {@code StigmergyCoordinator} (runtime-core) implements it.
 */
public interface AgentWorkStateNotifier {

  void transitionWorkState(UUID caseId, String agentId, AgentWorkState targetState);

  void taskAssigned(UUID caseId, String agentId, String taskId, Map<String, Object> context);
}
```

- [ ] **Step 3: Compile to verify**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkStateNotifier.java
git commit -m "feat: add AgentWorkLifecycleListener + AgentWorkStateNotifier SPIs

Refs casehubio/engine#1285"
```

---

## Batch 2: State Machine + Default Wiring

### Task 3: StigmergyCoordinator implements AgentWorkStateNotifier

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinator.java:42`
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinatorTest.java`

**Interfaces:**
- Consumes: `AgentWorkState` (from Task 1), `AgentWorkStateNotifier` + `AgentWorkLifecycleListener` (from Task 2)
- Produces: StigmergyCoordinator.transitionWorkState(UUID, String, AgentWorkState) — validates and applies transitions, fires listeners
- Produces: StigmergyCoordinator.taskAssigned(UUID, String, String, Map) — compound: transitions to ASSIGNED + fires onTaskAssigned

- [ ] **Step 1: Write failing test — valid work state transition**

Add to `StigmergyCoordinatorTest.java`:

```java
@Test
void transitionWorkStateChangesState() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");

    coordinator.transitionWorkState(caseId, "agent-1", AgentWorkState.ASSIGNED);
    assertEquals(AgentWorkState.ASSIGNED, coordinator.getAgent(caseId, "agent-1").workState());

    coordinator.transitionWorkState(caseId, "agent-1", AgentWorkState.EXECUTING);
    assertEquals(AgentWorkState.EXECUTING, coordinator.getAgent(caseId, "agent-1").workState());

    coordinator.transitionWorkState(caseId, "agent-1", AgentWorkState.IDLE);
    assertEquals(AgentWorkState.IDLE, coordinator.getAgent(caseId, "agent-1").workState());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest=StigmergyCoordinatorTest#transitionWorkStateChangesState -q`
Expected: FAIL — `transitionWorkState` method does not exist

- [ ] **Step 3: Write failing test — invalid transition rejected**

```java
@Test
void transitionWorkStateRejectsInvalidTransition() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");

    assertThrows(
        IllegalStateException.class,
        () -> coordinator.transitionWorkState(caseId, "agent-1", AgentWorkState.EXECUTING));
}
```

Add import: `import static org.junit.jupiter.api.Assertions.assertThrows;`

- [ ] **Step 4: Write failing test — listener receives events**

```java
@Test
void transitionWorkStateFiresListener() {
    var listener = new RecordingWorkLifecycleListener();
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");
    coordinator.addWorkLifecycleListener(listener);

    coordinator.transitionWorkState(caseId, "agent-1", AgentWorkState.ASSIGNED);

    assertEquals(1, listener.stateChanges.size());
    var change = listener.stateChanges.get(0);
    assertEquals(AgentWorkState.IDLE, change.from);
    assertEquals(AgentWorkState.ASSIGNED, change.to);
    assertEquals("agent-1", change.agentId);
}
```

Add the recording listener as an inner class:

```java
static class RecordingWorkLifecycleListener implements AgentWorkLifecycleListener {
    record StateChange(UUID caseId, String agentId, AgentWorkState from, AgentWorkState to) {}
    record TaskAssignment(UUID caseId, String agentId, String taskId, Map<String, Object> context) {}

    final List<StateChange> stateChanges = new ArrayList<>();
    final List<TaskAssignment> taskAssignments = new ArrayList<>();

    @Override
    public void onWorkStateChanged(UUID caseId, String agentId, AgentWorkState from, AgentWorkState to) {
        stateChanges.add(new StateChange(caseId, agentId, from, to));
    }

    @Override
    public void onTaskAssigned(UUID caseId, String agentId, String taskId, Map<String, Object> context) {
        taskAssignments.add(new TaskAssignment(caseId, agentId, taskId, context));
    }
}
```

Add import: `import io.casehub.api.spi.stigmergy.AgentWorkLifecycleListener;`

- [ ] **Step 5: Write failing test — taskAssigned compound operation**

```java
@Test
void taskAssignedTransitionsAndFiresListener() {
    var listener = new RecordingWorkLifecycleListener();
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test-tenant");
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");
    coordinator.addWorkLifecycleListener(listener);

    coordinator.taskAssigned(caseId, "agent-1", "worker-1", Map.of("key", "value"));

    assertEquals(AgentWorkState.ASSIGNED, coordinator.getAgent(caseId, "agent-1").workState());
    assertEquals(1, listener.stateChanges.size());
    assertEquals(1, listener.taskAssignments.size());
    assertEquals("worker-1", listener.taskAssignments.get(0).taskId);
}
```

Add import: `import java.util.Map;`

- [ ] **Step 6: Implement AgentWorkStateNotifier on StigmergyCoordinator**

Modify `StigmergyCoordinator.java`:

1. Add `implements AgentWorkStateNotifier` to class declaration (line 42)
2. Add imports for `AgentWorkLifecycleListener`, `AgentWorkStateNotifier`, `AgentWorkState`
3. Add a valid transitions set and listener list as fields
4. Add `addWorkLifecycleListener` method
5. Implement `transitionWorkState` and `taskAssigned`

Add these fields after line 57:

```java
private final List<AgentWorkLifecycleListener> workLifecycleListeners = new ArrayList<>();

private static final Map<AgentWorkState, Set<AgentWorkState>> VALID_WORK_TRANSITIONS =
    Map.of(
        AgentWorkState.IDLE, Set.of(AgentWorkState.ASSIGNED),
        AgentWorkState.ASSIGNED, Set.of(AgentWorkState.EXECUTING, AgentWorkState.IDLE),
        AgentWorkState.EXECUTING,
            Set.of(
                AgentWorkState.IDLE,
                AgentWorkState.ADAPTING,
                AgentWorkState.SIGNALING,
                AgentWorkState.WAITING),
        AgentWorkState.ADAPTING, Set.of(AgentWorkState.EXECUTING),
        AgentWorkState.SIGNALING, Set.of(AgentWorkState.EXECUTING),
        AgentWorkState.WAITING, Set.of(AgentWorkState.EXECUTING));
```

Add the methods before `evictByCase`:

```java
public void addWorkLifecycleListener(AgentWorkLifecycleListener listener) {
    workLifecycleListeners.add(listener);
}

@Override
public void transitionWorkState(UUID caseId, String agentId, AgentWorkState targetState) {
    var caseState = cases.get(caseId);
    if (caseState == null) return;
    var current = caseState.agents.get(agentId);
    if (current == null) return;

    AgentWorkState from = current.workState();
    Set<AgentWorkState> allowed = VALID_WORK_TRANSITIONS.getOrDefault(from, Set.of());
    if (!allowed.contains(targetState)) {
        throw new IllegalStateException(
            "Invalid work state transition for agent " + agentId + ": " + from + " → " + targetState);
    }

    caseState.agents.put(
        agentId,
        new AgentState(
            agentId, current.bindingName(), current.state(), targetState,
            current.joinedAt(), current.activatedAt(), current.departedAt(), Instant.now()));

    for (var listener : workLifecycleListeners) {
        try {
            listener.onWorkStateChanged(caseId, agentId, from, targetState);
        } catch (Exception e) {
            LOG.warnf(e, "AgentWorkLifecycleListener.onWorkStateChanged failed for agent %s", agentId);
        }
    }
}

@Override
public void taskAssigned(UUID caseId, String agentId, String taskId, Map<String, Object> context) {
    transitionWorkState(caseId, agentId, AgentWorkState.ASSIGNED);
    for (var listener : workLifecycleListeners) {
        try {
            listener.onTaskAssigned(caseId, agentId, taskId, context);
        } catch (Exception e) {
            LOG.warnf(e, "AgentWorkLifecycleListener.onTaskAssigned failed for agent %s", agentId);
        }
    }
}
```

Add imports:

```java
import io.casehub.api.model.stigmergy.AgentWorkState;
import io.casehub.api.spi.stigmergy.AgentWorkLifecycleListener;
import io.casehub.api.spi.stigmergy.AgentWorkStateNotifier;
import java.util.Set;
```

- [ ] **Step 7: Run all tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: All tests PASS

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinator.java runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/StigmergyCoordinatorTest.java
git commit -m "feat: StigmergyCoordinator implements AgentWorkStateNotifier

State machine validates transitions, fires AgentWorkLifecycleListener
events on each work state change.

Refs casehubio/engine#1285"
```

### Task 4: NoOpAgentWorkStateNotifier + SpiDefaultBeans producer

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentWorkStateNotifier.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java:285-299`

**Interfaces:**
- Consumes: `AgentWorkStateNotifier` (from Task 2)
- Produces: `NoOpAgentWorkStateNotifier` — no-op implementation for non-swarm deployments
- Produces: CDI `@DefaultBean` producer for `AgentWorkStateNotifier`

- [ ] **Step 1: Create NoOpAgentWorkStateNotifier**

Create `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentWorkStateNotifier.java`:

```java
package io.casehub.engine.runtime.coordination;

import io.casehub.api.model.stigmergy.AgentWorkState;
import io.casehub.api.spi.stigmergy.AgentWorkStateNotifier;
import java.util.Map;
import java.util.UUID;

public class NoOpAgentWorkStateNotifier implements AgentWorkStateNotifier {

  @Override
  public void transitionWorkState(UUID caseId, String agentId, AgentWorkState targetState) {}

  @Override
  public void taskAssigned(
      UUID caseId, String agentId, String taskId, Map<String, Object> context) {}
}
```

- [ ] **Step 2: Add @DefaultBean producer to SpiDefaultBeans**

In `runtime/src/main/java/.../quarkus/SpiDefaultBeans.java`, add after the `TopologyProvider` producer (after line 299):

```java
@Produces
@DefaultBean
@ApplicationScoped
AgentWorkStateNotifier noOpAgentWorkStateNotifier() {
    return new NoOpAgentWorkStateNotifier();
}
```

Add imports:

```java
import io.casehub.api.spi.stigmergy.AgentWorkStateNotifier;
import io.casehub.engine.runtime.coordination.NoOpAgentWorkStateNotifier;
```

- [ ] **Step 3: Compile to verify**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentWorkStateNotifier.java runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java
git commit -m "feat: add NoOpAgentWorkStateNotifier + @DefaultBean producer

Ensures non-swarm deployments have a working no-op notifier.
Follows NoOpAgentCapabilityGuard pattern from #1283.

Refs casehubio/engine#1285"
```

---

## Batch 3: Orchestrator Integration

### Task 5: WorkerExecutionOrchestrator + RuntimeBeans wiring

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java:61-104,106-124,159-310`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java:1278-1306`

**Interfaces:**
- Consumes: `AgentWorkStateNotifier` (from Task 2), `AgentWorkState` (from Task 1)

- [ ] **Step 1: Add AgentWorkStateNotifier to WorkerExecutionOrchestrator constructor**

In `common-core/src/main/java/.../executor/WorkerExecutionOrchestrator.java`:

Add field after `traceIdProvider` (line 77):

```java
private final io.casehub.api.spi.stigmergy.AgentWorkStateNotifier workStateNotifier;
```

Add parameter to constructor (after `traceIdProvider` parameter at line 91):

```java
io.casehub.api.spi.stigmergy.AgentWorkStateNotifier workStateNotifier
```

Add assignment at end of constructor body (after line 103):

```java
this.workStateNotifier = workStateNotifier;
```

- [ ] **Step 2: Add state transitions to executeWorker**

In the `executeWorker` method, after `bindingName` extraction (after line 189, where `bindingName` is assigned):

```java
if (bindingName != null && workStateNotifier != null) {
    try {
        workStateNotifier.taskAssigned(
            taskData.caseId(), bindingName, workerId, Map.of());
    } catch (Exception e) {
        LOG.warnf(e, "Work state transition ASSIGNED failed for agent %s", bindingName);
    }
}
```

Wrap the worker resolution, execution, and publishing in a try-finally to ensure IDLE transition on all paths (success, failure, exception). The ASSIGNED→IDLE transition is valid for the case where resolution fails after assignment:

Before the worker resolution block (after ASSIGNED transition), wrap through to `publishOutcome` in a try-finally:

```java
try {
    // ... existing worker resolution, capability resolution, bridge setup ...

    if (bindingName != null && workStateNotifier != null) {
        try {
            workStateNotifier.transitionWorkState(
                taskData.caseId(), bindingName,
                io.casehub.api.model.stigmergy.AgentWorkState.EXECUTING);
        } catch (Exception e) {
            LOG.warnf(e, "Work state transition EXECUTING failed for agent %s", bindingName);
        }
    }

    // ... existing workerExecutor.execute + publishOutcome ...

} finally {
    if (bindingName != null && workStateNotifier != null) {
        try {
            workStateNotifier.transitionWorkState(
                taskData.caseId(), bindingName,
                io.casehub.api.model.stigmergy.AgentWorkState.IDLE);
        } catch (Exception e) {
            LOG.warnf(e, "Work state transition IDLE failed for agent %s", bindingName);
        }
    }
}
```

This handles: normal completion (EXECUTING→IDLE), resolution failure (ASSIGNED→IDLE), and execution exception (EXECUTING→IDLE). No changes needed in `execute()`'s outer catch block — failures before bindingName extraction mean no ASSIGNED transition happened.

- [ ] **Step 4: Update RuntimeBeans wiring**

In `runtime/src/main/java/.../quarkus/RuntimeBeans.java`, update the `workerExecutionOrchestrator` producer method (lines 1278-1306).

Add parameter:

```java
AgentWorkStateNotifier workStateNotifier,
```

Add to constructor call (after `traceIdProvider`):

```java
workStateNotifier);
```

Add import:

```java
import io.casehub.api.spi.stigmergy.AgentWorkStateNotifier;
```

- [ ] **Step 5: Compile to verify**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 6: Run full runtime-core test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core -q`
Expected: All tests PASS

- [ ] **Step 7: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: All tests PASS (including scheduler modules that construct/inject the orchestrator)

- [ ] **Step 8: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java
git commit -m "feat: integrate AgentWorkStateNotifier into WorkerExecutionOrchestrator

Worker dispatch path now fires IDLE→ASSIGNED→EXECUTING→IDLE state
transitions for swarm agents (bindingName != null). Non-swarm workers
skip transitions. NoOp notifier ensures backward compatibility.

Closes casehubio/engine#1285"
```

---

## References

- [2026-11-02-agent-work-lifecycle-design.md] — design spec this plan implements
- [decisions-1285.md] — design decisions (D1-D6)
- [AgentLifecycleState.java:18] — existing 3-state lifecycle enum
- [AgentState.java:21] — record to extend with workState
- [StigmergyCoordinator.java:42] — state machine owner, implements AgentWorkStateNotifier
- [SwarmCoordinationListener.java:30] — listener SPI pattern precedent
- [WorkerExecutionOrchestrator.java:61] — integration point for state transitions
- [SpiDefaultBeans.java:86] — @DefaultBean producer pattern
- [NoOpAgentCapabilityGuard.java] — no-op default pattern
- [RuntimeBeans.java:1280] — CDI wiring for orchestrator
- [StigmergyCoordinatorTest.java:40] — test class to extend
- casehubio/engine#1285 — focal issue
- casehubio/engine#1281 — parent epic
- casehubio/agency#377 — cross-repo parent
