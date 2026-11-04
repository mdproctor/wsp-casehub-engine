# Design: LlmSwarmCoordinator → CentralisedTopology Refactoring

**Issue:** casehubio/agency#381
**Epic:** casehubio/agency#377 (distributed per-agent coordination)
**Scale:** L | **Complexity:** High

## Summary

Decompose `LlmSwarmCoordinator` so its three LLM invocations (team formation, collective reasoning, provisioning) become `AgentDecisionLoop` instances within `CentralisedTopology`. The coordinator class is eliminated — its responsibilities are fully absorbed by the topology and the decision loop infrastructure.

## Context

`LlmSwarmCoordinator` is an immutable value holder with one real method (`assembleCoordinatorPrompt`). It holds references to `SwarmRoutingStrategy`, `CollectiveReasoningAggregation`, and supporting components, but doesn't orchestrate the LLM invocations itself. `SwarmCoordinatorBuilder` already has a parallel `buildTopology()` path that creates `CentralisedTopology` with the same components.

`CentralisedTopology` implements `CoordinationTopology` and already defines three `DecisionLoopDescriptor`s (team-formation, collective-reasoning, provisioning) in its `configure()` method. However, the executor methods (`executeRouting`, `executeAggregation`) are stubs returning `SUCCESS` without acting on the decisions.

The `AgentDecisionLoop` infrastructure provides observation collection, budget-aware summarisation, and emission policies. Each loop wraps a `ContentSummariser<ObservationEvent, DecisionOutcome>` — the "reasoner" — that performs the actual LLM invocation and returns a typed decision outcome.

## Architecture

### Data Flow

```
SwarmCoordinationEvent
    │
    ▼
CentralisedTopology.onSwarmEvent()
    │
    ├─ updates SwarmContext (as before)
    │
    └─ feeds ObservationEvent into the relevant loop's collect()
         │
         ▼
    AgentDecisionLoop (3 instances, created in configure())
         │
         ├─ team-formation: RoutingReasoner → DecisionOutcome.Route
         ├─ collective-reasoning: AggregationReasoner → DecisionOutcome.Aggregate
         └─ provisioning: ProvisioningReasoner → DecisionOutcome.Provision
              │
              ▼
         EventStreamBus<DecisionOutcome> (shared, from TopologyContext)
              │
              ▼
         DecisionOutcomeExecutor (pattern match on outcome variant)
              │
              ├─ Route → executeRouting()
              ├─ Aggregate → executeAggregation()
              ├─ Provision → executeProvisioning()
              ├─ WriteBack → executeWriteBack()
              ├─ Composite → chain sub-outcomes
              └─ ScopeGated → reject
```

Each `AgentDecisionLoop` wraps its strategy in the `DecisionLoopFactory` pipeline: `ContentSummariser` → `SectionAwareContentSummariser` → `BudgetAwareSummariser` → `InFlightGate` → `SummarisationRunner`. Budget checking, prompt section injection, and concurrency gating come for free from the existing infrastructure.

### Event → Loop Routing

`onSwarmEvent()` updates `SwarmContext` as before, then feeds observations into the appropriate loop:

| SwarmCoordinationEvent | Target loop | ObservationEvent key |
|------------------------|-------------|---------------------|
| `SwarmInitialised` | team-formation | `"candidates-available"` |
| `RoleChanged` | team-formation | `"roles-changed"` |
| `TeamChanged` | collective-reasoning | `"team-changed"` |
| `ProgressUpdated` | all three | `"progress-updated"` |
| `MemberCompleted` | collective-reasoning | `"member-completed"` |
| `ProvisioningChanged` | provisioning | `"provisioning-changed"` |

## New Types

### DecisionOutcome.Provision

New variant in the sealed `DecisionOutcome` hierarchy (agency-core):

```java
record Provision(ProvisioningAdvice advice) implements DecisionOutcome {}
```

`ProvisioningAdvice` is the existing type from engine-api (`SwarmProvisioningAdvisor.ProvisioningAdvice`), carrying `shouldProvision`, `adjustedCapabilities`, `adjustedPolicy`, and `reasoning`.

All existing switch expressions on `DecisionOutcome` must add a `case Provision` arm. The compiler enforces exhaustiveness.

### ContentSummariser Adapters

Three adapter classes in `orchestration.topology`, each implementing `ContentSummariser<ObservationEvent, DecisionOutcome>`:

**RoutingReasoner** — Wraps `SwarmRoutingStrategy`. On `summarise()`:
1. Reads current candidates from `AgentProvider`
2. Builds `RoutingContext<SwarmContext>` from `SwarmContext` + candidates
3. Calls `routingStrategy.route(routingContext)`
4. Returns `DecisionOutcome.Route(routingDecision)`

**AggregationReasoner** — Wraps `CollectiveReasoningAggregation`. On `summarise()`:
1. Extracts agent results from observation events (key = `"member-completed"`, value = `AgentResult`)
2. Builds `AggregationContext<SwarmContext>` from `SwarmContext`
3. Calls `aggregation.aggregate(results, aggregationContext)`
4. Returns `DecisionOutcome.Aggregate(aggregationResult)`

**ProvisioningReasoner** — Wraps `SwarmProvisioningAdvisorImpl`. On `summarise()`:
1. Builds `ProvisioningContext` from `SwarmContext` state
2. Calls `advisor.advise(provisioningContext)`
3. Returns `DecisionOutcome.Provision(provisioningAdvice)`

Each adapter holds references to the strategy and `SwarmContext`. The `SwarmContext` is always up-to-date because `onSwarmEvent()` updates it before feeding observations into loops.

## Modified Types

### CentralisedTopology

Constructor gains a `SwarmProvisioningAdvisor` parameter (the SPI interface; `SwarmProvisioningAdvisorImpl` is the concrete implementation).

`configure(TopologyContext)` changes:
1. Creates `AutonomyBudgetPool` (as before)
2. Creates three `DecisionLoopDescriptor`s (as before)
3. Creates three adapter instances (`RoutingReasoner`, `AggregationReasoner`, `ProvisioningReasoner`)
4. Uses `context.loopFactory().create()` to build three `AgentDecisionLoop` instances from descriptors + adapters
5. Stores loops as fields
6. Returns `CoordinationPlan` with descriptors, executor, and pool (unchanged return type)

`onSwarmEvent(SwarmCoordinationEvent)` changes:
- After updating `SwarmContext` (as before), creates an `ObservationEvent` and calls `collect()` on the target loop(s) per the routing table above

New accessor:
- `List<AgentDecisionLoop> loops()` — exposes loops for runtime lifecycle management (tick, flush, close)

Executor dispatch:
- `executeRouting()` — adds `case Provision` arm; otherwise returns `SUCCESS` as current stubs do. No runtime consumes the executor today (`build()` has 0 callers). Real executor behavior is a future concern when the runtime is built.
- `executeAggregation()` — same: `SUCCESS` feedback for now
- `executeProvisioning()` — new arm; returns `SUCCESS` feedback. Actual provisioning action deferred to runtime layer.

### SwarmCoordinatorBuilder

- Remove `build()` method (0 callers — dead code)
- `buildTopology()` becomes the only build path
- `buildTopology()` creates `SwarmProvisioningAdvisorImpl` and passes it (as `SwarmProvisioningAdvisor`) to `CentralisedTopology` constructor

### SectionAwareContentSummariser

Fix existing bug (verified): `summarise()` at line 28-34 builds `promptContext` from `LoopPromptSection` contributions but line 35 calls `delegate.summarise(items, previous)` without passing it — the assembled context is silently discarded. This must be fixed for prompt sections to work correctly through the loop pipeline. The fix requires either passing the prompt context to the delegate (which means the delegate's interface must accept it) or prepending it to the observation items as a synthetic event.

### DecisionOutcome Switch Sites

All switch expressions on `DecisionOutcome` gain a `case Provision p -> ...` arm. The compiler enforces this via sealed interface exhaustiveness.

## Deleted Types

### LlmSwarmCoordinator

Eliminated entirely. After the refactoring:
- `assembleCoordinatorPrompt()` is redundant — `SectionAwareContentSummariser` handles prompt assembly in the loop pipeline
- All component references (strategies, context, budget, fallback) are held by `CentralisedTopology`
- `build()` in `SwarmCoordinatorBuilder` has 0 callers — dead code

## Test Plan

### New Tests

| Test class | What it verifies |
|------------|-----------------|
| `RoutingReasonerTest` | Observation→RoutingContext translation, strategy delegation, Route outcome wrapping, error handling |
| `AggregationReasonerTest` | Observation→AggregationContext translation, agent result extraction, Aggregate outcome wrapping |
| `ProvisioningReasonerTest` | Observation→ProvisioningContext translation, Provision outcome wrapping |
| `DecisionOutcome.Provision` | Sealed variant record equality, nullability |

### Modified Tests

| Test class | Changes |
|------------|---------|
| `CentralisedTopologyTest` | Verify `configure()` creates 3 loops. Verify `onSwarmEvent()` feeds observations into correct loop. Verify executor handles `Provision` outcome. Verify `loops()` accessor. |
| `SwarmCoordinatorBuilderTest` | Remove `build()`-related tests. Verify `buildTopology()` wires provisioning advisor. |

### Deleted Tests

| Test class | Reason |
|------------|--------|
| (none — `LlmSwarmCoordinatorTest` does not exist) | |

### Existing Tests

All existing tests must pass after refactoring. The `SwarmRoutingStrategyTest`, `CollectiveReasoningAggregationTest`, and `SwarmProvisioningAdvisorImplTest` verify the strategies independently and should not require changes.

## Constraints

- All existing tests must pass after refactoring
- `CentralisedTopology` must produce identical behavior to current `LlmSwarmCoordinator` for routing and aggregation
- `SwarmFallbackStrategy`, experience replay, and prompt customisation carry over through the existing strategy classes
- Budget management carries over through the `BudgetAwareSummariser` wrapper in the loop pipeline

## References

- `agency-core/.../orchestration/swarm/LlmSwarmCoordinator.java` — class being eliminated
- `agency-core/.../orchestration/swarm/SwarmCoordinatorBuilder.java` — dual build path (build/buildTopology)
- `agency-core/.../orchestration/topology/CentralisedTopology.java` — target topology implementation
- `agency-core/.../orchestration/loop/AgentDecisionLoop.java` — loop infrastructure
- `agency-core/.../orchestration/loop/DecisionLoopFactory.java` — loop creation with reasoner wiring
- `agency-core/.../orchestration/loop/DecisionOutcome.java` — sealed hierarchy (gains Provision variant)
- `agency-core/.../orchestration/swarm/SwarmRoutingStrategy.java` — team formation LLM strategy
- `agency-core/.../orchestration/swarm/CollectiveReasoningAggregation.java` — collective reasoning LLM strategy
- `agency-core/.../orchestration/swarm/SwarmProvisioningAdvisorImpl.java` — provisioning LLM strategy
- `summarisation-api/.../ContentSummariser.java` — adapter target interface
- `agency-core/.../orchestration/loop/SectionAwareContentSummariser.java` — prompt section bug to fix
- casehubio/agency#381 — issue
- casehubio/agency#377 — parent epic
- casehubio/agency#379 — AgentDecisionLoop (dependency, complete)
- casehubio/agency#380 — CoordinationTopology SPI (dependency, complete)
