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
5. **Memory integration deferred** — neocortex memory access is out of scope for this SPI issue. Bounded evaluation (<100ms per observer within the serializer gate) precludes synchronous memory queries. Memory integration will be provided via a follow-up issue that adds async memory pre-fetch to `ObservationContext`. See §Dependencies

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

`NoOpObservationEvaluator` is a no-op implementation of the observation evaluation step. Used when `CaseDefinition` has no observation config or when observation is explicitly disabled. The `observations()` method returns immediately without checking for observers, recording history, or evaluating anything. This avoids the overhead of `ContextHistoryBuffer` lookups for cases that never use observations.

## SPI Types

### EnvironmentObserver

```java
package io.casehub.api.spi.observation;

import java.util.List;
import java.util.Set;

public interface EnvironmentObserver {

    String observerType();

    Set<String> watchedKeys();

    List<Observation> observe(ObservationContext ctx);
}
```

- `observerType()` — type identifier for audit/logging (e.g., `"threshold"`, `"correlation"`, `"temporal-sequence"`). NOT a unique instance ID — multiple instances of the same type may exist. Unique instance IDs are generated at registration time by `ObservationRegistry`. Does NOT extend `NamedStrategy` — observers are not resolved via `EngineStrategyResolver`. They are created programmatically by workers and registered via `WorkerRuntime.registerObserver()`
- `watchedKeys()` — context keys this observer cares about. Empty set = watch all keys. Engine skips evaluation when none of the changed keys match the watch set. Must not return null — throws `IllegalArgumentException` at registration if null
- `observe()` — called on each evaluation cycle. Returns structured observations. Must complete within bounded time (<100ms enforced via virtual-thread timeout). Must not throw — exceptions are caught, logged at WARN, and the observer's contribution is lost for that cycle (D8)

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

import com.fasterxml.jackson.databind.JsonNode;
import java.time.Instant;
import java.util.Map;

public record Observation(
    String patternId,
    double confidence,
    Map<String, JsonNode> details,
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
- `details` — `Map<String, JsonNode>` structured data about the observation. Uses `JsonNode` for serialization safety and consistency with `ContextSnapshot.changedValues`. Schema depends on the pattern type
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

`agentId` = worker name (`Worker.name()`) — the agent identity in the routing model.

```java
package io.casehub.engine.common.internal.observation;

@ApplicationScoped
public class ObservationRegistry implements Resettable {

    // Per-case, per-agent observer registrations
    // ConcurrentHashMap<UUID caseId, ConcurrentHashMap<String agentId, List<ObserverRegistration>>>
    // ObserverRegistration(EnvironmentObserver observer, String agentId, String bindingName, String instanceId)

    boolean registerObserver(UUID caseId, String agentId, String bindingName,
                             EnvironmentObserver observer, int maxPerCase);
    void unregisterByAgent(UUID caseId, String agentId);
    void unregisterByBinding(UUID caseId, Set<String> bindingNames);
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

- `registerObserver` records both `agentId` (worker name) and `bindingName` for lifecycle management. Generates a unique `instanceId` (observerType + sequence) for audit. Throws `IllegalArgumentException` for null observer, null `observerType()`, or null `watchedKeys()`. Returns `false` ONLY when per-case cap is reached (D9, default 20) — capacity limits are not programming errors
- `unregisterByBinding` called by `ScopedWorkerTerminationHandler` on `COMPOUND_COMPLETED` — receives binding names from `CompoundCompletedEvent.scopedBindingNames()`, removes all observers registered under those bindings
- `unregisterByAgent` removes all observers for a given worker name — available for explicit programmatic cleanup
- `unregisterByCase` called by `CaseStatusChangedHandler` on terminal status
- `storeObservations` replaces previous observations for that agent (latest cycle only — observations are ephemeral)
- Implements `Resettable` for test reset

### ContextHistoryBuffer

```java
package io.casehub.engine.common.internal.observation;

@ApplicationScoped
public class ContextHistoryBuffer implements Resettable {

    // Per-case circular buffer of ContextSnapshots
    // Per-case "last processed" snapshot for diff-based changedKeys computation

    Set<String> computeChangedKeys(UUID caseId, JsonNode currentSnapshot);
    Map<String, JsonNode> extractChangedValues(JsonNode currentSnapshot, Set<String> changedKeys);
    void record(UUID caseId, ContextSnapshot snapshot);
    List<ContextSnapshot> getHistory(UUID caseId, int maxEntries, Duration maxAge);
    void evict(UUID caseId);
    void evictExpired(UUID caseId, int maxEntries, Duration maxAge);

    void reset(); // Resettable
}
```

- `@ApplicationScoped` — singleton that survives across evaluation cycles, accumulating history for the case lifetime
- `computeChangedKeys` — diffs `currentSnapshot` against a per-case `lastProcessedSnapshot` (stored internally). Returns the set of top-level keys that were added, removed, or whose values differ. Updates `lastProcessedSnapshot` to `currentSnapshot` after computation. First call for a case (no prior snapshot) returns all current keys as "changed"
- `extractChangedValues` — extracts the new values for each changed key from the current snapshot
- Records context changes ONLY when observers are registered for the case (`observerCount(caseId) > 0`) — avoids wasted memory for cases without observers
- Bounded by count AND time (D3): keeps last N entries, discards entries older than T
- `evict` called on terminal case status — removes both history entries and `lastProcessedSnapshot`
- NOT per-agent — the history is shared. Per-agent filtering happens in `ObservationContext` construction
- Aggregate memory: per-case `ContextSnapshot` records are lightweight (changed keys + values only, not full context). The `lastProcessedSnapshot` is the only full `JsonNode` stored per case. Per-case entry count is bounded by `ObservationConfig.maxHistoryEntries`

## Registration Mechanism

### Identity model

`agentId` = worker name (`Worker.name()`) — the agent identity in the routing model. A binding dispatches a worker; the worker IS the agent. The observation registry is keyed by worker name for queries (local rules ask "what has agent-X observed?"), and also records the binding name for compound-scoped cleanup (which has binding names, not worker names).

### WorkerRuntime API

```java
// Addition to WorkerRuntime (engine-api)
public interface WorkerRuntime extends WorkerScope {
    // ... existing methods ...
    default boolean registerObserver(EnvironmentObserver observer) {
        return false; // no-op default for backward compat
    }
}
```

### Required changes to WorkerRuntimeFactory

`WorkerRuntimeFactory.create()` must be extended to thread through the worker name and binding name:

```java
public WorkerRuntime create(UUID caseId, String taskId, WorkerContext context,
                            Map<String, Object> accumulatedState,
                            String workerName, String bindingName) {
    return new DefaultWorkerRuntime(caseId, taskId, context, accumulatedState,
        ..., observationRegistry, workerName, bindingName);
}
```

`DefaultWorkerRuntime` gains three new fields: `ObservationRegistry observationRegistry`, `String workerName`, `String bindingName`. The `registerObserver()` implementation:

```java
@Override
public boolean registerObserver(EnvironmentObserver observer) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    CaseDefinition definition = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
    ObservationConfig config = definition.getObservationConfig();
    return observationRegistry.registerObserver(
        caseId, workerName, bindingName, observer, config.maxObserversPerCase());
}
```

The `ObservationConfig` and `maxObserversPerCase` are resolved at registration time via the `CaseDefinitionRegistry` already available on `DefaultWorkerRuntime`. No `LifecycleScope` resolution is needed in the runtime — the registry records the binding name, and cleanup hooks use binding names from the lifecycle events they already receive.

### Observer lifecycle

Observer lifecycle follows the binding's `LifecycleScope`:
  - **BINDING** — registration rejected. `DefaultWorkerRuntime.registerObserver()` checks the binding's `LifecycleScope` and throws `IllegalStateException` if BINDING. BINDING-scoped workers have no lifecycle cleanup event, so observers registered during BINDING dispatch would persist until case termination — effectively becoming CASE-scoped and consuming the per-case observer quota. Workers needing observers must declare COMPOUND or CASE scope on their binding
  - **COMPOUND** — observer lives for compound duration. Removed by `ScopedWorkerTerminationHandler` on `COMPOUND_COMPLETED` via `observationRegistry.unregisterByBinding(caseId, scopedBindingNames)`
  - **CASE** — observer lives for case duration. Removed by `CaseStatusChangedHandler` on terminal status via `observationRegistry.unregisterByCase(caseId)`

## Integration Points

### CaseContextChangedEventHandler

New method `observations()` called after `rules()` and `goals()` in `evaluateAndDispatch()`:

```java
private void observations(CaseInstance caseInstance, CaseContext contextSnapshot,
                           CaseDefinition definition, String changedLayer) {
    // Early exit: no observers registered for this case
    if (observationRegistry.observerCount(caseInstance.getUuid()) == 0) {
        return;
    }

    ObservationConfig config = definition.getObservationConfig();
    // getObservationConfig() returns defaults when null — no consumer-side null check

    JsonNode snapshot = contextSnapshot.layer(ContextLayer.WORKING).asJsonNode();

    // Compute changedKeys by diffing current snapshot against last-processed snapshot
    Set<String> changedKeys = historyBuffer.computeChangedKeys(
        caseInstance.getUuid(), snapshot);
    Map<String, JsonNode> changedValues = historyBuffer.extractChangedValues(
        snapshot, changedKeys);

    // Record history
    historyBuffer.record(caseInstance.getUuid(),
        new ContextSnapshot(changedKeys, changedValues, Instant.now()));
    historyBuffer.evictExpired(caseInstance.getUuid(),
        config.maxHistoryEntries(), config.maxHistoryAge());

    // Evaluate observers
    Map<String, List<EnvironmentObserver>> observers =
        observationRegistry.getObservers(caseInstance.getUuid());

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

            // Timeout-enforced evaluation via virtual thread
            try {
                List<Observation> results = CompletableFuture
                    .supplyAsync(() -> observer.observe(ctx), virtualThreads)
                    .orTimeout(100, TimeUnit.MILLISECONDS)
                    .join();
                if (results != null) {
                    agentObservations.addAll(results);
                }
            } catch (CompletionException e) {
                if (e.getCause() instanceof TimeoutException) {
                    LOG.warnf("Observer %s timed out (>100ms) for case=%s agent=%s",
                        observer.observerType(), caseInstance.getUuid(), agentId);
                } else {
                    LOG.warnf(e.getCause(), "Observer %s failed for case=%s agent=%s",
                        observer.observerType(), caseInstance.getUuid(), agentId);
                }
            }
        }

        observationRegistry.storeObservations(
            caseInstance.getUuid(), agentId, agentObservations);
    }
}
```

### CaseStatusChangedHandler

On terminal case status, cleanup all observation state:
```java
observationRegistry.unregisterByCase(caseId);
historyBuffer.evict(caseId); // removes history entries AND lastProcessedSnapshot
```

### ScopedWorkerTerminationHandler

On `COMPOUND_COMPLETED`, cleanup compound-scoped observers:
```java
// scopedBindingNames comes from CompoundCompletedEvent.scopedBindingNames()
observationRegistry.unregisterByBinding(caseId, event.scopedBindingNames());
```

### CaseDefinition

New field:
```java
private ObservationConfig observationConfig; // nullable — getter applies defaults

public ObservationConfig getObservationConfig() {
    return observationConfig != null ? observationConfig : ObservationConfig.defaults();
}
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

Three built-in observers for the full pattern vocabulary. All live in `runtime/internal/observation/`. These are plain classes (NOT CDI beans) — each registration creates a new parameterized instance via static factory methods.

### ThresholdObserver

Detects when a numeric context value crosses a threshold.

```java
public class ThresholdObserver implements EnvironmentObserver {
    // observerType: "threshold"
    // Configured per-registration with: key, operator (GT, LT, GTE, LTE, EQ), threshold value
    // watchedKeys: {key}
    // Produces: Observation("threshold-crossing", 1.0,
    //   {key, value, threshold, operator, direction: "rising"|"falling"})

    public static ThresholdObserver of(String key, Operator operator, double threshold) { ... }
}
```

### CorrelationObserver

Detects when multiple keys satisfy a condition simultaneously. Distinct from `ContextChangeTrigger`: triggers produce a boolean dispatch decision, observers produce structured observations (patternId, confidence, details) consumed by local rules.

```java
public class CorrelationObserver implements EnvironmentObserver {
    // observerType: "correlation"
    // Configured per-registration with: keys, JQ condition expression
    // watchedKeys: {keys}
    // Produces: Observation("multi-key-correlation", 1.0,
    //   {keys, values, condition})

    public static CorrelationObserver of(Set<String> keys, String jqCondition) { ... }
}
```

### TemporalSequenceObserver

Detects when a sequence of key changes occurs within a time window.

```java
public class TemporalSequenceObserver implements EnvironmentObserver {
    // observerType: "temporal-sequence"
    // Configured per-registration with: sequence of (key, optional value predicate), window duration
    // watchedKeys: {all keys in sequence}
    // Uses history buffer to detect temporal ordering
    // Produces: Observation("temporal-sequence", 1.0,
    //   {sequence, window, matchedAt: [timestamps]})

    public static TemporalSequenceObserver of(List<SequenceStep> steps, Duration window) { ... }
}
```

Construction is via static factory methods — workers call e.g. `ThresholdObserver.of("temperature", Operator.GT, 100.0)` and register the returned instance via `WorkerRuntime.registerObserver()`. Multiple instances of the same type with different parameters coexist; each gets a unique `instanceId` from the `ObservationRegistry` at registration time.

## Thread Safety

- `ObservationRegistry` uses `ConcurrentHashMap` — concurrent registration from multiple workers is safe
- Observer evaluation is serialized per-case by `CaseEvaluationSerializer` — no concurrent evaluation for the same case
- `ContextHistoryBuffer` uses per-case locking for append — history records arrive from the serializer gate (no contention in practice)
- Observer `observe()` calls receive immutable `ObservationContext` — no shared mutable state

## Audit

New `CaseHubEventType`:
- `OBSERVER_REGISTERED` — when an agent registers an observer. Metadata: `agentId`, `observerType`, `watchedKeys`
- `OBSERVATION_DETECTED` — when an observer produces observations. Metadata: `agentId`, `observerType`, `patternIds`, `observationCount`

Audit events use `observerType()` (not instance ID) — the evaluation loop iterates raw `EnvironmentObserver` instances and has access to `observerType()` but not to registry-internal instance IDs. Instance-level audit tracking is not needed for v1.

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
- Observer evaluation bounded: <100ms per observer (enforced via `CompletableFuture.orTimeout()` on virtual threads — observers exceeding the bound have their contribution discarded for that cycle; the observer's virtual thread continues running until natural completion but its result is ignored. `orTimeout` does NOT interrupt the underlying thread. Virtual threads are cheap, so the resource cost of leaked observer threads is low. True cooperative interruption would require observers to check `Thread.interrupted()`, which cannot be enforced at the SPI level). 20 observers per case max (D9)
- `EnvironmentObserver` lives in `engine-api` — no dependency on `engine-common` or `runtime` types

## Dependencies

- **#1106** (Signal/pheromone model) — observers can detect pheromone signals; pheromones are context values that observers watch
- **#1107** (Dynamic interest registration) — extends the registration mechanism with runtime pattern negotiation
- **#1109** (Local rule evaluation) — primary consumer of observations; queries `ObservationRegistry`
- **blocks#284** (LLM-driven observation) — provides `EnvironmentObserver` implementations that detect trigger patterns and dispatch LLM calls

### Deferred: Neocortex memory integration

Issue #1105 lists "Integration with neocortex memory — observations can reference agent memory for context" as a requirement. This is explicitly deferred from the SPI foundation:

- **Rationale:** Observation evaluation runs within the serializer gate with a <100ms budget. Synchronous neocortex queries would violate this constraint. Memory integration requires an async pre-fetch pattern — retrieve relevant memories BEFORE the observation cycle, inject them into `ObservationContext`.
- **Design surface:** `ObservationContext` is a record; adding a `List<RetrievedMemory> memories` field in a follow-up issue is non-breaking. The field would be populated by the evaluation handler from pre-fetched memories.
- **Tracked as:** Follow-up issue under #1104 epic (to be filed at implementation time)

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
