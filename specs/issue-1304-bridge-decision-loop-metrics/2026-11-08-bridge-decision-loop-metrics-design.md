# Bridge DecisionLoopEvent Metrics into SwarmTelemetry LLM Fields

**Issue:** casehubio/engine#1304, casehubio/agency#444
**Date:** 2026-11-08

## Problem

`SwarmTelemetry` has three LLM-related fields — `invocationCount`, `tokensConsumed`, `totalLatency` — that `SwarmTelemetryCollector` populates with `(0, 0L, Duration.ZERO)`. The agency layer already emits `DecisionLoopEvent.ReasoningCompleted` with actual `tokensConsumed` and `latencyMs` per LLM call, but no bridge connects them.

The infrastructure for this bridge was partially designed: `ExecutionEventListener.onDecisionLoopEvent()` exists as a default no-op method, and `SwarmCoordinatorBuilder.buildTopology()` passes a no-op `loopEventSink` with the comment "wired at execution layer." Neither was completed.

## Design

### Architecture

Three layers, each with a clear responsibility:

```
DecisionLoopFactory                    (agency-core: emits events)
  └─ loopEventSink Consumer
       └─ AbstractExecutionDriver      (agency-core: routes to listeners)
            └─ notifyDecisionLoopEvent
                 └─ CheckpointingListener  (engine-adapter-core: accumulates)
                      └─ AgentLlmStatsProvider SPI  (engine-api: contract)
                           └─ SwarmTelemetryCollector  (engine-runtime: consumes)
```

### Agency Changes (casehubio/agency#444)

**1. `AbstractExecutionDriver` — add `notifyDecisionLoopEvent`**

Add a protected method following the existing `notifyXxx` pattern:

```java
protected void notifyDecisionLoopEvent(ExecutionModel<T> model,
                                        DecisionLoopEvent event) {
    for (var listener : model.listeners()) {
        listener.onDecisionLoopEvent(event);
    }
}
```

**2. `SwarmCoordinatorBuilder` — accept external `loopEventSink`**

Replace the hardcoded `e -> {}` with a builder-configurable `Consumer<DecisionLoopEvent>`. Default remains no-op for backward compatibility. The execution layer provides the real sink.

```java
private Consumer<DecisionLoopEvent> loopEventSink = e -> {};

public SwarmCoordinatorBuilder loopEventSink(Consumer<DecisionLoopEvent> sink) {
    this.loopEventSink = Objects.requireNonNull(sink);
    return this;
}
```

Pass `this.loopEventSink` instead of `e -> {}` in `buildTopology()`.

**3. `CentralisedTopology` — propagate `loopEventSink`**

The `loopEventSink` parameter already flows through `CentralisedTopology` into `DecisionLoopFactory.create()`. No change needed if `SwarmCoordinatorBuilder` passes the real sink. Verify the parameter propagation path.

**4. Execution layer wiring**

Where the `ExecutionModel` is constructed with its listener list and the topology is built, wire the `loopEventSink` to call `notifyDecisionLoopEvent`. The exact wiring point depends on how `SwarmCoordinatorBuilder` is invoked — likely in `PatternWorkerFunctionHandler` or the pattern execution setup in `engine-adapter-core`.

### Engine-Adapter Changes (casehubio/agency — engine-adapter-core module)

**5. `CheckpointingListener` — accumulate LLM stats**

Add a `ConcurrentHashMap<String, LlmAccumulator>` keyed by agentId. Override `onDecisionLoopEvent`:

```java
@Override
public void onDecisionLoopEvent(DecisionLoopEvent event) {
    if (event instanceof DecisionLoopEvent.ReasoningCompleted rc) {
        accumulators.computeIfAbsent(rc.agentId(), k -> new LlmAccumulator())
                    .record(rc.tokensConsumed(), rc.latencyMs());
    }
}
```

Where `LlmAccumulator` is a private inner class:

```java
private static class LlmAccumulator {
    private int invocationCount;
    private long tokensConsumed;
    private long totalLatencyMs;

    void record(long tokens, long latencyMs) {
        invocationCount++;
        tokensConsumed += tokens;
        totalLatencyMs += latencyMs;
    }
}
```

**6. `CheckpointingListener` — implement `AgentLlmStatsProvider`**

Have `CheckpointingListener` implement the engine SPI:

```java
public class CheckpointingListener implements ExecutionEventListener, AgentLlmStatsProvider {
    // ...
    @Override
    public AgentLlmStats getStats(UUID caseId, String agentId) {
        var acc = accumulators.get(agentId);
        if (acc == null) return AgentLlmStats.ZERO;
        return new AgentLlmStats(
            acc.invocationCount,
            acc.tokensConsumed,
            Duration.ofMillis(acc.totalLatencyMs));
    }
}
```

### Engine Changes (casehubio/engine#1304)

**7. `AgentLlmStatsProvider` SPI in `engine/api`**

```java
package io.casehub.api.spi.stigmergy;

import java.time.Duration;
import java.util.UUID;

public interface AgentLlmStatsProvider {
    record AgentLlmStats(int invocationCount, long tokensConsumed, Duration totalLatency) {
        public static final AgentLlmStats ZERO = new AgentLlmStats(0, 0L, Duration.ZERO);
    }
    AgentLlmStats getStats(UUID caseId, String agentId);
}
```

**8. `SwarmTelemetryCollector` — consume stats**

Add `AgentLlmStatsProvider` as a constructor parameter (nullable for backward compatibility):

```java
public SwarmTelemetryCollector(StigmergyCoordinator coordinator,
                                SignalRegistry signalRegistry,
                                AgentLlmStatsProvider llmStatsProvider) {
    this.coordinator = coordinator;
    this.signalRegistry = signalRegistry;
    this.llmStatsProvider = llmStatsProvider;
}
```

In `snapshot()`, replace the hardcoded zeros:

```java
var llmStats = llmStatsProvider != null
    ? llmStatsProvider.getStats(caseId, agent.agentId())
    : AgentLlmStats.ZERO;

result.add(new SwarmTelemetry(
    caseId, agent.agentId(), workState,
    signalsDeposited, messagesSent, tasksCompleted, activeTime,
    llmStats.invocationCount(),
    llmStats.tokensConsumed(),
    llmStats.totalLatency()));
```

### Testing

- **`SwarmTelemetryCollectorTest`** — extend existing tests to verify LLM fields are populated when a provider is supplied, and remain zero when provider is null.
- **`CheckpointingListenerTest`** — add tests for `onDecisionLoopEvent` accumulation: single event, multiple events for same agent, multiple agents, zero-valued events.
- **`AbstractExecutionDriverTest`** — verify `notifyDecisionLoopEvent` calls all listeners.

### Thread Safety

`CheckpointingListener` is used within a single execution — the `AbstractExecutionDriver` runs sequentially through iterations. The accumulator does not need to be thread-safe unless concurrent agent dispatch changes. Use `ConcurrentHashMap` for the map (agents may be dispatched concurrently within an iteration) but the `LlmAccumulator` fields can use simple `+=` since each agent's decision loop runs sequentially.

### Backward Compatibility

- `SwarmTelemetryCollector` accepts null `llmStatsProvider` (falls back to zeros) — existing callers unaffected.
- `SwarmCoordinatorBuilder.loopEventSink()` defaults to no-op — existing topology construction unchanged.
- `ExecutionEventListener.onDecisionLoopEvent()` already has a default no-op — existing listeners unaffected.

## References

- `SwarmTelemetryCollector.java:61-72` — hardcoded zeros
- `SwarmTelemetry.java:29-31` — LLM fields
- `DecisionLoopEvent.java:61-77` — `ReasoningCompleted` definition
- `DecisionLoopFactory.java:60-76` — `budgetBridge` that emits `ReasoningCompleted`
- `SwarmCoordinatorBuilder.java:132` — no-op `loopEventSink` with "wired at execution layer" comment
- `AbstractExecutionDriver.java:229-298` — existing `notifyXxx` pattern
- `ExecutionEventListener.java:29` — existing unused `onDecisionLoopEvent` method
- `CheckpointingListener.java:32-101` — engine-adapter bridge listener
- `MetricsListener.java:133` — only listener currently overriding `onDecisionLoopEvent`
