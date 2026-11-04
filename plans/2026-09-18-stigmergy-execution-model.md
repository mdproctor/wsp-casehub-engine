# Stigmergy Execution Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1111 — feat: Stigmergy execution model — indirect coordination via shared environment
**Issue group:** #1105, #1106, #1107, #1108, #1109, #1110, #1111

**Goal:** Compose the six foundation SPIs into a coherent stigmergy execution model with three layers: StigmergyConfig (configuration preset), StigmergyStrategy (dispatch), StigmergyCoordinator (lifecycle + coordination intelligence).

**Architecture:** Three-layer blend over existing SPIs. StigmergyConfig (api) provides coordinated defaults. StigmergyStrategy (planning-core) handles first-cycle agent dispatch as a named PlanningStrategy. StigmergyCoordinator (runtime-core) tracks agent lifecycle (JOINING→ACTIVE→DEPARTED), detects coordination patterns (signal consensus, coordination storms, interest convergence), and provides population queries. Agents use existing WorkerRuntime facets (signals, interests, neighbors, rules); the coordinator observes their aggregate behavior without adding new facets.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (`@ApplicationScoped`), ConcurrentHashMap (in-memory state)

## Global Constraints

- All new types follow the existing Apache 2.0 license header
- Records are preferred for value types; classes for mutable state
- `@ApplicationScoped` + `Resettable` for all per-case registries
- Constructor injection for CDI dependencies; `Instance<>` with `isResolvable()` for optional dependencies
- Package `io.casehub.api.model.stigmergy` for API types, `io.casehub.engine.internal.stigmergy` for coordinator, `io.casehub.engine.plan.strategy` for strategy (existing package)
- Every commit references `Refs casehubio/engine#1111`

---

## Batch 1: API Types + Foundation

### Task 1: Foundation types — config records, event types, RuleAction.Leave, WorkerRuntime.leave(), SignalRegistry.consensusSignals()

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/StigmergyConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/StigmergyDefaults.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/CoordinationConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/AgentLifecycleState.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add 7 stigmergy event types
- Modify: `api/src/main/java/io/casehub/api/spi/observation/RuleAction.java` — add `Leave` permit
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` — add `default void leave()`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `stigmergyConfig` field + getter/setter + builder method
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java` — add `consensusSignals()` method
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/StigmergyConfigTest.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryConsensusTest.java`

**Interfaces:**
- Produces: `StigmergyConfig(StigmergyDefaults, CoordinationConfig)` — consumed by Task 2 (coordinator), Task 4 (strategy)
- Produces: `AgentLifecycleState` enum (`JOINING`, `ACTIVE`, `DEPARTED`) — consumed by Task 2
- Produces: `AgentState` record — consumed by Task 2
- Produces: `RuleAction.Leave` — consumed by Task 3 (LocalRuleEvaluator wiring)
- Produces: `WorkerRuntime.leave()` default method — consumed by Task 3 (DefaultWorkerRuntime)
- Produces: `CaseHubEventType.STIGMERGY_*` — consumed by Task 2, Task 3
- Produces: `SignalRegistry.consensusSignals(UUID, int, double)` — consumed by Task 2 (coordinator consensus detection)
- Produces: `CaseDefinition.getStigmergyConfig()` / `setStigmergyConfig()` — consumed by Task 3, Task 4

- [ ] **Step 1: Write StigmergyDefaults record test**

```java
// api/src/test/java/io/casehub/api/model/stigmergy/StigmergyConfigTest.java
package io.casehub.api.model.stigmergy;

import static org.junit.jupiter.api.Assertions.*;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class StigmergyConfigTest {

    @Test
    void defaultsRecordStoresAllFields() {
        var defaults = new StigmergyDefaults(
                Duration.ofMinutes(5), 0.01, 100, 20, 50,
                Duration.ofSeconds(60), Duration.ofSeconds(30),
                10000, 50000);
        assertEquals(Duration.ofMinutes(5), defaults.signalHalfLife());
        assertEquals(0.01, defaults.effectiveZeroThreshold());
        assertEquals(100, defaults.maxSignalsPerCase());
        assertEquals(20, defaults.maxObserversPerCase());
        assertEquals(50, defaults.maxRulesPerCase());
        assertEquals(Duration.ofSeconds(60), defaults.rateWindow());
        assertEquals(Duration.ofSeconds(30), defaults.stabilityWindow());
        assertEquals(10000, defaults.maxDispatches());
        assertEquals(50000, defaults.maxEvaluationCycles());
    }

    @Test
    void allNullableFieldsAllowNull() {
        var defaults = new StigmergyDefaults(null, null, null, null, null, null, null, null, null);
        assertNull(defaults.signalHalfLife());
    }

    @Test
    void coordinationConfigHasDefaults() {
        var coord = new CoordinationConfig(2, 10.0, 0.6);
        assertEquals(2, coord.consensusThreshold());
        assertEquals(10.0, coord.stormRateMultiplier());
        assertEquals(0.6, coord.interestHotspotThreshold());
    }

    @Test
    void stigmergyConfigComposesSubRecords() {
        var defaults = new StigmergyDefaults(null, null, null, null, null, null, null, null, null);
        var coord = new CoordinationConfig(null, null, null);
        var config = new StigmergyConfig(defaults, coord);
        assertNotNull(config.defaults());
        assertNotNull(config.coordination());
    }

    @Test
    void stigmergyConfigAllowsNullSubRecords() {
        var config = new StigmergyConfig(null, null);
        assertNull(config.defaults());
        assertNull(config.coordination());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=StigmergyConfigTest -q`
Expected: FAIL — classes do not exist yet

- [ ] **Step 3: Create config records**

Use `ide_create_file` for each:

```java
// api/src/main/java/io/casehub/api/model/stigmergy/StigmergyDefaults.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.time.Duration;

public record StigmergyDefaults(
    @Nullable Duration signalHalfLife,
    @Nullable Double effectiveZeroThreshold,
    @Nullable Integer maxSignalsPerCase,
    @Nullable Integer maxObserversPerCase,
    @Nullable Integer maxRulesPerCase,
    @Nullable Duration rateWindow,
    @Nullable Duration stabilityWindow,
    @Nullable Integer maxDispatches,
    @Nullable Integer maxEvaluationCycles) {}
```

```java
// api/src/main/java/io/casehub/api/model/stigmergy/CoordinationConfig.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record CoordinationConfig(
    @Nullable Integer consensusThreshold,
    @Nullable Double stormRateMultiplier,
    @Nullable Double interestHotspotThreshold) {}
```

```java
// api/src/main/java/io/casehub/api/model/stigmergy/StigmergyConfig.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record StigmergyConfig(
    @Nullable StigmergyDefaults defaults,
    @Nullable CoordinationConfig coordination) {}
```

```java
// api/src/main/java/io/casehub/api/model/stigmergy/AgentLifecycleState.java
package io.casehub.api.model.stigmergy;

public enum AgentLifecycleState {
    JOINING,
    ACTIVE,
    DEPARTED
}
```

```java
// api/src/main/java/io/casehub/api/model/stigmergy/AgentState.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.time.Instant;

public record AgentState(
    String agentId,
    String bindingName,
    AgentLifecycleState state,
    Instant joinedAt,
    @Nullable Instant activatedAt,
    @Nullable Instant departedAt,
    Instant lastActivity) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=StigmergyConfigTest -q`
Expected: PASS

- [ ] **Step 5: Add 7 CaseHubEventType values**

Use `ide_edit_member` on `CaseHubEventType` enum to add before the closing brace. The last existing value is `OUTPUT_CONVERGENCE_DETECTED` (no trailing comma). Add a comma after it, then the new values:

```java
  OUTPUT_CONVERGENCE_DETECTED, // per-binding output structural similarity exceeds threshold

  STIGMERGY_CASE_INITIALIZED,
  STIGMERGY_AGENT_JOINED,
  STIGMERGY_AGENT_ACTIVATED,
  STIGMERGY_AGENT_DEPARTED,
  SIGNAL_CONSENSUS_DETECTED,
  COORDINATION_STORM_DETECTED,
  INTEREST_CONVERGENCE_DETECTED
```

- [ ] **Step 6: Add RuleAction.Leave permit**

Use `ide_insert_member` to add after `WriteContext` in `RuleAction.java`:

```java
  record Leave() implements RuleAction {}
```

- [ ] **Step 7: Add WorkerRuntime.leave() default method**

Use `ide_insert_member` to add at the end of `WorkerRuntime.java` (after `rules()`):

```java
  default void leave() {}
```

- [ ] **Step 8: Add stigmergyConfig field to CaseDefinition**

Use `ide_insert_member` to add after the `compounds` field (line 276):

Field:
```java
  private StigmergyConfig stigmergyConfig;
```

Getter (after `setCompounds`):
```java
  public StigmergyConfig getStigmergyConfig() {
      return stigmergyConfig;
  }
```

Setter:
```java
  public void setStigmergyConfig(StigmergyConfig stigmergyConfig) {
      this.stigmergyConfig = stigmergyConfig;
  }
```

Builder field (add to Builder class):
```java
  private StigmergyConfig stigmergyConfig;
```

Builder method:
```java
  public Builder stigmergyConfig(StigmergyConfig stigmergyConfig) {
      this.stigmergyConfig = stigmergyConfig;
      return this;
  }
```

In `Builder.build()`, add after compounds assignment:
```java
  def.setStigmergyConfig(this.stigmergyConfig);
```

- [ ] **Step 9: Write SignalRegistry.consensusSignals() test**

```java
// common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryConsensusTest.java
package io.casehub.engine.common.internal.signal;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.signal.Signal;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SignalRegistryConsensusTest {

    private SignalRegistry registry;
    private UUID caseId;

    @BeforeEach
    void setUp() {
        registry = new SignalRegistry();
        caseId = UUID.randomUUID();
    }

    @Test
    void returnsEmptyWhenNoSignals() {
        Map<String, Signal> result = registry.consensusSignals(caseId, 2, 0.01);
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsEmptyWhenNoSignalMeetsThreshold() {
        registry.deposit(caseId, "lonely", 0.8, Duration.ofMinutes(5), "agent-1", 100);
        Map<String, Signal> result = registry.consensusSignals(caseId, 2, 0.01);
        assertTrue(result.isEmpty());
    }

    @Test
    void returnsSignalWhenSourceCountMeetsThreshold() {
        registry.deposit(caseId, "consensus-signal", 0.8, Duration.ofMinutes(5), "agent-1", 100);
        registry.deposit(caseId, "consensus-signal", 0.9, Duration.ofMinutes(5), "agent-2", 100);
        Map<String, Signal> result = registry.consensusSignals(caseId, 2, 0.01);
        assertEquals(1, result.size());
        assertTrue(result.containsKey("consensus-signal"));
        assertEquals(2, result.get("consensus-signal").sources().size());
    }

    @Test
    void excludesExpiredSignals() {
        Instant past = Instant.now().minusSeconds(600);
        registry.deposit(caseId, "old", 0.5, Duration.ofSeconds(1), "agent-1", 100, past);
        registry.deposit(caseId, "old", 0.5, Duration.ofSeconds(1), "agent-2", 100, past);
        Map<String, Signal> result = registry.consensusSignals(caseId, 2, 0.01);
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 10: Run consensus test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest=SignalRegistryConsensusTest -q`
Expected: FAIL — method does not exist

- [ ] **Step 11: Implement SignalRegistry.consensusSignals()**

Use `ide_insert_member` to add to `SignalRegistry` after `getAllSignals()`:

```java
  public Map<String, Signal> consensusSignals(UUID caseId, int minSources, double effectiveZeroThreshold) {
      return consensusSignals(caseId, minSources, effectiveZeroThreshold, Instant.now());
  }

  public Map<String, Signal> consensusSignals(UUID caseId, int minSources, double effectiveZeroThreshold, Instant now) {
      ConcurrentHashMap<String, Signal> caseSignals = signals.get(caseId);
      if (caseSignals == null) return Map.of();
      Map<String, Signal> result = new LinkedHashMap<>();
      for (Signal signal : caseSignals.values()) {
          if (signal.expired()) continue;
          double effective = SignalDecay.effectiveStrength(
              signal.strength(), signal.lastReinforced(), signal.halfLife(), now);
          if (effective < effectiveZeroThreshold) continue;
          if (signal.sources().size() >= minSources) {
              result.put(signal.name(), signal);
          }
      }
      return Collections.unmodifiableMap(result);
  }
```

- [ ] **Step 12: Run consensus test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl common-core -Dtest=SignalRegistryConsensusTest -q`
Expected: PASS

- [ ] **Step 13: Run full api + common-core test suites**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common-core -q`
Expected: All tests pass

- [ ] **Step 14: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/ \
       api/src/test/java/io/casehub/api/model/stigmergy/ \
       api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java \
       api/src/main/java/io/casehub/api/spi/observation/RuleAction.java \
       api/src/main/java/io/casehub/api/engine/WorkerRuntime.java \
       api/src/main/java/io/casehub/api/model/CaseDefinition.java \
       common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java \
       common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryConsensusTest.java
git commit -m "feat: add stigmergy foundation types — config records, event types, RuleAction.Leave, consensusSignals

Refs casehubio/engine#1111"
```

---

## Batch 2: Coordinator + Runtime Wiring

### Task 2: StigmergyCoordinator — lifecycle tracking + coordination pattern detection

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/StigmergyCoordinator.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/StigmergyCoordinatorTest.java`

**Interfaces:**
- Consumes: `AgentLifecycleState`, `AgentState` (from Task 1)
- Consumes: `StigmergyConfig`, `CoordinationConfig` (from Task 1)
- Consumes: `CaseHubEventType.STIGMERGY_*` (from Task 1)
- Consumes: `SignalRegistry.consensusSignals(UUID, int, double)` (from Task 1)
- Consumes: `ObservationRegistry.getObservers(UUID)` — existing (returns `Map<String, List<EnvironmentObserver>>`)
- Consumes: `ActivityTracker.getState(UUID)` — existing (returns `CaseActivityState`)
- Produces: `StigmergyCoordinator.initializeCase(UUID, List<String>, StigmergyConfig)` — consumed by Task 4 (strategy)
- Produces: `StigmergyCoordinator.agentJoined(UUID, String, String)` — consumed by Task 3 (wiring)
- Produces: `StigmergyCoordinator.agentActivated(UUID, String)` — consumed by Task 3 (wiring)
- Produces: `StigmergyCoordinator.agentDeparted(UUID, String)` — consumed by Task 3 (wiring)
- Produces: `StigmergyCoordinator.isStigmergyCase(UUID)` — consumed by Task 3, Task 4
- Produces: `StigmergyCoordinator.activeAgents(UUID)`, `activeCount(UUID)`, `allActive(UUID)` — consumed by Task 4
- Produces: `StigmergyCoordinator.detectPatterns(UUID, StigmergyConfig)` — consumed by Task 3 (handler wiring)
- Produces: `StigmergyCoordinator.evictByCase(UUID)` — consumed by Task 3 (status handler)

- [ ] **Step 1: Write StigmergyCoordinator test — lifecycle tracking**

```java
// runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/StigmergyCoordinatorTest.java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.stigmergy.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class StigmergyCoordinatorTest {

    private StigmergyCoordinator coordinator;
    private SignalRegistry signalRegistry;
    private ObservationRegistry observationRegistry;
    private ActivityTracker activityTracker;
    private UUID caseId;
    private StigmergyConfig config;

    @BeforeEach
    void setUp() {
        signalRegistry = new SignalRegistry();
        observationRegistry = new ObservationRegistry();
        activityTracker = new ActivityTracker();
        coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
        caseId = UUID.randomUUID();
        config = new StigmergyConfig(null, new CoordinationConfig(2, 10.0, 0.6));
    }

    @Test
    void notStigmergyCaseBeforeInit() {
        assertFalse(coordinator.isStigmergyCase(caseId));
    }

    @Test
    void isStigmergyCaseAfterInit() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        assertTrue(coordinator.isStigmergyCase(caseId));
    }

    @Test
    void agentJoinedTransitionsToJoining() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        coordinator.agentJoined(caseId, "agent-1", "binding-1");
        var agents = coordinator.activeAgents(caseId);
        assertTrue(agents.isEmpty());
        var agent = coordinator.getAgent(caseId, "agent-1");
        assertNotNull(agent);
        assertEquals(AgentLifecycleState.JOINING, agent.state());
    }

    @Test
    void agentActivatedTransitionsToActive() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        coordinator.agentJoined(caseId, "agent-1", "binding-1");
        coordinator.agentActivated(caseId, "agent-1");
        var agents = coordinator.activeAgents(caseId);
        assertEquals(1, agents.size());
        assertEquals(AgentLifecycleState.ACTIVE, agents.get(0).state());
    }

    @Test
    void agentDepartedTransitionsToDeparted() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        coordinator.agentJoined(caseId, "agent-1", "binding-1");
        coordinator.agentActivated(caseId, "agent-1");
        coordinator.agentDeparted(caseId, "agent-1");
        var agents = coordinator.activeAgents(caseId);
        assertTrue(agents.isEmpty());
        var agent = coordinator.getAgent(caseId, "agent-1");
        assertEquals(AgentLifecycleState.DEPARTED, agent.state());
    }

    @Test
    void allActiveReturnsTrueWhenAllPastJoining() {
        coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
        coordinator.agentJoined(caseId, "a1", "b1");
        coordinator.agentJoined(caseId, "a2", "b2");
        assertFalse(coordinator.allActive(caseId));
        coordinator.agentActivated(caseId, "a1");
        assertFalse(coordinator.allActive(caseId));
        coordinator.agentActivated(caseId, "a2");
        assertTrue(coordinator.allActive(caseId));
    }

    @Test
    void evictByCaseClearsAll() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        coordinator.agentJoined(caseId, "agent-1", "binding-1");
        coordinator.evictByCase(caseId);
        assertFalse(coordinator.isStigmergyCase(caseId));
    }

    @Test
    void resetClearsAll() {
        coordinator.initializeCase(caseId, List.of("agent-1"), config);
        coordinator.reset();
        assertFalse(coordinator.isStigmergyCase(caseId));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement StigmergyCoordinator**

Use `ide_create_file`:

```java
// runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/StigmergyCoordinator.java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.signal.Signal;
import io.casehub.api.model.signal.SignalDecay;
import io.casehub.api.model.stigmergy.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class StigmergyCoordinator implements Resettable {

    private final SignalRegistry signalRegistry;
    private final ObservationRegistry observationRegistry;
    private final ActivityTracker activityTracker;

    private final ConcurrentHashMap<UUID, CaseCoordinationState> cases = new ConcurrentHashMap<>();

    @Inject
    public StigmergyCoordinator(SignalRegistry signalRegistry,
                                 ObservationRegistry observationRegistry,
                                 ActivityTracker activityTracker) {
        this.signalRegistry = signalRegistry;
        this.observationRegistry = observationRegistry;
        this.activityTracker = activityTracker;
    }

    public void initializeCase(UUID caseId, List<String> agentIds, StigmergyConfig config) {
        cases.put(caseId, new CaseCoordinationState(config, agentIds));
    }

    public boolean isStigmergyCase(UUID caseId) {
        return cases.containsKey(caseId);
    }

    public void agentJoined(UUID caseId, String agentId, String bindingName) {
        var state = cases.get(caseId);
        if (state == null) return;
        var now = Instant.now();
        state.agents.put(agentId, new AgentState(
                agentId, bindingName, AgentLifecycleState.JOINING, now, null, null, now));
    }

    public void agentActivated(UUID caseId, String agentId) {
        var state = cases.get(caseId);
        if (state == null) return;
        var current = state.agents.get(agentId);
        if (current == null || current.state() != AgentLifecycleState.JOINING) return;
        var now = Instant.now();
        state.agents.put(agentId, new AgentState(
                agentId, current.bindingName(), AgentLifecycleState.ACTIVE,
                current.joinedAt(), now, null, now));
    }

    public void agentDeparted(UUID caseId, String agentId) {
        var state = cases.get(caseId);
        if (state == null) return;
        var current = state.agents.get(agentId);
        if (current == null || current.state() == AgentLifecycleState.DEPARTED) return;
        var now = Instant.now();
        state.agents.put(agentId, new AgentState(
                agentId, current.bindingName(), AgentLifecycleState.DEPARTED,
                current.joinedAt(), current.activatedAt(), now, now));
    }

    public AgentState getAgent(UUID caseId, String agentId) {
        var state = cases.get(caseId);
        return state != null ? state.agents.get(agentId) : null;
    }

    public List<AgentState> activeAgents(UUID caseId) {
        var state = cases.get(caseId);
        if (state == null) return List.of();
        return state.agents.values().stream()
                .filter(a -> a.state() == AgentLifecycleState.ACTIVE)
                .toList();
    }

    public int activeCount(UUID caseId) {
        return activeAgents(caseId).size();
    }

    public boolean allActive(UUID caseId) {
        var state = cases.get(caseId);
        if (state == null) return false;
        if (state.agents.isEmpty()) return false;
        return state.agents.values().stream()
                .noneMatch(a -> a.state() == AgentLifecycleState.JOINING);
    }

    public List<CoordinationEvent> detectPatterns(UUID caseId, StigmergyConfig config) {
        var state = cases.get(caseId);
        if (state == null) return List.of();
        var coord = config != null && config.coordination() != null
                ? config.coordination()
                : new CoordinationConfig(2, 10.0, 0.6);
        List<CoordinationEvent> events = new ArrayList<>();
        detectSignalConsensus(caseId, coord, state, events);
        detectCoordinationStorm(caseId, coord, state, events);
        detectInterestConvergence(caseId, coord, state, events);
        return events;
    }

    private void detectSignalConsensus(UUID caseId, CoordinationConfig coord,
                                        CaseCoordinationState state, List<CoordinationEvent> events) {
        int threshold = coord.consensusThreshold() != null ? coord.consensusThreshold() : 2;
        double ezThreshold = 0.01;
        var consensus = signalRegistry.consensusSignals(caseId, threshold, ezThreshold);
        for (var entry : consensus.entrySet()) {
            if (state.consensusFired.add(entry.getKey())) {
                var signal = entry.getValue();
                double effective = SignalDecay.effectiveStrength(
                        signal.strength(), signal.lastReinforced(), signal.halfLife(), Instant.now());
                events.add(new CoordinationEvent(
                        CoordinationEvent.Type.SIGNAL_CONSENSUS,
                        Map.of("signalName", signal.name(),
                               "reinforcementCount", signal.reinforcementCount(),
                               "sources", signal.sources(),
                               "effectiveStrength", effective)));
            }
        }
        state.consensusFired.retainAll(consensus.keySet());
    }

    private void detectCoordinationStorm(UUID caseId, CoordinationConfig coord,
                                          CaseCoordinationState state, List<CoordinationEvent> events) {
        var actState = activityTracker.getState(caseId);
        if (actState == null) return;
        double multiplier = coord.stormRateMultiplier() != null ? coord.stormRateMultiplier() : 10.0;
        var now = Instant.now();
        boolean storming = false;
        List<String> stormingMetrics = new ArrayList<>();
        double dispatchRate = actState.dispatchRate(now);
        double signalRate = actState.signalDepositRate(now);
        double mutationRate = actState.contextMutationRate(now);
        double evalRate = actState.evaluationRate(now);
        if (dispatchRate > 0.1 * multiplier) { stormingMetrics.add("dispatches"); storming = true; }
        if (signalRate > 0.1 * multiplier) { stormingMetrics.add("signalDeposits"); storming = true; }
        if (mutationRate > 0.1 * multiplier) { stormingMetrics.add("contextMutations"); storming = true; }
        if (evalRate > 0.5 * multiplier) { stormingMetrics.add("evaluationCycles"); storming = true; }

        if (storming && !state.stormActive) {
            state.stormActive = true;
            events.add(new CoordinationEvent(
                    CoordinationEvent.Type.COORDINATION_STORM,
                    Map.of("stormingMetrics", stormingMetrics,
                           "currentRates", Map.of("dispatches", dispatchRate, "signalDeposits", signalRate,
                                                   "contextMutations", mutationRate, "evaluationCycles", evalRate))));
        } else if (!storming && state.stormActive) {
            state.stormActive = false;
        }
    }

    private void detectInterestConvergence(UUID caseId, CoordinationConfig coord,
                                            CaseCoordinationState state, List<CoordinationEvent> events) {
        double threshold = coord.interestHotspotThreshold() != null ? coord.interestHotspotThreshold() : 0.6;
        int totalActive = activeCount(caseId);
        if (totalActive == 0) return;
        var observers = observationRegistry.getObservers(caseId);
        Map<String, Set<String>> keyToAgents = new HashMap<>();
        for (var entry : observers.entrySet()) {
            String agentId = entry.getKey();
            for (var observer : entry.getValue()) {
                for (String key : observer.watchedKeys()) {
                    keyToAgents.computeIfAbsent(key, k -> new HashSet<>()).add(agentId);
                }
            }
        }
        Set<String> currentHotspots = new HashSet<>();
        for (var entry : keyToAgents.entrySet()) {
            double score = (double) entry.getValue().size() / totalActive;
            if (score >= threshold) {
                currentHotspots.add(entry.getKey());
                if (state.interestHotspotFired.add(entry.getKey())) {
                    events.add(new CoordinationEvent(
                            CoordinationEvent.Type.INTEREST_CONVERGENCE,
                            Map.of("hotspotKey", entry.getKey(),
                                   "watchingAgentCount", entry.getValue().size(),
                                   "hotspotScore", score)));
                }
            }
        }
        state.interestHotspotFired.retainAll(currentHotspots);
    }

    public void evictByCase(UUID caseId) {
        cases.remove(caseId);
    }

    @Override
    public void reset() {
        cases.clear();
    }

    private static class CaseCoordinationState {
        final StigmergyConfig config;
        final List<String> declaredAgentIds;
        final ConcurrentHashMap<String, AgentState> agents = new ConcurrentHashMap<>();
        final Set<String> consensusFired = ConcurrentHashMap.newKeySet();
        volatile boolean stormActive = false;
        final Set<String> interestHotspotFired = ConcurrentHashMap.newKeySet();

        CaseCoordinationState(StigmergyConfig config, List<String> agentIds) {
            this.config = config;
            this.declaredAgentIds = List.copyOf(agentIds);
        }
    }

    public record CoordinationEvent(Type type, Map<String, Object> metadata) {
        public enum Type { SIGNAL_CONSENSUS, COORDINATION_STORM, INTEREST_CONVERGENCE }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: PASS

- [ ] **Step 5: Write coordination pattern detection tests**

Add to existing test file:

```java
    @Test
    void detectsSignalConsensus() {
        coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
        coordinator.agentJoined(caseId, "a1", "b1");
        coordinator.agentJoined(caseId, "a2", "b2");
        coordinator.agentActivated(caseId, "a1");
        coordinator.agentActivated(caseId, "a2");
        signalRegistry.deposit(caseId, "trail", 0.8, Duration.ofMinutes(5), "a1", 100);
        signalRegistry.deposit(caseId, "trail", 0.9, Duration.ofMinutes(5), "a2", 100);
        var events = coordinator.detectPatterns(caseId, config);
        assertEquals(1, events.size());
        assertEquals(StigmergyCoordinator.CoordinationEvent.Type.SIGNAL_CONSENSUS,
                events.get(0).type());
    }

    @Test
    void consensusDeduplicates() {
        coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
        signalRegistry.deposit(caseId, "trail", 0.8, Duration.ofMinutes(5), "a1", 100);
        signalRegistry.deposit(caseId, "trail", 0.9, Duration.ofMinutes(5), "a2", 100);
        var first = coordinator.detectPatterns(caseId, config);
        assertEquals(1, first.size());
        var second = coordinator.detectPatterns(caseId, config);
        assertTrue(second.isEmpty());
    }
```

- [ ] **Step 6: Run to verify new tests pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=StigmergyCoordinatorTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/ \
       runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/
git commit -m "feat: add StigmergyCoordinator — lifecycle tracking + coordination pattern detection

Refs casehubio/engine#1111"
```

### Task 3: Runtime wiring — leave(), LocalRuleEvaluator, pipeline, status handler

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` — implement `leave()`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` — wire coordinator
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/observation/LocalRuleEvaluator.java` — handle `Leave` action
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — wire coordinator into convergenceDetection()
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — evict coordinator on terminal
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/LocalRuleEvaluatorLeaveTest.java`

**Interfaces:**
- Consumes: `StigmergyCoordinator` (from Task 2) — all methods
- Consumes: `RuleAction.Leave` (from Task 1)
- Consumes: `WorkerRuntime.leave()` (from Task 1)
- Produces: `DefaultWorkerRuntime.leave()` implementation — consumed by workers at runtime
- Produces: `LocalRuleEvaluator` handles `Leave` action — leaves the agent on behalf of a rule

- [ ] **Step 1: Write LocalRuleEvaluator Leave handling test**

```java
// runtime-core/src/test/java/io/casehub/engine/internal/observation/LocalRuleEvaluatorLeaveTest.java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class LocalRuleEvaluatorLeaveTest {

    @Test
    void leaveActionAppearsInFiring() {
        var signalRegistry = new SignalRegistry();
        var evaluator = new LocalRuleEvaluator(signalRegistry);
        var leaveRule = new LocalRule("leave-rule",
                new RuleCondition.PredicateCondition(ctx -> true),
                List.of(new RuleAction.Leave()),
                1);
        var context = new RuleContext(
                List.of(), Map.of(), null, Set.of(), null,
                "agent-1", "tenant-1", UUID.randomUUID());
        var config = new RuleConfig(50, 100, 100);
        var firings = evaluator.evaluate("agent-1", List.of(leaveRule), context, config);
        assertEquals(1, firings.size());
        assertEquals("leave-rule", firings.get(0).ruleId());
        assertTrue(firings.get(0).executedActions().get(0) instanceof RuleAction.Leave);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=LocalRuleEvaluatorLeaveTest -q`
Expected: FAIL — `Leave` not handled in switch

- [ ] **Step 3: Add Leave handling to LocalRuleEvaluator.executeCoordinationAction()**

Use `ide_replace_member` on `executeCoordinationAction` in `LocalRuleEvaluator.java`. New body:

```java
    switch (action) {
      case RuleAction.DepositSignal ds ->
          signalRegistry.deposit(
              context.caseId(),
              ds.name(),
              ds.strength(),
              ds.halfLife() != null ? ds.halfLife() : DEFAULT_HALF_LIFE,
              agentId,
              DEFAULT_MAX_SIGNALS);
      case RuleAction.WriteContext wc -> {}
      case RuleAction.RegisterInterest ri -> {}
      case RuleAction.DeregisterInterest di -> {}
      case RuleAction.Leave l -> {}
    }
```

The `Leave` case is a no-op in the evaluator — the actual departure is handled by the caller (`CaseContextChangedEventHandler.localRules()`) which checks for `Leave` actions in the firings and calls `coordinator.agentDeparted()`.

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=LocalRuleEvaluatorLeaveTest -q`
Expected: PASS

- [ ] **Step 5: Add coordinator field to DefaultWorkerRuntime**

Use `ide_insert_member` to add field after `ruleSpace`:

```java
    private final StigmergyCoordinator stigmergyCoordinator;
```

Add import: `import io.casehub.engine.internal.stigmergy.StigmergyCoordinator;`

Add coordinator parameter to the full constructor (the 14-arg one at lines 91-120). Add field assignment. Also add `leave()` override:

```java
    @Override
    public void leave() {
        if (stigmergyCoordinator == null) return;
        // Implemented by runtime wiring — actual deregistration handled via coordinator
    }
```

For now `leave()` is a placeholder — the full implementation wires through the coordinator in the handler. This satisfies the interface contract while the remaining wiring is in the handler itself.

- [ ] **Step 6: Wire coordinator into WorkerRuntimeFactory**

Use `ide_insert_member` to add `StigmergyCoordinator` field (nullable) to `WorkerRuntimeFactory`. Use `jakarta.enterprise.inject.Instance<StigmergyCoordinator>` for optional injection. Pass to `DefaultWorkerRuntime` constructor.

- [ ] **Step 7: Wire coordinator into CaseContextChangedEventHandler.convergenceDetection()**

Add `Instance<StigmergyCoordinator>` field. In `convergenceDetection()`, add before the null guard:

```java
    if (stigmergyCoordinator.isResolvable()) {
        var coordinator = stigmergyCoordinator.get();
        if (coordinator.isStigmergyCase(caseInstance.getUuid())) {
            var patterns = coordinator.detectPatterns(
                    caseInstance.getUuid(), caseDefinition.getStigmergyConfig());
            for (var pattern : patterns) {
                // Publish as EventLog entries
            }
        }
    }
```

- [ ] **Step 8: Wire eviction into CaseStatusChangedHandler**

Add `Instance<StigmergyCoordinator>` field. In the terminal-status handler, add:

```java
    if (stigmergyCoordinator.isResolvable()) {
        stigmergyCoordinator.get().evictByCase(caseId);
    }
```

- [ ] **Step 9: Run runtime-core tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -q`
Expected: All tests pass

- [ ] **Step 10: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java \
       runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java \
       runtime-core/src/main/java/io/casehub/engine/internal/observation/LocalRuleEvaluator.java \
       runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
       runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java \
       runtime-core/src/test/java/io/casehub/engine/internal/observation/LocalRuleEvaluatorLeaveTest.java
git commit -m "feat: wire StigmergyCoordinator into runtime — leave(), pipeline, status handler

Refs casehubio/engine#1111"
```

---

## Batch 3: Strategy + Integration

### Task 4: StigmergyStrategy — named PlanningStrategy with first-cycle dispatch

**Files:**
- Create: `planning-core/src/main/java/io/casehub/engine/planning/control/StigmergyStrategy.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/control/StigmergyStrategyTest.java`

**Interfaces:**
- Consumes: `PlanningStrategy` interface — `id()`, `getName()`, `select(CasePlanModel, PlanExecutionContext, List<Binding>)`
- Consumes: `StigmergyCoordinator.initializeCase()`, `isStigmergyCase()`, `activeAgents()` (from Task 2)
- Produces: `StigmergyStrategy` with `id() = "stigmergy"` — resolved by StrategyResolver

- [ ] **Step 1: Write StigmergyStrategy test**

```java
// planning/src/test/java/io/casehub/engine/planning/control/StigmergyStrategyTest.java
package io.casehub.engine.planning.control;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.Binding;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.stigmergy.StigmergyConfig;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.internal.stigmergy.StigmergyCoordinator;
import io.casehub.engine.planning.plan.CasePlanModel;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class StigmergyStrategyTest {

    private StigmergyStrategy strategy;
    private StigmergyCoordinator coordinator;

    @BeforeEach
    void setUp() {
        coordinator = new StigmergyCoordinator(
                new SignalRegistry(), new ObservationRegistry(), new ActivityTracker());
        strategy = new StigmergyStrategy(coordinator);
    }

    @Test
    void idIsStigmergy() {
        assertEquals("stigmergy", strategy.id());
    }

    @Test
    void firstSelectReturnsAllBindings() {
        var caseId = UUID.randomUUID();
        var binding1 = Binding.builder().name("b1")
                .capability(Capability.builder().name("c1").build())
                .on("true").build();
        var binding2 = Binding.builder().name("b2")
                .capability(Capability.builder().name("c2").build())
                .on("true").build();
        var def = CaseDefinition.builder()
                .namespace("test").name("test").version("1.0.0")
                .stigmergyConfig(new StigmergyConfig(null, null))
                .build();
        var plan = new CasePlanModel(caseId, def);
        var ctx = new PlanExecutionContext(caseId, def, null, null);
        var selected = strategy.select(plan, ctx, List.of(binding1, binding2));
        assertEquals(2, selected.size());
        assertTrue(coordinator.isStigmergyCase(caseId));
    }

    @Test
    void subsequentSelectReturnsEmpty() {
        var caseId = UUID.randomUUID();
        var binding = Binding.builder().name("b1")
                .capability(Capability.builder().name("c1").build())
                .on("true").build();
        var def = CaseDefinition.builder()
                .namespace("test").name("test").version("1.0.0")
                .stigmergyConfig(new StigmergyConfig(null, null))
                .build();
        var plan = new CasePlanModel(caseId, def);
        var ctx = new PlanExecutionContext(caseId, def, null, null);
        strategy.select(plan, ctx, List.of(binding));
        var second = strategy.select(plan, ctx, List.of(binding));
        assertTrue(second.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest=StigmergyStrategyTest -q`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement StigmergyStrategy**

Use `ide_create_file`:

```java
// planning-core/src/main/java/io/casehub/engine/planning/control/StigmergyStrategy.java
package io.casehub.engine.planning.control;

import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.Binding;
import io.casehub.engine.internal.stigmergy.StigmergyCoordinator;
import io.casehub.engine.planning.plan.CasePlanModel;
import java.util.List;
import java.util.UUID;
import org.jboss.logging.Logger;

public class StigmergyStrategy implements PlanningStrategy {

    private static final Logger LOG = Logger.getLogger(StigmergyStrategy.class);

    private final StigmergyCoordinator coordinator;

    public StigmergyStrategy(StigmergyCoordinator coordinator) {
        this.coordinator = coordinator;
    }

    @Override
    public String id() {
        return "stigmergy";
    }

    @Override
    public String getName() {
        return "Stigmergy Strategy";
    }

    @Override
    public List<Binding> select(CasePlanModel plan, PlanExecutionContext context,
                                 List<Binding> eligible) {
        UUID caseId = context.caseId();

        if (!coordinator.isStigmergyCase(caseId)) {
            var agentIds = eligible.stream().map(Binding::getName).toList();
            var stigmergyConfig = context.definition().getStigmergyConfig();
            coordinator.initializeCase(caseId, agentIds, stigmergyConfig);
            LOG.infof("Stigmergy case initialized: caseId=%s, agents=%d", caseId, agentIds.size());
            return List.copyOf(eligible);
        }

        return List.of();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest=StigmergyStrategyTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add planning-core/src/main/java/io/casehub/engine/planning/control/StigmergyStrategy.java \
       planning/src/test/java/io/casehub/engine/planning/control/StigmergyStrategyTest.java
git commit -m "feat: add StigmergyStrategy — first-cycle dispatch as named PlanningStrategy

Refs casehubio/engine#1111"
```

### Task 5: Integration test — end-to-end stigmergy case lifecycle

**Files:**
- Test: `runtime/src/test/java/io/casehub/engine/internal/stigmergy/StigmergyIntegrationTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1-4
- Tests: Full lifecycle — case init → agent dispatch → agent activation → signal deposit → consensus detection → convergence

- [ ] **Step 1: Write integration test**

```java
// runtime/src/test/java/io/casehub/engine/internal/stigmergy/StigmergyIntegrationTest.java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.stigmergy.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class StigmergyIntegrationTest {

    @Test
    void fullLifecycle() {
        var signalRegistry = new SignalRegistry();
        var observationRegistry = new ObservationRegistry();
        var activityTracker = new ActivityTracker();
        var coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
        var caseId = UUID.randomUUID();
        var config = new StigmergyConfig(null, new CoordinationConfig(2, 10.0, 0.6));

        coordinator.initializeCase(caseId, List.of("temp-monitor", "pressure-monitor"), config);
        assertTrue(coordinator.isStigmergyCase(caseId));

        coordinator.agentJoined(caseId, "temp-monitor", "run-temp-monitor");
        coordinator.agentJoined(caseId, "pressure-monitor", "run-pressure-monitor");
        assertFalse(coordinator.allActive(caseId));

        coordinator.agentActivated(caseId, "temp-monitor");
        coordinator.agentActivated(caseId, "pressure-monitor");
        assertTrue(coordinator.allActive(caseId));
        assertEquals(2, coordinator.activeCount(caseId));

        signalRegistry.deposit(caseId, "overheating", 0.8, Duration.ofMinutes(5), "temp-monitor", 100);
        signalRegistry.deposit(caseId, "overheating", 0.9, Duration.ofMinutes(5), "pressure-monitor", 100);

        var patterns = coordinator.detectPatterns(caseId, config);
        assertTrue(patterns.stream().anyMatch(
                e -> e.type() == StigmergyCoordinator.CoordinationEvent.Type.SIGNAL_CONSENSUS));

        coordinator.agentDeparted(caseId, "temp-monitor");
        assertEquals(1, coordinator.activeCount(caseId));
        assertEquals(AgentLifecycleState.DEPARTED,
                coordinator.getAgent(caseId, "temp-monitor").state());

        coordinator.evictByCase(caseId);
        assertFalse(coordinator.isStigmergyCase(caseId));
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=StigmergyIntegrationTest -q`
Expected: PASS (all types from prior tasks on classpath)

- [ ] **Step 3: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/internal/stigmergy/StigmergyIntegrationTest.java
git commit -m "test: add stigmergy integration test — full lifecycle verification

Refs casehubio/engine#1111"
```

---

## References

- [2026-09-18-stigmergy-execution-model-design.md](../specs/issue-1104-hive-mind/2026-09-18-stigmergy-execution-model-design.md) — design spec
- `api/src/main/java/io/casehub/api/spi/observation/RuleAction.java` — sealed interface, 4 existing permits
- `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` — coordination surface
- `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — existing enum (112 values)
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — case definition with nullable config fields
- `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java` — signal storage, needs `consensusSignals()`
- `common-core/src/main/java/io/casehub/engine/common/internal/observation/ObservationRegistry.java` — observer storage, has `getObservers()`, `computeLandscape()`
- `runtime-core/src/main/java/io/casehub/engine/internal/observation/LocalRuleEvaluator.java` — rule evaluation, switch on RuleAction
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` — WorkerRuntime implementation
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` — factory, wires registries
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:1384-1425` — convergenceDetection()
- `planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategy.java` — strategy SPI
- `planning-core/src/main/java/io/casehub/engine/planning/control/SequentialPlanningStrategy.java` — existing strategy pattern
- D60-D72 — design decisions
- GitHub #1111 — focal issue
- GitHub #1104 — Hive Mind epic
