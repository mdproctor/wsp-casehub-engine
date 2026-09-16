# Agent Discovery & Neighbor Awareness — Design Spec

**Issue:** casehubio/engine#1108
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-16

## Summary

Runtime agent discovery enabling workers to know who else is active on their case, what they're observing, what signals they're depositing, and how their work relates. Implemented as a pure read-only query facade over existing engine registries — no new storage. Third WorkerRuntime facet (`NeighborSpace`) following the SignalSpace/InterestSpace pattern from #1107.

Proximity is emergent from shared activity, not pre-computed capability similarity. Agents that watch the same keys are observationally proximate. Agents that deposit the same signals are coordinationally proximate. Agents whose outputs feed another's inputs are complementary.

## Design Principles

1. **Query, not storage** — all neighbor data is computed on-demand from existing registries (PlanItemStore, ScopedWorkerRegistry, ObservationRegistry, SignalRegistry)
2. **Emergent proximity** — similarity is derived from what agents actually do, not what they're declared capable of
3. **Full identity** — agents need to know WHO their neighbors are to coordinate (deposit targeted signals, register complementary interests)
4. **Facet consistency** — third facet on WorkerRuntime, follows the SignalSpace/InterestSpace pattern exactly

## Architecture

### NeighborSpace Facet (D32, D34)

Third coordination facet on WorkerRuntime. Pure read-only — no mutation methods.

```java
package io.casehub.api.engine;

public interface NeighborSpace {

  List<Neighbor> active();

  List<Neighbor> withSharedInterests();

  List<Neighbor> withSharedSignals();

  List<Neighbor> complementary();

  NeighborSpace NOOP = new NeighborSpace() {
    @Override public List<Neighbor> active() { return List.of(); }
    @Override public List<Neighbor> withSharedInterests() { return List.of(); }
    @Override public List<Neighbor> withSharedSignals() { return List.of(); }
    @Override public List<Neighbor> complementary() { return List.of(); }
  };
}
```

### WorkerRuntime Extension

```java
public interface WorkerRuntime extends WorkerScope {
    // ... existing methods ...
    default SignalSpace signals() { return SignalSpace.NOOP; }
    default InterestSpace interests() { return InterestSpace.NOOP; }
    default NeighborSpace neighbors() { return NeighborSpace.NOOP; }  // NEW
}
```

## SPI Types

### Neighbor (D33)

Bounded view of a neighboring agent. Identity exposed for coordination. Multiple relations possible per neighbor.

```java
package io.casehub.api.spi.observation;

public record Neighbor(
    String agentId,
    Set<String> capabilities,
    TaskStatus currentStatus,
    String bindingName,
    Set<NeighborRelation> relations
) {}
```

### NeighborRelation (D33)

Taxonomy of how agents relate to each other within a case.

```java
package io.casehub.api.spi.observation;

public enum NeighborRelation {
  COACTIVE,          // active on the same case
  SHARED_INTEREST,   // watching ≥1 same context key
  SHARED_SIGNAL,     // depositing the same signal
  COMPLEMENTARY      // my outputs feed their observations, or vice versa
}
```

## Signal Source Tracking (D35)

`Signal` gains `Set<String> sources` — the full set of agent IDs that have deposited or reinforced the signal. `deposit()` adds the depositor to the set alongside updating `lastSource`. `sources` is immutable on read (`Set.copyOf()`).

```java
// Signal record gains sources field
public record Signal(
    String name,
    double strength,
    Instant firstDeposited,
    Instant lastReinforced,
    Duration halfLife,
    String lastSource,
    int reinforcementCount,
    boolean expired,
    Set<String> sources          // NEW — all depositor agent IDs
) {}
```

`SignalRegistry.deposit()` updated:
- On new signal: `sources = Set.of(source)`
- On reinforcement: `sources = new HashSet<>(existing.sources()); sources.add(source); Set.copyOf(sources)`

## Implementation

### DefaultNeighborSpace (D32)

Pure query class — no state, no mutation. Constructed per worker invocation by `WorkerRuntimeFactory`.

```
runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultNeighborSpace.java
```

Constructor takes:
- `UUID caseId` — the calling agent's case
- `String tenancyId` — tenant for PlanItemStore queries
- `String selfAgentId` — the calling agent's worker name (for self-exclusion)
- `String selfBindingName` — the calling agent's binding (for complementary lookup)
- `PlanItemStore planItemStore` — for active neighbor queries
- `ObservationRegistry observationRegistry` — for shared interest queries
- `SignalRegistry signalRegistry` — for shared signal queries
- `CaseDefinitionRegistry definitionRegistry` — for capability and producedKeys lookup
- `CaseInstanceCache caseInstanceCache` — for CaseDefinition resolution
- `SignalConfig signalConfig` — for effective-zero threshold on signal perception

### Query Data Sources

| Method | Data source | Logic |
|--------|------------|-------|
| `active()` | `PlanItemStore.findByCaseId(caseId, tenancyId)` | Filter to RUNNING/DISPATCHING status. Group by `executorName`. Exclude self. Build `Neighbor` with `COACTIVE` relation. |
| `withSharedInterests()` | `ObservationRegistry.getObservers(caseId)` | For each other agent's observers, intersect `watchedKeys()` with self's `watchedKeys()`. Non-empty intersection → `SHARED_INTEREST` relation. |
| `withSharedSignals()` | `SignalRegistry.perceive(caseId, threshold)` | For each perceived signal, check if `sources` contains both `selfAgentId` and another agent. Shared sources → `SHARED_SIGNAL` relation. |
| `complementary()` | `CaseDefinition.getBindings()` + `ObservationRegistry` | For each other active agent, check: (1) do their binding's `producedKeys` overlap with my `watchedKeys`? (2) do my binding's `producedKeys` overlap with their `watchedKeys`? Either → `COMPLEMENTARY` relation. |

Self-exclusion: all methods filter out the calling agent by `selfAgentId`.

### DefaultWorkerRuntime Changes

`DefaultWorkerRuntime` gains `private final NeighborSpace neighborSpace` field. `neighbors()` returns it. `WorkerRuntimeFactory` creates `DefaultNeighborSpace` with injected registries.

### WorkerRuntimeFactory Changes

`WorkerRuntimeFactory.create()` (6-arg overload) gains `PlanItemStore` injection. Creates `DefaultNeighborSpace` with all required registries.

## Module Placement (D36)

| Type | Module | Package |
|------|--------|---------|
| `NeighborSpace` | engine-api | `io.casehub.api.engine` |
| `Neighbor` | engine-api | `io.casehub.api.spi.observation` |
| `NeighborRelation` | engine-api | `io.casehub.api.spi.observation` |
| `DefaultNeighborSpace` | runtime-core | `io.casehub.engine.internal.observation` |

## Testing

- Unit tests for `Neighbor` and `NeighborRelation` (construction, equality, multi-relation)
- Unit tests for `NeighborSpace.NOOP` (all methods return empty lists)
- Unit tests for `DefaultNeighborSpace`:
  - `active()` — registers PlanItems for multiple agents, verifies self-exclusion, verifies COACTIVE relation
  - `withSharedInterests()` — registers observers for two agents on overlapping keys, verifies SHARED_INTEREST
  - `withSharedSignals()` — deposits signals from two agents, verifies SHARED_SIGNAL
  - `complementary()` — sets up binding producedKeys vs other agent's watchedKeys, verifies COMPLEMENTARY
- Unit tests for `Signal.sources` tracking in `SignalRegistry`
- Unit tests for `DefaultSignalSpace` regression (deposit/perceive still work with new sources field)
- Integration test: full case with multiple agents → `runtime.neighbors().active()` returns correct neighbors

## Dependencies

- #1105 (Environment Observation SPI) — `ObservationRegistry.getObservers()` for shared interest queries
- #1106 (Signal/Pheromone model) — `SignalRegistry` for shared signal queries, `Signal` type extension
- #1107 (Dynamic Interest Registration) — `InterestSpace` facet pattern, `WorkerRuntime` faceting architecture

## Not in Scope

- Capability-space vector similarity (eidos concern — eidos#14)
- Cross-case neighbor discovery (intra-case only)
- Topic-based broadcast for neighbor communication (qhorus concern — qhorus#15)
- Neighbor notification on topology change (future — agents poll via `neighbors()`)
- Dynamic topology graph persistence (computed on-demand, never stored)

## References

- `PlanItemStore.java` (common-core) — active PlanItem queries
- `ScopedWorkerRegistry.java` (common-core) — persistent session tracking
- `ObservationRegistry.java` (common-core) — observer queries for interest overlap
- `SignalRegistry.java` (common-core) — signal perception and source tracking
- `Signal.java` (api/model/signal) — signal record gaining `sources` field
- `WorkerRuntime.java` (api/engine) — facet accessor pattern
- `DefaultWorkerRuntime.java` (runtime-core) — facet field pattern
- `WorkerRuntimeFactory.java` (runtime-core) — facet construction pattern
- `Binding.producedKeys` (api/model) — for complementary neighbor computation
- D32-D36 in `decisions.md` — all design decisions for this issue
- arXiv:2504.00587 (AgentNet) — decentralized coordination via dynamic DAG topology
