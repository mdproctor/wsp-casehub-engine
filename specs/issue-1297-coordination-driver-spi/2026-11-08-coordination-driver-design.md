# CoordinationDriver SPI — COORDINATOR Behavioral Contract

**Issue:** casehubio/engine#1297
**Parent epic:** casehubio/engine#1296 (PlatformRole Behavioral SPIs)
**Origin:** Refs casehubio/engine#1282 (coordination vocabulary types)

## Summary

Engine-api SPI defining the mechanical behavioral contract for `PlatformRole.COORDINATOR`. Coordinators orchestrate, delegate tasks, allocate agents, and aggregate results. Currently this behavior exists only in agency (`OrchestratedDriver`, pattern builders) with no engine-level contract. This issue moves the behavioral contract to engine-api so both engine and agency share the same interface.

## Design Principles

1. **Two-layer pattern.** `CoordinatorBehavior` marker interface in `model.coordination` tags what implements the role's behavior. `CoordinationDriver` in `spi.coordination` defines the concrete methods. Follows the established `SupervisorGateBehavior` / `OversightGateService` pattern.

2. **Complete behavioral surface.** All four COORDINATOR facets — delegation, aggregation, allocation, lifecycle — are defined on the SPI. Engine has the full contract; agency provides the intelligence layer.

3. **Coordination-level, not case-level.** `WorkOrchestrator` submits work at the case level. `CoordinationDriver` operates within a topology — delegating to agents that are already part of the coordination arrangement. Different scope, different concerns.

4. **Sealed interfaces for pattern matching.** `CoordinationResult` and `CoordinationLifecycleEvent` use sealed interfaces with typed variants. Engine can exhaustively switch on result/event type. Consistent with `TransitionTrigger` (#1290).

## Package

- Marker: `io.casehub.api.model.coordination.CoordinatorBehavior`
- SPI: `io.casehub.api.spi.coordination.CoordinationDriver`
- Vocabulary types: `io.casehub.api.model.coordination.*`
- Default: `io.casehub.engine.runtime.coordination.NoOpCoordinationDriver`

## Type Definitions

### CoordinatorBehavior (marker interface)

```java
public interface CoordinatorBehavior {}
```

Marker for components that implement `PlatformRole.COORDINATOR` behavior — the ability to delegate tasks, allocate agents, aggregate results, and manage coordination lifecycle. Parallel to `SupervisorGateBehavior` for `PlatformRole.SUPERVISOR`.

### CoordinationDriver (SPI)

```java
public interface CoordinationDriver extends CoordinatorBehavior {

  List<DelegationAssignment> delegate(
      CoordinationTask task, TopologyDescriptor topology);

  CoordinationResult aggregate(
      List<DelegationOutcome> outcomes, CoordinationTask originalTask);

  List<String> allocate(
      CoordinationTask task, TopologyDescriptor topology);

  void onLifecycleEvent(CoordinationLifecycleEvent event);
}
```

**delegate** — assigns work to agents within the topology. Takes a task and the current topology descriptor. Returns a list of assignments mapping agents to sub-tasks. An empty list means the coordinator declined to delegate (e.g., the task doesn't match the topology's capability).

**aggregate** — combines outputs from delegated agents into a single result. Called after delegation completes (fully or partially). Returns a `CoordinationResult` indicating success, partial completion, or failure.

**allocate** — selects which agents are eligible to receive a task. Distinct from `AgentRoutingStrategy` (which routes workers to tasks at the case level). This operates within a topology — the agents are already known participants. Returns a list of agent IDs.

**onLifecycleEvent** — receives coordination lifecycle transitions. Implementations can react to topology starts, stops, and reconfigurations. This is the hook for agency's `CoordinationTopology` to manage decision loops, budget pools, and degradation triggers.

### CoordinationTask (record)

The unit of work delegated within a topology.

```java
public record CoordinationTask(
    String taskId,
    String description,
    @Nullable String targetAgentConstraint,
    @Nullable String aggregationHint) {

  public CoordinationTask {
    Objects.requireNonNull(taskId, "taskId must not be null");
    if (taskId.isBlank())
      throw new IllegalArgumentException("taskId must not be blank");
    Objects.requireNonNull(description, "description must not be null");
  }
}
```

- **taskId** — unique identifier for the task within this coordination session
- **description** — what needs to be done (consumed by agency for LLM prompt construction)
- **targetAgentConstraint** — optional filter for eligible agents (e.g., a role name, capability, or agent ID pattern)
- **aggregationHint** — optional hint for how results should be combined (e.g., "merge", "vote", "first-wins"). The coordinator's `aggregate` method interprets this; the engine does not enforce it.

### DelegationAssignment (record)

A task assigned to a specific agent.

```java
public record DelegationAssignment(
    String agentId,
    CoordinationTask task) {

  public DelegationAssignment {
    Objects.requireNonNull(agentId, "agentId must not be null");
    Objects.requireNonNull(task, "task must not be null");
  }
}
```

### DelegationOutcome (record)

What a delegated agent produced.

```java
public record DelegationOutcome(
    String agentId,
    DelegationStatus status,
    @Nullable String payload) {

  public DelegationOutcome {
    Objects.requireNonNull(agentId, "agentId must not be null");
    Objects.requireNonNull(status, "status must not be null");
  }

  public enum DelegationStatus {
    COMPLETED,
    FAILED,
    PARTIAL
  }
}
```

### CoordinationResult (sealed interface)

Aggregated output from coordination.

```java
public sealed interface CoordinationResult
    permits CoordinationResult.Aggregated,
            CoordinationResult.Partial,
            CoordinationResult.Failed {

  record Aggregated(String payload, List<String> contributors)
      implements CoordinationResult {
    public Aggregated {
      Objects.requireNonNull(payload, "payload must not be null");
      contributors = contributors != null ? List.copyOf(contributors) : List.of();
    }
  }

  record Partial(String payload, List<String> completed, List<String> pending)
      implements CoordinationResult {
    public Partial {
      Objects.requireNonNull(payload, "payload must not be null");
      completed = completed != null ? List.copyOf(completed) : List.of();
      pending = pending != null ? List.copyOf(pending) : List.of();
    }
  }

  record Failed(String reason, List<String> failedAgents)
      implements CoordinationResult {
    public Failed {
      Objects.requireNonNull(reason, "reason must not be null");
      failedAgents = failedAgents != null ? List.copyOf(failedAgents) : List.of();
    }
  }
}
```

**Aggregated** — all delegated agents completed and results are combined. `payload` is the merged output. `contributors` lists the agent IDs that contributed.

**Partial** — some agents completed, others are still pending. Useful for phased coordination where the coordinator reports intermediate progress.

**Failed** — delegation failed. `reason` describes what went wrong. `failedAgents` lists the agents that failed.

### CoordinationLifecycleEvent (sealed interface)

Lifecycle transitions for coordination within a topology.

```java
public sealed interface CoordinationLifecycleEvent
    permits CoordinationLifecycleEvent.Started,
            CoordinationLifecycleEvent.Stopped,
            CoordinationLifecycleEvent.Reconfigured {

  record Started(String coordinatorId, TopologyDescriptor topology)
      implements CoordinationLifecycleEvent {
    public Started {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(topology, "topology must not be null");
    }
  }

  record Stopped(String coordinatorId, String reason)
      implements CoordinationLifecycleEvent {
    public Stopped {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(reason, "reason must not be null");
    }
  }

  record Reconfigured(String coordinatorId,
      TopologyDescriptor previousTopology, TopologyDescriptor newTopology)
      implements CoordinationLifecycleEvent {
    public Reconfigured {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(previousTopology, "previousTopology must not be null");
      Objects.requireNonNull(newTopology, "newTopology must not be null");
    }
  }
}
```

**Started** — coordination begins in a topology. The coordinator receives the initial topology descriptor.

**Stopped** — coordination ends. `reason` explains why (e.g., topology dissolved, case completed, manual stop).

**Reconfigured** — topology changed (e.g., agents added/removed, topology type transitioned). Carries both previous and new descriptors so the coordinator can diff and adapt.

## NoOpCoordinationDriver

Default implementation in `runtime-core`. Returns empty results for all methods. Registered as `@DefaultBean` (Quarkus) and `@ConditionalOnMissingBean` (Spring via the auto-configuration generator).

```java
public class NoOpCoordinationDriver implements CoordinationDriver {

  @Override
  public List<DelegationAssignment> delegate(
      CoordinationTask task, TopologyDescriptor topology) {
    return List.of();
  }

  @Override
  public CoordinationResult aggregate(
      List<DelegationOutcome> outcomes, CoordinationTask originalTask) {
    return new CoordinationResult.Failed("no coordinator configured", List.of());
  }

  @Override
  public List<String> allocate(
      CoordinationTask task, TopologyDescriptor topology) {
    return List.of();
  }

  @Override
  public void onLifecycleEvent(CoordinationLifecycleEvent event) {}
}
```

## PlatformRole Cross-Reference

Add Javadoc to `PlatformRole.COORDINATOR`:

```java
/**
 * Orchestrates, delegates tasks, allocates agents, and aggregates results.
 *
 * @see CoordinatorBehavior
 * @see io.casehub.api.spi.coordination.CoordinationDriver
 */
COORDINATOR(StructuralRole.INITIATOR),
```

## Relationship to Existing Types

| Type | Scope | Relationship |
|------|-------|-------------|
| `WorkOrchestrator` | Case-level | Submits WorkRequests. CoordinationDriver delegates within a topology. |
| `AgentRoutingStrategy` | Case-level | Routes workers to tasks. CoordinationDriver.allocate selects agents within a topology. |
| `TopologyProvider` | Topology-level | Describes the topology. CoordinationDriver operates within it. |
| `CoordinationTopology` (agency) | Runtime topology | Configures decision loops. Would implement CoordinationDriver. |
| `OrchestratedDriver` (agency) | Execution | Runs the orchestration loop. Would delegate to CoordinationDriver. |

## Test Coverage

- **CoordinationTaskTest** — taskId null/blank rejected, valid construction with and without optional fields
- **DelegationAssignmentTest** — null agentId/task rejected, valid construction
- **DelegationOutcomeTest** — null agentId/status rejected, valid construction with each DelegationStatus value
- **CoordinationResultTest** — sealed interface permits 3 variants, valid construction of each, defensive copies on lists, null payload rejected for Aggregated/Partial/Failed
- **CoordinationLifecycleEventTest** — sealed interface permits 3 variants, valid construction of each, null fields rejected
- **CoordinationDriverTest** — NoOpCoordinationDriver returns empty lists, Failed result, and no-ops lifecycle

## What This Issue Does NOT Include

- **Coordination execution runtime** — who calls `delegate`/`aggregate` and when. Owned by the engine runtime or agency's CoordinationTopology.
- **Agency implementation** — the LLM-driven implementation of CoordinationDriver. Agency depends on engine-api and provides the intelligence.
- **YAML surface** — DSL representation of coordination configuration. Deferred to schema/codegen work.
- **Sibling role SPIs** — JUDGE (#1298) and MODERATOR (#1299) have their own behavioral contracts.

## References

- `api/src/main/java/io/casehub/api/model/coordination/SupervisorGateBehavior.java` — marker interface pattern
- `api/src/main/java/io/casehub/api/spi/OversightGateService.java` — SPI extending marker pattern
- `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:25` — COORDINATOR enum value
- `api/src/main/java/io/casehub/api/model/coordination/TopologyDescriptor.java` — consumed by delegate/allocate
- `api/src/main/java/io/casehub/api/spi/routing/AgentRoutingStrategy.java` — case-level routing (distinct scope)
- `common-core/src/main/java/io/casehub/engine/common/spi/WorkOrchestrator.java` — case-level orchestration (distinct scope)
- `agency-core: CoordinationTopology.java` — agency integration point
- `agency-core: OrchestratedDriver.java` — agency execution driver
- `agency-core: CoordinationPlan.java` — agency's coordination configuration
- `specs/issue-1290-transition-policy-graduated-topology/2026-11-08-transition-policy-design.md` — sealed interface pattern
- casehubio/engine#1296 — parent epic (PlatformRole Behavioral SPIs)
- casehubio/engine#1282 — coordination vocabulary types (foundation)
