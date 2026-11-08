# Bridge DecisionLoopEvent Metrics into SwarmTelemetry LLM Fields

**Issue:** casehubio/engine#1304, casehubio/agency#444
**Date:** 2026-11-08

## Problem

`SwarmTelemetry` has three LLM-related fields — `invocationCount`, `tokensConsumed`, `totalLatency` — that `SwarmTelemetryCollector` populates with `(0, 0L, Duration.ZERO)`. The agency layer already emits `DecisionLoopEvent.ReasoningCompleted` with actual `tokensConsumed` and `latencyMs` per LLM call, but no bridge connects them.

The infrastructure for this bridge was partially designed: `ExecutionEventListener.onDecisionLoopEvent()` exists as a default no-op method, and `SwarmCoordinatorBuilder.buildTopology()` passes a no-op `loopEventSink` with the comment "wired at execution layer." Neither was completed.

## Execution Path Constraint

`DecisionLoopEvent` is emitted inside coordination topologies (`CentralisedTopology` → `DecisionLoopFactory` → `AgentDecisionLoop`). This is a separate execution path from the standard `ExecutionModel` → `AbstractExecutionDriver` loop. The driver's `notifyXxx` pattern and `ExecutionEventListener` operate on the driver path only — they never see coordination topology events.

The `loopEventSink` is captured at topology construction time by `SwarmCoordinatorBuilder.buildTopology()`, not at driver execution time. The accumulator must be wired directly as the `loopEventSink` consumer, not through the driver's listener chain.

## Design

### Architecture

```
DecisionLoopFactory                    (agency-core: emits events)
  └─ loopEventSink Consumer
       └─ LlmTelemetryAccumulator     (engine-adapter-core: accumulates)
            ├─ implements Consumer<DecisionLoopEvent>
            └─ implements AgentLlmStatsProvider  (engine-api: SPI contract)
                 └─ SwarmTelemetryCollector       (engine-runtime: consumes)
```

### Agency Changes (casehubio/agency#444)

**1. `SwarmCoordinatorBuilder` — accept external `loopEventSink`**

Add a builder field and method:

```java
private Consumer<DecisionLoopEvent> loopEventSink = e -> {};

public SwarmCoordinatorBuilder loopEventSink(Consumer<DecisionLoopEvent> sink) {
    this.loopEventSink = Objects.requireNonNull(sink);
    return this;
}
```

In `buildTopology()`, replace the hardcoded `e -> {}` with `this.loopEventSink`.

**2. `CentralisedTopology` — no change needed**

The `loopEventSink` parameter already flows through `CentralisedTopology` into `DecisionLoopFactory.create()`. Once `SwarmCoordinatorBuilder` passes a real sink, the propagation works.

### Engine-Adapter Changes (casehubio/agency — engine-adapter-core module)

**3. `LlmTelemetryAccumulator` — new class**

A standalone accumulator that serves as both the `loopEventSink` consumer and the `AgentLlmStatsProvider` implementation:

```java
package io.casehub.engine.agentic;

public class LlmTelemetryAccumulator
        implements Consumer<DecisionLoopEvent>, AgentLlmStatsProvider {

    private final UUID caseId;
    private final ConcurrentHashMap<String, LlmAccumulator> accumulators = new ConcurrentHashMap<>();

    public LlmTelemetryAccumulator(UUID caseId) {
        this.caseId = caseId;
    }

    @Override
    public void accept(DecisionLoopEvent event) {
        if (event instanceof DecisionLoopEvent.ReasoningCompleted rc) {
            accumulators.computeIfAbsent(rc.agentId(), k -> new LlmAccumulator())
                        .record(rc.tokensConsumed(), rc.latencyMs());
        }
    }

    @Override
    public AgentLlmStats getStats(UUID caseId, String agentId) {
        var acc = accumulators.get(agentId);
        if (acc == null) return AgentLlmStats.ZERO;
        return new AgentLlmStats(
            acc.invocationCount,
            acc.tokensConsumed,
            Duration.ofMillis(acc.totalLatencyMs));
    }

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
}
```

**4. Wiring at topology construction**

Wherever `SwarmCoordinatorBuilder` is invoked to create a coordination topology for the engine, the caller creates an `LlmTelemetryAccumulator` and passes it:

```java
var accumulator = new LlmTelemetryAccumulator(caseId);
var topology = Patterns.swarm()
    .agentProvider(provider)
    .loopEventSink(accumulator)
    .buildTopology();
```

The same `accumulator` instance is passed to `SwarmTelemetryCollector` as the `AgentLlmStatsProvider`. The accumulator bridges the two systems: it receives events from the coordination topology and provides stats to the engine's telemetry collector.

### Engine Changes (casehubio/engine#1304)

**5. `AgentLlmStatsProvider` SPI in `engine/api`**

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

**6. `SwarmTelemetryCollector` — consume stats**

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

**7. Update `CaseContextChangedEventHandler` — inject provider**

Where `SwarmTelemetryCollector` is instantiated (at line 1482-1488), pass the `AgentLlmStatsProvider` if available. When no coordination topology is active, pass null (falls back to zeros).

### Testing

- **`SwarmTelemetryCollectorTest`** — extend to verify LLM fields are populated when a provider is supplied, and remain zero when provider is null.
- **`LlmTelemetryAccumulatorTest`** — new test: single event, multiple events per agent, multiple agents, zero-valued events, `getStats` for unknown agent returns ZERO.
- **`SwarmCoordinatorBuilderTest`** — verify custom `loopEventSink` is propagated to the topology.

### Thread Safety

`LlmTelemetryAccumulator` uses `ConcurrentHashMap` for thread-safe per-agent lookup. Each agent's `LlmAccumulator` is accessed from that agent's decision loop, which runs sequentially — no concurrent writes to the same accumulator instance.

### Backward Compatibility

- `SwarmTelemetryCollector` accepts null `llmStatsProvider` (falls back to zeros) — existing callers unaffected.
- `SwarmCoordinatorBuilder.loopEventSink()` defaults to no-op — existing topology construction unchanged.

### What This Does NOT Change

- `AbstractExecutionDriver` — coordination topologies bypass the driver entirely. Adding `notifyDecisionLoopEvent` to the driver is out of scope (it would only serve the standard five-phase loop path, which doesn't emit decision loop events).
- `ExecutionEventListener.onDecisionLoopEvent()` — remains available for future use but is not part of this bridge. The coordination topology path has its own event routing.

## References

- `SwarmTelemetryCollector.java:61-72` — hardcoded zeros
- `SwarmTelemetry.java:29-31` — LLM fields
- `DecisionLoopEvent.java:61-77` — `ReasoningCompleted` definition
- `DecisionLoopFactory.java:60-76` — `budgetBridge` that emits `ReasoningCompleted`
- `SwarmCoordinatorBuilder.java:132` — no-op `loopEventSink` with "wired at execution layer" comment
- `CentralisedTopology.java:47` — coordination topology that captures `loopEventSink`
- `CheckpointingListener.java:32-101` — engine-adapter bridge listener (NOT used for this bridge — different execution path)
- `PatternWorkerFunctionHandler.java:91-98` — standard path creates `CheckpointingListener` for `ExecutionModel`
- `CaseContextChangedEventHandler.java:1482-1488` — where `SwarmTelemetryCollector` is instantiated
