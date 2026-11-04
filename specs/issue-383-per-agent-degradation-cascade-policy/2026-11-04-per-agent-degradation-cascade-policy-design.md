# Per-Agent Degradation + Topology-Owned Cascade Policy

**Issue:** casehubio/agency#383
**Epic:** casehubio/agency#377
**Depends on:** agency#380 (CoordinationTopology SPI), agency#382 (budget pool unification)

## Problem

When a decision loop's budget is exhausted, `AgentDecisionLoop.tick()` silently
skips — the agent stops reasoning entirely. `BudgetAwareSummariser` returns
`previous` on budget denial, which is indistinguishable from "no new observations."
There is no graceful degradation to heuristic reasoning, and no mechanism for the
topology to respond when multiple loops degrade simultaneously.

`SwarmFallbackStrategy` has heuristic methods (`fallbackRouting`,
`fallbackAggregation`, `fallbackProvisioningAdvice`) but they are disconnected
from the decision loop path. `SwarmDegradedEvent` exists as a scaffolded record
but is never emitted. `ExecutionEventListener.onSwarmDegraded()` is declared but
never invoked.

## Design

Two-level degradation model:

1. **Per-loop degradation** — each decision loop independently checks a
   `DegradationTrigger` before every LLM call. When triggered, the call is
   replaced with a heuristic response. Degradation is stateless per-call —
   the trigger is a pure function of current conditions. Recovery is implicit
   when the trigger clears (e.g., latency normalizes).

2. **Topology-owned cascade** — when the count of degraded loops crosses a
   configurable `CascadeThreshold`, the topology executes a one-way structural
   response. Individual loops may recover, but the cascade response persists
   for the execution.

## Architecture

### DegradationTrigger SPI

```
orchestration/degradation/DegradationTrigger.java          — @FunctionalInterface boolean shouldDegrade()
orchestration/degradation/CompositeDegradationTrigger.java  — OR-combines List<DegradationTrigger>
orchestration/degradation/BudgetDegradationTrigger.java     — adapts AgentBudgetView.isBudgetExhausted()
orchestration/degradation/LatencyDegradationTrigger.java    — tracks LLM call durations against LlmInvocationPolicy thresholds
```

`DegradationTrigger` is a functional interface:

```java
@FunctionalInterface
public interface DegradationTrigger {
    boolean shouldDegrade();
}
```

`CompositeDegradationTrigger` holds a `List<DegradationTrigger>` and returns
`true` if any trigger fires (OR semantics).

`BudgetDegradationTrigger` wraps `AgentBudgetView` and delegates to
`isBudgetExhausted()`. This is a state query — it reads the view's atomic
counters without acquiring budget. No circular dependency with
`BudgetAwareSummariser.tryAcquire()`, which is a budget acquisition.

`LatencyDegradationTrigger` tracks its own breach state. It observes
`BudgetTelemetryEvent.Outcome` events (which already record `latencyMs` for
every LLM call) and compares each duration against
`LlmInvocationPolicy.errorAfter()`. It maintains a sliding window of recent
results and fires `shouldDegrade()` when the breach count within the window
exceeds a configurable threshold (`DegradationPolicy.latencyBreachCount`).
When the loop is degraded (heuristic path), no `BudgetTelemetryEvent` is
emitted — correct, since there is no LLM latency to measure. On recovery,
the next LLM call emits an Outcome event, feeding the window.

`InvocationPolicyMonitor` is a stateless fire-and-forget logger — it cannot
be adapted. The latency trigger reuses `LlmInvocationPolicy` threshold
configuration but maintains independent state. It is wired as an additional
consumer on `BudgetAwareSummariser`'s existing `Consumer<BudgetTelemetryEvent>`
sink via a composite consumer.

### DegradationAwareSummariser

A new decorator in the summariser chain. Position in the chain:

```
reasoner → SectionAwareContentSummariser → BudgetAwareSummariser → DegradationAwareSummariser → InFlightGate
```

`DegradationAwareSummariser` is the outermost decorator before `InFlightGate`.
Before each `summarise()` call, it checks `trigger.shouldDegrade()`:

- **Not degraded:** delegates to the inner chain (sections → budget → LLM).
- **Degraded:** delegates to a heuristic `ContentSummariser`, bypassing budget
  acquisition, section enrichment, and the LLM call entirely. The heuristic
  path needs none of these.

On degradation state transition (non-degraded → degraded), emits a
`SwarmDegradedEvent` via a `Consumer<SwarmDegradedEvent>` sink. Maintains a
`previouslyDegraded` boolean to detect transitions. On recovery (degraded →
non-degraded), resets the flag so a subsequent re-degradation emits another
event.

```java
public class DegradationAwareSummariser<T, R> implements ContentSummariser<T, R> {

    private final ContentSummariser<T, R>        delegate;
    private final ContentSummariser<T, R>        heuristic;
    private final DegradationTrigger             trigger;
    private final Consumer<SwarmDegradedEvent>   eventSink;
    private final String                         loopName;
    private       boolean                        previouslyDegraded;

    @Override
    public CompletionStage<R> summarise(List<T> items, @Nullable R previous) {
        if (trigger.shouldDegrade()) {
            if (!previouslyDegraded) {
                previouslyDegraded = true;
                eventSink.accept(new SwarmDegradedEvent(
                    loopName, "degradation trigger fired", true));
            }
            return heuristic.summarise(items, previous);
        }
        previouslyDegraded = false;
        return delegate.summarise(items, previous);
    }
}
```

### Heuristic Summarisers

Per-loop-type heuristic `ContentSummariser` implementations that wrap
`SwarmFallbackStrategy` methods:

```
orchestration/degradation/HeuristicRoutingSummariser.java       — wraps fallbackRouting()
orchestration/degradation/HeuristicAggregationSummariser.java   — wraps fallbackAggregation()
orchestration/degradation/HeuristicProvisioningSummariser.java  — wraps fallbackProvisioningAdvice()
orchestration/degradation/NoOpHeuristicSummariser.java          — for non-Centralised loops: returns previous (pass-through)
```

Each heuristic summariser adapts a `SwarmFallbackStrategy` method to the
`ContentSummariser<ObservationEvent, DecisionOutcome>` interface. The
summariser parses the observation context to extract the inputs the fallback
method needs (routing candidates, agent results, swarm size), calls the
fallback, and wraps the result in the appropriate `DecisionOutcome` variant.

`NoOpHeuristicSummariser` is for non-Centralised topologies where each loop
does general reasoning. It returns `previous` — the agent stops making novel
decisions but does not block the swarm. This is the default heuristic.

### CascadeThreshold SPI

```
orchestration/degradation/CascadeThreshold.java              — boolean isBreached(Set<String> degradedLoopIds, int totalLoops)
orchestration/degradation/SimpleFractionThreshold.java        — fraction-based (e.g., > 0.5)
orchestration/degradation/PriorityWeightedThreshold.java      — RESERVED loops weighted higher
```

`CascadeThreshold` evaluates whether the cascade should fire:

```java
public interface CascadeThreshold {
    boolean isBreached(Set<String> degradedLoopIds, int totalLoops,
                       Map<String, BudgetPriority> loopPriorities);
}
```

`SimpleFractionThreshold` divides `degradedLoopIds.size()` by `totalLoops`
and compares against a configurable fraction.

`PriorityWeightedThreshold` assigns configurable weights per `BudgetPriority`
(default: RESERVED=2.0, NORMAL=1.0). Sums weights of degraded loops, divides
by total weighted capacity, compares against threshold.

### DegradationPolicy Configuration

```
orchestration/degradation/DegradationPolicy.java — configuration record
```

```java
public record DegradationPolicy(
    CascadeThresholdType thresholdType,    // SIMPLE_FRACTION, PRIORITY_WEIGHTED
    double cascadeThreshold,               // e.g., 0.5
    Map<BudgetPriority, Double> priorityWeights,  // for PRIORITY_WEIGHTED
    int latencyBreachWindow,               // sliding window size for LatencyDegradationTrigger
    int latencyBreachCount                 // breaches within window to trigger degradation
) {
    public static DegradationPolicy defaults() {
        return new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.5,
            Map.of(BudgetPriority.RESERVED, 2.0, BudgetPriority.NORMAL, 1.0),
            5, 3);
    }
}

public enum CascadeThresholdType { SIMPLE_FRACTION, PRIORITY_WEIGHTED }
```

Delivered via `TopologyContext`:

```java
public record TopologyContext(
    UUID caseId,
    TopologyDescriptor descriptor,
    AgentProvider agentProvider,
    DecisionLoopFactory loopFactory,
    AutonomyBudgetPolicy budgetPolicy,
    DegradationPolicy degradationPolicy    // NEW
) { ... }
```

### Cascade Tracking — Push Model

`DegradationAwareSummariser` emits `SwarmDegradedEvent` on degradation state
transitions. Events are delivered via a composite `Consumer<SwarmDegradedEvent>`
wired during topology `configure()`:

1. **Cascade path:** `CoordinationTopology.onDegradation(SwarmDegradedEvent)` —
   a new method on the interface. Each topology implementation maintains a
   `Set<String>` of degraded loop IDs (idempotent) and evaluates the
   `CascadeThreshold` on each event.

2. **Observability path:** `ExecutionEventListener.onSwarmDegraded(SwarmDegradedEvent)` —
   activates the existing scaffolded infrastructure.

The push model avoids a dead path in pull: when budget exhaustion causes
`AgentDecisionLoop.tick()` to short-circuit, no feedback fires and
`onFeedback()` is never called. Push ensures cascade evaluation happens
regardless of the trigger source.

`CoordinationTopology` gains a 4th method:

```java
public interface CoordinationTopology extends TopologyProvider {
    CoordinationPlan configure(TopologyContext context);
    void onSwarmEvent(SwarmCoordinationEvent event);
    void onFeedback(CoordinationFeedback feedback);
    void onDegradation(SwarmDegradedEvent event);    // NEW
}
```

### DecisionLoopFactory Changes

`DecisionLoopFactory.create()` gains parameters for the degradation decorator:

```java
public AgentDecisionLoop create(
        DecisionLoopDescriptor descriptor,
        DecisionContext context,
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner,
        ContentSummariser<ObservationEvent, DecisionOutcome> heuristic,  // NEW
        AutonomyBudgetPool pool,
        DegradationTrigger trigger,                                      // NEW
        Consumer<SwarmDegradedEvent> degradationSink) {                   // NEW

    // ... existing section + budget wrapping ...

    var degradationAware = new DegradationAwareSummariser<>(
        budgetAware, heuristic, trigger, degradationSink, descriptor.name());

    var gate = new InFlightGate<>(degradationAware);
    // ... rest unchanged ...
}
```

Each topology's `configure()` method constructs the appropriate
`DegradationTrigger` (composite of budget + latency) and heuristic
summariser per loop, then passes them to `DecisionLoopFactory.create()`.

### Per-Topology Cascade Responses

Each topology implements `onDegradation()` with its specific cascade
response. All cascade responses are **one-way ratchets** — once triggered,
the structural change persists for the execution.

| Topology | Cascade Response | Structural Change |
|----------|-----------------|-------------------|
| Distributed | Independent — no cascade | None. `onDegradation()` is a no-op. Each loop degrades independently. |
| Graduated | Centralise authority | `GraduationState.forceScope(agentId, STIMULUS_DRIVEN)` for all non-degraded agents. Concentrates decision-making at the root. Requires new `forceScope()` method (current `downgrade()` steps one level at a time). |
| Hybrid | Tighten local budgets | Reduce per-agent budget allocation for NORMAL-priority local loops via `AutonomyBudgetPool` adjustments. Central coordinator (RESERVED) retains full budget. |
| Centralised | All-heuristic | Force-degrade all remaining loops by setting a topology-level `cascadeTriggered` flag that each loop's `DegradationTrigger` reads. |
| Hierarchical | Collapse upward | Degraded leaf loops stop ticking. Parent loops absorb their observation scope — parent receives events that would have gone to collapsed children. |
| Hive | Simplify pattern | Downgrade `OrchestrationPatternType`: DEBATE → VOTING → SUPERVISOR. `HivePatternAdapter.execute()` dispatches dynamically on pattern type, so the existing dispatch path works once the topology passes the new type. |

**In-flight drain:** Cascade transitions complete any in-flight LLM
operations before applying structural changes. The one-way ratchet flag is
set immediately (preventing new LLM calls in the affected scope), but
structural mutation (scope changes, pattern switches) waits for in-flight
completions.

## Changes to Existing Files

| File | Change |
|------|--------|
| `CoordinationTopology.java` | Add `onDegradation(SwarmDegradedEvent)` method |
| `TopologyContext.java` | Add `DegradationPolicy degradationPolicy` field |
| `DecisionLoopFactory.java` | Add `heuristic`, `trigger`, `degradationSink` parameters to `create()` |
| `CentralisedTopology.java` | Implement `onDegradation()`, wire heuristic summarisers + triggers in `configure()` |
| `DistributedTopology.java` | Implement `onDegradation()` (no-op), wire triggers in `configure()` |
| `HybridTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `HierarchicalTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `GraduatedTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `HiveTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `GraduationState.java` | Add `forceScope(String agentId, DecisionScope scope)` method |
| `SwarmDegradedEvent.java` | No change — existing record is sufficient |
| `ExecutionEventListener.java` | No change — `onSwarmDegraded()` already declared |

## New Files

| File | Purpose |
|------|---------|
| `orchestration/degradation/DegradationTrigger.java` | Functional interface |
| `orchestration/degradation/CompositeDegradationTrigger.java` | OR-combines triggers |
| `orchestration/degradation/BudgetDegradationTrigger.java` | Adapts AgentBudgetView |
| `orchestration/degradation/LatencyDegradationTrigger.java` | Independent latency tracker |
| `orchestration/degradation/DegradationAwareSummariser.java` | Summariser decorator |
| `orchestration/degradation/HeuristicRoutingSummariser.java` | Wraps fallbackRouting() |
| `orchestration/degradation/HeuristicAggregationSummariser.java` | Wraps fallbackAggregation() |
| `orchestration/degradation/HeuristicProvisioningSummariser.java` | Wraps fallbackProvisioningAdvice() |
| `orchestration/degradation/NoOpHeuristicSummariser.java` | Pass-through for non-Centralised |
| `orchestration/degradation/CascadeThreshold.java` | Threshold interface |
| `orchestration/degradation/SimpleFractionThreshold.java` | Fraction-based threshold |
| `orchestration/degradation/PriorityWeightedThreshold.java` | Priority-weighted threshold |
| `orchestration/degradation/DegradationPolicy.java` | Configuration record |
| `orchestration/degradation/CascadeThresholdType.java` | Enum |

## Testing Strategy

### Unit Tests

- `DegradationTriggerTest` — composite OR semantics, individual trigger behavior
- `BudgetDegradationTriggerTest` — fires when `isBudgetExhausted()` returns true
- `LatencyDegradationTriggerTest` — sliding window breach counting, threshold crossing, recovery (breach count drops below threshold)
- `DegradationAwareSummariserTest` — delegates to heuristic when triggered, delegates to inner chain when not, emits `SwarmDegradedEvent` on transition only, recovery resets transition flag
- `SimpleFractionThresholdTest` — threshold arithmetic with various loop counts
- `PriorityWeightedThresholdTest` — weight calculations with RESERVED/NORMAL mixes
- `HeuristicRoutingSummariserTest` — produces correct `DecisionOutcome.Route` from `fallbackRouting()`
- `HeuristicAggregationSummariserTest` — produces correct `DecisionOutcome.Aggregate` from `fallbackAggregation()`
- `HeuristicProvisioningSummariserTest` — produces correct `DecisionOutcome.Provision` from `fallbackProvisioningAdvice()`

### Integration Tests (per topology)

- `CentralisedTopologyCascadeTest` — exhaust budget on 2/3 loops, verify remaining loop degrades to heuristic
- `DistributedTopologyCascadeTest` — exhaust budget on N loops, verify no cascade (independent)
- `GraduatedTopologyCascadeTest` — cross threshold, verify all scopes forced to STIMULUS_DRIVEN
- `HybridTopologyCascadeTest` — cross threshold, verify local loop budgets tightened
- `HierarchicalTopologyCascadeTest` — degrade leaf, verify parent absorbs observation scope
- `HiveTopologyCascadeTest` — cross threshold, verify pattern downgrade (DEBATE → VOTING)

### Edge Cases

- Latency recovery: trigger clears, loop resumes LLM, re-degradation emits new event
- Cascade with zero degraded loops post-recovery: cascade response stays (one-way ratchet)
- All loops degraded simultaneously: cascade fires once, not per-loop
- Single-loop topology (Distributed with 1 agent): cascade threshold never crossed (0/1 < 0.5)

## Out of Scope

- YAML configuration of `DegradationPolicy` (tracked by agency#385)
- `BudgetAwareSummariser` token estimator fix (separate concern)
- Cascade response for topologies not yet implemented beyond SPI
- Latency trigger integration with external circuit breakers

## References

- `CoordinationTopology.java` — SPI interface being extended
- `BudgetAwareSummariser.java` — existing budget decorator pattern
- `SwarmFallbackStrategy.java` — heuristic methods being adapted
- `SwarmDegradedEvent.java` — existing scaffolded event type
- `DecisionLoopFactory.java` — factory wiring point
- `GraduationState.java` — scope adjustment mechanism for Graduated cascade
- `HivePatternAdapter.java` — pattern dispatch for Hive cascade
- `AgentDecisionLoop.java:tick()` — dead-path evidence for push model (D6)
- `InvocationPolicyMonitor.java` — stateless logger, evidence for independent latency tracking (D1)
- Decision log: `decisions.md` — 11 decisions with rationale
