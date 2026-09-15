# Environment Observation SPI — Design Spec

**Issue:** casehubio/engine#1105
**Epic:** casehubio/engine#1104 (Hive Mind)
**Date:** 2026-09-16

## Summary

SPI enabling agents to observe full CaseContext state and detect patterns beyond what static `ContextChangeTrigger` JQ expressions can express. Foundation for stigmergic coordination — agents perceive the shared environment and feed observations into local rule evaluation (#1109).

This is a **separate perception layer**, orthogonal to the existing binding/trigger dispatch pipeline. Observations do NOT dispatch bindings — they produce structured results consumed by local rules.

## Design Principles

1. **Memory-first** — individual agent memory (neocortex) is prerequisite before environmental traces become useful (arXiv:2512.10166)
2. **Engine mechanics, blocks intelligence** — classical pattern detection in engine; LLM-backed observation in blocks (#284)
3. **Additive, not replacement** — `ContextChangeTrigger` and the binding dispatch pipeline are untouched
4. **Bounded evaluation** — observers execute synchronously within the serializer gate; unbounded work is rejected

## Architecture

### Evaluation Pipeline

```
CONTEXT_CHANGED event
    │
    ▼
CaseEvaluationSerializer (per-case gate)
    │
    ├── 1. rules()        — existing binding dispatch (untouched)
    ├── 2. goals()        — existing goal evaluation (untouched)
    └── 3. observations() — NEW: evaluate registered observers
                │
                ▼
        ObservationRegistry (in-memory, per-case)
                │
                ▼
        Available for local rules (#1109) in cycle N+1
```

Observation runs AFTER binding dispatch and goal evaluation, still within the serializer gate. Results are stored in the `ObservationRegistry` and available for the next evaluation cycle.

### Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `EnvironmentObserver` | engine-api | `io.casehub.api.spi.observation` |
| `ObservationContext` | engine-api | `io.casehub.api.spi.observation` |
| `Observation` | engine-api | `io.casehub.api.spi.observation` |
| `ObservationConfig` | engine-api | `io.casehub.api.spi.observation` |
| `ContextSnapshot` | engine-api | `io.casehub.api.spi.observation` |
| `ObservationRegistry` | engine-common | `io.casehub.engine.common.internal.observation` |
| `ContextHistoryBuffer` | engine-common | `io.casehub.engine.common.internal.observation` |
| Observer evaluation handler | runtime | `io.casehub.engine.internal.observation` |
| `NoOpObservationEvaluator` | runtime | `io.casehub.engine.internal.observation` |

## SPI Types

### EnvironmentObserver

```java
package io.casehub.api.spi.observation;

import io.casehub.platform.api.routing.NamedStrategy;
import java.util.List;
import java.util.Set;

public interface EnvironmentObserver extends NamedStrategy {

    Set<String> watchedKeys();

    List<Observation> observe(ObservationContext ctx);
}
```

- Extends `NamedStrategy` — consistent with platform SPI convention, resolvable via `EngineStrategyResolver`
- `watchedKeys()` — context keys this observer cares about. Empty set = watch all keys. Engine skips evaluation when none of the changed keys match the watch set
- `observe()` — called on each evaluation cycle. Returns structured observations. Must complete within bounded time (<100ms target). Must not throw — exceptions are caught, logged at WARN, and the observer's contribution is lost for that cycle (D8)

### ObservationContext

```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.List;
import java.util.Set;

public record ObservationContext(
    JsonNode snapshot,
    Set<String> changedKeys,
    List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    java.util.UUID caseId
) {}
```

- `snapshot` — current working layer as `JsonNode` (same surface as `ContextChangeTrigger` evaluation)
- `changedKeys` — keys that changed in this event. Observers can check what triggered the evaluation
- `history` — sliding window of recent context changes for temporal pattern detection (D3)
- `agentId` — the agent this observer belongs to. Enables per-agent personalized observation
- `tenancyId` — tenant scope for memory queries
- `caseId` — case identity

### ContextSnapshot

```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import java.time.Instant;
import java.util.Map;
import java.util.Set;

public record ContextSnapshot(
    Set<String> changedKeys,
    Map<String, JsonNode> changedValues,
    Instant timestamp
) {}
```

- Records what changed in each historical event — NOT a full context snapshot (prohibitively expensive)
- `changedValues` — map of key → new value for each changed key
- Temporal observers inspect the history to detect sequences, trends, and temporal correlations

### Observation

```java
package io.casehub.api.spi.observation;

import java.time.Instant;
import java.util.Map;

public record Observation(
    String patternId,
    double confidence,
    Map<String, Object> details,
    Instant timestamp
) {
    public Observation {
        if (confidence < 0.0 || confidence > 1.0) {
            throw new IllegalArgumentException("confidence must be in [0.0, 1.0]");
        }
    }
}
```

- `patternId` — identifies what pattern was detected (e.g., `"threshold-crossing"`, `"temporal-sequence"`)
- `confidence` — [0.0, 1.0] how confident the observer is. Classical engine observers produce 1.0; LLM-backed observers (blocks) produce probabilistic values
- `details` — structured data about the observation. Schema depends on the pattern type
- `timestamp` — when the observation was produced

### ObservationConfig

```java
package io.casehub.api.spi.observation;

import java.time.Duration;

public record ObservationConfig(
    int maxHistoryEntries,
    Duration maxHistoryAge,
    int maxObserversPerCase
) {
    public static final int DEFAULT_MAX_HISTORY_ENTRIES = 50;
    public static final Duration DEFAULT_MAX_HISTORY_AGE = Duration.ofMinutes(5);
    public static final int DEFAULT_MAX_OBSERVERS_PER_CASE = 20;

    public static ObservationConfig defaults() {
        return new ObservationConfig(
            DEFAULT_MAX_HISTORY_ENTRIES,
            DEFAULT_MAX_HISTORY_AGE,
            DEFAULT_MAX_OBSERVERS_PER_CASE
        );
    }
}
```

Configuration for the observation infrastructure, declared on `CaseDefinition`.

## Infrastructure Types

### ObservationRegistry

```java
package io.casehub.engine.common.internal.observation;

@ApplicationScoped
public class ObservationRegistry implements Resettable {

    // Per-case, per-agent observer registrations
    // ConcurrentHashMap<UUID caseId, ConcurrentHashMap<String agentId, List<EnvironmentObserver>>>
    boolean registerObserver(UUID caseId, String agentId, EnvironmentObserver observer, int maxPerCase);
    void unregisterByAgent(UUID caseId, String agentId);
    void unregisterByCase(UUID caseId);
    Map<String, List<EnvironmentObserver>> getObservers(UUID caseId);
    int observerCount(UUID caseId);

    // Per-case observation results (latest cycle only)
    void storeObservations(UUID caseId, String agentId, List<Observation> observations);
    List<Observation> getObservations(UUID caseId, String agentId);
    Map<String, List<Observation>> getAllObservations(UUID caseId);

    void reset(); // Resettable
}
```

- `registerObserver` returns `false` when per-case cap is reached (D9, default 20)
- `unregisterByCase` called by `CaseStatusChangedHandler` on terminal status
- `storeObservations` replaces previous observations for that agent (latest cycle only — observations are ephemeral)
- Implements `Resettable` for test reset

### ContextHistoryBuffer

```java
package io.casehub.engine.common.internal.observation;

public class ContextHistoryBuffer {

    // Per-case circular buffer of ContextSnapshots
    void record(UUID caseId, ContextSnapshot snapshot);
    List<ContextSnapshot> getHistory(UUID caseId, int maxEntries, Duration maxAge);
    void evict(UUID caseId);
    void evictExpired(UUID caseId, int maxEntries, Duration maxAge);
}
```

- Records context changes on every `CONTEXT_CHANGED` event, before observer evaluation
- Bounded by count AND time (D3): keeps last N entries, discards entries older than T
- `evict` called on terminal case status
- NOT per-agent — the history is shared. Per-agent filtering happens in `ObservationContext` construction

## Registration Mechanism

Agents register observers through `WorkerRuntime.registerObserver()` during worker execution (D2):

```java
// Addition to WorkerRuntime (engine-api)
public interface WorkerRuntime extends WorkerScope {
    // ... existing methods ...
    default boolean registerObserver(EnvironmentObserver observer) {
        return false; // no-op default for backward compat
    }
}
```

- `DefaultWorkerRuntime` delegates to `ObservationRegistry.registerObserver(caseId, agentId, observer, maxPerCase)`
- Returns `false` when the per-case observer cap is reached
- Observer lifecycle follows `LifecycleScope`:
  - **BINDING** — observer destroyed after single dispatch (not useful for temporal patterns)
  - **COMPOUND** — observer lives for compound duration. Removed by `ScopedWorkerTerminationHandler` on `COMPOUND_COMPLETED`
  - **CASE** — observer lives for case duration. Removed by `CaseStatusChangedHandler` on terminal status

The `DefaultWorkerRuntime` implementation resolves `LifecycleScope` from the binding's declaration and registers the observer with the appropriate cleanup hook.

## Integration Points

### CaseContextChangedEventHandler

New method `observations()` called after `rules()` and `goals()` in `evaluateAndDispatch()`:

```java
private void observations(CaseInstance caseInstance, CaseContext contextSnapshot,
                           CaseDefinition definition, String changedLayer) {
    ObservationConfig config = definition.getObservationConfig();
    if (config == null) {
        config = ObservationConfig.defaults();
    }

    // Record history
    Set<String> changedKeys = /* extract from diff */;
    historyBuffer.record(caseInstance.getUuid(),
        new ContextSnapshot(changedKeys, extractChangedValues(contextSnapshot, changedKeys),
                            Instant.now()));
    historyBuffer.evictExpired(caseInstance.getUuid(),
        config.maxHistoryEntries(), config.maxHistoryAge());

    // Evaluate observers
    Map<String, List<EnvironmentObserver>> observers =
        observationRegistry.getObservers(caseInstance.getUuid());
    if (observers.isEmpty()) {
        return;
    }

    JsonNode snapshot = contextSnapshot.layer(ContextLayer.WORKING).asJsonNode();
    List<ContextSnapshot> history = historyBuffer.getHistory(
        caseInstance.getUuid(), config.maxHistoryEntries(), config.maxHistoryAge());

    for (var entry : observers.entrySet()) {
        String agentId = entry.getKey();
        List<Observation> agentObservations = new ArrayList<>();

        for (EnvironmentObserver observer : entry.getValue()) {
            // Key filtering optimization
            if (!observer.watchedKeys().isEmpty()
                && Collections.disjoint(observer.watchedKeys(), changedKeys)) {
                continue;
            }

            ObservationContext ctx = new ObservationContext(
                snapshot, changedKeys, history, agentId,
                caseInstance.tenancyId, caseInstance.getUuid());

            try {
                List<Observation> results = observer.observe(ctx);
                if (results != null) {
                    agentObservations.addAll(results);
                }
            } catch (Exception e) {
                LOG.warnf(e, "Observer %s failed for case=%s agent=%s",
                    observer.id(), caseInstance.getUuid(), agentId);
            }
        }

        observationRegistry.storeObservations(
            caseInstance.getUuid(), agentId, agentObservations);
    }
}
```

### CaseStatusChangedHandler

On terminal case status, cleanup:
```java
observationRegistry.unregisterByCase(caseId);
historyBuffer.evict(caseId);
```

### ScopedWorkerTerminationHandler

On `COMPOUND_COMPLETED`, cleanup compound-scoped observers:
```java
// For each terminated scoped worker in the compound:
observationRegistry.unregisterByAgent(caseId, agentId);
```

### CaseDefinition

New field:
```java
private ObservationConfig observationConfig; // nullable, defaults to ObservationConfig.defaults()
```

Builder: `.observationConfig(ObservationConfig)`. YAML:
```yaml
spec:
  observation:
    maxHistoryEntries: 50
    maxHistoryAge: PT5M
    maxObserversPerCase: 20
```

## Classical Observer Implementations (Engine)

Three built-in observers for the full pattern vocabulary. All live in `runtime/internal/observation/`.

### ThresholdObserver

Detects when a numeric context value crosses a threshold.

```java
@ApplicationScoped
public class ThresholdObserver implements EnvironmentObserver {
    // id: "threshold"
    // Configured per-registration with: key, operator (GT, LT, GTE, LTE, EQ), threshold value
    // watchedKeys: {key}
    // Produces: Observation("threshold-crossing", 1.0,
    //   {key, value, threshold, operator, direction: "rising"|"falling"})
}
```

### CorrelationObserver

Detects when multiple keys satisfy a condition simultaneously.

```java
@ApplicationScoped
public class CorrelationObserver implements EnvironmentObserver {
    // id: "correlation"
    // Configured per-registration with: keys, JQ condition expression
    // watchedKeys: {keys}
    // Produces: Observation("multi-key-correlation", 1.0,
    //   {keys, values, condition})
}
```

### TemporalSequenceObserver

Detects when a sequence of key changes occurs within a time window.

```java
@ApplicationScoped
public class TemporalSequenceObserver implements EnvironmentObserver {
    // id: "temporal-sequence"
    // Configured per-registration with: sequence of (key, optional value predicate), window duration
    // watchedKeys: {all keys in sequence}
    // Uses history buffer to detect temporal ordering
    // Produces: Observation("temporal-sequence", 1.0,
    //   {sequence, window, matchedAt: [timestamps]})
}
```

These are **factory-style** — each registration creates a configured instance. The observer itself is parameterized at registration time via the `WorkerRuntime.registerObserver()` call. Static factory methods on each class provide the construction API.

## Thread Safety

- `ObservationRegistry` uses `ConcurrentHashMap` — concurrent registration from multiple workers is safe
- Observer evaluation is serialized per-case by `CaseEvaluationSerializer` — no concurrent evaluation for the same case
- `ContextHistoryBuffer` uses per-case locking for append — history records arrive from the serializer gate (no contention in practice)
- Observer `observe()` calls receive immutable `ObservationContext` — no shared mutable state

## Audit

New `CaseHubEventType`:
- `OBSERVER_REGISTERED` — when an agent registers an observer. Metadata: `agentId`, `observerId`, `watchedKeys`
- `OBSERVATION_DETECTED` — when an observer produces observations. Metadata: `agentId`, `observerId`, `patternIds`, `observationCount`

Both are fire-and-forget EventLog writes — they do not block the evaluation pipeline.

## YAML Schema Extension

```yaml
spec:
  observation:
    maxHistoryEntries: 50          # default 50
    maxHistoryAge: PT5M            # default 5 minutes, ISO-8601
    maxObserversPerCase: 20        # default 20
```

No YAML declaration for individual observers — registration is runtime-only via `WorkerRuntime.registerObserver()`. Static observer patterns may be added in a future issue if needed.

## Constraints

- Must NOT break existing `ContextChangeTrigger` model — observation is additive
- Must NOT write to `CaseContext` — avoids feedback loops (D7)
- Must NOT call LLM inside the observer — LLM observation dispatches as a separate worker (D6)
- Observer evaluation bounded: <100ms per observer, 20 observers per case max (D9)
- `EnvironmentObserver` lives in `engine-api` — no dependency on `engine-common` or `runtime` types

## Dependencies

- **#1106** (Signal/pheromone model) — observers can detect pheromone signals; pheromones are context values that observers watch
- **#1107** (Dynamic interest registration) — extends the registration mechanism with runtime pattern negotiation
- **#1109** (Local rule evaluation) — primary consumer of observations; queries `ObservationRegistry`
- **blocks#284** (LLM-driven observation) — provides `EnvironmentObserver` implementations that detect trigger patterns and dispatch LLM calls

## References

- `CaseContext.java:26` — the context being observed
- `ContextChangeTrigger.java:21` — existing static trigger model (preserved, not replaced)
- `CaseContextChangedEventHandler.java:252-354` — existing rules() dispatch method
- `CaseEvaluationSerializer.java:23` — per-case serialization gate
- `ScopedWorkerRegistry.java:23` — lifecycle scope pattern
- `WorkerRuntime.java:24` — registration surface
- `CaseContextChangedEvent.java:32` — event carrying context change information
- arXiv:2512.10166 — Emergent Collective Memory (memory-first principle)
- arXiv:2608.26081 — SwarmWorld (cognition/consequence split)
- casehubio/engine#1104 — Hive Mind epic
- casehubio/engine#1105 — this issue
