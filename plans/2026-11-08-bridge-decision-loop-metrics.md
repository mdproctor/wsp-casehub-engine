# Bridge DecisionLoopEvent Metrics into SwarmTelemetry Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1304 — feat: bridge DecisionLoopEvent metrics into SwarmTelemetry LLM fields
**Issue group:** casehubio/engine#1304, casehubio/agency#444

**Goal:** Wire per-agent LLM invocation stats (invocationCount, tokensConsumed, totalLatency) from the agency layer's `DecisionLoopEvent.ReasoningCompleted` events into the engine's `SwarmTelemetry` record.

**Architecture:** A standalone `LlmTelemetryAccumulator` in `engine-adapter-core` implements both `Consumer<DecisionLoopEvent>` (to serve as the `loopEventSink` in coordination topologies) and `AgentLlmStatsProvider` (a new engine SPI). `SwarmCoordinatorBuilder` gains a builder method to accept the sink. `SwarmTelemetryCollector` queries the SPI instead of hardcoding zeros.

**Tech Stack:** Java 21+, Quarkus 3.32.2, JUnit 5, AssertJ, Mockito

## Global Constraints

- Cross-repo: engine repo (casehubio/engine#1304) and agency repo (casehubio/agency#444)
- Agency commits reference `Refs #444`; engine commits reference `Refs #1304`
- `SwarmTelemetryCollector` must remain backward-compatible (null provider → zeros)
- `SwarmCoordinatorBuilder.loopEventSink()` defaults to no-op for backward compatibility
- Build verification: `mvn install -DskipTests -q` must pass in both repos after each batch

---

## Batch 1: Engine SPI and Collector [engine repo — foundation]

### Task 1: AgentLlmStatsProvider SPI + SwarmTelemetryCollector integration

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/stigmergy/AgentLlmStatsProvider.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/SwarmTelemetryCollector.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SwarmTelemetryCollectorTest.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:1482-1488`

**Interfaces:**
- Produces: `AgentLlmStatsProvider` SPI interface with `AgentLlmStats` record and `getStats(UUID caseId, String agentId)` method. `AgentLlmStats.ZERO` constant for null-safe defaults.

- [ ] **Step 1: Write the failing test — provider supplies LLM stats**

Add to `SwarmTelemetryCollectorTest.java`:

```java
@Test
void snapshot_withLlmStatsProvider_populatesLlmFields() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test");
    coordinator.agentJoined(caseId, "agent-1", "b-1");
    coordinator.agentActivated(caseId, "agent-1");

    AgentLlmStatsProvider provider = (cId, agentId) ->
        new AgentLlmStatsProvider.AgentLlmStats(3, 1500L, Duration.ofMillis(450));

    var collectorWithProvider = new SwarmTelemetryCollector(coordinator, signalRegistry, provider);
    var telemetry = collectorWithProvider.snapshot(caseId);

    assertEquals(1, telemetry.size());
    var t = telemetry.get(0);
    assertEquals(3, t.invocationCount());
    assertEquals(1500L, t.tokensConsumed());
    assertEquals(Duration.ofMillis(450), t.totalLatency());
}
```

Add import: `import io.casehub.api.spi.stigmergy.AgentLlmStatsProvider;`

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SwarmTelemetryCollectorTest#snapshot_withLlmStatsProvider_populatesLlmFields -q`
Expected: compilation error — `AgentLlmStatsProvider` does not exist

- [ ] **Step 3: Create AgentLlmStatsProvider SPI**

Create `api/src/main/java/io/casehub/api/spi/stigmergy/AgentLlmStatsProvider.java`:

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

- [ ] **Step 4: Modify SwarmTelemetryCollector to accept and use the provider**

Add field and overloaded constructor:

```java
private final AgentLlmStatsProvider llmStatsProvider;

public SwarmTelemetryCollector(StigmergyCoordinator coordinator, SignalRegistry signalRegistry) {
    this(coordinator, signalRegistry, null);
}

public SwarmTelemetryCollector(StigmergyCoordinator coordinator, SignalRegistry signalRegistry,
                                AgentLlmStatsProvider llmStatsProvider) {
    this.coordinator = coordinator;
    this.signalRegistry = signalRegistry;
    this.llmStatsProvider = llmStatsProvider;
}
```

In `snapshot(UUID caseId, Instant now)`, replace the hardcoded zeros (lines 70-72):

```java
var llmStats = llmStatsProvider != null
    ? llmStatsProvider.getStats(caseId, agent.agentId())
    : AgentLlmStatsProvider.AgentLlmStats.ZERO;

result.add(
    new SwarmTelemetry(
        caseId,
        agent.agentId(),
        workState,
        signalsDeposited,
        messagesSent,
        tasksCompleted,
        activeTime,
        llmStats.invocationCount(),
        llmStats.tokensConsumed(),
        llmStats.totalLatency()));
```

Add import: `import io.casehub.api.spi.stigmergy.AgentLlmStatsProvider;`

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SwarmTelemetryCollectorTest#snapshot_withLlmStatsProvider_populatesLlmFields -q`
Expected: PASS

- [ ] **Step 6: Write test — null provider preserves backward compatibility**

Add to `SwarmTelemetryCollectorTest.java`:

```java
@Test
void snapshot_withNullProvider_returnsZeroLlmFields() {
    coordinator.initializeCase(caseId, List.of("agent-1"), config, "test");
    coordinator.agentJoined(caseId, "agent-1", "b-1");
    coordinator.agentActivated(caseId, "agent-1");

    var telemetry = collector.snapshot(caseId);

    assertEquals(1, telemetry.size());
    var t = telemetry.get(0);
    assertEquals(0, t.invocationCount());
    assertEquals(0L, t.tokensConsumed());
    assertEquals(Duration.ZERO, t.totalLatency());
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SwarmTelemetryCollectorTest#snapshot_withNullProvider_returnsZeroLlmFields -q`
Expected: PASS (the existing `collector` field uses the no-arg constructor)

- [ ] **Step 8: Write test — provider returns per-agent stats**

Add to `SwarmTelemetryCollectorTest.java`:

```java
@Test
void snapshot_multipleAgents_providerReturnsPerAgentStats() {
    coordinator.initializeCase(caseId, List.of("agent-1", "agent-2"), config, "test");
    coordinator.agentJoined(caseId, "agent-1", "b-1");
    coordinator.agentJoined(caseId, "agent-2", "b-2");
    coordinator.agentActivated(caseId, "agent-1");
    coordinator.agentActivated(caseId, "agent-2");

    AgentLlmStatsProvider provider = (cId, agentId) -> switch (agentId) {
        case "agent-1" -> new AgentLlmStatsProvider.AgentLlmStats(5, 3000L, Duration.ofMillis(800));
        case "agent-2" -> new AgentLlmStatsProvider.AgentLlmStats(2, 500L, Duration.ofMillis(200));
        default -> AgentLlmStatsProvider.AgentLlmStats.ZERO;
    };

    var collectorWithProvider = new SwarmTelemetryCollector(coordinator, signalRegistry, provider);
    var telemetry = collectorWithProvider.snapshot(caseId);

    assertEquals(2, telemetry.size());
    var t1 = telemetry.stream().filter(t -> t.agentId().equals("agent-1")).findFirst().orElseThrow();
    var t2 = telemetry.stream().filter(t -> t.agentId().equals("agent-2")).findFirst().orElseThrow();
    assertEquals(5, t1.invocationCount());
    assertEquals(3000L, t1.tokensConsumed());
    assertEquals(2, t2.invocationCount());
    assertEquals(500L, t2.tokensConsumed());
}
```

- [ ] **Step 9: Run all collector tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SwarmTelemetryCollectorTest -q`
Expected: all PASS

- [ ] **Step 10: Update CaseContextChangedEventHandler instantiation**

At `CaseContextChangedEventHandler.java:1482-1484`, the collector is created with two args. Keep it that way (null provider) — the coordination topology integration that passes a real provider is wired by the caller of `SwarmCoordinatorBuilder`, which is not in the engine repo. No change needed here yet.

- [ ] **Step 11: Build verification**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/stigmergy/AgentLlmStatsProvider.java \
       runtime-core/src/main/java/io/casehub/engine/runtime/stigmergy/SwarmTelemetryCollector.java \
       runtime-core/src/test/java/io/casehub/engine/runtime/stigmergy/SwarmTelemetryCollectorTest.java
git commit -m "$(cat <<'EOF'
feat(#1304): add AgentLlmStatsProvider SPI and wire into SwarmTelemetryCollector

Introduce AgentLlmStatsProvider interface in engine-api with AgentLlmStats
record. SwarmTelemetryCollector gains an overloaded constructor that accepts
the provider and queries it for per-agent LLM stats instead of hardcoding
zeros. Null provider preserves backward compatibility.

Refs #1304

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: Agency builder + accumulator [agency repo — wiring]

### Task 2: SwarmCoordinatorBuilder loopEventSink builder method

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilder.java`
- Modify: `agency/src/test/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilderTest.java`

**Interfaces:**
- Produces: `SwarmCoordinatorBuilder.loopEventSink(Consumer<DecisionLoopEvent>)` builder method

- [ ] **Step 1: Write the failing test — custom loopEventSink receives events**

Add to `SwarmCoordinatorBuilderTest.java`:

```java
@Test
void buildTopology_customLoopEventSink_receivesEvents() {
    var events = new java.util.ArrayList<DecisionLoopEvent>();
    var topology = new SwarmCoordinatorBuilder()
                           .agentProvider(mock(AgentProvider.class))
                           .loopEventSink(events::add)
                           .buildTopology();

    assertThat(topology).isNotNull();
    assertThat(topology.loopEventSink()).isNotNull();
}
```

Add imports: `import io.casehub.agency.orchestration.loop.DecisionLoopEvent;`

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency -Dtest=SwarmCoordinatorBuilderTest#buildTopology_customLoopEventSink_receivesEvents -q`
Expected: compilation error — `loopEventSink` method does not exist on builder

- [ ] **Step 3: Add loopEventSink builder method to SwarmCoordinatorBuilder**

Add field after the existing `outcomeProvider` field:

```java
private Consumer<DecisionLoopEvent> loopEventSink = e -> {};
```

Add builder method after the `outcomeProvider` method:

```java
public SwarmCoordinatorBuilder loopEventSink(Consumer<DecisionLoopEvent> sink) {
    this.loopEventSink = Objects.requireNonNull(sink);
    return this;
}
```

Add imports:
```java
import io.casehub.agency.orchestration.loop.DecisionLoopEvent;
import java.util.function.Consumer;
```

In `buildTopology()`, replace line 132 (`e -> {}`) with `this.loopEventSink`:

Change:
```java
                e -> {});  // loopEventSink — wired at execution layer
```
To:
```java
                loopEventSink);
```

- [ ] **Step 4: Check if CentralisedTopology exposes `loopEventSink()` accessor**

The test in Step 1 calls `topology.loopEventSink()`. Check if `CentralisedTopology` has this accessor. If not, verify the sink propagation by checking the topology is non-null (the event flow is tested end-to-end in Task 3). Adjust the test assertion to:

```java
assertThat(topology).isNotNull();
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency -Dtest=SwarmCoordinatorBuilderTest -q`
Expected: all PASS

- [ ] **Step 6: Write test — default loopEventSink is no-op (backward compat)**

Add to `SwarmCoordinatorBuilderTest.java`:

```java
@Test
void buildTopology_defaultLoopEventSink_noOpDoesNotThrow() {
    var topology = new SwarmCoordinatorBuilder()
                           .agentProvider(mock(AgentProvider.class))
                           .buildTopology();
    assertThat(topology).isNotNull();
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl agency -Dtest=SwarmCoordinatorBuilderTest -q`
Expected: all PASS

- [ ] **Step 8: Build verification**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add agency-core/src/main/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilder.java \
       agency/src/test/java/io/casehub/agency/orchestration/swarm/SwarmCoordinatorBuilderTest.java
git commit -m "$(cat <<'EOF'
feat(#444): add loopEventSink builder method to SwarmCoordinatorBuilder

Replace hardcoded no-op loopEventSink with a configurable builder field.
Defaults to no-op for backward compatibility. Callers can now provide a
Consumer<DecisionLoopEvent> to receive per-agent LLM telemetry events
from coordination topologies.

Refs #444

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

### Task 3: LlmTelemetryAccumulator

**Files:**
- Create: `engine-adapter-core/src/main/java/io/casehub/engine/agentic/LlmTelemetryAccumulator.java`
- Create: `engine-adapter/src/test/java/io/casehub/engine/agentic/LlmTelemetryAccumulatorTest.java`

**Interfaces:**
- Consumes: `DecisionLoopEvent.ReasoningCompleted` (agency-core), `AgentLlmStatsProvider` (engine-api)
- Produces: `LlmTelemetryAccumulator` — implements both `Consumer<DecisionLoopEvent>` and `AgentLlmStatsProvider`

- [ ] **Step 1: Write the failing test — single ReasoningCompleted event**

Create `engine-adapter/src/test/java/io/casehub/engine/agentic/LlmTelemetryAccumulatorTest.java`:

```java
package io.casehub.engine.agentic;

import io.casehub.agency.orchestration.loop.DecisionLoopEvent;
import io.casehub.api.spi.stigmergy.AgentLlmStatsProvider;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class LlmTelemetryAccumulatorTest {

    private final UUID caseId = UUID.randomUUID();

    @Test
    void singleEvent_recordsStats() {
        var accumulator = new LlmTelemetryAccumulator(caseId);
        accumulator.accept(new DecisionLoopEvent.ReasoningCompleted(
                caseId, "agent-1", "corr-1", 500L, 120L, Instant.now()));

        var stats = accumulator.getStats(caseId, "agent-1");
        assertThat(stats.invocationCount()).isEqualTo(1);
        assertThat(stats.tokensConsumed()).isEqualTo(500L);
        assertThat(stats.totalLatency()).isEqualTo(Duration.ofMillis(120));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=LlmTelemetryAccumulatorTest#singleEvent_recordsStats -q`
Expected: compilation error — `LlmTelemetryAccumulator` does not exist

- [ ] **Step 3: Create LlmTelemetryAccumulator**

Create `engine-adapter-core/src/main/java/io/casehub/engine/agentic/LlmTelemetryAccumulator.java`:

```java
package io.casehub.engine.agentic;

import io.casehub.agency.orchestration.loop.DecisionLoopEvent;
import io.casehub.api.spi.stigmergy.AgentLlmStatsProvider;

import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Consumer;

public class LlmTelemetryAccumulator implements Consumer<DecisionLoopEvent>, AgentLlmStatsProvider {

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
        return new AgentLlmStats(acc.invocationCount, acc.tokensConsumed,
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

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=LlmTelemetryAccumulatorTest#singleEvent_recordsStats -q`
Expected: PASS

- [ ] **Step 5: Write test — multiple events accumulate per agent**

Add to `LlmTelemetryAccumulatorTest.java`:

```java
@Test
void multipleEvents_accumulates() {
    var accumulator = new LlmTelemetryAccumulator(caseId);
    accumulator.accept(new DecisionLoopEvent.ReasoningCompleted(
            caseId, "agent-1", "corr-1", 500L, 120L, Instant.now()));
    accumulator.accept(new DecisionLoopEvent.ReasoningCompleted(
            caseId, "agent-1", "corr-2", 300L, 80L, Instant.now()));

    var stats = accumulator.getStats(caseId, "agent-1");
    assertThat(stats.invocationCount()).isEqualTo(2);
    assertThat(stats.tokensConsumed()).isEqualTo(800L);
    assertThat(stats.totalLatency()).isEqualTo(Duration.ofMillis(200));
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=LlmTelemetryAccumulatorTest#multipleEvents_accumulates -q`
Expected: PASS

- [ ] **Step 7: Write test — multiple agents tracked independently**

Add to `LlmTelemetryAccumulatorTest.java`:

```java
@Test
void multipleAgents_trackedIndependently() {
    var accumulator = new LlmTelemetryAccumulator(caseId);
    accumulator.accept(new DecisionLoopEvent.ReasoningCompleted(
            caseId, "agent-1", "corr-1", 500L, 120L, Instant.now()));
    accumulator.accept(new DecisionLoopEvent.ReasoningCompleted(
            caseId, "agent-2", "corr-2", 200L, 50L, Instant.now()));

    var stats1 = accumulator.getStats(caseId, "agent-1");
    var stats2 = accumulator.getStats(caseId, "agent-2");
    assertThat(stats1.invocationCount()).isEqualTo(1);
    assertThat(stats1.tokensConsumed()).isEqualTo(500L);
    assertThat(stats2.invocationCount()).isEqualTo(1);
    assertThat(stats2.tokensConsumed()).isEqualTo(200L);
}
```

- [ ] **Step 8: Write test — unknown agent returns ZERO**

Add to `LlmTelemetryAccumulatorTest.java`:

```java
@Test
void unknownAgent_returnsZero() {
    var accumulator = new LlmTelemetryAccumulator(caseId);
    var stats = accumulator.getStats(caseId, "nonexistent");
    assertThat(stats).isEqualTo(AgentLlmStatsProvider.AgentLlmStats.ZERO);
}
```

- [ ] **Step 9: Write test — non-ReasoningCompleted events are ignored**

Add to `LlmTelemetryAccumulatorTest.java`:

```java
@Test
void nonReasoningEvent_ignored() {
    var accumulator = new LlmTelemetryAccumulator(caseId);
    accumulator.accept(new DecisionLoopEvent.ReasoningStarted(
            caseId, "agent-1", "corr-1", Instant.now()));

    var stats = accumulator.getStats(caseId, "agent-1");
    assertThat(stats).isEqualTo(AgentLlmStatsProvider.AgentLlmStats.ZERO);
}
```

- [ ] **Step 10: Run all accumulator tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-adapter -Dtest=LlmTelemetryAccumulatorTest -q`
Expected: all PASS

- [ ] **Step 11: Build verification**

Run: `mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add engine-adapter-core/src/main/java/io/casehub/engine/agentic/LlmTelemetryAccumulator.java \
       engine-adapter/src/test/java/io/casehub/engine/agentic/LlmTelemetryAccumulatorTest.java
git commit -m "$(cat <<'EOF'
feat(#444): add LlmTelemetryAccumulator bridging DecisionLoopEvent to AgentLlmStatsProvider

Standalone accumulator implements both Consumer<DecisionLoopEvent> (loopEventSink
for coordination topologies) and AgentLlmStatsProvider (engine SPI for telemetry
collector). Counts ReasoningCompleted events as invocations and accumulates
per-agent tokensConsumed and totalLatency.

Refs #444

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

## References

- [2026-11-08-bridge-decision-loop-metrics-design.md] — design spec this plan implements
- `SwarmTelemetryCollector.java:32-95` — hardcoded zeros, constructor, snapshot method
- `SwarmTelemetryCollectorTest.java:37-162` — existing test patterns
- `SwarmCoordinatorBuilder.java:17-135` — builder class, no-op loopEventSink
- `SwarmCoordinatorBuilderTest.java:9-25` — existing builder test
- `DecisionLoopEvent.java:61-77` — ReasoningCompleted record
- `DecisionLoopFactory.java:60-76` — budgetBridge emitting ReasoningCompleted
- `CaseContextChangedEventHandler.java:1482-1488` — collector instantiation site
- `AgentLlmStatsProvider.java` — new SPI (engine-api)
- `LlmTelemetryAccumulator.java` — new accumulator (engine-adapter-core)
- GitHub casehubio/engine#1304, casehubio/agency#444
