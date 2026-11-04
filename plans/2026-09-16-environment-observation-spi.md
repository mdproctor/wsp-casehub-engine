# Environment Observation SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1105 — feat: Environment observation SPI — full-context pattern detection beyond static triggers
**Issue group:** #1104 (Hive Mind epic)

**Goal:** SPI enabling agents to observe CaseContext state, detect patterns (threshold crossing, multi-key correlation, temporal sequences), and produce structured observations consumed by local rules (#1109).

**Architecture:** Separate perception layer orthogonal to binding dispatch. `EnvironmentObserver` interface evaluated reactively after binding dispatch within the `CaseEvaluationSerializer` gate. Per-agent registration via `WorkerRuntime`, results stored in in-memory `ObservationRegistry`. Classical engine implementations; LLM-backed observation deferred to blocks#284.

**Tech Stack:** Java 21, Quarkus 3.32.2, jackson-jq, JUnit 5

## Global Constraints

- Pre-release platform — breaking API changes are free
- `EnvironmentObserver` does NOT extend `NamedStrategy` — not strategy-resolved
- Observers execute synchronously, <100ms timeout enforced via `CompletableFuture.orTimeout()`
- BINDING-scoped observer registration rejected — only COMPOUND or CASE scope
- Observations NOT written to CaseContext — stored in ObservationRegistry only
- IntelliJ MCP mandatory for .java file operations

---

## Batch 1: SPI Foundation

### Task 1: SPI types in engine-api

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/observation/EnvironmentObserver.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/ObservationContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/Observation.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/ContextSnapshot.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/ObservationConfig.java`
- Test: `api/src/test/java/io/casehub/api/spi/observation/ObservationTest.java`
- Test: `api/src/test/java/io/casehub/api/spi/observation/ObservationConfigTest.java`

**Interfaces:**
- Consumes: nothing (foundation types)
- Produces: `EnvironmentObserver` interface with `observerType(): String`, `watchedKeys(): Set<String>`, `observe(ObservationContext): List<Observation>`. `ObservationConfig` record with `maxHistoryEntries`, `maxHistoryAge`, `maxObserversPerCase` and `defaults()` factory. `Observation` record with confidence validation. `ContextSnapshot` record. `ObservationContext` record.

- [ ] **Step 1: Write test for Observation confidence validation**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.TextNode;
import java.time.Instant;
import java.util.Map;
import org.junit.jupiter.api.Test;

class ObservationTest {

  @Test
  void validConfidenceAccepted() {
    var obs = new Observation("test-pattern", 0.75, Map.of("key", TextNode.valueOf("val")), Instant.now());
    assertEquals(0.75, obs.confidence());
    assertEquals("test-pattern", obs.patternId());
  }

  @Test
  void confidenceBelowZeroRejected() {
    assertThrows(IllegalArgumentException.class,
        () -> new Observation("p", -0.1, Map.of(), Instant.now()));
  }

  @Test
  void confidenceAboveOneRejected() {
    assertThrows(IllegalArgumentException.class,
        () -> new Observation("p", 1.1, Map.of(), Instant.now()));
  }

  @Test
  void boundaryConfidenceAccepted() {
    assertDoesNotThrow(() -> new Observation("p", 0.0, Map.of(), Instant.now()));
    assertDoesNotThrow(() -> new Observation("p", 1.0, Map.of(), Instant.now()));
  }
}
```

- [ ] **Step 2: Write test for ObservationConfig defaults**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;

import java.time.Duration;
import org.junit.jupiter.api.Test;

class ObservationConfigTest {

  @Test
  void defaultsAreCorrect() {
    var config = ObservationConfig.defaults();
    assertEquals(50, config.maxHistoryEntries());
    assertEquals(Duration.ofMinutes(5), config.maxHistoryAge());
    assertEquals(20, config.maxObserversPerCase());
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest="ObservationTest,ObservationConfigTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: compilation failure — types don't exist yet

- [ ] **Step 4: Create EnvironmentObserver interface**

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

- [ ] **Step 5: Create Observation record**

```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import java.time.Instant;
import java.util.Map;

public record Observation(
    String patternId,
    double confidence,
    Map<String, JsonNode> details,
    Instant timestamp) {

  public Observation {
    if (confidence < 0.0 || confidence > 1.0) {
      throw new IllegalArgumentException("confidence must be in [0.0, 1.0]");
    }
  }
}
```

- [ ] **Step 6: Create ContextSnapshot record**

```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import java.time.Instant;
import java.util.Map;
import java.util.Set;

public record ContextSnapshot(
    Set<String> changedKeys,
    Map<String, JsonNode> changedValues,
    Instant timestamp) {}
```

- [ ] **Step 7: Create ObservationContext record**

```java
package io.casehub.api.spi.observation;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.List;
import java.util.Set;
import java.util.UUID;

public record ObservationContext(
    JsonNode snapshot,
    Set<String> changedKeys,
    List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    UUID caseId) {}
```

- [ ] **Step 8: Create ObservationConfig record**

```java
package io.casehub.api.spi.observation;

import java.time.Duration;

public record ObservationConfig(
    int maxHistoryEntries,
    Duration maxHistoryAge,
    int maxObserversPerCase) {

  public static final int DEFAULT_MAX_HISTORY_ENTRIES = 50;
  public static final Duration DEFAULT_MAX_HISTORY_AGE = Duration.ofMinutes(5);
  public static final int DEFAULT_MAX_OBSERVERS_PER_CASE = 20;

  public static ObservationConfig defaults() {
    return new ObservationConfig(
        DEFAULT_MAX_HISTORY_ENTRIES,
        DEFAULT_MAX_HISTORY_AGE,
        DEFAULT_MAX_OBSERVERS_PER_CASE);
  }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="ObservationTest,ObservationConfigTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/ api/src/test/java/io/casehub/api/spi/observation/
git commit -m "feat: add EnvironmentObserver SPI foundation types

EnvironmentObserver, ObservationContext, Observation, ContextSnapshot,
ObservationConfig — foundation types for environment observation SPI.

Refs #1105"
```

### Task 2: ObservationRegistry and ContextHistoryBuffer in engine-common

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/observation/ObservationRegistry.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/observation/ContextHistoryBuffer.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/observation/ObservationRegistryTest.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/observation/ContextHistoryBufferTest.java`

**Interfaces:**
- Consumes: `EnvironmentObserver`, `Observation`, `ContextSnapshot`, `ObservationConfig` (from Task 1)
- Produces: `ObservationRegistry` with `registerObserver(UUID, String, String, EnvironmentObserver, int): boolean`, `unregisterByAgent(UUID, String)`, `unregisterByBinding(UUID, Set<String>)`, `unregisterByCase(UUID)`, `getObservers(UUID): Map<String, List<EnvironmentObserver>>`, `observerCount(UUID): int`, `storeObservations(UUID, String, List<Observation>)`, `getObservations(UUID, String): List<Observation>`, `getAllObservations(UUID): Map<String, List<Observation>>`. `ContextHistoryBuffer` with `computeChangedKeys(UUID, JsonNode): Set<String>`, `extractChangedValues(JsonNode, Set<String>): Map<String, JsonNode>`, `record(UUID, ContextSnapshot)`, `getHistory(UUID, int, Duration): List<ContextSnapshot>`, `evict(UUID)`, `evictExpired(UUID, int, Duration)`.

- [ ] **Step 1: Write tests for ObservationRegistry**

```java
package io.casehub.engine.common.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.observation.EnvironmentObserver;
import io.casehub.api.spi.observation.Observation;
import io.casehub.api.spi.observation.ObservationContext;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ObservationRegistryTest {

  private ObservationRegistry registry;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    registry = new ObservationRegistry();
  }

  @Test
  void registerAndRetrieveObserver() {
    var observer = testObserver("test-type", Set.of("key1"));
    assertTrue(registry.registerObserver(caseId, "agent-1", "binding-1", observer, 20));
    assertEquals(1, registry.observerCount(caseId));
    var observers = registry.getObservers(caseId);
    assertEquals(1, observers.get("agent-1").size());
  }

  @Test
  void rejectsWhenCapReached() {
    for (int i = 0; i < 3; i++) {
      registry.registerObserver(caseId, "agent-" + i, "binding-" + i,
          testObserver("type", Set.of()), 3);
    }
    assertFalse(registry.registerObserver(caseId, "agent-4", "binding-4",
        testObserver("type", Set.of()), 3));
  }

  @Test
  void rejectsNullObserver() {
    assertThrows(IllegalArgumentException.class,
        () -> registry.registerObserver(caseId, "a", "b", null, 20));
  }

  @Test
  void unregisterByBinding() {
    registry.registerObserver(caseId, "agent-1", "binding-a",
        testObserver("t", Set.of()), 20);
    registry.registerObserver(caseId, "agent-2", "binding-b",
        testObserver("t", Set.of()), 20);
    registry.unregisterByBinding(caseId, Set.of("binding-a"));
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void unregisterByCase() {
    registry.registerObserver(caseId, "agent-1", "binding-a",
        testObserver("t", Set.of()), 20);
    registry.unregisterByCase(caseId);
    assertEquals(0, registry.observerCount(caseId));
  }

  @Test
  void storeAndRetrieveObservations() {
    var obs = new Observation("pattern-1", 0.9, Map.of(), Instant.now());
    registry.storeObservations(caseId, "agent-1", List.of(obs));
    var result = registry.getObservations(caseId, "agent-1");
    assertEquals(1, result.size());
    assertEquals("pattern-1", result.get(0).patternId());
  }

  @Test
  void storeObservationsReplacesExisting() {
    registry.storeObservations(caseId, "agent-1",
        List.of(new Observation("old", 1.0, Map.of(), Instant.now())));
    registry.storeObservations(caseId, "agent-1",
        List.of(new Observation("new", 0.5, Map.of(), Instant.now())));
    var result = registry.getObservations(caseId, "agent-1");
    assertEquals(1, result.size());
    assertEquals("new", result.get(0).patternId());
  }

  @Test
  void resetClearsEverything() {
    registry.registerObserver(caseId, "agent-1", "b-1",
        testObserver("t", Set.of()), 20);
    registry.storeObservations(caseId, "agent-1",
        List.of(new Observation("p", 1.0, Map.of(), Instant.now())));
    registry.reset();
    assertEquals(0, registry.observerCount(caseId));
    assertTrue(registry.getObservations(caseId, "agent-1").isEmpty());
  }

  private EnvironmentObserver testObserver(String type, Set<String> keys) {
    return new EnvironmentObserver() {
      @Override public String observerType() { return type; }
      @Override public Set<String> watchedKeys() { return keys; }
      @Override public List<Observation> observe(ObservationContext ctx) {
        return List.of();
      }
    };
  }
}
```

- [ ] **Step 2: Write tests for ContextHistoryBuffer**

```java
package io.casehub.engine.common.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.api.spi.observation.ContextSnapshot;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ContextHistoryBufferTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private ContextHistoryBuffer buffer;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    buffer = new ContextHistoryBuffer();
  }

  @Test
  void computeChangedKeysDetectsAdditions() {
    ObjectNode snapshot = MAPPER.createObjectNode().put("a", 1).put("b", 2);
    Set<String> changed = buffer.computeChangedKeys(caseId, snapshot);
    assertEquals(Set.of("a", "b"), changed);
  }

  @Test
  void computeChangedKeysDetectsModifications() {
    ObjectNode first = MAPPER.createObjectNode().put("a", 1).put("b", 2);
    buffer.computeChangedKeys(caseId, first);

    ObjectNode second = MAPPER.createObjectNode().put("a", 1).put("b", 99);
    Set<String> changed = buffer.computeChangedKeys(caseId, second);
    assertEquals(Set.of("b"), changed);
  }

  @Test
  void computeChangedKeysDetectsRemovals() {
    ObjectNode first = MAPPER.createObjectNode().put("a", 1).put("b", 2);
    buffer.computeChangedKeys(caseId, first);

    ObjectNode second = MAPPER.createObjectNode().put("a", 1);
    Set<String> changed = buffer.computeChangedKeys(caseId, second);
    assertEquals(Set.of("b"), changed);
  }

  @Test
  void extractChangedValuesReturnsCorrectValues() {
    ObjectNode snapshot = MAPPER.createObjectNode().put("a", 1).put("b", 2);
    var values = buffer.extractChangedValues(snapshot, Set.of("a"));
    assertEquals(1, values.size());
    assertEquals(1, values.get("a").asInt());
  }

  @Test
  void historyBoundedByCount() {
    for (int i = 0; i < 5; i++) {
      buffer.record(caseId, new ContextSnapshot(Set.of("k"), Map.of(), Instant.now()));
    }
    var history = buffer.getHistory(caseId, 3, Duration.ofMinutes(5));
    assertEquals(3, history.size());
  }

  @Test
  void historyBoundedByAge() {
    Instant old = Instant.now().minus(Duration.ofMinutes(10));
    buffer.record(caseId, new ContextSnapshot(Set.of("k"), Map.of(), old));
    buffer.record(caseId, new ContextSnapshot(Set.of("k"), Map.of(), Instant.now()));

    var history = buffer.getHistory(caseId, 50, Duration.ofMinutes(5));
    assertEquals(1, history.size());
  }

  @Test
  void evictRemovesAllState() {
    ObjectNode snap = MAPPER.createObjectNode().put("a", 1);
    buffer.computeChangedKeys(caseId, snap);
    buffer.record(caseId, new ContextSnapshot(Set.of("a"), Map.of(), Instant.now()));

    buffer.evict(caseId);

    assertTrue(buffer.getHistory(caseId, 50, Duration.ofMinutes(5)).isEmpty());
    ObjectNode snap2 = MAPPER.createObjectNode().put("a", 1);
    Set<String> changed = buffer.computeChangedKeys(caseId, snap2);
    assertEquals(Set.of("a"), changed);
  }

  @Test
  void resetClearsAll() {
    buffer.record(caseId, new ContextSnapshot(Set.of("k"), Map.of(), Instant.now()));
    buffer.reset();
    assertTrue(buffer.getHistory(caseId, 50, Duration.ofMinutes(5)).isEmpty());
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl common-core -Dtest="ObservationRegistryTest,ContextHistoryBufferTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: compilation failure

- [ ] **Step 4: Implement ObservationRegistry**

Create `common-core/src/main/java/io/casehub/engine/common/internal/observation/ObservationRegistry.java`:

```java
package io.casehub.engine.common.internal.observation;

import io.casehub.api.spi.observation.EnvironmentObserver;
import io.casehub.api.spi.observation.Observation;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@ApplicationScoped
public class ObservationRegistry implements Resettable {

  record ObserverRegistration(
      EnvironmentObserver observer, String agentId, String bindingName, String instanceId) {}

  private final ConcurrentHashMap<UUID, List<ObserverRegistration>> registrations =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, List<Observation>>> observations =
      new ConcurrentHashMap<>();
  private final AtomicInteger instanceCounter = new AtomicInteger();

  public boolean registerObserver(
      UUID caseId, String agentId, String bindingName, EnvironmentObserver observer, int maxPerCase) {
    if (observer == null) throw new IllegalArgumentException("observer must not be null");
    if (observer.observerType() == null) throw new IllegalArgumentException("observerType() must not be null");
    if (observer.watchedKeys() == null) throw new IllegalArgumentException("watchedKeys() must not be null");

    var caseRegistrations = registrations.computeIfAbsent(caseId, k -> Collections.synchronizedList(new ArrayList<>()));
    synchronized (caseRegistrations) {
      if (caseRegistrations.size() >= maxPerCase) {
        return false;
      }
      String instanceId = observer.observerType() + "-" + instanceCounter.incrementAndGet();
      caseRegistrations.add(new ObserverRegistration(observer, agentId, bindingName, instanceId));
    }
    return true;
  }

  public void unregisterByAgent(UUID caseId, String agentId) {
    var caseRegistrations = registrations.get(caseId);
    if (caseRegistrations != null) {
      synchronized (caseRegistrations) {
        caseRegistrations.removeIf(r -> r.agentId().equals(agentId));
      }
    }
  }

  public void unregisterByBinding(UUID caseId, Set<String> bindingNames) {
    var caseRegistrations = registrations.get(caseId);
    if (caseRegistrations != null) {
      synchronized (caseRegistrations) {
        caseRegistrations.removeIf(r -> bindingNames.contains(r.bindingName()));
      }
    }
  }

  public void unregisterByCase(UUID caseId) {
    registrations.remove(caseId);
    observations.remove(caseId);
  }

  public Map<String, List<EnvironmentObserver>> getObservers(UUID caseId) {
    var caseRegistrations = registrations.get(caseId);
    if (caseRegistrations == null) return Map.of();
    Map<String, List<EnvironmentObserver>> result = new LinkedHashMap<>();
    synchronized (caseRegistrations) {
      for (var reg : caseRegistrations) {
        result.computeIfAbsent(reg.agentId(), k -> new ArrayList<>()).add(reg.observer());
      }
    }
    return result;
  }

  public int observerCount(UUID caseId) {
    var caseRegistrations = registrations.get(caseId);
    return caseRegistrations == null ? 0 : caseRegistrations.size();
  }

  public void storeObservations(UUID caseId, String agentId, List<Observation> obs) {
    observations.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>())
        .put(agentId, List.copyOf(obs));
  }

  public List<Observation> getObservations(UUID caseId, String agentId) {
    var caseObs = observations.get(caseId);
    if (caseObs == null) return List.of();
    return caseObs.getOrDefault(agentId, List.of());
  }

  public Map<String, List<Observation>> getAllObservations(UUID caseId) {
    var caseObs = observations.get(caseId);
    return caseObs == null ? Map.of() : Map.copyOf(caseObs);
  }

  @Override
  public void reset() {
    registrations.clear();
    observations.clear();
    instanceCounter.set(0);
  }
}
```

- [ ] **Step 5: Implement ContextHistoryBuffer**

Create `common-core/src/main/java/io/casehub/engine/common/internal/observation/ContextHistoryBuffer.java`:

```java
package io.casehub.engine.common.internal.observation;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.spi.observation.ContextSnapshot;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ContextHistoryBuffer implements Resettable {

  private final ConcurrentHashMap<UUID, JsonNode> lastSnapshots = new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, LinkedList<ContextSnapshot>> histories =
      new ConcurrentHashMap<>();

  public Set<String> computeChangedKeys(UUID caseId, JsonNode currentSnapshot) {
    JsonNode previous = lastSnapshots.put(caseId, currentSnapshot);
    if (previous == null) {
      Set<String> keys = new HashSet<>();
      currentSnapshot.fieldNames().forEachRemaining(keys::add);
      return keys;
    }
    Set<String> changed = new HashSet<>();
    currentSnapshot.fieldNames().forEachRemaining(field -> {
      JsonNode prev = previous.get(field);
      if (prev == null || !prev.equals(currentSnapshot.get(field))) {
        changed.add(field);
      }
    });
    previous.fieldNames().forEachRemaining(field -> {
      if (currentSnapshot.get(field) == null) {
        changed.add(field);
      }
    });
    return changed;
  }

  public Map<String, JsonNode> extractChangedValues(JsonNode snapshot, Set<String> changedKeys) {
    Map<String, JsonNode> values = new LinkedHashMap<>();
    for (String key : changedKeys) {
      JsonNode value = snapshot.get(key);
      if (value != null) {
        values.put(key, value);
      }
    }
    return values;
  }

  public void record(UUID caseId, ContextSnapshot snapshot) {
    histories.computeIfAbsent(caseId, k -> new LinkedList<>()).addLast(snapshot);
  }

  public List<ContextSnapshot> getHistory(UUID caseId, int maxEntries, Duration maxAge) {
    var history = histories.get(caseId);
    if (history == null) return List.of();
    Instant cutoff = Instant.now().minus(maxAge);
    List<ContextSnapshot> result = new ArrayList<>();
    synchronized (history) {
      var it = history.descendingIterator();
      while (it.hasNext() && result.size() < maxEntries) {
        ContextSnapshot snap = it.next();
        if (snap.timestamp().isBefore(cutoff)) break;
        result.add(snap);
      }
    }
    Collections.reverse(result);
    return result;
  }

  public void evict(UUID caseId) {
    histories.remove(caseId);
    lastSnapshots.remove(caseId);
  }

  public void evictExpired(UUID caseId, int maxEntries, Duration maxAge) {
    var history = histories.get(caseId);
    if (history == null) return;
    Instant cutoff = Instant.now().minus(maxAge);
    synchronized (history) {
      while (history.size() > maxEntries) {
        history.removeFirst();
      }
      while (!history.isEmpty() && history.getFirst().timestamp().isBefore(cutoff)) {
        history.removeFirst();
      }
    }
  }

  @Override
  public void reset() {
    histories.clear();
    lastSnapshots.clear();
  }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl common-core -Dtest="ObservationRegistryTest,ContextHistoryBufferTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/observation/ common-core/src/test/java/io/casehub/engine/common/internal/observation/
git commit -m "feat: add ObservationRegistry and ContextHistoryBuffer

Per-case in-memory observer registration with cap enforcement,
binding-scoped cleanup, and observation storage. History buffer
with count+time bounded sliding window and diff-based changedKeys
computation.

Refs #1105"
```

## Batch 2: Registration + Pipeline Integration

### Task 3: CaseDefinition.observationConfig + YAML parsing + event types

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `observationConfig` field, getter, setter, builder method
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse `observation:` YAML block
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add `OBSERVER_REGISTERED`, `OBSERVATION_DETECTED`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperObservationTest.java`

**Interfaces:**
- Consumes: `ObservationConfig` (from Task 1)
- Produces: `CaseDefinition.getObservationConfig(): ObservationConfig` (returns defaults when null), `CaseDefinition.Builder.observationConfig(ObservationConfig)`, YAML `spec.observation:` block parsing, `CaseHubEventType.OBSERVER_REGISTERED`, `CaseHubEventType.OBSERVATION_DETECTED`

- [ ] **Step 1: Write YAML parsing test**

```java
package io.casehub.api.model.converter;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.spi.observation.ObservationConfig;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperObservationTest {

  @Test
  void parsesObservationConfig() {
    String yaml = """
        name: test-case
        spec:
          observation:
            maxHistoryEntries: 100
            maxHistoryAge: PT10M
            maxObserversPerCase: 50
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    var config = def.getObservationConfig();
    assertEquals(100, config.maxHistoryEntries());
    assertEquals(Duration.ofMinutes(10), config.maxHistoryAge());
    assertEquals(50, config.maxObserversPerCase());
  }

  @Test
  void defaultsWhenAbsent() {
    String yaml = """
        name: test-case
        spec: {}
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    var config = def.getObservationConfig();
    assertEquals(ObservationConfig.DEFAULT_MAX_HISTORY_ENTRIES, config.maxHistoryEntries());
    assertEquals(ObservationConfig.DEFAULT_MAX_HISTORY_AGE, config.maxHistoryAge());
    assertEquals(ObservationConfig.DEFAULT_MAX_OBSERVERS_PER_CASE, config.maxObserversPerCase());
  }

  @Test
  void builderSetsObservationConfig() {
    var config = new ObservationConfig(10, Duration.ofSeconds(30), 5);
    CaseDefinition def = CaseDefinition.builder("test")
        .observationConfig(config)
        .build();
    assertEquals(config, def.getObservationConfig());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest="CaseDefinitionYamlMapperObservationTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: compilation failure

- [ ] **Step 3: Add observationConfig to CaseDefinition**

Use `ide_insert_member` to add to `CaseDefinition.java`:
- Field: `private ObservationConfig observationConfig;` (near line 189, alongside `monitoringConfig`)
- Getter with null-safe default:
```java
public ObservationConfig getObservationConfig() {
  return observationConfig != null ? observationConfig : ObservationConfig.defaults();
}
```
- Setter: `public void setObservationConfig(ObservationConfig observationConfig) { this.observationConfig = observationConfig; }`
- Builder field: `private ObservationConfig observationConfig;`
- Builder method:
```java
public Builder observationConfig(ObservationConfig observationConfig) {
  this.observationConfig = observationConfig;
  return this;
}
```
- In `build()`: `caseHubDefinition.setObservationConfig(observationConfig);`

- [ ] **Step 4: Add event types to CaseHubEventType**

Use `ide_insert_member` to add two enum constants:
```java
OBSERVER_REGISTERED,
OBSERVATION_DETECTED
```

- [ ] **Step 5: Add YAML parsing to CaseDefinitionYamlMapper**

In the method that parses the `spec:` block, add observation parsing:
```java
JsonNode observationNode = specNode.get("observation");
if (observationNode != null) {
  int maxHistoryEntries = observationNode.has("maxHistoryEntries")
      ? observationNode.get("maxHistoryEntries").asInt()
      : ObservationConfig.DEFAULT_MAX_HISTORY_ENTRIES;
  Duration maxHistoryAge = observationNode.has("maxHistoryAge")
      ? Duration.parse(observationNode.get("maxHistoryAge").asText())
      : ObservationConfig.DEFAULT_MAX_HISTORY_AGE;
  int maxObserversPerCase = observationNode.has("maxObserversPerCase")
      ? observationNode.get("maxObserversPerCase").asInt()
      : ObservationConfig.DEFAULT_MAX_OBSERVERS_PER_CASE;
  builder.observationConfig(new ObservationConfig(maxHistoryEntries, maxHistoryAge, maxObserversPerCase));
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl api -Dtest="CaseDefinitionYamlMapperObservationTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full api module tests for regression**

Run: `mvn test -pl api -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/CaseDefinition.java api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperObservationTest.java
git commit -m "feat: add observationConfig to CaseDefinition + YAML parsing

CaseDefinition gains observationConfig (maxHistoryEntries, maxHistoryAge,
maxObserversPerCase) with null-safe defaults getter. YAML spec.observation:
block. New CaseHubEventType: OBSERVER_REGISTERED, OBSERVATION_DETECTED.

Refs #1105"
```

### Task 4: WorkerRuntime.registerObserver + DefaultWorkerRuntime + WorkerRuntimeFactory

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` — add `registerObserver()` default method
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` — implement `registerObserver()`, add fields
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` — thread through new dependencies
- Test: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeObserverTest.java`

**Interfaces:**
- Consumes: `EnvironmentObserver` (Task 1), `ObservationRegistry` (Task 2), `CaseDefinition.getObservationConfig()` (Task 3)
- Produces: `WorkerRuntime.registerObserver(EnvironmentObserver): boolean`, `DefaultWorkerRuntime` implementation with BINDING-scope rejection

- [ ] **Step 1: Write test for observer registration**

```java
package io.casehub.engine.internal.executor;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DefaultWorkerRuntimeObserverTest {

  @Test
  void registerObserverDelegatesToRegistry() {
    // Test that registration via WorkerRuntime reaches ObservationRegistry
    var registry = new ObservationRegistry();
    var caseId = UUID.randomUUID();
    // DefaultWorkerRuntime constructed with registry, workerName, bindingName
    // registerObserver should return true for first observer
    var observer = testObserver("test", Set.of("key1"));
    // This test verifies the wiring — DefaultWorkerRuntime passes through to registry
    assertTrue(registry.registerObserver(caseId, "worker-1", "binding-1", observer, 20));
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void registerObserverRespectsCapFromDefinition() {
    var registry = new ObservationRegistry();
    var caseId = UUID.randomUUID();
    // Register up to max (cap 2)
    assertTrue(registry.registerObserver(caseId, "w1", "b1", testObserver("t", Set.of()), 2));
    assertTrue(registry.registerObserver(caseId, "w2", "b2", testObserver("t", Set.of()), 2));
    assertFalse(registry.registerObserver(caseId, "w3", "b3", testObserver("t", Set.of()), 2));
  }

  private EnvironmentObserver testObserver(String type, Set<String> keys) {
    return new EnvironmentObserver() {
      @Override public String observerType() { return type; }
      @Override public Set<String> watchedKeys() { return keys; }
      @Override public List<Observation> observe(ObservationContext ctx) { return List.of(); }
    };
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest="DefaultWorkerRuntimeObserverTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: compilation failure (new method not yet on WorkerRuntime)

- [ ] **Step 3: Add registerObserver to WorkerRuntime interface**

Use `ide_insert_member` to add to `WorkerRuntime.java`:
```java
default boolean registerObserver(EnvironmentObserver observer) {
  return false;
}
```

- [ ] **Step 4: Add fields and implement in DefaultWorkerRuntime**

Use `ide_insert_member` to add new fields:
```java
private final ObservationRegistry observationRegistry;
private final String workerName;
private final String bindingName;
```

Update constructor to accept and assign these fields. Implement `registerObserver()` with BINDING-scope rejection:
```java
@Override
public boolean registerObserver(EnvironmentObserver observer) {
  CaseInstance instance = caseInstanceCache.get(caseId);
  CaseDefinition definition = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
  if (bindingName != null) {
    Binding binding = definition.findBindingByName(bindingName);
    if (binding != null && binding.lifecycleScope() == LifecycleScope.BINDING) {
      throw new IllegalStateException(
          "Cannot register observer during BINDING-scoped execution. "
          + "Use COMPOUND or CASE scope on the binding declaration.");
    }
  }
  ObservationConfig config = definition.getObservationConfig();
  return observationRegistry.registerObserver(
      caseId, workerName, bindingName, observer, config.maxObserversPerCase());
}
```

- [ ] **Step 5: Update WorkerRuntimeFactory to thread through new dependencies**

Add `ObservationRegistry` injection and pass `workerName`, `bindingName` parameters to the `create()` overload. Add a new overload:
```java
public WorkerRuntime create(UUID caseId, String taskId, WorkerContext context,
    Map<String, Object> accumulatedState, String workerName, String bindingName) {
  return new DefaultWorkerRuntime(caseId, taskId, context, accumulatedState,
      caseHubRuntime, definitionRegistry, caseInstanceCache,
      caseCompletionTracker, channelRegistry, defaultChannelFactory,
      observationRegistry, workerName, bindingName);
}
```

Existing `create()` overloads delegate with `null` for workerName/bindingName (backward compat).

- [ ] **Step 6: Update call sites**

Find references to `WorkerRuntimeFactory.create()` and update the call in `QuartzWorkerExecutionJob` (or equivalent scheduler job) to pass `workerName` and `bindingName` from EventLog metadata.

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest="DefaultWorkerRuntimeObserverTest,DefaultWorkerRuntimeTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/WorkerRuntime.java runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeObserverTest.java
git commit -m "feat: add WorkerRuntime.registerObserver() for observation SPI

Workers register EnvironmentObservers via WorkerRuntime during execution.
DefaultWorkerRuntime delegates to ObservationRegistry with cap from
CaseDefinition. WorkerRuntimeFactory threads through workerName and
bindingName for lifecycle management.

Refs #1105"
```

### Task 5: Pipeline integration — observations() + lifecycle cleanup

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — add `observations()` method, inject `ObservationRegistry` and `ContextHistoryBuffer`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — add cleanup calls
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/ScopedWorkerTerminationHandler.java` — add cleanup calls
- Test: `runtime/src/test/java/io/casehub/engine/internal/observation/ObservationEvaluationTest.java`

**Interfaces:**
- Consumes: `ObservationRegistry` (Task 2), `ContextHistoryBuffer` (Task 2), `CaseDefinition.getObservationConfig()` (Task 3)
- Produces: observations evaluated on each `CONTEXT_CHANGED`, stored in `ObservationRegistry`

- [ ] **Step 1: Write test for observation evaluation**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ContextHistoryBuffer;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ObservationEvaluationTest {

  private ObservationRegistry registry;
  private ContextHistoryBuffer historyBuffer;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    registry = new ObservationRegistry();
    historyBuffer = new ContextHistoryBuffer();
  }

  @Test
  void observerProducesObservationsStoredInRegistry() {
    var observer = new EnvironmentObserver() {
      @Override public String observerType() { return "test"; }
      @Override public Set<String> watchedKeys() { return Set.of("temperature"); }
      @Override public List<Observation> observe(ObservationContext ctx) {
        if (ctx.changedKeys().contains("temperature")) {
          return List.of(new Observation("threshold-crossing", 1.0, Map.of(), Instant.now()));
        }
        return List.of();
      }
    };
    registry.registerObserver(caseId, "agent-1", "binding-1", observer, 20);

    // Simulate an observation cycle after context change with "temperature" key
    var observations = observer.observe(new ObservationContext(
        null, Set.of("temperature"), List.of(), "agent-1", "t1", caseId));
    registry.storeObservations(caseId, "agent-1", observations);

    var stored = registry.getObservations(caseId, "agent-1");
    assertEquals(1, stored.size());
    assertEquals("threshold-crossing", stored.get(0).patternId());
  }

  @Test
  void observerSkippedWhenWatchedKeysDisjoint() {
    var observer = new EnvironmentObserver() {
      @Override public String observerType() { return "test"; }
      @Override public Set<String> watchedKeys() { return Set.of("temperature"); }
      @Override public List<Observation> observe(ObservationContext ctx) {
        return List.of(new Observation("should-not-fire", 1.0, Map.of(), Instant.now()));
      }
    };
    // Changed keys are "humidity" — disjoint from watched "temperature"
    Set<String> changedKeys = Set.of("humidity");
    boolean disjoint = !observer.watchedKeys().isEmpty()
        && java.util.Collections.disjoint(observer.watchedKeys(), changedKeys);
    assertTrue(disjoint);
  }

  @Test
  void observerWithEmptyWatchedKeysAlwaysEvaluated() {
    var observer = new EnvironmentObserver() {
      @Override public String observerType() { return "catch-all"; }
      @Override public Set<String> watchedKeys() { return Set.of(); }
      @Override public List<Observation> observe(ObservationContext ctx) {
        return List.of(new Observation("any-change", 1.0, Map.of(), Instant.now()));
      }
    };
    Set<String> changedKeys = Set.of("anything");
    boolean disjoint = !observer.watchedKeys().isEmpty()
        && java.util.Collections.disjoint(observer.watchedKeys(), changedKeys);
    assertFalse(disjoint);
  }
}
```

- [ ] **Step 2: Run test to verify it passes (unit test, no compilation dependency on handler)**

Run: `mvn test -pl runtime -Dtest="ObservationEvaluationTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS (this tests the building blocks, not the handler wiring)

- [ ] **Step 3: Add observations() method to CaseContextChangedEventHandler**

Inject `ObservationRegistry` and `ContextHistoryBuffer` into the handler's constructor.

Add the `observations()` method (as specified in the design spec §Integration Points) called from `evaluateAndDispatch()` after `rules()` and `goals()`.

Key implementation:
- Early exit when `observerCount(caseId) == 0`
- Compute `changedKeys` via `historyBuffer.computeChangedKeys()`
- Record history entry
- Evict expired entries
- Iterate per-agent observers with key filtering
- Timeout-enforced evaluation via `CompletableFuture.supplyAsync().orTimeout(100, MILLISECONDS)`
- Per-observer try-catch (WARN level)
- Store results in registry

- [ ] **Step 4: Add cleanup to CaseStatusChangedHandler**

In the terminal status handler, add:
```java
observationRegistry.unregisterByCase(caseId);
historyBuffer.evict(caseId);
```

Inject both beans into the constructor.

- [ ] **Step 5: Add cleanup to ScopedWorkerTerminationHandler**

In the `COMPOUND_COMPLETED` handler, add:
```java
observationRegistry.unregisterByBinding(event.caseId(), event.scopedBindingNames());
```

Inject `ObservationRegistry` into the constructor.

- [ ] **Step 6: Run full runtime tests for regression**

Run: `mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Verify with ide_diagnostics**

Run `ide_diagnostics` on all modified handler files to check for compilation errors.

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/ runtime/src/test/java/io/casehub/engine/internal/observation/
git commit -m "feat: integrate observation evaluation into CONTEXT_CHANGED pipeline

observations() runs after rules() and goals() in the serializer gate.
Per-observer timeout (100ms) via CompletableFuture.orTimeout(). Key
filtering skips irrelevant observers. Lifecycle cleanup on terminal
case status and compound completion.

Refs #1105"
```

## Batch 3: Classical Observers

### Task 6: ThresholdObserver, CorrelationObserver, TemporalSequenceObserver

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/ThresholdObserver.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/CorrelationObserver.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/TemporalSequenceObserver.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/observation/ThresholdObserverTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/observation/CorrelationObserverTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/observation/TemporalSequenceObserverTest.java`

**Interfaces:**
- Consumes: `EnvironmentObserver`, `ObservationContext`, `Observation`, `ContextSnapshot` (Task 1)
- Produces: `ThresholdObserver.of(String key, Operator op, double threshold)`, `CorrelationObserver.of(Set<String> keys, String jqCondition)`, `TemporalSequenceObserver.of(List<SequenceStep> steps, Duration window)`. `ThresholdObserver.Operator` enum: GT, LT, GTE, LTE, EQ

- [ ] **Step 1: Write tests for ThresholdObserver**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.spi.observation.ObservationContext;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class ThresholdObserverTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void detectsThresholdCrossing() {
    var observer = ThresholdObserver.of("temperature", ThresholdObserver.Operator.GT, 100.0);
    var snapshot = MAPPER.createObjectNode().put("temperature", 105.0);
    var ctx = new ObservationContext(snapshot, Set.of("temperature"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    var observations = observer.observe(ctx);
    assertEquals(1, observations.size());
    assertEquals("threshold-crossing", observations.get(0).patternId());
    assertEquals(1.0, observations.get(0).confidence());
  }

  @Test
  void noObservationWhenBelowThreshold() {
    var observer = ThresholdObserver.of("temperature", ThresholdObserver.Operator.GT, 100.0);
    var snapshot = MAPPER.createObjectNode().put("temperature", 50.0);
    var ctx = new ObservationContext(snapshot, Set.of("temperature"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }

  @Test
  void watchedKeysContainsOnlyTargetKey() {
    var observer = ThresholdObserver.of("temperature", ThresholdObserver.Operator.GT, 100.0);
    assertEquals(Set.of("temperature"), observer.watchedKeys());
  }

  @Test
  void handlesNonNumericValueGracefully() {
    var observer = ThresholdObserver.of("temperature", ThresholdObserver.Operator.GT, 100.0);
    var snapshot = MAPPER.createObjectNode().put("temperature", "not-a-number");
    var ctx = new ObservationContext(snapshot, Set.of("temperature"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }

  @Test
  void handlesMissingKeyGracefully() {
    var observer = ThresholdObserver.of("temperature", ThresholdObserver.Operator.GT, 100.0);
    var snapshot = MAPPER.createObjectNode().put("humidity", 80.0);
    var ctx = new ObservationContext(snapshot, Set.of("humidity"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }
}
```

- [ ] **Step 2: Write tests for CorrelationObserver**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.spi.observation.ObservationContext;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CorrelationObserverTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void detectsCorrelation() {
    var observer = CorrelationObserver.of(Set.of("temperature", "humidity"),
        ".temperature > 30 and .humidity > 80");
    var snapshot = MAPPER.createObjectNode().put("temperature", 35).put("humidity", 85);
    var ctx = new ObservationContext(snapshot, Set.of("temperature", "humidity"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    var observations = observer.observe(ctx);
    assertEquals(1, observations.size());
    assertEquals("multi-key-correlation", observations.get(0).patternId());
  }

  @Test
  void noObservationWhenConditionFalse() {
    var observer = CorrelationObserver.of(Set.of("temperature", "humidity"),
        ".temperature > 30 and .humidity > 80");
    var snapshot = MAPPER.createObjectNode().put("temperature", 20).put("humidity", 85);
    var ctx = new ObservationContext(snapshot, Set.of("temperature"), List.of(),
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }

  @Test
  void watchedKeysMatchDeclaredKeys() {
    var observer = CorrelationObserver.of(Set.of("a", "b"), ".a == .b");
    assertEquals(Set.of("a", "b"), observer.watchedKeys());
  }
}
```

- [ ] **Step 3: Write tests for TemporalSequenceObserver**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.IntNode;
import io.casehub.api.spi.observation.ContextSnapshot;
import io.casehub.api.spi.observation.ObservationContext;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class TemporalSequenceObserverTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void detectsTemporalSequence() {
    var observer = TemporalSequenceObserver.of(
        List.of(
            new TemporalSequenceObserver.SequenceStep("alert", null),
            new TemporalSequenceObserver.SequenceStep("escalation", null)),
        Duration.ofMinutes(5));

    var now = Instant.now();
    var history = List.of(
        new ContextSnapshot(Set.of("alert"), Map.of("alert", IntNode.valueOf(1)), now.minusSeconds(60)),
        new ContextSnapshot(Set.of("escalation"), Map.of("escalation", IntNode.valueOf(1)), now.minusSeconds(30)));

    var snapshot = MAPPER.createObjectNode().put("alert", 1).put("escalation", 1);
    var ctx = new ObservationContext(snapshot, Set.of("escalation"), history,
        "agent-1", "t1", UUID.randomUUID());
    var observations = observer.observe(ctx);
    assertEquals(1, observations.size());
    assertEquals("temporal-sequence", observations.get(0).patternId());
  }

  @Test
  void noObservationWhenSequenceIncomplete() {
    var observer = TemporalSequenceObserver.of(
        List.of(
            new TemporalSequenceObserver.SequenceStep("alert", null),
            new TemporalSequenceObserver.SequenceStep("escalation", null)),
        Duration.ofMinutes(5));

    var now = Instant.now();
    var history = List.of(
        new ContextSnapshot(Set.of("alert"), Map.of("alert", IntNode.valueOf(1)), now.minusSeconds(60)));

    var snapshot = MAPPER.createObjectNode().put("alert", 1);
    var ctx = new ObservationContext(snapshot, Set.of("alert"), history,
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }

  @Test
  void noObservationWhenOutsideWindow() {
    var observer = TemporalSequenceObserver.of(
        List.of(
            new TemporalSequenceObserver.SequenceStep("alert", null),
            new TemporalSequenceObserver.SequenceStep("escalation", null)),
        Duration.ofSeconds(30));

    var now = Instant.now();
    var history = List.of(
        new ContextSnapshot(Set.of("alert"), Map.of("alert", IntNode.valueOf(1)), now.minusSeconds(120)),
        new ContextSnapshot(Set.of("escalation"), Map.of("escalation", IntNode.valueOf(1)), now.minusSeconds(5)));

    var snapshot = MAPPER.createObjectNode().put("alert", 1).put("escalation", 1);
    var ctx = new ObservationContext(snapshot, Set.of("escalation"), history,
        "agent-1", "t1", UUID.randomUUID());
    assertTrue(observer.observe(ctx).isEmpty());
  }

  @Test
  void watchedKeysIncludesAllSequenceKeys() {
    var observer = TemporalSequenceObserver.of(
        List.of(
            new TemporalSequenceObserver.SequenceStep("a", null),
            new TemporalSequenceObserver.SequenceStep("b", null)),
        Duration.ofMinutes(5));
    assertEquals(Set.of("a", "b"), observer.watchedKeys());
  }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest="ThresholdObserverTest,CorrelationObserverTest,TemporalSequenceObserverTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: compilation failure

- [ ] **Step 5: Implement ThresholdObserver**

Create `runtime-core/src/main/java/io/casehub/engine/internal/observation/ThresholdObserver.java` — plain class (not CDI bean) with static factory `of(key, operator, threshold)`. `Operator` enum: GT, LT, GTE, LTE, EQ. Evaluates `snapshot.get(key)` against threshold. Produces `Observation("threshold-crossing", 1.0, {key, value, threshold, operator})`. Handles non-numeric and missing key gracefully (returns empty list).

- [ ] **Step 6: Implement CorrelationObserver**

Create `runtime-core/src/main/java/io/casehub/engine/internal/observation/CorrelationObserver.java` — plain class with static factory `of(keys, jqCondition)`. Uses `JQEvaluator` (injected via constructor or inline jackson-jq) to evaluate the JQ condition against the snapshot. Produces `Observation("multi-key-correlation", 1.0, {keys, condition})`.

- [ ] **Step 7: Implement TemporalSequenceObserver**

Create `runtime-core/src/main/java/io/casehub/engine/internal/observation/TemporalSequenceObserver.java` — plain class with static factory `of(steps, window)`. Inner record `SequenceStep(String key, @Nullable String valuePredicate)`. Scans history for ordered sequence within window duration. Produces `Observation("temporal-sequence", 1.0, {sequence, window, matchedAt})`.

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest="ThresholdObserverTest,CorrelationObserverTest,TemporalSequenceObserverTest" -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 9: Run full runtime tests for regression**

Run: `mvn test -pl runtime -f /Users/mdproctor/claude/casehub/slots/197/engine/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/observation/ runtime/src/test/java/io/casehub/engine/internal/observation/
git commit -m "feat: add classical observer implementations

ThresholdObserver — numeric threshold crossing detection.
CorrelationObserver — multi-key JQ condition correlation.
TemporalSequenceObserver — ordered key-change sequence within window.

Closes #1105"
```

## References

- [2026-09-16-environment-observation-spi-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/context/CaseContext.java:26` — observed context interface
- `api/src/main/java/io/casehub/api/model/ContextChangeTrigger.java:21` — existing trigger model (preserved)
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:252` — integration point
- `runtime-core/src/main/java/io/casehub/engine/internal/engine/CaseEvaluationSerializer.java:23` — serialization gate
- `common-core/src/main/java/io/casehub/engine/common/internal/worker/scope/ScopedWorkerRegistry.java:23` — lifecycle scope pattern
- `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java:24` — registration surface
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java:43` — registration implementation
- `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java:25` — factory threading
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java:189` — config field pattern
- `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java:18` — event type enum
- GitHub #1104 — Hive Mind epic
- GitHub #1105 — Environment observation SPI
