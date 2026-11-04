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
   replaced with a heuristic response. Budget degradation is stateless
   (reads current atomic counters). Latency degradation maintains a
   time-bounded sliding window — entries expire after a configurable
   duration, enabling implicit recovery without external intervention.

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
results — each entry carries a timestamp — and fires `shouldDegrade()` when
the breach count within the window exceeds a configurable threshold
(`DegradationPolicy.latencyBreachCount`).

Entries in the sliding window expire after
`DegradationPolicy.latencyBreachWindowDuration`. Before evaluating
`shouldDegrade()`, expired entries are pruned. When all entries have expired
(e.g., during extended degradation where no LLM calls produce new entries),
the window is empty, `shouldDegrade()` returns `false`, and the next
`summarise()` call routes through the LLM chain — a recovery probe. If the
probe breaches the threshold, the new entry enters the window and
re-triggers degradation. If it does not breach, recovery succeeds and
subsequent LLM calls continue normally.

Unlike `BudgetDegradationTrigger` (which is stateless — it reads the view's
atomic counters), `LatencyDegradationTrigger` is stateful. However, recovery
is still implicit: entries expire by time, the trigger clears without
external intervention, and the system self-heals.

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
`SwarmDegradedEvent` with `fallbackActive=true` via a
`Consumer<SwarmDegradedEvent>` sink. On recovery (degraded → non-degraded),
emits a `SwarmDegradedEvent` with `fallbackActive=false` so the topology
can remove the loop from its degraded set and observers can distinguish
current from historical degradation. Uses an `AtomicBoolean` with
`compareAndSet()` for exactly-once event emission under concurrent access.

```java
public class DegradationAwareSummariser<T, R> implements ContentSummariser<T, R> {

    private final ContentSummariser<T, R>        delegate;
    private final ContentSummariser<T, R>        heuristic;
    private final DegradationTrigger             trigger;
    private final Consumer<SwarmDegradedEvent>   eventSink;
    private final String                         loopName;
    private final AtomicBoolean                  previouslyDegraded = new AtomicBoolean(false);

    @Override
    public CompletionStage<R> summarise(List<T> items, @Nullable R previous) {
        if (trigger.shouldDegrade()) {
            if (previouslyDegraded.compareAndSet(false, true)) {
                eventSink.accept(new SwarmDegradedEvent(
                    loopName, "degradation trigger fired", true));
            }
            return heuristic.summarise(items, previous);
        }
        if (previouslyDegraded.compareAndSet(true, false)) {
            eventSink.accept(new SwarmDegradedEvent(
                loopName, "degradation cleared", false));
        }
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
`ContentSummariser<ObservationEvent, DecisionOutcome>` interface. Each
heuristic mirrors its LLM-based counterpart's data access pattern —
constructor-injected dependencies where the reasoner uses them, event
extraction only where the reasoner uses it.

Adaptation contracts per heuristic:

- **`HeuristicRoutingSummariser`:** holds a
  `Supplier<List<RoutingCandidate>>` (same dependency as
  `RoutingReasoner`). Calls `fallbackRouting(candidateSupplier.get())`
  → `RoutingDecision`. Wraps as `DecisionOutcome.Route(decision)`.
  Does not extract from `ObservationEvent` items — `RoutingReasoner`
  ignores them too.

- **`HeuristicAggregationSummariser`:** extracts `AgentResult` list from
  events keyed `"member-completed"` (same pattern as
  `AggregationReasoner`). Calls `fallbackAggregation(results)` →
  `AggregationResult`. Wraps as `DecisionOutcome.Aggregate(result)`.

- **`HeuristicProvisioningSummariser`:** holds a `SwarmContext` reference
  (same dependency as `ProvisioningReasoner`). Reads provisioning state
  from `SwarmContext` to obtain `currentSize` and `maxSize`. Calls
  `fallbackProvisioningAdvice(currentSize, maxSize)` → `boolean`. Wraps
  as `DecisionOutcome.Provision(new ProvisioningAdvice(shouldProvision,
  null, null, "heuristic-fallback"))`. The `boolean` maps directly to
  `ProvisioningAdvice.shouldProvision` — the nullable enrichment fields
  (`adjustedCapabilities`, `adjustedPolicy`) are legitimately absent for
  a heuristic fallback. Does not extract from `ObservationEvent` items —
  `ProvisioningReasoner` ignores them too.

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
    int latencyBreachWindow,               // max sliding window size for LatencyDegradationTrigger
    int latencyBreachCount,                // breaches within window to trigger degradation
    Duration latencyBreachWindowDuration   // time-based expiry for sliding window entries
) {
    public DegradationPolicy {
        Objects.requireNonNull(thresholdType, "thresholdType must not be null");
        if (cascadeThreshold <= 0.0 || cascadeThreshold > 1.0)
            throw new IllegalArgumentException("cascadeThreshold must be in (0.0, 1.0]");
        Objects.requireNonNull(priorityWeights, "priorityWeights must not be null");
        if (latencyBreachWindow < 1)
            throw new IllegalArgumentException("latencyBreachWindow must be >= 1");
        if (latencyBreachCount < 1 || latencyBreachCount > latencyBreachWindow)
            throw new IllegalArgumentException(
                "latencyBreachCount must be in [1, latencyBreachWindow]");
        Objects.requireNonNull(latencyBreachWindowDuration,
            "latencyBreachWindowDuration must not be null");
        if (latencyBreachWindowDuration.isNegative() || latencyBreachWindowDuration.isZero())
            throw new IllegalArgumentException(
                "latencyBreachWindowDuration must be positive");
    }

    public static DegradationPolicy defaults() {
        return new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.5,
            Map.of(BudgetPriority.RESERVED, 2.0, BudgetPriority.NORMAL, 1.0),
            5, 3, Duration.ofMinutes(2));
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
   `Set<String>` of degraded loop IDs backed by
   `ConcurrentHashMap.newKeySet()` and evaluates the `CascadeThreshold`
   on each event. When `fallbackActive=true`, the loop is added to the
   set. When `fallbackActive=false` (recovery), the loop is removed from
   the set. This ensures the degraded set reflects *current* degradation
   state, not historical. The cascade threshold is only evaluated on
   `fallbackActive=true` events — recovery events update the set but do
   not reverse a cascade (D7 one-way ratchet).

   `onDegradation()` is called concurrently from multiple loops' threads
   when degradation events arrive simultaneously (e.g., budget exhaustion
   across a shared pool). Cascade activation is guarded by an
   `AtomicBoolean cascadeTriggered` with `compareAndSet(false, true)` —
   exactly-once activation even under concurrent threshold breaches.

2. **Observability path:** `ExecutionEventListener.onSwarmDegraded(SwarmDegradedEvent)` —
   activates the existing scaffolded infrastructure. Receives both
   degradation (`fallbackActive=true`) and recovery (`fallbackActive=false`)
   events, enabling observers to track current degradation state.

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

`DecisionLoopFactory.create()` gains parameters for the degradation decorator
and switches from the 2-arg to 5-arg `BudgetAwareSummariser` constructor to
enable `BudgetTelemetryEvent` emission (required for `LatencyDegradationTrigger`
to receive latency data):

```java
public AgentDecisionLoop create(
        DecisionLoopDescriptor descriptor,
        DecisionContext context,
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner,
        ContentSummariser<ObservationEvent, DecisionOutcome> heuristic,  // NEW
        AutonomyBudgetPool pool,
        DegradationTrigger trigger,                                      // NEW
        Consumer<SwarmDegradedEvent> degradationSink,                    // NEW
        Consumer<BudgetTelemetryEvent> budgetEventSink) {                // NEW

    // ... existing section wrapping ...

    ContentSummariser<...> budgetAware = new BudgetAwareSummariser<>(
        withSections, view, budgetEventSink, r -> 0L, "unknown");

    var degradationAware = new DegradationAwareSummariser<>(
        budgetAware, heuristic, trigger, degradationSink, descriptor.name());

    var gate = new InFlightGate<>(degradationAware);
    // ... rest unchanged ...
}
```

The `budgetEventSink` is a composite consumer wired by the topology to
deliver events to both `LatencyDegradationTrigger` (for latency window
updates) and any external telemetry subscribers. The token estimator
(`r -> 0L`) is a placeholder — the token estimator fix is a separate
concern (tracked by agency#402).

Each topology's `configure()` method constructs the appropriate
`DegradationTrigger` (composite of budget + latency) and heuristic
summariser per loop, then passes them to `DecisionLoopFactory.create()`.

### AgentDecisionLoop Changes

The budget guards in `tick()` and `flush()` are removed.
`DegradationAwareSummariser` now handles budget exhaustion via
`BudgetDegradationTrigger` — when the trigger fires, the summariser routes
to the heuristic path, producing a heuristic response (rather than
returning `null` from `tick()` or throwing `BudgetExhaustedException` from
`flush()`). This is the primary change that enables the spec's design goal:
replacing silent `tick()` short-circuiting with heuristic responses.

### Per-Topology Cascade Responses

Each topology implements `onDegradation()` with its specific cascade
response. All cascade responses are **one-way ratchets** — once triggered,
the structural change persists for the execution.

| Topology | Cascade Response | Structural Change |
|----------|-----------------|-------------------|
| Distributed | Independent — no cascade | None. `onDegradation()` is a no-op. Each loop degrades independently. |
| Graduated | Reduce autonomy | `GraduationState.forceScope(agentId, STIMULUS_DRIVEN)` for all agents. All agents stop proactive reasoning and respond only to incoming signals. `forceScope()` also sets a `volatile DecisionScope scopeFloor` that prevents subsequent `upgrade()` calls from exceeding the forced scope — `upgrade()` clamps the result against the floor. This ensures the cascade ratchet holds even as `onFeedback()` → `adjustScope()` continues processing feedback signals. Requires new `forceScope()` method with scope floor enforcement (current `downgrade()` steps one level at a time). |
| Hybrid | Force-degrade NORMAL loops | Each NORMAL-priority local loop's `CompositeDegradationTrigger` includes a `CascadeDegradationTrigger`. On cascade, `CascadeDegradationTrigger.activate()` forces NORMAL loops to heuristic. The RESERVED central coordinator does not include a `CascadeDegradationTrigger` and retains LLM-backed reasoning. |
| Centralised | All-heuristic | Each loop's `CompositeDegradationTrigger` includes a `CascadeDegradationTrigger`. On cascade, all triggers are activated, forcing all remaining loops to heuristic. |
| Hierarchical | Force-degrade leaves | Each leaf loop's `CompositeDegradationTrigger` includes a `CascadeDegradationTrigger`. On cascade, leaf triggers are activated, forcing leaf loops to heuristic. Parent loops retain LLM-backed reasoning. |
| Hive | Simplify pattern + force-degrade workers | Downgrade `OrchestrationPatternType`: DEBATE → VOTING → SUPERVISOR. `HiveTopology.patternType` changes from `final` to `volatile` to enable runtime mutation. `HivePatternAdapter.execute()` dispatches dynamically on pattern type. Additionally, each NORMAL-priority worker loop's `CompositeDegradationTrigger` includes a `CascadeDegradationTrigger` — on cascade, workers are force-degraded to heuristic. The RESERVED coordinator retains LLM-backed reasoning. This ensures the cascade is meaningful even when the topology is already at SUPERVISOR. |

**`CascadeDegradationTrigger`:** A `DegradationTrigger` implementation
backed by a `volatile boolean` flag. `activate()` sets the flag,
`shouldDegrade()` reads it. Wired into `CompositeDegradationTrigger`
alongside budget and latency triggers for loops that should respond to
cascade. The topology's `onDegradation()` calls `activate()` on the
appropriate triggers when the cascade threshold is breached.

**In-flight safety:** The `DegradationTrigger` mechanism provides natural
in-flight safety. `shouldDegrade()` is checked at the start of each
`summarise()` call. An in-flight LLM call (started before the cascade
flag was set) completes normally — the `InFlightGate` prevents
overlapping calls, so no new LLM call starts until the in-flight one
completes. The next `summarise()` call sees the cascade trigger and
routes to the heuristic.

Per-topology specifics:

- **Graduated:** `forceScope()` writes to `ConcurrentHashMap` and takes
  effect immediately for new calls. In-flight calls that started under the
  old scope complete normally — their results are the last LLM decisions
  from the pre-cascade state. Scope enforcement at execution time catches
  violations.
- **Hive:** Pattern type change (volatile write) applies to new
  `HivePatternAdapter.execute()` calls only. In-flight pattern executions
  (e.g., DEBATE rounds) complete with their original pattern semantics.
  New calls dispatch on the downgraded pattern type. Worker
  `CascadeDegradationTrigger` activation is immediate alongside the
  pattern change.
- **Hierarchical / Hybrid / Centralised:** `CascadeDegradationTrigger`
  activation is immediate. In-flight completions are the last pre-cascade
  results. No structural mutation needed beyond the flag.

## Changes to Existing Files

| File | Change |
|------|--------|
| `AgentDecisionLoop.java` | Remove budget guards from `tick()` and `flush()` — degradation is handled by `DegradationAwareSummariser` in the chain |
| `CoordinationTopology.java` | Add `onDegradation(SwarmDegradedEvent)` method |
| `TopologyContext.java` | Add `DegradationPolicy degradationPolicy` field |
| `DecisionLoopFactory.java` | Add `heuristic`, `trigger`, `degradationSink`, `budgetEventSink` parameters to `create()`; switch `BudgetAwareSummariser` to 5-arg constructor |
| `CentralisedTopology.java` | Implement `onDegradation()`, wire heuristic summarisers + triggers in `configure()` |
| `DistributedTopology.java` | Implement `onDegradation()` (no-op), wire triggers in `configure()` |
| `HybridTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `HierarchicalTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `GraduatedTopology.java` | Implement `onDegradation()`, wire triggers in `configure()` |
| `HiveTopology.java` | Change `patternType` from `final` to `volatile`; implement `onDegradation()` (pattern downgrade + worker `CascadeDegradationTrigger` activation); wire triggers in `configure()` |
| `GraduationState.java` | Add `forceScope(String agentId, DecisionScope scope)` method with `volatile DecisionScope scopeFloor`; `upgrade()` clamps result against floor to preserve cascade ratchet |
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
| `orchestration/degradation/CascadeDegradationTrigger.java` | Volatile-flag trigger activated by topology on cascade |
| `orchestration/degradation/CascadeThresholdType.java` | Enum |

## Testing Strategy

### Unit Tests

- `DegradationTriggerTest` — composite OR semantics, individual trigger behavior
- `BudgetDegradationTriggerTest` — fires when `isBudgetExhausted()` returns true
- `LatencyDegradationTriggerTest` — sliding window breach counting, threshold crossing, time-based entry expiry, recovery after window empties, re-degradation on probe breach
- `DegradationAwareSummariserTest` — delegates to heuristic when triggered, delegates to inner chain when not, emits `SwarmDegradedEvent(fallbackActive=true)` on degradation transition, emits `SwarmDegradedEvent(fallbackActive=false)` on recovery transition, exactly-once emission under concurrent access
- `CascadeDegradationTriggerTest` — `activate()` causes `shouldDegrade()` to return true, volatile visibility across threads
- `SimpleFractionThresholdTest` — threshold arithmetic with various loop counts
- `PriorityWeightedThresholdTest` — weight calculations with RESERVED/NORMAL mixes
- `HeuristicRoutingSummariserTest` — produces correct `DecisionOutcome.Route` from `fallbackRouting()`
- `HeuristicAggregationSummariserTest` — produces correct `DecisionOutcome.Aggregate` from `fallbackAggregation()`
- `HeuristicProvisioningSummariserTest` — produces correct `DecisionOutcome.Provision` from `fallbackProvisioningAdvice()`

### Integration Tests (per topology)

- `CentralisedTopologyCascadeTest` — exhaust budget on 2/3 loops, verify remaining loop degrades to heuristic
- `DistributedTopologyCascadeTest` — exhaust budget on N loops, verify no cascade (independent)
- `GraduatedTopologyCascadeTest` — cross threshold, verify all scopes forced to STIMULUS_DRIVEN
- `HybridTopologyCascadeTest` — cross threshold, verify NORMAL loops force-degraded, RESERVED coordinator retains LLM
- `HierarchicalTopologyCascadeTest` — cross threshold, verify leaf loops force-degraded, parent loops retain LLM
- `HiveTopologyCascadeTest` — cross threshold, verify pattern downgrade (DEBATE → VOTING) and worker loops force-degraded; verify cascade at SUPERVISOR still force-degrades workers

### Edge Cases

- Latency recovery: window entries expire, trigger clears, loop probes LLM, re-degradation emits new event
- Recovery event: `SwarmDegradedEvent(fallbackActive=false)` removes loop from degraded set — premature cascade prevented
- Cascade with zero degraded loops post-recovery: cascade response stays (one-way ratchet)
- All loops degraded simultaneously: cascade fires once, not per-loop
- Single-loop topology (Distributed with 1 agent): cascade threshold never crossed (0/1 < 0.5)

## Out of Scope

- YAML configuration of `DegradationPolicy` (tracked by agency#385)
- `BudgetAwareSummariser` token estimator fix (tracked by agency#402)
- Cascade response for topologies not yet implemented beyond SPI (tracked by agency#403)
- Latency trigger integration with external circuit breakers (tracked by agency#404)
- Hierarchical observation routing infrastructure — parent absorbing child event scope (tracked by agency#405)

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
