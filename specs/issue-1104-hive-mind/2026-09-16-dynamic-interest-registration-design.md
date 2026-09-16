# Dynamic Interest Registration — Design Spec

**Issue:** casehubio/engine#1107
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-16

## Summary

Agents register what they want to observe at runtime via a declarative interest API, rather than constructing `EnvironmentObserver` implementations directly. Includes a full WorkerRuntime restructure into domain-organized coordination facets (`SignalSpace`, `InterestSpace`), an anonymous aggregate `InterestLandscape` for collective attention awareness, and an `InterestDeclaration` sealed hierarchy for type-safe, auditable interest registration.

This is the dynamic face of the observation SPI (#1105). Static binding triggers are untouched. Agents decide what to observe based on their own state and goals — consistent with the self-organization principle from #1104.

## Design Principles

1. **Declarative over programmatic** — agents describe what they care about; the engine creates the right observer
2. **Anonymous aggregate visibility** — agents see what the collective attends to, without identity or implementation coupling
3. **Pre-release clean break** — no deprecated methods, no migration bridges; facets are the only coordination path
4. **Auditable by default** — typed interest permits are fully inspectable; JQ catchall is gated for compliance

## Architecture

### WorkerRuntime Faceting (D19)

WorkerRuntime is restructured into domain-organized coordination facets. Flat coordination methods are removed entirely (pre-release). WorkerRuntime retains only execution methods and facet accessors.

```java
public interface WorkerRuntime extends WorkerScope {
    // Tier 1 execution (unchanged)
    WorkerContext context();
    <R> WorkerResult<R> execute(WorkerFunction<?, R> fn, Object input);
    WorkerResult<Map<String, Object>> execute(String workerName, Map<String, Object> input);
    UUID spawnCase(String caseType, Map<String, Object> input);
    CaseContext awaitCase(UUID caseId, Duration timeout);
    CaseContext spawnAndAwaitCase(String caseType, Map<String, Object> input, Duration timeout);

    // Coordination facets
    default SignalSpace signals() { return SignalSpace.NOOP; }
    default InterestSpace interests() { return InterestSpace.NOOP; }
}
```

**Removed from WorkerRuntime:** `depositSignal()`, `perceiveSignals()`, `registerObserver()`. All callers migrate to facets.

### SignalSpace (D19)

Moved from flat WorkerRuntime methods. Pure domain facet for the signal/pheromone model (#1106).

```java
package io.casehub.api.engine;

public interface SignalSpace {
    void deposit(String name, double strength);
    void deposit(String name, double strength, Duration halfLife);
    Map<String, PerceivedSignal> perceive();

    SignalSpace NOOP = new SignalSpace() {
        @Override public void deposit(String name, double strength) {}
        @Override public void deposit(String name, double strength, Duration halfLife) {}
        @Override public Map<String, PerceivedSignal> perceive() { return Map.of(); }
    };
}
```

### InterestSpace (D19, D20, D21, D22)

New coordination facet for dynamic interest registration, deregistration, self-query, landscape awareness, and low-level observer registration.

```java
package io.casehub.api.engine;

public interface InterestSpace {
    InterestRegistration register(InterestDeclaration interest);
    boolean registerObserver(EnvironmentObserver observer);
    void deregister(String interestId);
    List<InterestRegistration> mine();
    InterestLandscape landscape();

    InterestSpace NOOP = new InterestSpace() {
        @Override public InterestRegistration register(InterestDeclaration interest) {
            return null;
        }
        @Override public boolean registerObserver(EnvironmentObserver observer) {
            return false;
        }
        @Override public void deregister(String interestId) {}
        @Override public List<InterestRegistration> mine() { return List.of(); }
        @Override public InterestLandscape landscape() { return InterestLandscape.EMPTY; }
    };
}
```

## SPI Types

### InterestDeclaration (D20)

Sealed hierarchy with five permits. Each of the first four maps 1:1 to a classical observer implementation. `JqInterest` is a general-purpose catchall gated by `ObservationConfig.allowJqInterests` (default `true`).

```java
package io.casehub.api.spi.observation;

public sealed interface InterestDeclaration
    permits InterestDeclaration.KeyThreshold,
            InterestDeclaration.KeyCorrelation,
            InterestDeclaration.TemporalSequence,
            InterestDeclaration.SignalThreshold,
            InterestDeclaration.JqInterest {

    record KeyThreshold(
        String key,
        ThresholdObserver.Operator operator,
        double threshold
    ) implements InterestDeclaration {}

    record KeyCorrelation(
        Set<String> keys,
        String jqCondition
    ) implements InterestDeclaration {}

    record TemporalSequence(
        List<TemporalSequenceObserver.SequenceStep> steps,
        Duration window
    ) implements InterestDeclaration {}

    record SignalThreshold(
        String signalName,
        ThresholdObserver.Operator operator,
        double threshold
    ) implements InterestDeclaration {}

    record JqInterest(
        String expression,
        Set<String> watchedKeys
    ) implements InterestDeclaration {}
}
```

**Observer mapping:** `register()` creates the corresponding observer internally:

| InterestDeclaration | Created Observer |
|---------------------|-----------------|
| `KeyThreshold` | `ThresholdObserver.of(key, operator, threshold)` |
| `KeyCorrelation` | `CorrelationObserver.of(keys, jqCondition)` |
| `TemporalSequence` | `TemporalSequenceObserver.of(steps, window)` |
| `SignalThreshold` | `SignalStrengthObserver.of(signalName, operator, threshold)` |
| `JqInterest` | `CorrelationObserver.of(watchedKeys, expression)` |

**JQ gate:** `JqInterest` registration checks `ObservationConfig.allowJqInterests()`. When `false`, throws `IllegalArgumentException`. The four typed permits are always allowed — they are fully auditable (keys, operators, thresholds are inspectable). Compliance teams in regulated domains set `allowJqInterests: false` per case definition.

### InterestRegistration (D21)

Immutable value record returned by `register()`. The `interestId` is engine-generated, following the `observerType-N` pattern from `ObservationRegistry`.

```java
package io.casehub.api.spi.observation;

public record InterestRegistration(
    String interestId,
    InterestDeclaration declaration,
    Instant registeredAt
) {}
```

### InterestLandscape (D24)

Anonymous aggregate view of collective observation attention. No agent identity, no parameters, no observer instances.

```java
package io.casehub.api.spi.observation;

public record InterestLandscape(
    Map<String, Integer> keyObserverCounts,
    Map<String, Integer> signalObserverCounts,
    Map<String, Integer> interestTypeCounts,
    int totalObserverCount
) {
    public static final InterestLandscape EMPTY =
        new InterestLandscape(Map.of(), Map.of(), Map.of(), 0);
}
```

- `keyObserverCounts` — context key → number of observers watching it (e.g. `"riskScore" → 3`)
- `signalObserverCounts` — signal name → number of observers monitoring it (e.g. `"danger" → 2`)
- `interestTypeCounts` — interest type → count (e.g. `"threshold" → 4, "correlation" → 2`)
- `totalObserverCount` — total registered observers for this case

Computed on-demand from `ObservationRegistry` data. O(N) where N ≤ `maxObserversPerCase` (20). For `ObservationContext`, computed once per evaluation cycle by the handler and passed as a pre-computed snapshot.

## Pipeline Integration

### ObservationContext (D25)

`ObservationContext` gains `InterestLandscape interestLandscape` as the 8th field.

```java
public record ObservationContext(
    JsonNode snapshot,
    Set<String> changedKeys,
    List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    UUID caseId,
    Map<String, PerceivedSignal> signals,
    InterestLandscape interestLandscape
) {
    // Backward-compatible constructors
    public ObservationContext(JsonNode snapshot, Set<String> changedKeys,
            List<ContextSnapshot> history, String agentId, String tenancyId, UUID caseId) {
        this(snapshot, changedKeys, history, agentId, tenancyId, caseId,
             Map.of(), InterestLandscape.EMPTY);
    }

    public ObservationContext(JsonNode snapshot, Set<String> changedKeys,
            List<ContextSnapshot> history, String agentId, String tenancyId,
            UUID caseId, Map<String, PerceivedSignal> signals) {
        this(snapshot, changedKeys, history, agentId, tenancyId, caseId,
             signals, InterestLandscape.EMPTY);
    }
}
```

### Handler Integration

`CaseContextChangedEventHandler.observations()` computes the `InterestLandscape` once per evaluation cycle (before iterating observers) and passes it into every `ObservationContext`.

### ObservationConfig Extension

`ObservationConfig` gains `allowJqInterests` (boolean, default `true`):

```java
public record ObservationConfig(
    int maxObserversPerCase,
    int maxHistoryEntries,
    Duration maxHistoryAge,
    boolean allowJqInterests
) {
    // Existing defaults + new default
    public ObservationConfig() {
        this(20, 50, Duration.ofMinutes(5), true);
    }

    // Backward-compatible 3-arg constructor
    public ObservationConfig(int maxObserversPerCase, int maxHistoryEntries, Duration maxHistoryAge) {
        this(maxObserversPerCase, maxHistoryEntries, maxHistoryAge, true);
    }
}
```

YAML: `allowJqInterests: false` under `observationConfig:` block.

## Scope Enforcement (D26)

Same as existing `registerObserver()`: BINDING scope rejected with `IllegalStateException` at registration time. Only COMPOUND or CASE scope allowed. Check is in `DefaultInterestSpace.register()`.

Deregistration via `deregister(interestId)` removes the observer from `ObservationRegistry`. `mine()` returns only this agent's interests for the current case.

## Implementation

### DefaultSignalSpace

Extracted from `DefaultWorkerRuntime`. Holds `SignalRegistry` reference and `SignalConfig`. Three methods delegate directly.

```
runtime-core/src/main/java/io/casehub/engine/internal/signal/DefaultSignalSpace.java
```

### DefaultInterestSpace

New implementation. Holds `ObservationRegistry`, `CaseInstanceCache`, `CaseDefinitionRegistry`, and per-invocation state (`caseId`, `workerName`, `bindingName`).

```
runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultInterestSpace.java
```

Key behaviors:
- `register(InterestDeclaration)` — validates scope (D26), checks JQ gate (D20), creates the corresponding `EnvironmentObserver`, delegates to `ObservationRegistry.registerObserver()`, returns `InterestRegistration` with generated ID
- `registerObserver(EnvironmentObserver)` — direct delegation to `ObservationRegistry`, same logic as the former `DefaultWorkerRuntime.registerObserver()`
- `deregister(interestId)` — removes observer by ID from `ObservationRegistry`
- `mine()` — queries `ObservationRegistry` for this agent's registrations, returns as `List<InterestRegistration>`
- `landscape()` — aggregates from `ObservationRegistry.getObservers()` into `InterestLandscape`

### DefaultWorkerRuntime Changes

`DefaultWorkerRuntime` creates `DefaultSignalSpace` and `DefaultInterestSpace` in its constructor (or factory). Returns them from `signals()` and `interests()`. All signal and observer methods removed from the class.

### ObservationRegistry Extensions

`ObservationRegistry` gains:
- `deregisterByInstanceId(UUID caseId, String instanceId)` — for interest deregistration by ID
- `getRegistrationsForAgent(UUID caseId, String agentId)` — returns `List<ObserverRegistration>` for `mine()` query
- `computeLandscape(UUID caseId)` — aggregates observer data into `InterestLandscape`

The `ObserverRegistration` record gains a nullable `InterestDeclaration declaration` field — populated when registered via `InterestSpace.register()`, null when registered via `registerObserver()`.

## Audit (D27)

Two new `CaseHubEventType` values:
- `INTEREST_REGISTERED` — metadata: `interestId`, `interestType`, `agentId`, declaration details
- `INTEREST_DEREGISTERED` — metadata: `interestId`, `agentId`

EventLog publishing deferred to the same wiring pass as `PHEROMONE_DEPOSITED`/`PHEROMONE_EXPIRED` from #1106.

## Module Placement (D23)

| Type | Module | Package |
|------|--------|---------|
| `SignalSpace` | engine-api | `io.casehub.api.engine` |
| `InterestSpace` | engine-api | `io.casehub.api.engine` |
| `InterestDeclaration` | engine-api | `io.casehub.api.spi.observation` |
| `InterestRegistration` | engine-api | `io.casehub.api.spi.observation` |
| `InterestLandscape` | engine-api | `io.casehub.api.spi.observation` |
| `DefaultSignalSpace` | runtime-core | `io.casehub.engine.internal.signal` |
| `DefaultInterestSpace` | runtime-core | `io.casehub.engine.internal.observation` |

## Migration Impact (Pre-release)

All existing call sites for removed flat methods must be updated:

| Flat method | Facet replacement |
|-------------|-------------------|
| `runtime.depositSignal(name, strength)` | `runtime.signals().deposit(name, strength)` |
| `runtime.depositSignal(name, strength, halfLife)` | `runtime.signals().deposit(name, strength, halfLife)` |
| `runtime.perceiveSignals()` | `runtime.signals().perceive()` |
| `runtime.registerObserver(observer)` | `runtime.interests().registerObserver(observer)` |

Call sites in: `DefaultWorkerRuntime` (self), `CaseContextChangedEventHandler` (signal perception in `observations()`), `WorkerRuntimeFactory`, tests.

## YAML Support (D28)

Not in scope. Interests are runtime-registered by agents. Static triggers remain via `ContextChangeTrigger` on bindings.

## Testing

- Unit tests for `InterestDeclaration` sealed hierarchy (each permit, construction, validation)
- Unit tests for `DefaultInterestSpace` (register, deregister, mine, landscape, scope enforcement, JQ gate)
- Unit tests for `DefaultSignalSpace` (deposit, perceive — regression from extracted code)
- Unit tests for `InterestLandscape` computation from registry data
- Unit tests for `ObservationContext` 8-arg constructor and backward compat
- Integration test: agent registers interest → context changes → observation fires → landscape reflects
- Integration test: JQ gate enforcement (allowJqInterests=false rejects JqInterest registration)

## Dependencies

- #1105 (Environment Observation SPI) — foundation: `EnvironmentObserver`, `ObservationRegistry`, `ObservationContext`
- #1106 (Signal/Pheromone model) — foundation: `SignalRegistry`, `PerceivedSignal`, `depositSignal`/`perceiveSignals` being moved

## Not in Scope

- EventLog publishing for interest events (deferred with #1106 pheromone events)
- YAML interest declaration syntax
- Interest persistence across JVM restart (in-memory only, consistent with `ObservationRegistry`)
- Cross-case interest visibility (intra-case only)

## References

- `WorkerRuntime.java` (api/engine) — current flat coordination surface
- `DefaultWorkerRuntime.java:286-343` (runtime-core) — existing registerObserver, depositSignal, perceiveSignals
- `ObservationRegistry.java` (common-core) — per-case observer storage
- `ThresholdObserver.java`, `CorrelationObserver.java`, `TemporalSequenceObserver.java`, `SignalStrengthObserver.java` (runtime-core) — classical observers mapped to interest types
- `ObservationConfig.java` (engine-api) — observation policy surface
- `CaseContextChangedEventHandler.java:1171-1267` (runtime-core) — observation evaluation pipeline
- `SignalRegistry.java` (common-core) — signal storage, perceive pattern
- D19-D28 in `decisions.md` — all design decisions for this issue
- arXiv:2603.28990 — "Drop the Hierarchy and Roles" (mission + protocol, not assigned roles)
- engine#1108-#1115 — future facet growth trajectory
