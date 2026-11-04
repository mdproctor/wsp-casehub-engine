# Agent Discovery & Neighbor Awareness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1108 — feat: Agent discovery & neighbor awareness — dynamic topology
**Issue group:** #1105, #1106, #1107, #1108

**Goal:** Add a NeighborSpace facet to WorkerRuntime that enables agents to discover who else is active on their case and how they relate (coactive, shared interests, shared signals, complementary).

**Architecture:** `NeighborSpace` is the third WorkerRuntime facet — a pure read-only query facade over existing engine registries (PlanItemStore, ObservationRegistry, SignalRegistry). `Signal` gains `Set<String> sources` for multi-depositor tracking. `Neighbor` record with identity, capabilities, status, and relation taxonomy. `DefaultNeighborSpace` computes everything on-demand — no new storage.

**Tech Stack:** Java 21, Quarkus 3.32.2

## Global Constraints

- Pre-release: no backward compatibility obligation
- All new types in engine-api; implementations in runtime-core
- TDD: every production type has a corresponding test
- Use `ide_replace_text_in_file` for editing existing Java files
- Use Write for new files

---

## Batch 1: Foundation types + Signal source tracking

### Task 1: Neighbor + NeighborRelation + NeighborSpace types

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/observation/Neighbor.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/NeighborRelation.java`
- Create: `api/src/main/java/io/casehub/api/engine/NeighborSpace.java`
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` (add `neighbors()` accessor)
- Test: `api/src/test/java/io/casehub/api/spi/observation/NeighborTest.java`
- Test: `api/src/test/java/io/casehub/api/engine/NeighborSpaceTest.java`

**Interfaces:**
- Produces: `Neighbor(String agentId, Set<String> capabilities, TaskStatus currentStatus, String bindingName, Set<NeighborRelation> relations)`, `NeighborRelation` enum (COACTIVE, SHARED_INTEREST, SHARED_SIGNAL, COMPLEMENTARY), `NeighborSpace` interface with `active()`, `withSharedInterests()`, `withSharedSignals()`, `complementary()`, `NOOP` constant

- [ ] **Step 1: Write failing tests for Neighbor and NeighborRelation**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.TaskStatus;
import java.util.Set;
import org.junit.jupiter.api.Test;

class NeighborTest {

  @Test
  void construction() {
    var neighbor = new Neighbor("agent-1", Set.of("analysis", "search"),
        TaskStatus.RUNNING, "binding-a", Set.of(NeighborRelation.COACTIVE));
    assertEquals("agent-1", neighbor.agentId());
    assertEquals(Set.of("analysis", "search"), neighbor.capabilities());
    assertEquals(TaskStatus.RUNNING, neighbor.currentStatus());
    assertEquals("binding-a", neighbor.bindingName());
    assertEquals(Set.of(NeighborRelation.COACTIVE), neighbor.relations());
  }

  @Test
  void multipleRelations() {
    var neighbor = new Neighbor("agent-2", Set.of("review"),
        TaskStatus.RUNNING, "binding-b",
        Set.of(NeighborRelation.COACTIVE, NeighborRelation.SHARED_INTEREST));
    assertEquals(2, neighbor.relations().size());
    assertTrue(neighbor.relations().contains(NeighborRelation.COACTIVE));
    assertTrue(neighbor.relations().contains(NeighborRelation.SHARED_INTEREST));
  }

  @Test
  void allRelationValues() {
    assertEquals(4, NeighborRelation.values().length);
    assertNotNull(NeighborRelation.valueOf("COACTIVE"));
    assertNotNull(NeighborRelation.valueOf("SHARED_INTEREST"));
    assertNotNull(NeighborRelation.valueOf("SHARED_SIGNAL"));
    assertNotNull(NeighborRelation.valueOf("COMPLEMENTARY"));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=NeighborTest -Dcheckstyle.skip=true -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement NeighborRelation**

```java
package io.casehub.api.spi.observation;

public enum NeighborRelation {
  COACTIVE,
  SHARED_INTEREST,
  SHARED_SIGNAL,
  COMPLEMENTARY
}
```

- [ ] **Step 4: Implement Neighbor**

```java
package io.casehub.api.spi.observation;

import io.casehub.api.model.TaskStatus;
import java.util.Set;

public record Neighbor(
    String agentId,
    Set<String> capabilities,
    TaskStatus currentStatus,
    String bindingName,
    Set<NeighborRelation> relations) {}
```

- [ ] **Step 5: Write failing tests for NeighborSpace NOOP**

```java
package io.casehub.api.engine;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class NeighborSpaceTest {

  @Test
  void noopActiveReturnsEmpty() {
    assertTrue(NeighborSpace.NOOP.active().isEmpty());
  }

  @Test
  void noopWithSharedInterestsReturnsEmpty() {
    assertTrue(NeighborSpace.NOOP.withSharedInterests().isEmpty());
  }

  @Test
  void noopWithSharedSignalsReturnsEmpty() {
    assertTrue(NeighborSpace.NOOP.withSharedSignals().isEmpty());
  }

  @Test
  void noopComplementaryReturnsEmpty() {
    assertTrue(NeighborSpace.NOOP.complementary().isEmpty());
  }
}
```

- [ ] **Step 6: Implement NeighborSpace**

```java
package io.casehub.api.engine;

import io.casehub.api.spi.observation.Neighbor;
import java.util.List;

public interface NeighborSpace {
  List<Neighbor> active();
  List<Neighbor> withSharedInterests();
  List<Neighbor> withSharedSignals();
  List<Neighbor> complementary();

  NeighborSpace NOOP = new NeighborSpace() {
    @Override public List<Neighbor> active() { return List.of(); }
    @Override public List<Neighbor> withSharedInterests() { return List.of(); }
    @Override public List<Neighbor> withSharedSignals() { return List.of(); }
    @Override public List<Neighbor> complementary() { return List.of(); }
  };
}
```

- [ ] **Step 7: Add neighbors() to WorkerRuntime**

Use `ide_replace_text_in_file` to add after the `interests()` method:

```java
  default InterestSpace interests() {
    return InterestSpace.NOOP;
  }

  default NeighborSpace neighbors() {
    return NeighborSpace.NOOP;
  }
```

- [ ] **Step 8: Run tests**

Run: `mvn test -pl api -Dtest="NeighborTest,NeighborSpaceTest" -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/Neighbor.java api/src/main/java/io/casehub/api/spi/observation/NeighborRelation.java api/src/main/java/io/casehub/api/engine/NeighborSpace.java api/src/main/java/io/casehub/api/engine/WorkerRuntime.java api/src/test/java/io/casehub/api/spi/observation/NeighborTest.java api/src/test/java/io/casehub/api/engine/NeighborSpaceTest.java
git commit -m "feat: add Neighbor, NeighborRelation, NeighborSpace facet + WorkerRuntime accessor Refs #1108"
```

### Task 2: Signal source tracking

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/signal/Signal.java:21-37` (add `sources` field)
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java:37-85` (track sources on deposit)
- Modify: `api/src/test/java/io/casehub/api/model/signal/SignalTest.java` (update for new field)
- Modify: `common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryTest.java` (add source tracking tests)
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/signal/DefaultSignalSpaceTest.java` (regression)

**Interfaces:**
- Consumes: `Signal` (existing), `SignalRegistry` (existing)
- Produces: `Signal` gains `Set<String> sources` (9th field), `SignalRegistry.deposit()` adds source to the set

- [ ] **Step 1: Write failing test for Signal sources**

Add to `SignalTest.java`:

```java
@Test
void sourcesFieldPresent() {
  var signal = new Signal("test", 1.0, Instant.now(), Instant.now(),
      Duration.ofMinutes(5), "agent-1", 1, false, Set.of("agent-1"));
  assertEquals(Set.of("agent-1"), signal.sources());
}

@Test
void backwardCompatConstructor() {
  var signal = new Signal("test", 1.0, Instant.now(), Instant.now(),
      Duration.ofMinutes(5), "agent-1", 1, false);
  assertEquals(Set.of("agent-1"), signal.sources());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=SignalTest -Dcheckstyle.skip=true -q`
Expected: FAIL — no 9-arg constructor

- [ ] **Step 3: Add sources field to Signal**

Use `ide_replace_text_in_file` to update the `Signal` record. Add `Set<String> sources` as the 9th field. Add a backward-compatible 8-arg constructor that defaults `sources` to `Set.of(lastSource)`:

```java
public record Signal(
    String name,
    double strength,
    Instant firstDeposited,
    Instant lastReinforced,
    Duration halfLife,
    String lastSource,
    int reinforcementCount,
    boolean expired,
    Set<String> sources) {

  public Signal(
      String name, double strength, Instant firstDeposited, Instant lastReinforced,
      Duration halfLife, String lastSource, int reinforcementCount, boolean expired) {
    this(name, strength, firstDeposited, lastReinforced, halfLife, lastSource,
        reinforcementCount, expired, lastSource != null ? Set.of(lastSource) : Set.of());
  }
}
```

- [ ] **Step 4: Write failing test for SignalRegistry source tracking**

Add to `SignalRegistryTest.java`:

```java
@Test
void depositTracksSources() {
  registry.deposit(caseId, "food", 1.0, Duration.ofMinutes(5), "ant-1", 100);
  var signals = registry.getAllSignals(caseId);
  assertEquals(Set.of("ant-1"), signals.get("food").sources());
}

@Test
void reinforcementAddsSources() {
  registry.deposit(caseId, "food", 1.0, Duration.ofMinutes(5), "ant-1", 100);
  registry.deposit(caseId, "food", 0.8, Duration.ofMinutes(5), "ant-2", 100);
  var signals = registry.getAllSignals(caseId);
  assertEquals(Set.of("ant-1", "ant-2"), signals.get("food").sources());
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn test -pl common-core -Dtest=SignalRegistryTest -Dcheckstyle.skip=true -q`
Expected: FAIL — sources is `Set.of("ant-1")` after reinforcement (no tracking yet)

- [ ] **Step 6: Update SignalRegistry.deposit() to track sources**

In `SignalRegistry.java`, update the reinforcement path (line 60-71) to merge sources. On new signal (line 75) set `Set.of(source)`. On reinforce: `new HashSet<>(existing.sources()); sources.add(source); Set.copyOf(sources)`.

Use `ide_replace_text_in_file` to update the three `new Signal(...)` constructor calls in `deposit()`:

Reinforcement path:
```java
      Set<String> mergedSources = new java.util.HashSet<>(existing.sources());
      mergedSources.add(source);
      caseSignals.put(
          name,
          new Signal(name, newStrength, existing.firstDeposited(), now, halfLife,
              source, existing.reinforcementCount() + 1, false, Set.copyOf(mergedSources)));
```

Expired re-deposit path:
```java
      caseSignals.put(name, new Signal(name, strength, now, now, halfLife, source, 1, false, Set.of(source)));
```

New signal path:
```java
    caseSignals.put(name, new Signal(name, strength, now, now, halfLife, source, 1, false, Set.of(source)));
```

- [ ] **Step 7: Add getAllSignals() helper to SignalRegistry if missing**

If `getAllSignals(UUID caseId)` doesn't exist, add it:

```java
public Map<String, Signal> getAllSignals(UUID caseId) {
  var caseSignals = signals.get(caseId);
  return caseSignals == null ? Map.of() : Map.copyOf(caseSignals);
}
```

- [ ] **Step 8: Run all tests to verify pass**

Run: `mvn test -pl api,common-core,runtime-core -Dtest="SignalTest,SignalRegistryTest,DefaultSignalSpaceTest" -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/signal/Signal.java common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java api/src/test/java/io/casehub/api/model/signal/SignalTest.java common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryTest.java
git commit -m "feat: add sources tracking to Signal for multi-depositor neighbor awareness Refs #1108"
```

## Batch 2: DefaultNeighborSpace + wiring + docs

### Task 3: DefaultNeighborSpace + DefaultWorkerRuntime + WorkerRuntimeFactory wiring

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultNeighborSpace.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` (add neighborSpace field + accessor)
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` (inject PlanItemStore, create DefaultNeighborSpace)
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultNeighborSpaceTest.java`

**Interfaces:**
- Consumes: `NeighborSpace` (Task 1), `Neighbor` (Task 1), `NeighborRelation` (Task 1), `Signal.sources` (Task 2), `PlanItemStore.findByCaseId()` (existing), `ObservationRegistry.getObservers()` (existing), `SignalRegistry.perceive()` (existing)
- Produces: `DefaultNeighborSpace` with all four query methods implemented, `WorkerRuntime.neighbors()` returning live instance

- [ ] **Step 1: Write failing tests for DefaultNeighborSpace**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.TaskStatus;
import io.casehub.api.model.signal.SignalConfig;
import io.casehub.api.spi.observation.Neighbor;
import io.casehub.api.spi.observation.NeighborRelation;
import io.casehub.api.spi.observation.ObservationConfig;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultNeighborSpaceTest {

  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private DefaultNeighborSpace neighborSpace;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    neighborSpace = new DefaultNeighborSpace(
        caseId, "tenant-1", "self-agent", "self-binding",
        null, observationRegistry, signalRegistry,
        null, null,
        new SignalConfig(Duration.ofMinutes(5), 0.01, 100));
  }

  @Test
  void activeReturnsEmptyWhenNoOtherAgents() {
    assertTrue(neighborSpace.active().isEmpty());
  }

  @Test
  void withSharedInterestsFindsOverlap() {
    var selfObserver = io.casehub.engine.internal.observation.ThresholdObserver.of(
        "riskScore", io.casehub.engine.internal.observation.ThresholdObserver.Operator.GT, 0.5);
    observationRegistry.registerObserver(caseId, "self-agent", "self-binding", selfObserver, 20);

    var otherObserver = io.casehub.engine.internal.observation.ThresholdObserver.of(
        "riskScore", io.casehub.engine.internal.observation.ThresholdObserver.Operator.GT, 0.7);
    observationRegistry.registerObserver(caseId, "other-agent", "other-binding", otherObserver, 20);

    var neighbors = neighborSpace.withSharedInterests();
    assertEquals(1, neighbors.size());
    assertEquals("other-agent", neighbors.get(0).agentId());
    assertTrue(neighbors.get(0).relations().contains(NeighborRelation.SHARED_INTEREST));
  }

  @Test
  void withSharedInterestsExcludesSelf() {
    var observer = io.casehub.engine.internal.observation.ThresholdObserver.of(
        "x", io.casehub.engine.internal.observation.ThresholdObserver.Operator.GT, 0.5);
    observationRegistry.registerObserver(caseId, "self-agent", "self-binding", observer, 20);
    assertTrue(neighborSpace.withSharedInterests().isEmpty());
  }

  @Test
  void withSharedSignalsFindsSharedDepositors() {
    signalRegistry.deposit(caseId, "danger", 1.0, Duration.ofMinutes(5), "self-agent", 100);
    signalRegistry.deposit(caseId, "danger", 0.8, Duration.ofMinutes(5), "other-agent", 100);
    var neighbors = neighborSpace.withSharedSignals();
    assertEquals(1, neighbors.size());
    assertEquals("other-agent", neighbors.get(0).agentId());
    assertTrue(neighbors.get(0).relations().contains(NeighborRelation.SHARED_SIGNAL));
  }

  @Test
  void withSharedSignalsExcludesSelf() {
    signalRegistry.deposit(caseId, "food", 1.0, Duration.ofMinutes(5), "self-agent", 100);
    assertTrue(neighborSpace.withSharedSignals().isEmpty());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime-core -Dtest=DefaultNeighborSpaceTest -Dcheckstyle.skip=true -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DefaultNeighborSpace**

```java
package io.casehub.engine.internal.observation;

import io.casehub.api.engine.NeighborSpace;
import io.casehub.api.model.TaskStatus;
import io.casehub.api.model.signal.SignalConfig;
import io.casehub.api.spi.observation.EnvironmentObserver;
import io.casehub.api.spi.observation.Neighbor;
import io.casehub.api.spi.observation.NeighborRelation;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.CaseInstanceCache;
import java.util.*;
import java.util.stream.Collectors;

public class DefaultNeighborSpace implements NeighborSpace {

  private final UUID caseId;
  private final String tenancyId;
  private final String selfAgentId;
  private final String selfBindingName;
  private final PlanItemStore planItemStore;
  private final ObservationRegistry observationRegistry;
  private final SignalRegistry signalRegistry;
  private final CaseDefinitionRegistry definitionRegistry;
  private final CaseInstanceCache caseInstanceCache;
  private final SignalConfig signalConfig;

  public DefaultNeighborSpace(
      UUID caseId, String tenancyId, String selfAgentId, String selfBindingName,
      PlanItemStore planItemStore, ObservationRegistry observationRegistry,
      SignalRegistry signalRegistry, CaseDefinitionRegistry definitionRegistry,
      CaseInstanceCache caseInstanceCache, SignalConfig signalConfig) {
    this.caseId = caseId;
    this.tenancyId = tenancyId;
    this.selfAgentId = selfAgentId;
    this.selfBindingName = selfBindingName;
    this.planItemStore = planItemStore;
    this.observationRegistry = observationRegistry;
    this.signalRegistry = signalRegistry;
    this.definitionRegistry = definitionRegistry;
    this.caseInstanceCache = caseInstanceCache;
    this.signalConfig = signalConfig;
  }

  @Override
  public List<Neighbor> active() {
    if (planItemStore == null) return List.of();
    List<PlanItemRecord> items = planItemStore.findByCaseId(caseId, tenancyId);
    Map<String, PlanItemRecord> byExecutor = new LinkedHashMap<>();
    for (PlanItemRecord item : items) {
      if (item.executorName() == null) continue;
      if (item.executorName().equals(selfAgentId)) continue;
      if (item.status() != TaskStatus.RUNNING && item.status() != TaskStatus.DISPATCHING) continue;
      byExecutor.putIfAbsent(item.executorName(), item);
    }
    return byExecutor.entrySet().stream()
        .map(e -> new Neighbor(e.getKey(), Set.of(), e.getValue().status(),
            e.getValue().bindingName(), Set.of(NeighborRelation.COACTIVE)))
        .toList();
  }

  @Override
  public List<Neighbor> withSharedInterests() {
    Map<String, List<EnvironmentObserver>> allObservers =
        observationRegistry.getObservers(caseId);
    List<EnvironmentObserver> myObservers = allObservers.getOrDefault(selfAgentId, List.of());
    if (myObservers.isEmpty()) return List.of();

    Set<String> myKeys = myObservers.stream()
        .flatMap(o -> o.watchedKeys().stream())
        .collect(Collectors.toSet());
    if (myKeys.isEmpty()) return List.of();

    List<Neighbor> result = new ArrayList<>();
    for (var entry : allObservers.entrySet()) {
      if (entry.getKey().equals(selfAgentId)) continue;
      Set<String> theirKeys = entry.getValue().stream()
          .flatMap(o -> o.watchedKeys().stream())
          .collect(Collectors.toSet());
      if (!Collections.disjoint(myKeys, theirKeys)) {
        result.add(new Neighbor(entry.getKey(), Set.of(), null, null,
            Set.of(NeighborRelation.SHARED_INTEREST)));
      }
    }
    return result;
  }

  @Override
  public List<Neighbor> withSharedSignals() {
    double threshold = signalConfig != null
        ? signalConfig.effectiveZeroThreshold() : 0.01;
    var signals = signalRegistry.perceive(caseId, threshold);
    Map<String, Set<String>> allSignals = signalRegistry.getAllSignals(caseId)
        .entrySet().stream()
        .filter(e -> signals.containsKey(e.getKey()))
        .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().sources()));

    Set<String> sharedAgents = new LinkedHashSet<>();
    for (var entry : allSignals.entrySet()) {
      if (entry.getValue().contains(selfAgentId)) {
        for (String source : entry.getValue()) {
          if (!source.equals(selfAgentId)) {
            sharedAgents.add(source);
          }
        }
      }
    }

    return sharedAgents.stream()
        .map(agentId -> new Neighbor(agentId, Set.of(), null, null,
            Set.of(NeighborRelation.SHARED_SIGNAL)))
        .toList();
  }

  @Override
  public List<Neighbor> complementary() {
    if (definitionRegistry == null || caseInstanceCache == null || planItemStore == null) {
      return List.of();
    }
    var instance = caseInstanceCache.get(caseId);
    if (instance == null) return List.of();
    var definition = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
    if (definition == null) return List.of();

    var myBinding = definition.getBindings().stream()
        .filter(b -> selfBindingName != null && selfBindingName.equals(b.getName()))
        .findFirst().orElse(null);
    Set<String> myProducedKeys = myBinding != null && myBinding.producedKeys() != null
        ? myBinding.producedKeys() : Set.of();

    Map<String, List<EnvironmentObserver>> allObservers =
        observationRegistry.getObservers(caseId);
    Set<String> myWatchedKeys = allObservers.getOrDefault(selfAgentId, List.of()).stream()
        .flatMap(o -> o.watchedKeys().stream())
        .collect(Collectors.toSet());

    List<PlanItemRecord> activeItems = planItemStore.findByCaseId(caseId, tenancyId);
    Set<String> result = new LinkedHashSet<>();

    for (PlanItemRecord item : activeItems) {
      if (item.executorName() == null || item.executorName().equals(selfAgentId)) continue;
      var theirBinding = definition.getBindings().stream()
          .filter(b -> item.bindingName() != null && item.bindingName().equals(b.getName()))
          .findFirst().orElse(null);
      Set<String> theirProducedKeys = theirBinding != null && theirBinding.producedKeys() != null
          ? theirBinding.producedKeys() : Set.of();
      Set<String> theirWatchedKeys = allObservers.getOrDefault(item.executorName(), List.of())
          .stream().flatMap(o -> o.watchedKeys().stream()).collect(Collectors.toSet());

      boolean iProduceTheyWatch = !myProducedKeys.isEmpty() && !theirWatchedKeys.isEmpty()
          && !Collections.disjoint(myProducedKeys, theirWatchedKeys);
      boolean theyProduceIWatch = !theirProducedKeys.isEmpty() && !myWatchedKeys.isEmpty()
          && !Collections.disjoint(theirProducedKeys, myWatchedKeys);
      if (iProduceTheyWatch || theyProduceIWatch) {
        result.add(item.executorName());
      }
    }

    return result.stream()
        .map(agentId -> new Neighbor(agentId, Set.of(), null, null,
            Set.of(NeighborRelation.COMPLEMENTARY)))
        .toList();
  }
}
```

- [ ] **Step 4: Add neighborSpace field to DefaultWorkerRuntime**

Use `ide_replace_text_in_file` to add `private final NeighborSpace neighborSpace;` field and `@Override public NeighborSpace neighbors() { return neighborSpace; }` method. Update constructors to accept and store the facet.

- [ ] **Step 5: Update WorkerRuntimeFactory to inject PlanItemStore and create DefaultNeighborSpace**

Add `PlanItemStore` as a constructor-injected field. In the `create()` overload that builds facets, construct `DefaultNeighborSpace` with all required registries and pass it to `DefaultWorkerRuntime`.

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl api,common-core,runtime-core -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultNeighborSpace.java runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultNeighborSpaceTest.java
git commit -m "feat: implement DefaultNeighborSpace query facade + wire into WorkerRuntime Refs #1108"
```

### Task 4: CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: All types from Tasks 1-3

- [ ] **Step 1: Run full test suite**

Run: `mvn test -pl api,common-core,runtime-core -Dcheckstyle.skip=true -q`
Expected: PASS

- [ ] **Step 2: Update CLAUDE.md**

Add after the "## Dynamic Interest Registration" section a new "## Agent Discovery & Neighbor Awareness" section documenting the NeighborSpace facet, Neighbor/NeighborRelation types, four query methods with their data sources, Signal source tracking extension, and DefaultNeighborSpace as a query facade.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add agent discovery and neighbor awareness to CLAUDE.md Refs #1108"
```

## References

- [2026-09-16-agent-discovery-neighbor-awareness-design.md] — design spec this plan implements
- [WorkerRuntime.java:34-40] — facet accessor pattern
- [DefaultWorkerRuntime.java] — facet field pattern
- [WorkerRuntimeFactory.java] — facet construction pattern
- [Signal.java:21-37] — signal record gaining sources field
- [SignalRegistry.java:37-85] — deposit logic for source tracking
- [PlanItemStore.java:41] — findByCaseId query
- [PlanItemRecord.java:22-80] — executorName, bindingName, status fields
- [ObservationRegistry.java:89-101] — getObservers per case
- [D32-D36 in decisions.md] — design decisions
- [GitHub #1108] — focal issue
- [GitHub #1105, #1106, #1107] — foundation issues
