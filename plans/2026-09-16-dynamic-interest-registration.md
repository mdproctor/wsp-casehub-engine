# Dynamic Interest Registration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1107 — feat: Dynamic interest registration — agents register observation interests at runtime
**Issue group:** #1105, #1106, #1107

**Goal:** Restructure WorkerRuntime into domain-organized coordination facets (SignalSpace, InterestSpace) and add a declarative interest registration API with aggregate landscape visibility.

**Architecture:** Two new facet interfaces in `api/engine/` — `SignalSpace` (moved from flat methods) and `InterestSpace` (new interest registration + moved registerObserver). Flat coordination methods removed from WorkerRuntime (pre-release clean break). `InterestDeclaration` sealed hierarchy (5 permits) maps to existing classical observers. `InterestLandscape` provides anonymous aggregate view. `DefaultWorkerRuntime` delegates to `DefaultSignalSpace` and `DefaultInterestSpace`.

**Tech Stack:** Java 21, Quarkus 3.32.2, jackson-jq 1.6

## Global Constraints

- Pre-release: no backward compatibility — flat methods removed, not deprecated
- All new types in engine-api; implementations in runtime-core
- TDD: every production type has a corresponding test
- All tests run with `mvn test -pl api` or `mvn test -pl runtime-core`
- Use `ide_insert_member` for new methods, `ide_replace_member` for body rewrites
- Use `ide_move_file` for file moves, never bash cp/mv

---

## Batch 1: Foundation types + SignalSpace facet

### Task 1: InterestDeclaration sealed hierarchy + InterestRegistration + InterestLandscape

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/observation/InterestDeclaration.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/InterestRegistration.java`
- Create: `api/src/main/java/io/casehub/api/spi/observation/InterestLandscape.java`
- Test: `api/src/test/java/io/casehub/api/spi/observation/InterestDeclarationTest.java`
- Test: `api/src/test/java/io/casehub/api/spi/observation/InterestLandscapeTest.java`

**Interfaces:**
- Produces: `InterestDeclaration` sealed interface (5 permits: `KeyThreshold`, `KeyCorrelation`, `TemporalSequence`, `SignalThreshold`, `JqInterest`), `InterestRegistration(String interestId, InterestDeclaration declaration, Instant registeredAt)`, `InterestLandscape(Map<String,Integer> keyObserverCounts, Map<String,Integer> signalObserverCounts, Map<String,Integer> interestTypeCounts, int totalObserverCount)` with `EMPTY` constant

- [ ] **Step 1: Write failing tests for InterestDeclaration permits**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;
import java.time.Duration;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;

class InterestDeclarationTest {

  @Test
  void keyThresholdConstruction() {
    var interest = new InterestDeclaration.KeyThreshold("riskScore",
        InterestDeclaration.ComparisonOperator.GTE, 0.7);
    assertEquals("riskScore", interest.key());
    assertEquals(InterestDeclaration.ComparisonOperator.GTE, interest.operator());
    assertEquals(0.7, interest.threshold());
  }

  @Test
  void keyCorrelationConstruction() {
    var interest = new InterestDeclaration.KeyCorrelation(
        Set.of("amount", "country"), ".amount > 10000 and .country == \"US\"");
    assertEquals(Set.of("amount", "country"), interest.keys());
    assertEquals(".amount > 10000 and .country == \"US\"", interest.jqCondition());
  }

  @Test
  void temporalSequenceConstruction() {
    var steps = List.of(
        new InterestDeclaration.SequenceStep("login", null),
        new InterestDeclaration.SequenceStep("transfer", null));
    var interest = new InterestDeclaration.TemporalSequence(steps, Duration.ofMinutes(5));
    assertEquals(2, interest.steps().size());
    assertEquals(Duration.ofMinutes(5), interest.window());
  }

  @Test
  void signalThresholdConstruction() {
    var interest = new InterestDeclaration.SignalThreshold("danger",
        InterestDeclaration.ComparisonOperator.GT, 0.5);
    assertEquals("danger", interest.signalName());
    assertEquals(InterestDeclaration.ComparisonOperator.GT, interest.operator());
  }

  @Test
  void jqInterestConstruction() {
    var interest = new InterestDeclaration.JqInterest(
        ".riskScore > .threshold", Set.of("riskScore", "threshold"));
    assertEquals(".riskScore > .threshold", interest.expression());
    assertEquals(Set.of("riskScore", "threshold"), interest.watchedKeys());
  }

  @Test
  void sealedHierarchyPatternMatching() {
    InterestDeclaration interest = new InterestDeclaration.KeyThreshold("x",
        InterestDeclaration.ComparisonOperator.GT, 1.0);
    String type = switch (interest) {
      case InterestDeclaration.KeyThreshold t -> "threshold";
      case InterestDeclaration.KeyCorrelation c -> "correlation";
      case InterestDeclaration.TemporalSequence s -> "sequence";
      case InterestDeclaration.SignalThreshold s -> "signal";
      case InterestDeclaration.JqInterest j -> "jq";
    };
    assertEquals("threshold", type);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=InterestDeclarationTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement InterestDeclaration**

```java
package io.casehub.api.spi.observation;

import java.time.Duration;
import java.util.List;
import java.util.Set;

public sealed interface InterestDeclaration
    permits InterestDeclaration.KeyThreshold,
            InterestDeclaration.KeyCorrelation,
            InterestDeclaration.TemporalSequence,
            InterestDeclaration.SignalThreshold,
            InterestDeclaration.JqInterest {

  enum ComparisonOperator { GT, LT, GTE, LTE, EQ }

  record SequenceStep(String key, String valuePredicate) {}

  record KeyThreshold(String key, ComparisonOperator operator, double threshold)
      implements InterestDeclaration {}

  record KeyCorrelation(Set<String> keys, String jqCondition)
      implements InterestDeclaration {}

  record TemporalSequence(List<SequenceStep> steps, Duration window)
      implements InterestDeclaration {}

  record SignalThreshold(String signalName, ComparisonOperator operator, double threshold)
      implements InterestDeclaration {}

  record JqInterest(String expression, Set<String> watchedKeys)
      implements InterestDeclaration {}
}
```

- [ ] **Step 4: Implement InterestRegistration**

```java
package io.casehub.api.spi.observation;

import java.time.Instant;

public record InterestRegistration(
    String interestId,
    InterestDeclaration declaration,
    Instant registeredAt) {}
```

- [ ] **Step 5: Write failing tests for InterestLandscape**

```java
package io.casehub.api.spi.observation;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class InterestLandscapeTest {

  @Test
  void emptyLandscape() {
    var landscape = InterestLandscape.EMPTY;
    assertTrue(landscape.keyObserverCounts().isEmpty());
    assertTrue(landscape.signalObserverCounts().isEmpty());
    assertTrue(landscape.interestTypeCounts().isEmpty());
    assertEquals(0, landscape.totalObserverCount());
  }

  @Test
  void populatedLandscape() {
    var landscape = new InterestLandscape(
        Map.of("riskScore", 3, "amount", 1),
        Map.of("danger", 2),
        Map.of("threshold", 4, "correlation", 2),
        8);
    assertEquals(3, landscape.keyObserverCounts().get("riskScore"));
    assertEquals(2, landscape.signalObserverCounts().get("danger"));
    assertEquals(8, landscape.totalObserverCount());
  }
}
```

- [ ] **Step 6: Implement InterestLandscape**

```java
package io.casehub.api.spi.observation;

import java.util.Map;

public record InterestLandscape(
    Map<String, Integer> keyObserverCounts,
    Map<String, Integer> signalObserverCounts,
    Map<String, Integer> interestTypeCounts,
    int totalObserverCount) {

  public static final InterestLandscape EMPTY =
      new InterestLandscape(Map.of(), Map.of(), Map.of(), 0);
}
```

- [ ] **Step 7: Run all tests to verify they pass**

Run: `mvn test -pl api -Dtest="InterestDeclarationTest,InterestLandscapeTest" -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/InterestDeclaration.java api/src/main/java/io/casehub/api/spi/observation/InterestRegistration.java api/src/main/java/io/casehub/api/spi/observation/InterestLandscape.java api/src/test/java/io/casehub/api/spi/observation/InterestDeclarationTest.java api/src/test/java/io/casehub/api/spi/observation/InterestLandscapeTest.java
git commit -m "feat: add InterestDeclaration sealed hierarchy, InterestRegistration, InterestLandscape Refs #1107"
```

### Task 2: SignalSpace + InterestSpace facet interfaces + WorkerRuntime restructure

**Files:**
- Create: `api/src/main/java/io/casehub/api/engine/SignalSpace.java`
- Create: `api/src/main/java/io/casehub/api/engine/InterestSpace.java`
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java:34-44` (remove flat methods, add facet accessors)
- Test: `api/src/test/java/io/casehub/api/engine/SignalSpaceTest.java`
- Test: `api/src/test/java/io/casehub/api/engine/InterestSpaceTest.java`

**Interfaces:**
- Consumes: `InterestDeclaration`, `InterestRegistration`, `InterestLandscape` (from Task 1), `EnvironmentObserver` (existing), `PerceivedSignal` (existing)
- Produces: `SignalSpace` interface with `NOOP`, `InterestSpace` interface with `NOOP`, updated `WorkerRuntime` with `signals()` and `interests()` facet methods

- [ ] **Step 1: Write failing tests for NOOP implementations**

```java
package io.casehub.api.engine;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class SignalSpaceTest {

  @Test
  void noopDepositDoesNothing() {
    assertDoesNotThrow(() -> SignalSpace.NOOP.deposit("test", 1.0));
  }

  @Test
  void noopDepositWithHalfLifeDoesNothing() {
    assertDoesNotThrow(() ->
        SignalSpace.NOOP.deposit("test", 1.0, java.time.Duration.ofMinutes(5)));
  }

  @Test
  void noopPerceiveReturnsEmpty() {
    assertTrue(SignalSpace.NOOP.perceive().isEmpty());
  }
}
```

```java
package io.casehub.api.engine;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.spi.observation.InterestDeclaration;
import io.casehub.api.spi.observation.InterestLandscape;
import org.junit.jupiter.api.Test;

class InterestSpaceTest {

  @Test
  void noopRegisterReturnsSentinel() {
    var reg = InterestSpace.NOOP.register(
        new InterestDeclaration.KeyThreshold("x",
            InterestDeclaration.ComparisonOperator.GT, 1.0));
    assertNotNull(reg);
    assertEquals("noop-0", reg.interestId());
  }

  @Test
  void noopRegisterObserverReturnsFalse() {
    assertFalse(InterestSpace.NOOP.registerObserver(ctx -> java.util.List.of()));
  }

  @Test
  void noopDeregisterDoesNothing() {
    assertDoesNotThrow(() -> InterestSpace.NOOP.deregister("any-id"));
  }

  @Test
  void noopMineReturnsEmpty() {
    assertTrue(InterestSpace.NOOP.mine().isEmpty());
  }

  @Test
  void noopLandscapeReturnsEmpty() {
    assertEquals(InterestLandscape.EMPTY, InterestSpace.NOOP.landscape());
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest="SignalSpaceTest,InterestSpaceTest" -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — classes not found

- [ ] **Step 3: Implement SignalSpace**

```java
package io.casehub.api.engine;

import io.casehub.api.model.signal.PerceivedSignal;
import java.time.Duration;
import java.util.Map;

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

- [ ] **Step 4: Implement InterestSpace**

```java
package io.casehub.api.engine;

import io.casehub.api.spi.observation.EnvironmentObserver;
import io.casehub.api.spi.observation.InterestDeclaration;
import io.casehub.api.spi.observation.InterestLandscape;
import io.casehub.api.spi.observation.InterestRegistration;
import java.time.Instant;
import java.util.List;

public interface InterestSpace {
  InterestRegistration register(InterestDeclaration interest);
  boolean registerObserver(EnvironmentObserver observer);
  void deregister(String interestId);
  List<InterestRegistration> mine();
  InterestLandscape landscape();

  InterestSpace NOOP = new InterestSpace() {
    @Override public InterestRegistration register(InterestDeclaration interest) {
      return new InterestRegistration("noop-0", interest, Instant.now());
    }
    @Override public boolean registerObserver(EnvironmentObserver observer) { return false; }
    @Override public void deregister(String interestId) {}
    @Override public List<InterestRegistration> mine() { return List.of(); }
    @Override public InterestLandscape landscape() { return InterestLandscape.EMPTY; }
  };
}
```

- [ ] **Step 5: Update WorkerRuntime — remove flat methods, add facet accessors**

Replace the entire body of `WorkerRuntime` (lines 24-45) with:

```java
public interface WorkerRuntime extends io.casehub.worker.api.WorkerScope {

  io.casehub.worker.api.WorkerContext context();

  java.util.UUID spawnCase(String caseType, java.util.Map<String, Object> input);

  io.casehub.api.context.CaseContext awaitCase(
      java.util.UUID childCaseId, java.time.Duration timeout);

  io.casehub.api.context.CaseContext spawnAndAwaitCase(
      String caseType, java.util.Map<String, Object> input, java.time.Duration timeout);

  default SignalSpace signals() { return SignalSpace.NOOP; }

  default InterestSpace interests() { return InterestSpace.NOOP; }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="SignalSpaceTest,InterestSpaceTest" -q`
Expected: PASS

- [ ] **Step 7: Verify compilation across affected modules**

Run: `mvn compile -pl api,common-core,runtime-core -q`
Expected: compilation failures in `DefaultWorkerRuntime` (flat methods now have no interface method to override). These are expected and will be fixed in Task 3.

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/SignalSpace.java api/src/main/java/io/casehub/api/engine/InterestSpace.java api/src/main/java/io/casehub/api/engine/WorkerRuntime.java api/src/test/java/io/casehub/api/engine/SignalSpaceTest.java api/src/test/java/io/casehub/api/engine/InterestSpaceTest.java
git commit -m "feat: add SignalSpace + InterestSpace facets, restructure WorkerRuntime Refs #1107"
```

### Task 3: DefaultSignalSpace + DefaultInterestSpace + DefaultWorkerRuntime migration

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/signal/DefaultSignalSpace.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultInterestSpace.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java:62-63,94-125,286-343` (remove signal/observer fields and methods, add facet fields and accessors)
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java:79-103` (create facet instances, pass to DefaultWorkerRuntime)
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/signal/DefaultSignalSpaceTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultInterestSpaceTest.java`

**Interfaces:**
- Consumes: `SignalSpace` (Task 2), `InterestSpace` (Task 2), `SignalRegistry` (existing), `ObservationRegistry` (existing), `InterestDeclaration` (Task 1), `InterestRegistration` (Task 1), `InterestLandscape` (Task 1)
- Produces: `DefaultSignalSpace` (deposit/perceive delegation to SignalRegistry), `DefaultInterestSpace` (register/deregister/mine/landscape delegation to ObservationRegistry), updated `DefaultWorkerRuntime` with facet fields

- [ ] **Step 1: Write failing tests for DefaultSignalSpace**

```java
package io.casehub.engine.internal.signal;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.model.signal.SignalConfig;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultSignalSpaceTest {

  private SignalRegistry registry;
  private DefaultSignalSpace signalSpace;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    registry = new SignalRegistry();
    var config = new SignalConfig(
        Duration.ofMinutes(5), 0.01, 100);
    signalSpace = new DefaultSignalSpace(
        registry, caseId, "agent-1", config);
  }

  @Test
  void depositAndPerceive() {
    signalSpace.deposit("food", 1.0);
    var signals = signalSpace.perceive();
    assertEquals(1, signals.size());
    assertTrue(signals.containsKey("food"));
    assertTrue(signals.get("food").effectiveStrength() > 0.9);
  }

  @Test
  void depositWithCustomHalfLife() {
    signalSpace.deposit("danger", 0.8, Duration.ofSeconds(30));
    var signals = signalSpace.perceive();
    assertTrue(signals.containsKey("danger"));
  }

  @Test
  void perceiveFiltersExpired() {
    registry.deposit(caseId, "old", 0.001, Duration.ofMillis(1), "agent-1", 100);
    try { Thread.sleep(10); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    var signals = signalSpace.perceive();
    assertFalse(signals.containsKey("old"));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime-core -Dtest=DefaultSignalSpaceTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DefaultSignalSpace**

```java
package io.casehub.engine.internal.signal;

import io.casehub.api.engine.SignalSpace;
import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.model.signal.SignalConfig;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;

public class DefaultSignalSpace implements SignalSpace {

  private final SignalRegistry signalRegistry;
  private final UUID caseId;
  private final String workerName;
  private final SignalConfig config;

  public DefaultSignalSpace(
      SignalRegistry signalRegistry, UUID caseId,
      String workerName, SignalConfig config) {
    this.signalRegistry = signalRegistry;
    this.caseId = caseId;
    this.workerName = workerName;
    this.config = config;
  }

  @Override
  public void deposit(String name, double strength) {
    deposit(name, strength, config.defaultHalfLife());
  }

  @Override
  public void deposit(String name, double strength, Duration halfLife) {
    signalRegistry.deposit(caseId, name, strength, halfLife,
        workerName, config.maxSignalsPerCase());
  }

  @Override
  public Map<String, PerceivedSignal> perceive() {
    return signalRegistry.perceive(caseId, config.effectiveZeroThreshold());
  }
}
```

- [ ] **Step 4: Write failing tests for DefaultInterestSpace**

```java
package io.casehub.engine.internal.observation;

import static org.junit.jupiter.api.Assertions.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultInterestSpaceTest {

  private ObservationRegistry registry;
  private DefaultInterestSpace interestSpace;
  private final UUID caseId = UUID.randomUUID();

  @BeforeEach
  void setUp() {
    registry = new ObservationRegistry();
    interestSpace = new DefaultInterestSpace(
        registry, caseId, "agent-1", null,
        ObservationConfig.defaults());
  }

  @Test
  void registerKeyThresholdCreatesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.KeyThreshold("riskScore",
            InterestDeclaration.ComparisonOperator.GTE, 0.7));
    assertNotNull(reg);
    assertNotNull(reg.interestId());
    assertNotNull(reg.registeredAt());
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void registerSignalThresholdCreatesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.SignalThreshold("danger",
            InterestDeclaration.ComparisonOperator.GT, 0.5));
    assertNotNull(reg);
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void registerKeyCorrelationCreatesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.KeyCorrelation(
            java.util.Set.of("a", "b"), ".a > .b"));
    assertNotNull(reg);
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void registerTemporalSequenceCreatesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.TemporalSequence(
            java.util.List.of(
                new InterestDeclaration.SequenceStep("login", null),
                new InterestDeclaration.SequenceStep("transfer", null)),
            java.time.Duration.ofMinutes(5)));
    assertNotNull(reg);
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void registerJqInterestCreatesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.JqInterest(".x > 1",
            java.util.Set.of("x")));
    assertNotNull(reg);
    assertEquals(1, registry.observerCount(caseId));
  }

  @Test
  void jqInterestRejectedWhenDisabled() {
    var config = new ObservationConfig(50,
        java.time.Duration.ofMinutes(5), 20, false);
    var restricted = new DefaultInterestSpace(
        registry, caseId, "agent-1", null, config);
    assertThrows(IllegalArgumentException.class, () ->
        restricted.register(new InterestDeclaration.JqInterest(
            ".x > 1", java.util.Set.of("x"))));
  }

  @Test
  void deregisterRemovesObserver() {
    var reg = interestSpace.register(
        new InterestDeclaration.KeyThreshold("x",
            InterestDeclaration.ComparisonOperator.GT, 1.0));
    assertEquals(1, registry.observerCount(caseId));
    interestSpace.deregister(reg.interestId());
    assertEquals(0, registry.observerCount(caseId));
  }

  @Test
  void mineReturnsOwnRegistrations() {
    interestSpace.register(new InterestDeclaration.KeyThreshold("a",
        InterestDeclaration.ComparisonOperator.GT, 1.0));
    interestSpace.register(new InterestDeclaration.KeyThreshold("b",
        InterestDeclaration.ComparisonOperator.LT, 0.5));
    var mine = interestSpace.mine();
    assertEquals(2, mine.size());
  }

  @Test
  void landscapeAggregatesCorrectly() {
    interestSpace.register(new InterestDeclaration.KeyThreshold("riskScore",
        InterestDeclaration.ComparisonOperator.GT, 0.7));
    interestSpace.register(new InterestDeclaration.SignalThreshold("danger",
        InterestDeclaration.ComparisonOperator.GTE, 0.5));
    var landscape = interestSpace.landscape();
    assertEquals(1, landscape.keyObserverCounts().getOrDefault("riskScore", 0));
    assertEquals(1, landscape.signalObserverCounts().getOrDefault("danger", 0));
    assertEquals(2, landscape.totalObserverCount());
  }
}
```

- [ ] **Step 5: Implement DefaultInterestSpace**

```java
package io.casehub.engine.internal.observation;

import io.casehub.api.engine.InterestSpace;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.internal.observation.ThresholdObserver;
import io.casehub.engine.internal.observation.CorrelationObserver;
import io.casehub.engine.internal.observation.TemporalSequenceObserver;
import io.casehub.engine.internal.observation.SignalStrengthObserver;
import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

public class DefaultInterestSpace implements InterestSpace {

  private final ObservationRegistry registry;
  private final UUID caseId;
  private final String agentId;
  private final String bindingName;
  private final ObservationConfig config;
  private final List<InterestRegistration> registrations =
      Collections.synchronizedList(new ArrayList<>());

  public DefaultInterestSpace(
      ObservationRegistry registry, UUID caseId, String agentId,
      String bindingName, ObservationConfig config) {
    this.registry = registry;
    this.caseId = caseId;
    this.agentId = agentId;
    this.bindingName = bindingName;
    this.config = config;
  }

  @Override
  public InterestRegistration register(InterestDeclaration interest) {
    if (interest instanceof InterestDeclaration.JqInterest && !config.allowJqInterests()) {
      throw new IllegalArgumentException(
          "JQ interests are disabled for this case definition (allowJqInterests=false)");
    }

    EnvironmentObserver observer = createObserver(interest);
    String instanceId = registry.registerObserver(
        caseId, agentId, bindingName, observer, config.maxObserversPerCase());
    if (instanceId == null) {
      return null;
    }

    var reg = new InterestRegistration(instanceId, interest, Instant.now());
    registrations.add(reg);
    return reg;
  }

  @Override
  public boolean registerObserver(EnvironmentObserver observer) {
    return registry.registerObserver(
        caseId, agentId, bindingName, observer, config.maxObserversPerCase());
  }

  @Override
  public void deregister(String interestId) {
    registrations.removeIf(r -> r.interestId().equals(interestId));
    registry.deregisterByInstanceId(caseId, interestId);
  }

  @Override
  public List<InterestRegistration> mine() {
    return List.copyOf(registrations);
  }

  @Override
  public InterestLandscape landscape() {
    return registry.computeLandscape(caseId);
  }

  private EnvironmentObserver createObserver(InterestDeclaration interest) {
    return switch (interest) {
      case InterestDeclaration.KeyThreshold t ->
          ThresholdObserver.of(t.key(), mapOperator(t.operator()), t.threshold());
      case InterestDeclaration.KeyCorrelation c ->
          CorrelationObserver.of(c.keys(), c.jqCondition());
      case InterestDeclaration.TemporalSequence s ->
          TemporalSequenceObserver.of(
              s.steps().stream()
                  .map(step -> new TemporalSequenceObserver.SequenceStep(
                      step.key(), step.valuePredicate()))
                  .toList(),
              s.window());
      case InterestDeclaration.SignalThreshold s ->
          SignalStrengthObserver.of(s.signalName(), mapOperator(s.operator()), s.threshold());
      case InterestDeclaration.JqInterest j ->
          CorrelationObserver.of(j.watchedKeys(), j.expression());
    };
  }

  private ThresholdObserver.Operator mapOperator(InterestDeclaration.ComparisonOperator op) {
    return switch (op) {
      case GT -> ThresholdObserver.Operator.GT;
      case LT -> ThresholdObserver.Operator.LT;
      case GTE -> ThresholdObserver.Operator.GTE;
      case LTE -> ThresholdObserver.Operator.LTE;
      case EQ -> ThresholdObserver.Operator.EQ;
    };
  }
}
```

- [ ] **Step 6: Add ObservationConfig 4-arg constructor with allowJqInterests**

Modify `api/src/main/java/io/casehub/api/spi/observation/ObservationConfig.java` — add the 4th field and backward-compatible constructor:

```java
public record ObservationConfig(
    int maxHistoryEntries, java.time.Duration maxHistoryAge,
    int maxObserversPerCase, boolean allowJqInterests) {

  public static final int DEFAULT_MAX_HISTORY_ENTRIES = 50;
  public static final java.time.Duration DEFAULT_MAX_HISTORY_AGE =
      java.time.Duration.ofMinutes(5);
  public static final int DEFAULT_MAX_OBSERVERS_PER_CASE = 20;

  public ObservationConfig(int maxHistoryEntries,
      java.time.Duration maxHistoryAge, int maxObserversPerCase) {
    this(maxHistoryEntries, maxHistoryAge, maxObserversPerCase, true);
  }

  public static ObservationConfig defaults() {
    return new ObservationConfig(
        DEFAULT_MAX_HISTORY_ENTRIES, DEFAULT_MAX_HISTORY_AGE,
        DEFAULT_MAX_OBSERVERS_PER_CASE, true);
  }
}
```

- [ ] **Step 7: Add registry extensions — registerObserver returns instanceId, deregisterByInstanceId, computeLandscape**

Update `ObservationRegistry.registerObserver()` to return the generated `instanceId` (currently returns `boolean`). Change return type to `String` — returns the instanceId on success, `null` when at capacity. Update all call sites.

Add to `ObservationRegistry`:

```java
public void deregisterByInstanceId(UUID caseId, String instanceId) {
  var caseRegistrations = registrations.get(caseId);
  if (caseRegistrations != null) {
    synchronized (caseRegistrations) {
      caseRegistrations.removeIf(r -> r.instanceId().equals(instanceId));
    }
  }
}

public InterestLandscape computeLandscape(UUID caseId) {
  var caseRegistrations = registrations.get(caseId);
  if (caseRegistrations == null) {
    return InterestLandscape.EMPTY;
  }
  Map<String, Integer> keyCounts = new LinkedHashMap<>();
  Map<String, Integer> signalCounts = new LinkedHashMap<>();
  Map<String, Integer> typeCounts = new LinkedHashMap<>();
  int total;
  synchronized (caseRegistrations) {
    total = caseRegistrations.size();
    for (var reg : caseRegistrations) {
      typeCounts.merge(reg.observer().observerType(), 1, Integer::sum);
      for (String key : reg.observer().watchedKeys()) {
        keyCounts.merge(key, 1, Integer::sum);
      }
      if ("signal-strength".equals(reg.observer().observerType())) {
        signalCounts.merge("signal", 1, Integer::sum);
      }
    }
  }
  return new InterestLandscape(
      Map.copyOf(keyCounts), Map.copyOf(signalCounts),
      Map.copyOf(typeCounts), total);
}
```

- [ ] **Step 8: Migrate DefaultWorkerRuntime — replace signal/observer methods with facet fields**

Remove from `DefaultWorkerRuntime`:
- Fields: `signalRegistry` (line 62), `signalConfig` (line 63)
- The 15-arg constructor's signal/observer parameters
- Methods: `registerObserver()` (lines 286-309), `depositSignal()` (lines 312-319, 322-331), `perceiveSignals()` (lines 334-343)

Add:
- Fields: `private final SignalSpace signalSpace;` and `private final InterestSpace interestSpace;`
- Methods: `@Override public SignalSpace signals() { return signalSpace; }` and `@Override public InterestSpace interests() { return interestSpace; }`
- Update constructors to accept and store the facet instances

- [ ] **Step 9: Update WorkerRuntimeFactory to create facet instances**

In `WorkerRuntimeFactory.create()` (the 6-arg overload at line 79): create `DefaultSignalSpace` and `DefaultInterestSpace` with the resolved `SignalConfig` and `ObservationConfig`, pass them to `DefaultWorkerRuntime`.

- [ ] **Step 10: Run all tests to verify compilation and passing**

Run: `mvn test -pl api,common-core,runtime-core -q`
Expected: PASS (existing tests still pass after migration)

- [ ] **Step 11: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/signal/DefaultSignalSpace.java runtime-core/src/main/java/io/casehub/engine/internal/observation/DefaultInterestSpace.java runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java common-core/src/main/java/io/casehub/engine/common/internal/observation/ObservationRegistry.java api/src/main/java/io/casehub/api/spi/observation/ObservationConfig.java runtime-core/src/test/java/io/casehub/engine/internal/signal/DefaultSignalSpaceTest.java runtime-core/src/test/java/io/casehub/engine/internal/observation/DefaultInterestSpaceTest.java
git commit -m "feat: implement DefaultSignalSpace + DefaultInterestSpace, migrate DefaultWorkerRuntime to facets Refs #1107"
```

## Batch 2: Pipeline integration + YAML + CLAUDE.md

### Task 4: ObservationContext gains interestLandscape + handler wiring + allowJqInterests YAML

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/observation/ObservationContext.java:23-41` (add 8th field)
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:1230-1238` (compute landscape, pass to ObservationContext)
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java:1142-1157` (parse allowJqInterests)
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add INTEREST_REGISTERED, INTEREST_DEREGISTERED to CaseHubEventType)
- Test: `api/src/test/java/io/casehub/api/spi/observation/ObservationContextTest.java` (update existing or create)
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` (add allowJqInterests test)

**Interfaces:**
- Consumes: `InterestLandscape` (Task 1), `ObservationRegistry.computeLandscape()` (Task 3)
- Produces: Updated `ObservationContext` with `interestLandscape()` field, YAML parsing for `allowJqInterests`

- [ ] **Step 1: Write failing test for ObservationContext 8-arg constructor**

```java
@Test
void eightArgConstructorIncludesLandscape() {
  var landscape = new InterestLandscape(
      Map.of("x", 2), Map.of(), Map.of("threshold", 2), 2);
  var ctx = new ObservationContext(
      null, Set.of(), List.of(), "agent", "tenant",
      UUID.randomUUID(), Map.of(), landscape);
  assertEquals(landscape, ctx.interestLandscape());
}

@Test
void sevenArgConstructorDefaultsEmptyLandscape() {
  var ctx = new ObservationContext(
      null, Set.of(), List.of(), "agent", "tenant",
      UUID.randomUUID(), Map.of());
  assertEquals(InterestLandscape.EMPTY, ctx.interestLandscape());
}

@Test
void sixArgConstructorDefaultsEmptyLandscape() {
  var ctx = new ObservationContext(
      null, Set.of(), List.of(), "agent", "tenant", UUID.randomUUID());
  assertEquals(InterestLandscape.EMPTY, ctx.interestLandscape());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=ObservationContextTest -Dsurefire.failIfNoSpecifiedTests=false -q`
Expected: FAIL — no 8-arg constructor

- [ ] **Step 3: Update ObservationContext with 8th field**

Replace `ObservationContext.java` record definition:

```java
public record ObservationContext(
    com.fasterxml.jackson.databind.JsonNode snapshot,
    java.util.Set<String> changedKeys,
    java.util.List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    java.util.UUID caseId,
    java.util.Map<String, io.casehub.api.model.signal.PerceivedSignal> signals,
    InterestLandscape interestLandscape) {

  public ObservationContext(
      com.fasterxml.jackson.databind.JsonNode snapshot,
      java.util.Set<String> changedKeys,
      java.util.List<ContextSnapshot> history,
      String agentId, String tenancyId, java.util.UUID caseId) {
    this(snapshot, changedKeys, history, agentId, tenancyId, caseId,
        java.util.Map.of(), InterestLandscape.EMPTY);
  }

  public ObservationContext(
      com.fasterxml.jackson.databind.JsonNode snapshot,
      java.util.Set<String> changedKeys,
      java.util.List<ContextSnapshot> history,
      String agentId, String tenancyId, java.util.UUID caseId,
      java.util.Map<String, io.casehub.api.model.signal.PerceivedSignal> signals) {
    this(snapshot, changedKeys, history, agentId, tenancyId, caseId,
        signals, InterestLandscape.EMPTY);
  }
}
```

- [ ] **Step 4: Update CaseContextChangedEventHandler.observations() — compute and pass landscape**

In `observations()` method, after line 1213 (where `observers` is fetched), add landscape computation:

```java
io.casehub.api.spi.observation.InterestLandscape landscape =
    observationRegistry.computeLandscape(caseInstance.getUuid());
```

Then update the `ObservationContext` constructor call at line 1231 to pass `landscape` as the 8th argument:

```java
io.casehub.api.spi.observation.ObservationContext ctx =
    new io.casehub.api.spi.observation.ObservationContext(
        snapshot, changedKeys, history, agentId,
        caseInstance.tenancyId, caseInstance.getUuid(),
        signals, landscape);
```

- [ ] **Step 5: Update YamlCaseDefinitionConverter.convertObservation() — parse allowJqInterests**

At `YamlCaseDefinitionConverter.java:1142-1157`, update `convertObservation()`:

```java
private static io.casehub.api.spi.observation.ObservationConfig convertObservation(
    com.fasterxml.jackson.databind.JsonNode node) {
  int maxHistoryEntries = node.has("maxHistoryEntries")
      ? node.get("maxHistoryEntries").asInt()
      : io.casehub.api.spi.observation.ObservationConfig.DEFAULT_MAX_HISTORY_ENTRIES;
  java.time.Duration maxHistoryAge = node.has("maxHistoryAge")
      ? java.time.Duration.parse(node.get("maxHistoryAge").asText())
      : io.casehub.api.spi.observation.ObservationConfig.DEFAULT_MAX_HISTORY_AGE;
  int maxObserversPerCase = node.has("maxObserversPerCase")
      ? node.get("maxObserversPerCase").asInt()
      : io.casehub.api.spi.observation.ObservationConfig.DEFAULT_MAX_OBSERVERS_PER_CASE;
  boolean allowJqInterests = !node.has("allowJqInterests")
      || node.get("allowJqInterests").asBoolean(true);
  return new io.casehub.api.spi.observation.ObservationConfig(
      maxHistoryEntries, maxHistoryAge, maxObserversPerCase, allowJqInterests);
}
```

- [ ] **Step 6: Add CaseHubEventType entries for interest audit**

Add to the `CaseHubEventType` enum:

```java
INTEREST_REGISTERED,
INTEREST_DEREGISTERED,
```

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl api,common-core,runtime-core -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/ObservationContext.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java
git commit -m "feat: add interestLandscape to ObservationContext, YAML allowJqInterests parsing, audit event types Refs #1107"
```

### Task 5: CLAUDE.md update + full test suite verification

**Files:**
- Modify: `CLAUDE.md` (add Dynamic Interest Registration section)

**Interfaces:**
- Consumes: All types from Tasks 1-4

- [ ] **Step 1: Run full test suite across all affected modules**

Run: `mvn test -pl api,common-core,runtime-core,planning -q`
Expected: PASS — all existing and new tests green

- [ ] **Step 2: Update CLAUDE.md with Dynamic Interest Registration section**

Add after the "## Signal/Pheromone Model" section, a new section documenting the faceted WorkerRuntime architecture, InterestDeclaration types, InterestSpace API, InterestLandscape, and ObservationConfig.allowJqInterests.

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add dynamic interest registration to CLAUDE.md Refs #1107"
```

## References

- [2026-09-16-dynamic-interest-registration-design.md] — design spec this plan implements
- [WorkerRuntime.java:24-45] — current flat coordination surface being restructured
- [DefaultWorkerRuntime.java:286-343] — signal/observer methods being extracted to facets
- [ObservationRegistry.java] — per-case observer storage gaining deregister/landscape methods
- [ObservationConfig.java:20-31] — gaining allowJqInterests 4th field
- [ThresholdObserver.java, CorrelationObserver.java, TemporalSequenceObserver.java, SignalStrengthObserver.java] — classical observers mapped from InterestDeclaration permits
- [WorkerRuntimeFactory.java:79-103] — factory creating facet instances
- [CaseContextChangedEventHandler.java:1171-1267] — observation pipeline gaining landscape computation
- [YamlCaseDefinitionConverter.java:1142-1157] — YAML parsing for allowJqInterests
- [D19-D28 in decisions.md] — design decisions
- [GitHub #1107] — focal issue
- [GitHub #1105, #1106] — foundation issues (observation SPI, signal model)
