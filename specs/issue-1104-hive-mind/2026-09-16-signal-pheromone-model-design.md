# Signal/Pheromone Model — Design Spec

**Issue:** casehubio/engine#1106
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-16

## Summary

Temporal signal model for stigmergic coordination. Agents deposit named signals with strength values that decay exponentially over time and are reinforced when multiple agents confirm the same signal. Signals are the core coordination primitive for indirect agent communication — agents perceive the shared environment (via signals) rather than communicating directly.

Signals are stored in a dedicated `SignalRegistry` (not CaseContext), avoiding the feedback loops identified in #1105 D7. Decay is computed lazily at read time — no physical mutation of stored values. Configuration is per-case via `SignalConfig` on `CaseDefinition`.

## Design Principles

1. **Lazy decay** — strength is never physically decremented; `effectiveStrength` is computed at read time via exponential decay
2. **No feedback loops** — signals live in `SignalRegistry`, not CaseContext; writes don't trigger `CONTEXT_CHANGED`
3. **Bounded** — `maxSignalsPerCase` caps registry size; effective-zero threshold filters dead signals from perception
4. **Audit trail** — deposit and expiry events in EventLog; signals are never physically deleted within a case's lifetime
5. **Consistent patterns** — follows `ObservationRegistry`, `ContextHistoryBuffer`, `DispositionSignalStore` conventions

## Architecture

### Signal Flow

```
Agent A                    Agent B
  │                          │
  ▼                          ▼
depositSignal("path-x", 0.8) depositSignal("path-x", 0.9)
  │                          │
  ▼                          ▼
SignalRegistry (engine-common)
  ├── Signal("path-x", strength=0.9, reinforcementCount=2,
  │         lastSource="agent-b", lastReinforced=now, halfLife=5min)
  │
  ▼
observations() pipeline (CaseContextChangedEventHandler)
  ├── reads from SignalRegistry
  ├── computes effectiveStrength = strength * e^(-λ * elapsed)
  ├── filters below effectiveZeroThreshold
  ├── fires PHEROMONE_EXPIRED for newly-expired signals
  └── passes Map<String, PerceivedSignal> into ObservationContext
        │
        ▼
  Observers perceive signals via ctx.signals()
```

### Evaluation Pipeline (updated from #1105)

```
CONTEXT_CHANGED event
    │
    ▼
CaseEvaluationSerializer (per-case gate)
    │
    ├── 1. rules()        — existing binding dispatch (untouched)
    ├── 2. goals()        — existing goal evaluation (untouched)
    └── 3. observations() — evaluate registered observers
                │
                ├── read signals from SignalRegistry
                ├── compute effective strengths, filter expired
                ├── fire PHEROMONE_EXPIRED events (lazy)
                ├── build ObservationContext with signals()
                └── evaluate observers (unchanged from #1105)
```

### Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `Signal` | engine-api | `io.casehub.api.model.signal` |
| `PerceivedSignal` | engine-api | `io.casehub.api.model.signal` |
| `SignalConfig` | engine-api | `io.casehub.api.model.signal` |
| `SignalRegistry` | engine-common | `io.casehub.engine.common.internal.signal` |
| `WorkerRuntime.depositSignal()` | engine-api | existing `io.casehub.api.engine` |
| `WorkerRuntime.perceiveSignals()` | engine-api | existing `io.casehub.api.engine` |
| Signal event publishing | runtime | `io.casehub.engine.internal.signal` |
| `SignalStrengthObserver` | runtime | `io.casehub.engine.internal.observation` |

## Types

### Signal (stored value)

```java
package io.casehub.api.model.signal;

public record Signal(
    String name,
    double strength,
    Instant firstDeposited,
    Instant lastReinforced,
    Duration halfLife,
    String lastSource,
    int reinforcementCount,
    boolean expired) {

  public Signal {
    if (strength < 0.0 || strength > 1.0)
      throw new IllegalArgumentException("strength must be in [0.0, 1.0]");
    if (halfLife.isNegative() || halfLife.isZero())
      throw new IllegalArgumentException("halfLife must be positive");
  }
}
```

- `strength` — the deposited/reinforced strength at `lastReinforced` time (not current perceived strength)
- `firstDeposited` — timestamp of initial signal creation; used for `lifetimeMs` in `PHEROMONE_EXPIRED` audit
- `lastReinforced` — timestamp of last deposit/reinforcement; decay is computed from this
- `halfLife` — per-signal decay rate; defaults to `SignalConfig.defaultHalfLife` when not specified at deposit time
- `lastSource` — agent ID of the most recent depositor (audit only)
- `reinforcementCount` — total number of deposits to this signal name (starts at 1)
- `expired` — set to `true` when effective strength drops below `effectiveZeroThreshold`; never reset

### PerceivedSignal (read model)

```java
package io.casehub.api.model.signal;

public record PerceivedSignal(
    String name,
    double effectiveStrength,
    int reinforcementCount,
    String lastSource,
    Duration age) {

  public PerceivedSignal {
    if (effectiveStrength < 0.0 || effectiveStrength > 1.0)
      throw new IllegalArgumentException(
          "effectiveStrength must be in [0.0, 1.0], got: " + effectiveStrength);
  }
}
```

- `effectiveStrength` — `strength * e^(-λ * elapsed)` where `λ = ln(2) / halfLife.toMillis()` and `elapsed = now - lastReinforced`
- `age` — `Duration.between(lastReinforced, now)`
- Only signals with `effectiveStrength >= effectiveZeroThreshold` are included in perception results

### SignalConfig

```java
package io.casehub.api.model.signal;

public record SignalConfig(
    Duration defaultHalfLife,
    double effectiveZeroThreshold,
    int maxSignalsPerCase) {

  public static final Duration DEFAULT_HALF_LIFE = Duration.ofMinutes(5);
  public static final double DEFAULT_EFFECTIVE_ZERO_THRESHOLD = 0.01;
  public static final int DEFAULT_MAX_SIGNALS_PER_CASE = 100;

  public static SignalConfig defaults() {
    return new SignalConfig(DEFAULT_HALF_LIFE,
        DEFAULT_EFFECTIVE_ZERO_THRESHOLD, DEFAULT_MAX_SIGNALS_PER_CASE);
  }
}
```

YAML:
```yaml
spec:
  signalConfig:
    defaultHalfLife: PT5M
    effectiveZeroThreshold: 0.01
    maxSignalsPerCase: 100
```

### SignalRegistry

```java
package io.casehub.engine.common.internal.signal;

@ApplicationScoped
public class SignalRegistry implements Resettable {

  // Per-case signal storage
  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, Signal>> signals;

  // Deposit or reinforce a signal
  public boolean deposit(UUID caseId, String name, double strength,
      Duration halfLife, String source, int maxPerCase);

  // Read all signals with effective strength above threshold
  public Map<String, PerceivedSignal> perceive(UUID caseId,
      double effectiveZeroThreshold);

  // Mark a signal as expired (after PHEROMONE_EXPIRED event is fired)
  public void markExpired(UUID caseId, String name);

  // Find signals that have crossed below threshold since last check
  public List<Signal> findNewlyExpired(UUID caseId,
      double effectiveZeroThreshold);

  // Case termination cleanup
  public void evictByCase(UUID caseId);

  // Signal count for a case
  public int signalCount(UUID caseId);

  @Override
  public void reset();
}
```

`deposit()` returns `false` and logs WARN when `maxPerCase` is reached (same pattern as `ObservationRegistry.registerObserver()`).

Reinforcement logic in `deposit()`:
1. If signal name doesn't exist: create new `Signal(name, strength, now, now, halfLife, source, 1, false)` (both `firstDeposited` and `lastReinforced` = now)
2. If signal name exists and not expired: compute current `effectiveStrength`; set new strength to `max(effectiveStrength, newStrength)`, reset `lastReinforced` to now, preserve `firstDeposited`, increment `reinforcementCount`, update `lastSource`
3. If signal name exists and expired: treat as new deposit (replace entirely, `reinforcementCount = 1`, new `firstDeposited`)

### Decay Computation

Static utility method (no CDI, pure math):

```java
package io.casehub.api.model.signal;

public final class SignalDecay {
  public static double effectiveStrength(double strength,
      Instant lastReinforced, Duration halfLife, Instant now) {
    long elapsedMs = Duration.between(lastReinforced, now).toMillis();
    if (elapsedMs <= 0) return strength;
    double lambda = Math.log(2) / halfLife.toMillis();
    return strength * Math.exp(-lambda * elapsedMs);
  }
}
```

## Worker API

`WorkerRuntime` gains two default methods:

```java
default void depositSignal(String name, double strength) {}
default void depositSignal(String name, double strength, Duration halfLife) {}
default Map<String, PerceivedSignal> perceiveSignals() { return Map.of(); }
```

- `depositSignal(name, strength)` — uses `SignalConfig.defaultHalfLife` from the case definition
- `depositSignal(name, strength, halfLife)` — per-signal half-life override
- `perceiveSignals()` — returns all signals above effective-zero threshold for the current case

`DefaultWorkerRuntime` implementation delegates to `SignalRegistry`:
- `depositSignal` calls `registry.deposit(caseId, name, strength, halfLife, workerName, maxPerCase)` and publishes `PHEROMONE_DEPOSITED` EventLog
- `perceiveSignals` calls `registry.perceive(caseId, threshold)`

## Observation Integration

`ObservationContext` gains a 7th field:

```java
public record ObservationContext(
    JsonNode snapshot,
    Set<String> changedKeys,
    List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    UUID caseId,
    Map<String, PerceivedSignal> signals) {}  // NEW
```

Backward-compatible 6-arg constructor passes `Map.of()`.

In `CaseContextChangedEventHandler.observations()`:
1. Read `SignalConfig` from `CaseDefinition` (or defaults)
2. Call `signalRegistry.perceive(caseId, threshold)` to get active signals
3. Call `signalRegistry.findNewlyExpired(caseId, threshold)` — for each newly expired signal, publish `PHEROMONE_EXPIRED` EventLog and call `markExpired()`
4. Steps 1-3 run BEFORE the `observerCount == 0` early-return guard — signal expiry detection is independent of observer registration
5. Pass signals map into `ObservationContext` constructor (after the guard)

### SignalStrengthObserver (classical observer)

```java
package io.casehub.engine.internal.observation;

public final class SignalStrengthObserver implements EnvironmentObserver {
  private final String signalName;
  private final ThresholdObserver.Operator operator;
  private final double threshold;

  public static SignalStrengthObserver of(
      String signalName, ThresholdObserver.Operator operator, double threshold);

  @Override public String observerType() { return "signal-strength"; }
  @Override public Set<String> watchedKeys() { return Set.of(); }
  @Override public List<Observation> observe(ObservationContext ctx) {
    PerceivedSignal signal = ctx.signals().get(signalName);
    // compare effectiveStrength against threshold using operator
  }
}
```

`watchedKeys()` returns empty — signal observers fire on every evaluation cycle (they observe registry state, not CaseContext keys). This is acceptable because the observer itself is O(1) — a single map lookup and comparison.

## CaseDefinition Integration

`CaseDefinition` gains:
- `signalConfig` (nullable `SignalConfig`) — `getSignalConfig()` returns `SignalConfig.defaults()` when null
- Builder: `.signalConfig(SignalConfig)`
- YAML: `signalConfig:` block under `spec:`

`CaseDefinitionYamlMapper` parses `defaultHalfLife` as ISO-8601 Duration, `effectiveZeroThreshold` as double, `maxSignalsPerCase` as int.

## EventLog Integration

Two new `CaseHubEventType` values:

### PHEROMONE_DEPOSITED
Published by `DefaultWorkerRuntime.depositSignal()`.

Metadata:
```json
{
  "signalName": "promising-path",
  "strength": 0.8,
  "reinforcementCount": 2,
  "source": "analyst-agent",
  "halfLifeMs": 300000,
  "reinforced": true
}
```

`reinforced: true` when the signal already existed; `false` for new signals.

### PHEROMONE_EXPIRED
Published lazily by the `observations()` pipeline when a signal crosses below `effectiveZeroThreshold`.

Metadata:
```json
{
  "signalName": "promising-path",
  "finalStrength": 0.008,
  "totalReinforcementCount": 5,
  "lifetimeMs": 1200000
}
```

`lifetimeMs` is the duration from first deposit to expiry detection.

## Lifecycle

- **Case start:** No signal initialization — signals are deposited dynamically by agents
- **Case running:** Agents deposit/reinforce via `WorkerRuntime`; observation pipeline computes decay, filters, and detects expiry
- **Case terminal:** `CaseStatusChangedHandler` calls `signalRegistry.evictByCase(caseId)` — same eviction point as `ObservationRegistry.unregisterByCase()` and `ContextHistoryBuffer.evict()`
- **Reset:** `EngineResetService` discovers `SignalRegistry` via `Instance<Resettable>` and calls `reset()`

## Dependencies

- **#1105 (Environment Observation SPI)** — complete; `ObservationContext` is extended with `signals()`
- **#1109 (Local rule evaluation)** — future consumer; local rules will query `ObservationRegistry` observations produced by signal-aware observers
- **#1111 (Stigmergy execution model)** — future consumer; stigmergy coordination patterns build on the signal model

## References

- `ObservationRegistry.java:27` — per-case in-memory registry pattern
- `ContextHistoryBuffer.java:28` — Resettable, eviction pattern
- `WritableLayerImpl.java:672` — `engineSet()` CaseContext write suppression (considered and rejected for signals, D10)
- `CaseContextChangedEventHandler.java:1168` — `observations()` pipeline integration point
- `ThresholdObserver.java:30` — classical observer pattern for `SignalStrengthObserver`
- `CbrConfig.temporalDecayHalfLifeDays` — temporal decay precedent in the platform
- `DispositionSignalStore` (eidos) — per-agent signal store with decay (different scope — personality is per-agent, signals are per-case)
- `CaseStatusChangedHandler` — terminal state eviction pattern
- `Resettable` interface — demo/test reset SPI
- engine#1106 issue spec — ACO decay formula, four trail surfaces
- arXiv:2512.10166 — Emergent Collective Memory (memory-first principle)
- D7, D10-D18 — design decisions
