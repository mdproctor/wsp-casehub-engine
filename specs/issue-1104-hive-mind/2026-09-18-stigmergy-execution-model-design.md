# Stigmergy Execution Model — Design Spec

**Issue:** casehubio/engine#1111
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-18
**Decisions:** D60–D72

## Problem

The Hive Mind epic has delivered six foundation SPIs for stigmergic coordination:

| SPI | Issue | What it provides |
|-----|-------|-----------------|
| Environment observation | #1105 | `EnvironmentObserver`, `ObservationRegistry` — agents perceive CaseContext patterns |
| Signal/pheromone model | #1106 | `SignalRegistry`, `SignalSpace` — temporal signals with decay and reinforcement |
| Dynamic interest registration | #1107 | `InterestSpace`, `InterestDeclaration` — agents register observation interests at runtime |
| Agent discovery & neighbors | #1108 | `NeighborSpace` — emergent topology from shared activity |
| Local rule evaluation | #1109 | `RuleSpace`, `RuleRegistry`, `LocalRule` — per-agent condition→action rules |
| Convergence detection | #1110 | `ConvergenceDetector`, `BudgetEnforcer`, `ActivityTracker` — emergent termination |

These SPIs are individually functional and independently tested. The evaluation pipeline already drives the perceive→decide→act cycle: `rules()` → `goals()` → `observations()` → `localRules()` → `convergenceDetection()`.

**What's missing is the composing abstraction.** A case author who wants stigmergy must currently configure 6+ separate config blocks, understand which lifecycle scopes to use, know that bindings need trivially-true triggers, and wire everything manually. There is no "this is a stigmergy case" declaration, no agent lifecycle tracking, no coordination pattern detection, and no audit model that captures the emergent coordination story.

Stigmergy — from Greek *stigma* (mark) + *ergon* (work) — is coordination through environment modification. Agents don't communicate directly; they observe and modify a shared environment, and coordination emerges from their individual reactions to environmental changes. The canonical example is ant pheromone trails: an ant deposits pheromone, other ants detect and follow it, trails to exhausted food sources decay naturally.

The unified execution model spec (§3.1) classifies stigmergy as a choreographed dispatch archetype with self-selected routing, environment-change activation, merge-to-environment aggregation, and emergent termination.

## Architecture Overview

The StigmergyExecutionModel is a composition of three layers, each addressing a different concern:

```
┌─────────────────────────────────────────────────────────┐
│                StigmergyExecutionModel                   │
│                                                         │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────┐  │
│  │StigmergyConfig│  │StigmergyStrategy│  │Stigmergy-  │  │
│  │  (api)        │  │ (planning-core) │  │Coordinator │  │
│  │               │  │                 │  │(runtime-   │  │
│  │ • defaults    │  │ • first-cycle   │  │ core)      │  │
│  │ • coordination│  │   dispatch      │  │            │  │
│  │   thresholds  │  │ • health mon.   │  │ • agent    │  │
│  │               │  │ • re-dispatch   │  │   lifecycle│  │
│  │               │  │                 │  │ • pattern  │  │
│  │               │  │                 │  │   detection│  │
│  └───────┬───────┘  └────────┬────────┘  └──────┬─────┘  │
│          │                   │                   │        │
│          ▼                   ▼                   ▼        │
│  ┌───────────────────────────────────────────────────┐   │
│  │         Existing Foundation SPIs (#1105-#1110)     │   │
│  │  SignalRegistry · ObservationRegistry · RuleReg.  │   │
│  │  InterestSpace · NeighborSpace · ActivityTracker  │   │
│  │  ConvergenceDetector · BudgetEnforcer             │   │
│  └───────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Why three layers, not one.** The unified execution model (§2.7) identifies three orthogonal axes: structure, dispatch, and technique. Stigmergy doesn't fit cleanly into any of these — it is a **coordination model**, a fourth axis describing how agents interact after dispatch. This coordination concern touches dispatch (agents need to be dispatched), configuration (SPIs need coherent defaults), and post-dispatch lifecycle (agents perceive, decide, act). The three-layer decomposition maps one layer per concern. Each is independently testable and extensible.

## 1. StigmergyConfig

`StigmergyConfig` (`api`, `io.casehub.api.model.stigmergy`) — configuration surface that activates stigmergy mode and provides coordinated defaults.

### Structure

Two sub-records:

```java
record StigmergyConfig(
    StigmergyDefaults defaults,
    CoordinationConfig coordination
)

record StigmergyDefaults(
    Duration signalHalfLife,           // default PT5M
    Double effectiveZeroThreshold,     // default 0.01
    Integer maxSignalsPerCase,         // default 100
    Integer maxObserversPerCase,       // default 20
    Integer maxRulesPerCase,           // default 50
    Duration rateWindow,               // default PT60S
    Duration stabilityWindow,          // default PT30S
    Integer maxDispatches,             // default 10000
    Integer maxEvaluationCycles        // default 50000
)

record CoordinationConfig(
    Integer consensusThreshold,        // default 2
    Double stormRateMultiplier,        // default 10.0
    Double interestHotspotThreshold    // default 0.6
)
```

All fields are nullable. Null means use the per-SPI default or system default.

### Config Resolution

`StigmergyConfig` is a preset — it provides default values that fill in missing per-SPI config fields on `CaseDefinition`. Explicit per-SPI configuration always wins.

**Resolution order:** explicit per-SPI config > StigmergyDefaults > system defaults (null = disabled).

Example: if a case declares both `stigmergyConfig.defaults.signalHalfLife = PT5M` and `signalConfig.defaultHalfLife = PT2M`, the explicit `signalConfig` value (PT2M) is used.

This means a case author can write `stigmergyConfig: {}` with zero configuration to get a complete working stigmergy setup with sensible defaults. Power users override specific aspects without losing the rest.

### Field Mapping: StigmergyDefaults → CaseDefinition Config Objects

| StigmergyDefaults field | Target config object | Target field |
|------------------------|---------------------|-------------|
| `signalHalfLife` | `SignalConfig` | `defaultHalfLife` |
| `effectiveZeroThreshold` | `SignalConfig` | `effectiveZeroThreshold` |
| `maxSignalsPerCase` | `SignalConfig` | `maxSignalsPerCase` |
| `maxObserversPerCase` | `ObservationConfig` | `maxObserversPerCase` |
| `maxRulesPerCase` | `RuleConfig` | `maxRulesPerCase` |
| `rateWindow` | `ConvergenceThresholdConfig` | `rateWindow` |
| `stabilityWindow` | `ConvergenceThresholdConfig` | `stabilityWindow` |
| `maxDispatches` | `BudgetConfig` | `maxDispatches` |
| `maxEvaluationCycles` | `BudgetConfig` | `maxEvaluationCycles` |

Fields not in StigmergyDefaults (e.g., `BudgetConfig.maxSignalDeposits`, `BudgetConfig.maxContextMutations`, the four individual rate thresholds on `ConvergenceThresholdConfig`) are left at their per-SPI defaults (null = no enforcement / system default). The stigmergy preset provides the most commonly needed knobs; power users use explicit per-SPI configs for full control.

**Initialization guarantee:** The case initializer MUST ensure that `budgetConfig` and `convergenceThresholdConfig` are non-null for any case with `stigmergyConfig` present. If neither explicit configs nor StigmergyDefaults provide values, the initializer creates these config objects with system defaults. This prevents the `convergenceDetection()` null guard from silently skipping budget/convergence checks.

### CaseDefinition Integration

`CaseDefinition` gains a nullable `stigmergyConfig` field. Its presence activates stigmergy mode. The case initializer resolves effective per-SPI configs using the resolution order above during case initialization.

### YAML

```yaml
spec:
  planningStrategy: stigmergy

  stigmergyConfig:
    defaults:
      signalHalfLife: PT5M
      effectiveZeroThreshold: 0.01
      maxSignalsPerCase: 100
      stabilityWindow: PT30S
      maxDispatches: 10000
    coordination:
      consensusThreshold: 2
      stormRateMultiplier: 10.0
      interestHotspotThreshold: 0.6
```

Both `defaults:` and `coordination:` sub-blocks are optional. An empty `stigmergyConfig:` block activates stigmergy mode with all defaults.

### Trade-offs

Two resolution paths (preset vs explicit) add a layer of indirection. Mitigated: resolution is a simple null-check per field at case initialization time. An EventLog entry at case start logs the effective config source per block for diagnostics.

## 2. StigmergyStrategy

`StigmergyStrategy` (`planning-core`, `io.casehub.engine.plan.strategy`) — named `PlanningStrategy` that manages agent population dispatch.

### Strategy Identity

Registered as a `NamedStrategy` with `id() = "stigmergy"`, resolved by `StrategyResolver`. Follows the `DefaultPlanningStrategy` pattern. Cases declare `planningStrategy: stigmergy` in YAML or `CaseDefinition.builder().planningStrategy("stigmergy")` in Java.

**Dependency:** Requires the planning module on the classpath. `PlanningStrategyLoopControl` activates when the planning module is present, replacing `ChoreographyLoopControl` as the active `LoopControl` implementation.

### Agent Model

When a compound PlanItem's `planningStrategy` is `"stigmergy"`, ALL bindings within that compound are stigmergy agents. No explicit agent list — the compound's strategy scopes to all its children.

If a case needs non-stigmergic bindings alongside stigmergy agents, it uses nested compounds: a root compound (choreography) containing a stigmergy compound (stigmergy agents) alongside normal bindings. This follows the unified execution model's composition-via-nesting principle (§2.1).

### Trigger-Less Bindings

Bindings in a stigmergy compound do not require trigger conditions (`on:`, `when:`). The strategy handles dispatch.

**Implementation path:** the case initializer automatically adds `ScopeActivatedTrigger` to bindings within a stigmergy compound when no trigger is specified. This is transparent to the YAML author — they omit `on:` and the initializer fills in the scope-activated trigger. `PlanningStrategyLoopControl.collectScopeActivatedBindings()` handles dispatch via its existing path when the compound activates.

If a binding in a stigmergy compound has an explicit trigger, the strategy logs WARN on first encounter — triggers are redundant when the strategy manages dispatch.

### Dispatch Behavior

**First cycle:** `select()` dispatches all bindings. Bindings in the root compound (case-level `planningStrategy: stigmergy`) receive the implicit root compound's scope — effectively `CASE` lifecycle scope since the root compound spans the entire case. The case initializer adds `ScopeActivatedTrigger` to trigger-less bindings, and `PlanningStrategyLoopControl.collectScopeActivatedBindings()` dispatches them when the root compound activates at case start. All agents join simultaneously at case start. The coordinator is notified (`initializeCase()`), and each agent enters the `JOINING` lifecycle state.

**Subsequent cycles:** `select()` returns an empty list (no new dispatches). The strategy monitors agent health via coordinator queries:
- Agents stuck in `JOINING` past a timeout threshold are logged at WARN
- Failed agents (worker execution error) are re-dispatched

No condition-gated or population-managed dispatch in v1. These are natural extension points for #1112 (swarm) and #1113 (self-provisioning) — `select()` gains additional logic without architectural change.

### Trade-offs

All agents start simultaneously — no staggered or conditional joining. Acceptable for rule-based stigmergy where the agent population is known at case definition time.

## 3. StigmergyCoordinator

`StigmergyCoordinator` (`runtime-core`, `io.casehub.engine.internal.stigmergy`, `@ApplicationScoped`, `Resettable`) — tracks agent lifecycle and detects coordination patterns.

### Agent Lifecycle Tracking

Three lifecycle states per agent per case:

```
JOINING ──────────→ ACTIVE ──────────→ DEPARTED
  (dispatched,        (setup done,       (voluntarily left
   setup in progress)  participating)     or case terminating)
```

**State data:**

```java
record AgentState(
    String agentId,
    String bindingName,
    AgentLifecycleState state,      // JOINING, ACTIVE, DEPARTED
    Instant joinedAt,
    Instant activatedAt,            // null while JOINING
    Instant departedAt,             // null until DEPARTED
    Instant lastActivity            // updated on rule firing or signal deposit
)
```

**Transitions:**

| Trigger | From | To | Event |
|---------|------|----|-------|
| Worker dispatched | — | `JOINING` | `STIGMERGY_AGENT_JOINED` |
| Worker `execute()` completes successfully | `JOINING` | `ACTIVE` | `STIGMERGY_AGENT_ACTIVATED` |
| `WorkerRuntime.leave()` called | `ACTIVE` | `DEPARTED` | `STIGMERGY_AGENT_DEPARTED` |
| Case terminates | any | evicted | — (handled by `evictByCase()`) |

**Storage:** `ConcurrentHashMap<UUID, Map<String, AgentState>>` keyed by `(caseId, agentId)`. Evicted on case termination via `CaseStatusChangedHandler`, same pattern as all other per-case registries.

### Population Queries

| Method | Returns |
|--------|---------|
| `activeAgents(caseId)` | Agents in `ACTIVE` state |
| `activeCount(caseId)` | Count of `ACTIVE` agents |
| `allActive(caseId)` | True when all agents are past `JOINING` |
| `isStigmergyCase(caseId)` | Whether this case uses stigmergy |

### Coordination Pattern Detection

Three detectors run during the `convergenceDetection()` pipeline phase for stigmergy cases. Each is a method on `StigmergyCoordinator`, examining existing registry state. No new storage systems — pure computation over existing data.

#### 3a. Signal Consensus Detection

Detects when a signal has been independently reinforced by multiple agents — the core stigmergy mechanism (pheromone trail formation).

**Trigger:** `signal.sources().size() >= coordinationConfig.consensusThreshold()` (default: 2).

**Computation:** O(S) where S ≤ `maxSignalsPerCase` (default 100). `SignalRegistry` gains a `consensusSignals(UUID caseId, int minSources)` method that returns `Map<String, Signal>` — raw Signal records (which have `sources()`) filtered to those with `sources.size() >= minSources` and above effective-zero threshold. This avoids exposing `sources` on `PerceivedSignal` (which is the worker-facing read model) while giving the coordinator the data it needs.

**Event:** `SIGNAL_CONSENSUS_DETECTED` with metadata: `signalName`, `reinforcementCount`, `sources` (agent IDs), `effectiveStrength`.

**Dedup:** Tracks which signals have already fired consensus events (per-case `Set<String>`). Fires once per signal crossing the threshold. Resets when the signal decays below effective-zero (signal evicted from perception view).

#### 3b. Coordination Storm Detection

Detects pathological coordination — agents in a feedback loop, each reacting to each other's changes faster than convergence can establish. This is the early warning; `BudgetEnforcer` is the hard gate.

**Trigger:** Any `ActivityTracker` rate exceeds its corresponding `ConvergenceThresholdConfig` threshold × `coordinationConfig.stormRateMultiplier()`. Each rate uses its own threshold: dispatch rate vs `dispatchRateThreshold × 10.0`, signal deposit rate vs `signalDepositRateThreshold × 10.0`, context mutation rate vs `contextMutationRateThreshold × 10.0`, evaluation rate vs `evaluationRateThreshold × 10.0`. The four rates have different natural baselines (evaluation at 0.5/s vs dispatch at 0.1/s), so per-rate storm thresholds are essential.

**Computation:** O(1) — reads four rate values from `ActivityTracker`.

**Event:** `COORDINATION_STORM_DETECTED` with metadata: `stormingMetrics` (which rates exceeded), `currentRates`, `stormThreshold`.

**Dedup:** Fires once when storm begins. Resets (can fire again) after ALL rates drop below storm threshold (hysteresis — prevents repeated firing during sustained storms).

#### 3c. Interest Convergence Detection

Detects when the swarm's collective attention is focusing on specific keys — often a leading indicator of behavioral convergence.

**Trigger:** Interest landscape hotspot score exceeds `coordinationConfig.interestHotspotThreshold()` (default: 0.6).

**Hotspot score algorithm:** For each watched key, count the number of distinct agents observing it. The hotspot score for a key is `watchingAgentCount / totalActiveAgents`. A score of 1.0 means every agent watches that key. A score of 0.6 (default threshold) means 60% of agents share that interest — strong collective attention. The coordinator iterates `ObservationRegistry.getObservers(caseId)`, extracts `watchedKeys()` per observer, groups by key, and counts unique agent IDs per key.

**Computation:** O(N × K) where N ≤ `maxObserversPerCase` (default 20) and K = average watched keys per observer (typically small). Single pass over observer registrations.

**Event:** `INTEREST_CONVERGENCE_DETECTED` with metadata: `hotspotKeys`, `watchingAgentCount`, `hotspotScore`.

**Dedup:** Fires once per hotspot key. Resets when hotspot dissolves (agents deregister interests that reduced the score below threshold).

### Pipeline Integration

The `convergenceDetection()` phase in `CaseContextChangedEventHandler` already runs `ConvergenceDetector` and `BudgetEnforcer`. For stigmergy cases, it additionally runs coordination pattern detection:

```
convergenceDetection(caseInstance, caseDefinition):
    // Stigmergy coordination detection runs BEFORE the existing null guard,
    // since a stigmergy case may not have explicit budgetConfig/convergenceThresholdConfig
    // (they're filled from StigmergyDefaults during initialization, but the guard
    // must not block coordination detection).
    if coordinator.isStigmergyCase(caseInstance.id()):
        coordinator.detectPatterns(caseInstance.id(),
            caseDefinition.stigmergyConfig())

    // Existing — requires non-null budgetConfig or convergenceThresholdConfig
    if budgetConfig == null && convergenceConfig == null:
        return
    budgetEnforcer.check(...)
    convergenceDetector.evaluate(...)
```

**Note:** Config resolution (§1) MUST ensure `budgetConfig` and `convergenceThresholdConfig` are non-null for stigmergy cases — the StigmergyDefaults provide `maxDispatches`, `maxEvaluationCycles`, and `stabilityWindow` which populate these configs. Without this guarantee, the existing budget/convergence checks would be silently skipped.

`StigmergyCoordinator` is `@ApplicationScoped` in `runtime-core` with constructor-injected dependencies (`SignalRegistry`, `ObservationRegistry`, `ActivityTracker`). It is injected into `CaseContextChangedEventHandler` via `Instance<StigmergyCoordinator>` with `isResolvable()` guard — transparent no-op when no stigmergy case is active. Follows the `Instance<OutputConvergenceMonitor>` pattern (D51). `detectPatterns()` uses constructor-injected registries, not method parameters — consistent with the CDI convention used by all other infrastructure beans.

### Lifecycle

`CaseStatusChangedHandler` calls `coordinator.evictByCase(caseId)` on terminal case status. `StigmergyCoordinator implements Resettable` for demo/test replay.

### Per-Case Tracking State for Dedup

| Detector | State | Bounded by |
|----------|-------|-----------|
| Signal consensus | `Set<String>` of signals that have fired | `maxSignalsPerCase` (default 100) |
| Coordination storm | `boolean` indicating storm is active | O(1) |
| Interest convergence | `Set<String>` of hotspot keys that have fired | `maxObserversPerCase` (default 20) |

All evicted on case termination alongside agent states.

## 4. WorkerRuntime Integration

`WorkerRuntime` gains one method for voluntary departure:

```java
default void leave() {
    // no-op for non-stigmergy cases
}
```

`DefaultWorkerRuntime` implementation:

1. `observationRegistry.unregisterByAgent(caseId, agentId)` — removes all observers
2. `ruleRegistry.unregisterByAgent(caseId, agentId)` — removes all rules
3. `coordinator.agentDeparted(caseId, agentId)` — lifecycle transition to `DEPARTED`
4. Publishes `STIGMERGY_AGENT_DEPARTED` event

**Signals are NOT removed.** Signals deposited by a departing agent decay naturally. This is consistent with biological stigmergy — an ant that leaves doesn't erase its pheromone trail. The signal's `sources` set retains the departed agent's ID for audit purposes.

**Calling `leave()` outside stigmergy mode** is a no-op. The coordinator is not tracking the agent, so there is nothing to transition. The `default` method on `WorkerRuntime` returns immediately. This follows the established pattern where WorkerRuntime facet methods are no-ops when the backing infrastructure is absent.

### Call Site: Leave as a RuleAction

Since `execute()` returns before the agent enters ACTIVE state, and the `WorkerRuntime` reference is not retained after execution, agents trigger departure via a **`Leave` rule action** — the 5th permit on the `RuleAction` sealed hierarchy (extending D41):

```java
record Leave() implements RuleAction {}
```

When a rule fires a `Leave` action, the `LocalRuleEvaluator` calls `WorkerRuntime.leave()` on behalf of the agent. This gives agents a clean departure path: a rule evaluates "I've accomplished my purpose" and fires Leave. The engine handles deregistration.

This extends the RuleAction hierarchy to five permits: `DepositSignal`, `RegisterInterest`, `DeregisterInterest`, `WriteContext`, `Leave`.

### RuleRegistry.unregisterByAgent()

`RuleRegistry.unregisterByAgent(UUID caseId, String agentId)` already exists — `leave()` uses it directly. No new API needed.

### JOINING → ACTIVE Transition

The activation transition happens when the worker's `execute()` method returns successfully. The worker completion handler (`WorkflowExecutionCompletedHandler`) notifies the coordinator:

```
onWorkerCompleted(caseId, agentId, bindingName):
    if coordinator.isStigmergyCase(caseId):
        coordinator.agentActivated(caseId, agentId)
        // publishes STIGMERGY_AGENT_ACTIVATED with interestCount, ruleCount
```

## 5. Audit Events

Seven new `CaseHubEventType` values in two groups:

### Lifecycle Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `STIGMERGY_CASE_INITIALIZED` | Case starts with stigmergy mode | `agentCount`, config summary |
| `STIGMERGY_AGENT_JOINED` | Agent dispatched (→ JOINING) | `agentId`, `bindingName` |
| `STIGMERGY_AGENT_ACTIVATED` | Agent setup complete (→ ACTIVE) | `agentId`, `interestCount`, `ruleCount` |
| `STIGMERGY_AGENT_DEPARTED` | Agent voluntarily left (→ DEPARTED) | `agentId`, `reason`, `activeTimeMs` |

### Coordination Intelligence Events

| Event | When | Key Metadata |
|-------|------|-------------|
| `SIGNAL_CONSENSUS_DETECTED` | Signal reinforced by N agents | `signalName`, `reinforcementCount`, `sources`, `effectiveStrength` |
| `COORDINATION_STORM_DETECTED` | Activity rates exceed storm thresholds | `stormingMetrics`, `currentRates`, `stormThreshold` |
| `INTEREST_CONVERGENCE_DETECTED` | Collective attention focusing | `hotspotKeys`, `watchingAgentCount`, `hotspotScore` |

### The Coordination Story

Together, these events capture the emergent coordination narrative in the EventLog:

```
t=0:   STIGMERGY_CASE_INITIALIZED    (5 agents, defaults)
t=1:   STIGMERGY_AGENT_JOINED        (temperature-monitor)
t=1:   STIGMERGY_AGENT_JOINED        (pressure-monitor)
t=1:   STIGMERGY_AGENT_JOINED        (cooling-controller)
t=2:   STIGMERGY_AGENT_ACTIVATED     (temperature-monitor, 2 interests, 3 rules)
t=2:   STIGMERGY_AGENT_ACTIVATED     (pressure-monitor, 1 interest, 2 rules)
t=3:   STIGMERGY_AGENT_ACTIVATED     (cooling-controller, 2 interests, 4 rules)
...
t=10:  PHEROMONE_DEPOSITED           (overheating, strength=0.8, source=temp-monitor)
t=12:  PHEROMONE_DEPOSITED           (overheating, strength=0.9, source=pressure-monitor)
t=12:  SIGNAL_CONSENSUS_DETECTED     (overheating, 2 agents, strength=0.9)
t=15:  RULE_FIRED                    (emergency-cool, agent=cooling-controller)
t=20:  INTEREST_CONVERGENCE_DETECTED (hotspot: cooling-status, 3 agents)
...
t=90:  CONVERGENCE_DETECTED          (all rates below threshold for 30s)
```

This is readable without correlating hundreds of individual SPI events.

## 6. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `StigmergyConfig`, `StigmergyDefaults`, `CoordinationConfig` | api | `io.casehub.api.model.stigmergy` |
| `AgentLifecycleState` (enum), `AgentState` (record) | api | `io.casehub.api.model.stigmergy` |
| New `CaseHubEventType` values | api | `io.casehub.api.event` (existing enum) |
| `StigmergyCoordinator` | runtime-core | `io.casehub.engine.internal.stigmergy` |
| `StigmergyStrategy` | planning-core | `io.casehub.engine.plan.strategy` (existing) |

Follows the established tier model: API types in Tier 1, infrastructure in Tier 2/3, strategy in planning module.

## 7. Complete YAML Example

A chemical reactor control case where agents coordinate through environmental signals to achieve reactor stability:

```yaml
dsl: "1.0.0"
namespace: industrial
name: reactor-control
version: "1.0.0"

spec:
  planningStrategy: stigmergy

  stigmergyConfig:
    defaults:
      signalHalfLife: PT2M
      stabilityWindow: PT30S
      maxDispatches: 5000
    coordination:
      consensusThreshold: 2
      stormRateMultiplier: 10.0

  capabilities:
    - name: monitorTemperature
      outputProjection: "{ tempReading: .reading, tempStatus: .status }"
    - name: monitorPressure
      outputProjection: "{ pressureReading: .reading, pressureStatus: .status }"
    - name: controlCooling
      outputProjection: "{ coolingAction: .action, coolingResult: .result }"
    - name: controlVenting
      outputProjection: "{ ventAction: .action, ventResult: .result }"
    - name: assessStability
      outputProjection: "{ reactorStatus: .status, stabilityScore: .score }"

  workers:
    - name: temp-monitor
      capabilities: [monitorTemperature]
    - name: pressure-monitor
      capabilities: [monitorPressure]
    - name: cooling-controller
      capabilities: [controlCooling]
    - name: vent-controller
      capabilities: [controlVenting]
    - name: stability-assessor
      capabilities: [assessStability]

  bindings:
    - name: run-temp-monitor
      capability: monitorTemperature
    - name: run-pressure-monitor
      capability: monitorPressure
    - name: run-cooling-controller
      capability: controlCooling
    - name: run-vent-controller
      capability: controlVenting
    - name: run-stability-assessor
      capability: assessStability

  goals:
    - name: reactorStable
      condition: '.reactorStatus == "STABLE" and .stabilityScore > 0.9'
      kind: success

  completion:
    success:
      anyOf: [reactorStable, _converged]
```

No `on:` triggers on bindings — the `StigmergyStrategy` handles dispatch. The case initializer automatically adds `ScopeActivatedTrigger` to each binding. All agents join at case start, register their interests and rules programmatically in their `execute()` methods, and the pipeline drives the perceive→decide→act cycle. Convergence detection fires `_converged` when all activity rates drop below threshold for the stability window.

### Java Worker Example

Each agent is a Worker that registers interests and rules during setup:

```java
@Worker(capability = "monitorTemperature")
public class TemperatureMonitorWorker implements WorkerFunction {
    @Override
    public WorkerResult execute(WorkerScope scope) {
        var runtime = (WorkerRuntime) scope;

        // Register what we observe
        runtime.interests().register(
            InterestDeclaration.keyThreshold("tempReading", Operator.GT, 100));

        // Register local rules
        runtime.rules().register(new LocalRule("signal-overheating",
            new PredicateCondition(ctx ->
                ctx.observations().stream()
                    .anyMatch(o -> "threshold_crossed".equals(o.type()))),
            List.of(new DepositSignal("overheating", 0.8)),
            10));  // priority

        runtime.rules().register(new LocalRule("write-temp-status",
            new PredicateCondition(ctx ->
                ctx.signals().containsKey("overheating")
                    && ctx.signals().get("overheating").effectiveStrength() > 0.5),
            List.of(new WriteContext("tempAlert", JsonNodeFactory.instance.textNode("HIGH"))),
            5));

        return WorkerResult.of(Map.of("status", "monitoring"));
    }
}
```

After `execute()` returns, the agent transitions to `ACTIVE`. Its registered observers evaluate each cycle, rules fire based on observations and signals, and actions modify the shared environment — all driven by the pipeline without re-invoking the worker.

## 8. Cross-Cutting Concerns

### 8a. Evaluation Backpressure (D71)

**Explicit v1 trade-off:** No backpressure mechanism beyond the existing `CaseEvaluationSerializer` per-case serialization and `BudgetConfig` cumulative caps. The serializer guarantees at most one evaluation per case at a time. Cross-case evaluation runs concurrently on virtual threads. Budget enforcement detects and terminates runaway cases after the fact.

A case can queue many evaluation cycles before budget enforcement stops it. Each queued evaluation runs to completion — potentially wasted work when the result would be identical. Acceptable for v1 because budget enforcement catches pathological cases, and the per-case serializer prevents fan-out.

Event coalescing in the serializer's pending-event queue is the natural next step when production workloads surface the need.

### 8d. Crash Recovery (D29)

**Explicit v1 limitation:** Stigmergy coordination state is in-memory only (per D29). On engine restart, all coordinator state is lost — agent lifecycle tracking, consensus dedup sets, storm detection state. A stigmergy case that survives a restart (case instance persisted, CaseContext persisted) loses all coordination state. Agents would need to be re-dispatched and re-register their observers and rules. This is consistent with all other coordination state (ObservationRegistry, SignalRegistry, RuleRegistry, ActivityTracker) which is also in-memory only. Stigmergy cases cannot survive engine restarts in v1.

### 8b. Virtual Thread Compatibility (D72)

Per-case registries (`ObservationRegistry`, `RuleRegistry`) currently use `synchronized` blocks for internal synchronization. This pins platform threads when observers or rules run on virtual threads (D6 specifies parallel observer evaluation on virtual threads).

All registry-internal synchronization should use `java.util.concurrent.locks.ReentrantLock` instead of `synchronized`. This aligns with D6's own guidance that implementations "should avoid `synchronized` blocks (which pin platform threads — a known virtual thread anti-pattern)." The registry's internal synchronization should follow the same rule it imposes on observer implementations.

Critical sections remain short (list add/remove). Lock scope is per-case (each case has its own registration list), so contention is limited to concurrent registrations for the same case.

### 8c. Agent-Visible Metrics (D56)

**Explicit v1 trade-off:** Agents cannot read convergence metrics (activity rates, budget usage). Convergence detection is a system-level supervisory function. Agents coordinate via signals, observations, interests, neighbors, and rules — the agent-facing coordination primitives. The engine monitors aggregate behavior and intervenes when thresholds are breached.

When swarm scenarios (#1112/#1113) demonstrate a concrete need for agent-level metric visibility, a read-only `MetricsSpace` facet with `activityRates() → Map<String, Double>` is the planned extension path — no tracker changes required.

## 9. Relationship to Existing Infrastructure

| Existing | Relationship |
|----------|-------------|
| **ChoreographyLoopControl** | Replaced by `PlanningStrategyLoopControl` when planning module is present. `StigmergyStrategy` is resolved per-compound alongside existing strategies. |
| **ConvergenceDetector / BudgetEnforcer** | Complementary. System-level convergence and budget enforcement run first; coordination pattern detection runs after for stigmergy cases. |
| **Watchdog** (qhorus) | Complementary. Watchdog monitors conversations. StigmergyCoordinator monitors coordination state. No dependency. |
| **GoalBasedCompletion** | Reused. `_converged` goal integrates with existing completion system. |
| **ObservationRegistry / SignalRegistry / RuleRegistry** | Consumed. Coordinator queries these registries for pattern detection — no changes to registry APIs (except `RuleRegistry.unregisterByAgent()`). |
| **WorkerRuntimeFactory** | Extended. Wires coordinator reference into `DefaultWorkerRuntime` for `leave()` support and JOINING→ACTIVE notification. |
| **ScopeActivatedTrigger** | Reused. Case initializer adds this trigger to trigger-less bindings in stigmergy compounds. |
| **engine#604** (original stigmergy issue) | Superseded by #1111. #604 sketched the platform mapping; #1111 delivers the full model using the Hive Mind foundation SPIs. |

## Decisions

This spec implements the following design decisions:

| Decision | Title |
|----------|-------|
| D60 | Stigmergy execution model architecture — three-layer blend |
| D61 | Module placement — StigmergyStrategy requires planning module |
| D62 | StigmergyConfig as coordinated defaults preset |
| D63 | Agent model — all bindings are agents in stigmergy compound |
| D64 | Agent lifecycle — three states with voluntary departure |
| D65 | Strategy dispatch — first-cycle dispatch with health monitoring |
| D66 | StigmergyConfig structure — defaults + coordination thresholds |
| D67 | Coordination pattern detection — three detectors in convergenceDetection phase |
| D68 | Seven new CaseHubEventTypes for stigmergy |
| D69 | Trigger-less bindings in stigmergy compounds |
| D70 | Module placement — package structure |
| D71 | Evaluation backpressure — per-case event coalescing (v1 trade-off) |
| D72 | Registry synchronization — ReentrantLock for virtual thread compatibility |

Cross-references to foundation SPI decisions: D1-D9 (observation), D10-D18 (signals), D19-D28 (interests/facets), D29-D31 (coordination state), D32-D36 (neighbors), D37-D44 (local rules), D45-D59 (convergence).

## References

- `CaseContextChangedEventHandler.java:228-279` — evaluation pipeline (`evaluateAndDispatch`)
- `CaseContextChangedEventHandler.java:1384-1425` — convergence detection phase
- `WorkerRuntime.java` — api coordination surface
- `DefaultWorkerRuntime.java` — runtime-core WorkerRuntime implementation
- `WorkerRuntimeFactory.java` — factory wiring registries into runtime
- `PlanningStrategyLoopControl.java` — per-case strategy resolution
- `DefaultPlanningStrategy.java` — existing strategy pattern (planning-core)
- `ScopeActivatedTrigger` — scope-based trigger type for strategy-managed dispatch
- `SignalRegistry.java` — signal storage, `deposit()`, `perceiveAll()`
- `ObservationRegistry.java` — observer storage, `registerObserver()`, `unregisterByAgent()`
- `RuleRegistry.java` — rule storage (needs `unregisterByAgent()`)
- `ActivityTracker.java` — sliding-window rate computation
- `ConvergenceDetector.java` — activity quiescence detection
- `BudgetEnforcer.java` — cumulative budget enforcement
- `OutputConvergenceMonitor.java` — `Instance<>` injection pattern
- `CaseStatusChangedHandler.java` — terminal state cleanup
- `WorkflowExecutionCompletedHandler.java` — worker completion handling
- `CaseHubEventType.java` — existing event type enum
- `Resettable` interface — demo/test replay
- Unified execution model spec §2.1-§2.7, §3.1 — dispatch archetypes, composable strategies
- engine#604 — original stigmergy issue
- engine#1104 — Hive Mind epic
- engine#1111 — this issue
- arXiv:2608.26081 (SwarmWorld) — cognition/consequence split validating engine/blocks boundary
- D60-D72 — design decisions
