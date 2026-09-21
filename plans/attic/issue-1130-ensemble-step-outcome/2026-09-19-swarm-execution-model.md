# Swarm Execution Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1112 — feat: Swarm execution model — self-organizing agents with role emergence
**Issue group:** #1104 (Hive Mind epic), #1105-#1111 (foundation SPIs + stigmergy)

**Goal:** Extend the stigmergy execution model with behavioral fingerprinting, role emergence detection, team affinity clustering, swarm progress tracking, and agent-visible system metrics via the MetricsSpace facet.

**Architecture:** Four new `@ApplicationScoped` beans (RoleTracker, TeamDetector, SwarmProgressTracker, DefaultMetricsSpace) extend stigmergy with detection-only intelligence. All query existing registries — no new storage systems. MetricsSpace is the 5th WorkerRuntime facet, read-only. SwarmConfig nests inside StigmergyConfig with presence-as-activation. Seven new CaseHubEventType values capture role/team/progress events.

**Tech Stack:** Java 21+, Quarkus 3.32.2, JUnit 5

## Global Constraints

- All new types in existing stigmergy packages (no new packages)
- All tracker state is in-memory only (ConcurrentHashMap, per D29)
- All trackers implement `Resettable` for demo/test replay
- Use `ReentrantLock` instead of `synchronized` (per D72, virtual thread compat)
- Tests must be `*Test.java` (never `*IT.java`)
- All commits reference `Refs #1112`
- Maven compile: `/opt/homebrew/bin/mvn compile -pl <module> -q`
- Maven test: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl <module>`

---

## Batch 1: API Types

### Task 1: Foundation types — SwarmConfig, BehavioralFingerprint, MetricsSpace, events

All API-level type definitions. No logic — records, interfaces, enum values.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/SwarmConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/RoleDomainWeights.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/BehavioralFingerprint.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/DetectedRole.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/DetectedTeam.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/SwarmProgress.java`
- Create: `api/src/main/java/io/casehub/api/engine/MetricsSpace.java`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/StigmergyConfig.java:20` — add `swarm` field
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java:49` — add `metrics()` default method
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java:120` — add 7 enum values

**Interfaces:**
- Consumes: `StigmergyConfig`, `StigmergyDefaults`, `CoordinationConfig` (existing records)
- Produces: `SwarmConfig`, `RoleDomainWeights`, `BehavioralFingerprint`, `DetectedRole`, `DetectedTeam`, `SwarmProgress`, `MetricsSpace` — all consumed by Batch 2-4

- [ ] **Step 1: Create SwarmConfig record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record SwarmConfig(
    @Nullable Integer maxSwarmSize,
    @Nullable Double roleSimilarityThreshold,
    @Nullable Integer roleMinClusterSize,
    @Nullable Integer roleDetectionWindow,
    @Nullable Integer detectionInterval,
    @Nullable RoleDomainWeights domainWeights,
    @Nullable Double teamAffinityThreshold,
    @Nullable Integer teamMinSize,
    @Nullable Double progressChangeThreshold) {

  public double effectiveRoleSimilarityThreshold() {
    return roleSimilarityThreshold != null ? roleSimilarityThreshold : 0.7;
  }

  public int effectiveRoleMinClusterSize() {
    return roleMinClusterSize != null ? roleMinClusterSize : 2;
  }

  public int effectiveRoleDetectionWindow() {
    return roleDetectionWindow != null ? roleDetectionWindow : 20;
  }

  public int effectiveDetectionInterval() {
    return detectionInterval != null ? detectionInterval : 10;
  }

  public RoleDomainWeights effectiveDomainWeights() {
    return domainWeights != null ? domainWeights : RoleDomainWeights.EQUAL;
  }

  public double effectiveTeamAffinityThreshold() {
    return teamAffinityThreshold != null ? teamAffinityThreshold : 0.5;
  }

  public int effectiveTeamMinSize() {
    return teamMinSize != null ? teamMinSize : 2;
  }

  public double effectiveProgressChangeThreshold() {
    return progressChangeThreshold != null ? progressChangeThreshold : 0.1;
  }
}
```

- [ ] **Step 2: Create RoleDomainWeights record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record RoleDomainWeights(
    @Nullable Double perception,
    @Nullable Double communication,
    @Nullable Double decision,
    @Nullable Double effect) {

  public static final RoleDomainWeights EQUAL = new RoleDomainWeights(0.25, 0.25, 0.25, 0.25);

  public double[] normalized() {
    double p = perception != null ? perception : 0.25;
    double c = communication != null ? communication : 0.25;
    double d = decision != null ? decision : 0.25;
    double e = effect != null ? effect : 0.25;
    double sum = p + c + d + e;
    if (sum <= 0) return new double[] {0.25, 0.25, 0.25, 0.25};
    return new double[] {p / sum, c / sum, d / sum, e / sum};
  }
}
```

- [ ] **Step 3: Create BehavioralFingerprint record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import java.util.Map;

public record BehavioralFingerprint(
    Map<String, Double> perception,
    Map<String, Double> communication,
    Map<String, Double> decision,
    Map<String, Double> effect) {

  public static final BehavioralFingerprint EMPTY =
      new BehavioralFingerprint(Map.of(), Map.of(), Map.of(), Map.of());

  public BehavioralFingerprint {
    perception = Map.copyOf(perception);
    communication = Map.copyOf(communication);
    decision = Map.copyOf(decision);
    effect = Map.copyOf(effect);
  }

  public static double cosineSimilarity(Map<String, Double> a, Map<String, Double> b) {
    if (a.isEmpty() || b.isEmpty()) return 0.0;
    double dot = 0.0, magA = 0.0, magB = 0.0;
    for (var entry : a.entrySet()) {
      double va = entry.getValue();
      magA += va * va;
      Double vb = b.get(entry.getKey());
      if (vb != null) dot += va * vb;
    }
    for (double vb : b.values()) magB += vb * vb;
    double denom = Math.sqrt(magA) * Math.sqrt(magB);
    return denom > 0 ? dot / denom : 0.0;
  }

  public double weightedSimilarity(BehavioralFingerprint other, RoleDomainWeights weights) {
    double[] w = weights.normalized();
    return w[0] * cosineSimilarity(perception, other.perception)
        + w[1] * cosineSimilarity(communication, other.communication)
        + w[2] * cosineSimilarity(decision, other.decision)
        + w[3] * cosineSimilarity(effect, other.effect);
  }
}
```

- [ ] **Step 4: Create DetectedRole record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import java.util.List;
import java.util.Set;

public record DetectedRole(
    String roleId,
    Set<String> memberAgents,
    BehavioralFingerprint centroid,
    List<String> dominantFeatures,
    int stabilityCount,
    double centroidDrift) {

  public DetectedRole {
    memberAgents = Set.copyOf(memberAgents);
    dominantFeatures = List.copyOf(dominantFeatures);
  }
}
```

- [ ] **Step 5: Create DetectedTeam record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import io.casehub.api.spi.neighbor.NeighborRelation;
import java.util.Set;

public record DetectedTeam(
    String teamId,
    Set<String> memberAgents,
    Set<NeighborRelation> dominantRelations,
    double avgAffinity,
    int stabilityCount) {

  public DetectedTeam {
    memberAgents = Set.copyOf(memberAgents);
    dominantRelations = Set.copyOf(dominantRelations);
  }
}
```

- [ ] **Step 6: Create SwarmProgress record**

Use `ide_create_file`:

```java
package io.casehub.api.model.stigmergy;

import java.time.Instant;

public record SwarmProgress(
    double explorationPace,
    double consensusScore,
    double stabilityScore,
    Instant computedAt) {

  public static final SwarmProgress EMPTY = new SwarmProgress(0.0, 0.0, 0.0, Instant.EPOCH);
}
```

- [ ] **Step 7: Create MetricsSpace interface**

Use `ide_create_file`:

```java
package io.casehub.api.engine;

import io.casehub.api.model.stigmergy.BehavioralFingerprint;
import io.casehub.api.model.stigmergy.DetectedRole;
import io.casehub.api.model.stigmergy.DetectedTeam;
import io.casehub.api.model.stigmergy.SwarmProgress;
import java.util.List;
import java.util.Map;

public interface MetricsSpace {

  Map<String, Double> activityRates();

  Map<String, Long> budgetUsage();

  BehavioralFingerprint myFingerprint();

  SwarmProgress swarmProgress();

  List<DetectedRole> detectedRoles();

  List<DetectedTeam> detectedTeams();

  MetricsSpace NOOP =
      new MetricsSpace() {
        @Override
        public Map<String, Double> activityRates() {
          return Map.of();
        }

        @Override
        public Map<String, Long> budgetUsage() {
          return Map.of();
        }

        @Override
        public BehavioralFingerprint myFingerprint() {
          return BehavioralFingerprint.EMPTY;
        }

        @Override
        public SwarmProgress swarmProgress() {
          return SwarmProgress.EMPTY;
        }

        @Override
        public List<DetectedRole> detectedRoles() {
          return List.of();
        }

        @Override
        public List<DetectedTeam> detectedTeams() {
          return List.of();
        }
      };
}
```

- [ ] **Step 8: Modify StigmergyConfig — add swarm field**

Use `ide_replace_text_in_file` to replace the record definition:

```java
public record StigmergyConfig(
    @Nullable StigmergyDefaults defaults,
    @Nullable CoordinationConfig coordination,
    @Nullable SwarmConfig swarm) {

  public StigmergyConfig(@Nullable StigmergyDefaults defaults, @Nullable CoordinationConfig coordination) {
    this(defaults, coordination, null);
  }
}
```

The 2-arg constructor preserves backward compat with existing call sites (`new StigmergyConfig(defaults, coordination)`).

- [ ] **Step 9: Modify WorkerRuntime — add metrics() default**

Use `ide_replace_text_in_file` to add after `default void leave() {}`:

```java
  default MetricsSpace metrics() {
    return MetricsSpace.NOOP;
  }
```

- [ ] **Step 10: Add 7 CaseHubEventType values**

Use `ide_replace_text_in_file` to replace `INTEREST_CONVERGENCE_DETECTED // collective attention...` (last enum value without trailing comma) with:

```java
  INTEREST_CONVERGENCE_DETECTED, // collective attention focusing on specific keys

  SWARM_ROLE_EMERGED, // new behavioral role cluster detected among agents
  SWARM_ROLE_DISSOLVED, // role cluster no longer meets minimum membership
  SWARM_ROLE_SHIFT, // agent moved from one role cluster to another
  SWARM_TEAM_FORMED, // team affinity cluster detected among agents
  SWARM_TEAM_DISSOLVED, // team cluster no longer exists
  SWARM_TEAM_SHIFT, // agent moved from one team to another
  SWARM_PROGRESS // swarm progress scores changed significantly
```

- [ ] **Step 11: Compile and verify**

Run: `/opt/homebrew/bin/mvn compile -pl api -q`
Expected: BUILD SUCCESS

- [ ] **Step 12: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/SwarmConfig.java \
  api/src/main/java/io/casehub/api/model/stigmergy/RoleDomainWeights.java \
  api/src/main/java/io/casehub/api/model/stigmergy/BehavioralFingerprint.java \
  api/src/main/java/io/casehub/api/model/stigmergy/DetectedRole.java \
  api/src/main/java/io/casehub/api/model/stigmergy/DetectedTeam.java \
  api/src/main/java/io/casehub/api/model/stigmergy/SwarmProgress.java \
  api/src/main/java/io/casehub/api/engine/MetricsSpace.java \
  api/src/main/java/io/casehub/api/model/stigmergy/StigmergyConfig.java \
  api/src/main/java/io/casehub/api/engine/WorkerRuntime.java \
  api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java
git commit -m "feat: add swarm API types — SwarmConfig, BehavioralFingerprint, MetricsSpace, events

Refs #1112"
```

---

## Batch 2: RoleTracker

### Task 2: RoleTracker — fingerprinting, cosine similarity, clustering, evolution

Core swarm intelligence: accumulates per-agent behavioral data, computes fingerprints, detects role clusters via pairwise cosine similarity + connected components with internal average check, and tracks role evolution across detection cycles.

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmEvent.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/RoleTracker.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/RoleTrackerTest.java`

**Interfaces:**
- Consumes: `ObservationRegistry.getRegistrationsForAgent(UUID, String) → List<ObserverRegistration>`, `ObservationRegistry.getObservers(UUID) → Map<String, List<EnvironmentObserver>>`, `SignalRegistry.getAllSignals(UUID) → Map<String, Signal>`, `RuleRegistry.getFirings(UUID, String) → List<RuleFiring>`, `StigmergyCoordinator.activeAgents(UUID) → List<AgentState>`, `StigmergyCoordinator.isStigmergyCase(UUID) → boolean`, `InterestDeclaration` sealed hierarchy (5 variants), `RuleAction.WriteContext`, `BehavioralFingerprint.cosineSimilarity()`, `BehavioralFingerprint.weightedSimilarity()`, `SwarmConfig.effective*()` methods, `CaseHubEventType.SWARM_ROLE_EMERGED/DISSOLVED/SHIFT`
- Produces: `SwarmEvent(CaseHubEventType, Map<String, Object>)` — returned from `detect()`. `RoleTracker.getFingerprint(UUID caseId, String agentId) → BehavioralFingerprint` — consumed by DefaultMetricsSpace. `RoleTracker.getDetectedRoles(UUID caseId) → List<DetectedRole>` — consumed by DefaultMetricsSpace and SwarmProgressTracker. `RoleTracker.effectKeys(UUID caseId, String agentId) → Set<String>` — consumed by TeamDetector for COMPLEMENTARY affinity. `RoleTracker.shouldDetect(UUID caseId) → boolean` — consumed by handler for trigger check. `RoleTracker.accumulate(UUID caseId) → void` — consumed by handler each cycle. `RoleTracker.markDirty(UUID caseId) → void` — consumed by handler on structural changes. `RoleTracker.evictByCase(UUID caseId) → void` — consumed by CaseStatusChangedHandler.

- [ ] **Step 1: Create SwarmEvent record**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.event.CaseHubEventType;
import java.util.Map;

public record SwarmEvent(CaseHubEventType type, Map<String, Object> metadata) {

  public SwarmEvent {
    metadata = Map.copyOf(metadata);
  }
}
```

- [ ] **Step 2: Write the failing test — BehavioralFingerprint cosine similarity**

Use `ide_create_file` for test file. Start with cosine similarity tests (the math is foundational):

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.stigmergy.*;
import java.util.Map;
import org.junit.jupiter.api.Test;

class RoleTrackerTest {

  @Test
  void cosineSimilarityIdenticalVectors() {
    var a = Map.of("x", 0.5, "y", 0.5);
    assertEquals(1.0, BehavioralFingerprint.cosineSimilarity(a, a), 0.001);
  }

  @Test
  void cosineSimilarityOrthogonalVectors() {
    var a = Map.of("x", 1.0);
    var b = Map.of("y", 1.0);
    assertEquals(0.0, BehavioralFingerprint.cosineSimilarity(a, b), 0.001);
  }

  @Test
  void cosineSimilarityEmptyVector() {
    assertEquals(0.0, BehavioralFingerprint.cosineSimilarity(Map.of(), Map.of("x", 1.0)), 0.001);
  }

  @Test
  void cosineSimilarityPartialOverlap() {
    var a = Map.of("x", 1.0, "y", 1.0);
    var b = Map.of("x", 1.0, "z", 1.0);
    double expected = 1.0 / (Math.sqrt(2) * Math.sqrt(2));
    assertEquals(expected, BehavioralFingerprint.cosineSimilarity(a, b), 0.001);
  }
}
```

- [ ] **Step 3: Run test to verify it passes** (cosineSimilarity is already implemented in BehavioralFingerprint from Task 1)

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=RoleTrackerTest -q`
Expected: PASS (4 tests)

- [ ] **Step 4: Write failing test — RoleTracker fingerprint computation**

Add tests to `RoleTrackerTest.java`:

```java
  private RoleTracker roleTracker;
  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private RuleRegistry ruleRegistry;
  private StigmergyCoordinator coordinator;
  private ActivityTracker activityTracker;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    roleTracker = new RoleTracker(observationRegistry, signalRegistry, ruleRegistry, coordinator);
    caseId = UUID.randomUUID();
  }

  @Test
  void fingerprintExtractsPerceptionFromInterests() {
    var config = new StigmergyConfig(null, null, new SwarmConfig(
        null, null, null, null, null, null, null, null, null));
    coordinator.initializeCase(caseId, List.of("agent-1"), config);
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");

    observationRegistry.registerObserver(caseId, "agent-1", "binding-1",
        new TestObserver("obs-1", Set.of("tempReading", "pressure")), 20);

    roleTracker.accumulate(caseId);
    var fp = roleTracker.getFingerprint(caseId, "agent-1");

    assertEquals(1.0, fp.perception().get("tempReading"));
    assertEquals(1.0, fp.perception().get("pressure"));
    assertEquals(2, fp.perception().size());
  }

  @Test
  void fingerprintExtractsCommunicationFromSignals() {
    var config = new StigmergyConfig(null, null, new SwarmConfig(
        null, null, null, null, null, null, null, null, null));
    coordinator.initializeCase(caseId, List.of("agent-1"), config);
    coordinator.agentJoined(caseId, "agent-1", "binding-1");
    coordinator.agentActivated(caseId, "agent-1");

    signalRegistry.deposit(caseId, "overheating", 0.8, Duration.ofMinutes(5), "agent-1", 100);
    signalRegistry.deposit(caseId, "overheating", 0.9, Duration.ofMinutes(5), "agent-1", 100);
    signalRegistry.deposit(caseId, "cooldown", 0.5, Duration.ofMinutes(5), "agent-1", 100);

    roleTracker.accumulate(caseId);
    var fp = roleTracker.getFingerprint(caseId, "agent-1");

    assertTrue(fp.communication().containsKey("overheating"));
    assertTrue(fp.communication().containsKey("cooldown"));
  }
```

Import additions for the test file: `import io.casehub.engine.common.internal.convergence.ActivityTracker;`, `import io.casehub.engine.common.internal.observation.ObservationRegistry;`, `import io.casehub.engine.common.internal.signal.SignalRegistry;`, `import io.casehub.engine.common.internal.observation.RuleRegistry;`, `import io.casehub.api.spi.observation.*;`, `import java.time.Duration;`, `import java.time.Instant;`, `import java.util.*;`, `import org.junit.jupiter.api.BeforeEach;`.

Also add a `TestObserver` helper class at the bottom of the test file:

```java
  static class TestObserver implements EnvironmentObserver {
    private final String type;
    private final Set<String> keys;

    TestObserver(String type, Set<String> keys) {
      this.type = type;
      this.keys = keys;
    }

    @Override
    public String observerType() {
      return type;
    }

    @Override
    public Set<String> watchedKeys() {
      return keys;
    }

    @Override
    public List<Observation> observe(ObservationContext context) {
      return List.of();
    }
  }
```

- [ ] **Step 5: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=RoleTrackerTest#fingerprintExtractsPerceptionFromInterests -q`
Expected: FAIL — `RoleTracker` class does not exist

- [ ] **Step 6: Implement RoleTracker**

Use `ide_create_file` to create `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/RoleTracker.java`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

@ApplicationScoped
public class RoleTracker implements Resettable {

  private final ObservationRegistry observationRegistry;
  private final SignalRegistry signalRegistry;
  private final RuleRegistry ruleRegistry;
  private final StigmergyCoordinator coordinator;

  private final ConcurrentHashMap<UUID, CaseRoleState> cases = new ConcurrentHashMap<>();

  @Inject
  public RoleTracker(
      ObservationRegistry observationRegistry,
      SignalRegistry signalRegistry,
      RuleRegistry ruleRegistry,
      StigmergyCoordinator coordinator) {
    this.observationRegistry = observationRegistry;
    this.signalRegistry = signalRegistry;
    this.ruleRegistry = ruleRegistry;
    this.coordinator = coordinator;
  }

  public void accumulate(UUID caseId) {
    var state = cases.computeIfAbsent(caseId, k -> new CaseRoleState());
    state.cycleCount++;
    var agents = coordinator.activeAgents(caseId);
    for (var agent : agents) {
      accumulateAgent(caseId, agent.agentId(), state);
    }
  }

  private void accumulateAgent(UUID caseId, String agentId, CaseRoleState state) {
    var agentState =
        state.agentAccumulators.computeIfAbsent(agentId, k -> new AgentAccumulator());

    var firings = ruleRegistry.getFirings(caseId, agentId);
    Map<String, Integer> cycleFirings = new HashMap<>();
    Set<String> cycleWriteKeys = new HashSet<>();
    for (var firing : firings) {
      cycleFirings.merge(firing.ruleId(), 1, Integer::sum);
      for (var action : firing.executedActions()) {
        if (action instanceof RuleAction.WriteContext wc) {
          cycleWriteKeys.add(wc.key());
          agentState.allEffectKeys.add(wc.key());
        }
      }
    }
    agentState.decisionWindow.add(cycleFirings);
    agentState.effectWindow.add(cycleWriteKeys.stream()
        .collect(Collectors.toMap(k -> k, k -> 1, Integer::sum)));

    int window = getWindow(caseId);
    while (agentState.decisionWindow.size() > window) agentState.decisionWindow.removeFirst();
    while (agentState.effectWindow.size() > window) agentState.effectWindow.removeFirst();
  }

  public BehavioralFingerprint getFingerprint(UUID caseId, String agentId) {
    Map<String, Double> perception = extractPerception(caseId, agentId);
    Map<String, Double> communication = extractCommunication(caseId, agentId);
    var state = cases.get(caseId);
    Map<String, Double> decision = Map.of();
    Map<String, Double> effect = Map.of();
    if (state != null) {
      var acc = state.agentAccumulators.get(agentId);
      if (acc != null) {
        decision = normalizeWindow(acc.decisionWindow);
        effect = normalizeWindow(acc.effectWindow);
      }
    }
    return new BehavioralFingerprint(perception, communication, decision, effect);
  }

  private Map<String, Double> extractPerception(UUID caseId, String agentId) {
    var registrations = observationRegistry.getRegistrationsForAgent(caseId, agentId);
    Map<String, Double> keys = new HashMap<>();
    for (var reg : registrations) {
      if (reg.declaration() != null) {
        extractKeysFromDeclaration(reg.declaration(), keys);
      } else {
        for (String key : reg.observer().watchedKeys()) {
          keys.put(key, 1.0);
        }
      }
    }
    return keys;
  }

  private void extractKeysFromDeclaration(InterestDeclaration decl, Map<String, Double> keys) {
    switch (decl) {
      case InterestDeclaration.KeyThreshold kt -> keys.put(kt.key(), 1.0);
      case InterestDeclaration.KeyCorrelation kc -> kc.keys().forEach(k -> keys.put(k, 1.0));
      case InterestDeclaration.TemporalSequence ts ->
          ts.steps().forEach(s -> keys.put(s.key(), 1.0));
      case InterestDeclaration.JqInterest jq -> jq.watchedKeys().forEach(k -> keys.put(k, 1.0));
      case InterestDeclaration.SignalThreshold st -> {}
    }
  }

  private Map<String, Double> extractCommunication(UUID caseId, String agentId) {
    var allSignals = signalRegistry.getAllSignals(caseId);
    Map<String, Double> comm = new HashMap<>();
    int totalDeposits = 0;
    Map<String, Integer> depositCounts = new HashMap<>();
    for (var entry : allSignals.entrySet()) {
      if (entry.getValue().sources().contains(agentId)) {
        depositCounts.put(entry.getKey(), 1);
        totalDeposits++;
      }
    }
    var registrations = observationRegistry.getRegistrationsForAgent(caseId, agentId);
    for (var reg : registrations) {
      if (reg.declaration() instanceof InterestDeclaration.SignalThreshold st) {
        comm.put("signal-interest:" + st.signalName(), 1.0);
      }
    }
    if (totalDeposits > 0) {
      for (var entry : depositCounts.entrySet()) {
        comm.put(entry.getKey(), (double) entry.getValue() / totalDeposits);
      }
    }
    return comm;
  }

  private Map<String, Double> normalizeWindow(Deque<Map<String, Integer>> window) {
    Map<String, Integer> totals = new HashMap<>();
    int sum = 0;
    for (var cycle : window) {
      for (var entry : cycle.entrySet()) {
        totals.merge(entry.getKey(), entry.getValue(), Integer::sum);
        sum += entry.getValue();
      }
    }
    if (sum == 0) return Map.of();
    Map<String, Double> normalized = new HashMap<>();
    for (var entry : totals.entrySet()) {
      normalized.put(entry.getKey(), (double) entry.getValue() / sum);
    }
    return normalized;
  }

  public Set<String> effectKeys(UUID caseId, String agentId) {
    var state = cases.get(caseId);
    if (state == null) return Set.of();
    var acc = state.agentAccumulators.get(agentId);
    return acc != null ? Set.copyOf(acc.allEffectKeys) : Set.of();
  }

  public List<DetectedRole> getDetectedRoles(UUID caseId) {
    var state = cases.get(caseId);
    return state != null ? List.copyOf(state.currentRoles) : List.of();
  }

  public boolean shouldDetect(UUID caseId) {
    var state = cases.get(caseId);
    if (state == null) return false;
    if (state.dirty) return true;
    int interval = getInterval(caseId);
    return state.cycleCount % interval == 0;
  }

  public void markDirty(UUID caseId) {
    var state = cases.get(caseId);
    if (state != null) state.dirty = true;
  }

  public List<SwarmEvent> detect(UUID caseId, SwarmConfig config) {
    var state = cases.get(caseId);
    if (state == null) return List.of();
    state.dirty = false;

    var agents = coordinator.activeAgents(caseId);
    if (agents.size() < config.effectiveRoleMinClusterSize()) {
      if (!state.currentRoles.isEmpty()) {
        var events = dissolveAllRoles(state);
        state.currentRoles.clear();
        state.previousRoles.clear();
        return events;
      }
      return List.of();
    }

    Map<String, BehavioralFingerprint> fingerprints = new HashMap<>();
    for (var agent : agents) {
      fingerprints.put(agent.agentId(), getFingerprint(caseId, agent.agentId()));
    }

    var weights = config.effectiveDomainWeights();
    double threshold = config.effectiveRoleSimilarityThreshold();
    int minSize = config.effectiveRoleMinClusterSize();

    List<Set<String>> clusters = clusterAgents(fingerprints, weights, threshold, minSize);

    List<DetectedRole> newRoles = new ArrayList<>();
    for (var cluster : clusters) {
      var centroid = computeCentroid(cluster, fingerprints);
      var dominant = dominantFeatures(centroid, 3);
      String roleId = matchExistingRole(cluster, state);
      double drift = 0.0;
      int stability = 1;
      if (roleId != null) {
        var origCentroid = state.originalCentroids.get(roleId);
        if (origCentroid != null) {
          drift = 1.0 - centroid.weightedSimilarity(origCentroid, weights);
        }
        var prev = state.currentRoles.stream()
            .filter(r -> r.roleId().equals(roleId)).findFirst();
        stability = prev.map(r -> r.stabilityCount() + 1).orElse(1);
      } else {
        roleId = "role-" + state.roleCounter.incrementAndGet();
        state.originalCentroids.put(roleId, centroid);
      }
      newRoles.add(new DetectedRole(roleId, cluster, centroid, dominant, stability, drift));
    }

    List<SwarmEvent> events = computeEvolution(state, newRoles);
    state.previousRoles = state.currentRoles;
    state.currentRoles = newRoles;
    return events;
  }

  private List<Set<String>> clusterAgents(
      Map<String, BehavioralFingerprint> fingerprints,
      RoleDomainWeights weights,
      double threshold,
      int minSize) {
    List<String> agentIds = new ArrayList<>(fingerprints.keySet());
    int n = agentIds.size();
    double[][] sim = new double[n][n];
    for (int i = 0; i < n; i++) {
      sim[i][i] = 1.0;
      for (int j = i + 1; j < n; j++) {
        double s = fingerprints.get(agentIds.get(i))
            .weightedSimilarity(fingerprints.get(agentIds.get(j)), weights);
        sim[i][j] = s;
        sim[j][i] = s;
      }
    }

    boolean[][] adj = new boolean[n][n];
    for (int i = 0; i < n; i++)
      for (int j = i + 1; j < n; j++)
        if (sim[i][j] >= threshold) {
          adj[i][j] = true;
          adj[j][i] = true;
        }

    boolean[] visited = new boolean[n];
    List<Set<String>> result = new ArrayList<>();
    for (int i = 0; i < n; i++) {
      if (visited[i]) continue;
      Set<Integer> component = new LinkedHashSet<>();
      Queue<Integer> queue = new ArrayDeque<>();
      queue.add(i);
      visited[i] = true;
      while (!queue.isEmpty()) {
        int curr = queue.poll();
        component.add(curr);
        for (int j = 0; j < n; j++) {
          if (adj[curr][j] && !visited[j]) {
            visited[j] = true;
            queue.add(j);
          }
        }
      }
      if (component.size() >= minSize) {
        double avgSim = averageInternalSimilarity(component, sim);
        if (avgSim >= threshold) {
          result.add(component.stream().map(agentIds::get).collect(Collectors.toSet()));
        }
      }
    }
    return result;
  }

  private double averageInternalSimilarity(Set<Integer> component, double[][] sim) {
    if (component.size() <= 1) return 1.0;
    double sum = 0;
    int count = 0;
    var list = new ArrayList<>(component);
    for (int i = 0; i < list.size(); i++) {
      for (int j = i + 1; j < list.size(); j++) {
        sum += sim[list.get(i)][list.get(j)];
        count++;
      }
    }
    return count > 0 ? sum / count : 0.0;
  }

  private BehavioralFingerprint computeCentroid(
      Set<String> cluster, Map<String, BehavioralFingerprint> fingerprints) {
    Map<String, Double> percAcc = new HashMap<>(), commAcc = new HashMap<>(),
        decAcc = new HashMap<>(), effAcc = new HashMap<>();
    for (String agentId : cluster) {
      var fp = fingerprints.get(agentId);
      fp.perception().forEach((k, v) -> percAcc.merge(k, v, Double::sum));
      fp.communication().forEach((k, v) -> commAcc.merge(k, v, Double::sum));
      fp.decision().forEach((k, v) -> decAcc.merge(k, v, Double::sum));
      fp.effect().forEach((k, v) -> effAcc.merge(k, v, Double::sum));
    }
    int size = cluster.size();
    return new BehavioralFingerprint(
        divideAll(percAcc, size), divideAll(commAcc, size),
        divideAll(decAcc, size), divideAll(effAcc, size));
  }

  private Map<String, Double> divideAll(Map<String, Double> map, int divisor) {
    Map<String, Double> result = new HashMap<>();
    map.forEach((k, v) -> result.put(k, v / divisor));
    return result;
  }

  private List<String> dominantFeatures(BehavioralFingerprint centroid, int topK) {
    Map<String, Double> all = new HashMap<>();
    centroid.perception().forEach((k, v) -> all.put("interest:" + k, v));
    centroid.communication().forEach((k, v) -> all.put("signal:" + k, v));
    centroid.decision().forEach((k, v) -> all.put("rule:" + k, v));
    centroid.effect().forEach((k, v) -> all.put("output:" + k, v));
    return all.entrySet().stream()
        .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
        .limit(topK)
        .map(Map.Entry::getKey)
        .toList();
  }

  private String matchExistingRole(Set<String> cluster, CaseRoleState state) {
    for (var prev : state.currentRoles) {
      long overlap = cluster.stream().filter(prev.memberAgents()::contains).count();
      if (overlap > prev.memberAgents().size() / 2.0 && overlap > cluster.size() / 2.0) {
        return prev.roleId();
      }
    }
    return null;
  }

  private List<SwarmEvent> computeEvolution(CaseRoleState state, List<DetectedRole> newRoles) {
    List<SwarmEvent> events = new ArrayList<>();
    Set<String> prevRoleIds = state.currentRoles.stream()
        .map(DetectedRole::roleId).collect(Collectors.toSet());
    Set<String> newRoleIds = newRoles.stream()
        .map(DetectedRole::roleId).collect(Collectors.toSet());

    for (var role : newRoles) {
      if (!prevRoleIds.contains(role.roleId())) {
        events.add(new SwarmEvent(CaseHubEventType.SWARM_ROLE_EMERGED, Map.of(
            "roleId", role.roleId(),
            "memberAgents", role.memberAgents(),
            "dominantFeatures", role.dominantFeatures(),
            "clusterSize", role.memberAgents().size(),
            "avgSimilarity", 0.0)));
      }
    }
    for (var prev : state.currentRoles) {
      if (!newRoleIds.contains(prev.roleId())) {
        events.add(new SwarmEvent(CaseHubEventType.SWARM_ROLE_DISSOLVED, Map.of(
            "roleId", prev.roleId(),
            "previousMembers", prev.memberAgents(),
            "lifetimeCycles", prev.stabilityCount())));
      }
    }

    Map<String, String> prevAssignment = new HashMap<>();
    for (var prev : state.currentRoles) {
      for (String agent : prev.memberAgents()) {
        prevAssignment.put(agent, prev.roleId());
      }
    }
    for (var role : newRoles) {
      for (String agent : role.memberAgents()) {
        String oldRole = prevAssignment.get(agent);
        if (oldRole != null && !oldRole.equals(role.roleId())) {
          events.add(new SwarmEvent(CaseHubEventType.SWARM_ROLE_SHIFT, Map.of(
              "agentId", agent,
              "fromRoleId", oldRole,
              "toRoleId", role.roleId(),
              "similarityToNewRole", 0.0)));
        }
      }
    }
    return events;
  }

  private List<SwarmEvent> dissolveAllRoles(CaseRoleState state) {
    List<SwarmEvent> events = new ArrayList<>();
    for (var role : state.currentRoles) {
      events.add(new SwarmEvent(CaseHubEventType.SWARM_ROLE_DISSOLVED, Map.of(
          "roleId", role.roleId(),
          "previousMembers", role.memberAgents(),
          "lifetimeCycles", role.stabilityCount())));
    }
    return events;
  }

  private int getWindow(UUID caseId) {
    var coord = coordinator.getAgent(caseId, "");
    var stigConfig = getStigmergyConfig(caseId);
    if (stigConfig != null && stigConfig.swarm() != null) {
      return stigConfig.swarm().effectiveRoleDetectionWindow();
    }
    return 20;
  }

  private int getInterval(UUID caseId) {
    var stigConfig = getStigmergyConfig(caseId);
    if (stigConfig != null && stigConfig.swarm() != null) {
      return stigConfig.swarm().effectiveDetectionInterval();
    }
    return 10;
  }

  private StigmergyConfig getStigmergyConfig(UUID caseId) {
    return null;
  }

  public void evictByCase(UUID caseId) {
    cases.remove(caseId);
  }

  @Override
  public void reset() {
    cases.clear();
  }

  private static class CaseRoleState {
    int cycleCount;
    volatile boolean dirty;
    final AtomicInteger roleCounter = new AtomicInteger();
    final ConcurrentHashMap<String, AgentAccumulator> agentAccumulators =
        new ConcurrentHashMap<>();
    List<DetectedRole> currentRoles = List.of();
    List<DetectedRole> previousRoles = List.of();
    final ConcurrentHashMap<String, BehavioralFingerprint> originalCentroids =
        new ConcurrentHashMap<>();
  }

  private static class AgentAccumulator {
    final Deque<Map<String, Integer>> decisionWindow = new ArrayDeque<>();
    final Deque<Map<String, Integer>> effectWindow = new ArrayDeque<>();
    final Set<String> allEffectKeys = ConcurrentHashMap.newKeySet();
  }
}
```

- [ ] **Step 7: Run tests to verify fingerprint extraction passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=RoleTrackerTest -q`
Expected: PASS

- [ ] **Step 8: Add clustering tests**

Add to `RoleTrackerTest.java`:

```java
  @Test
  void detectsRoleClustersFromSimilarBehavior() {
    var swarmConfig = new SwarmConfig(null, 0.5, 2, 20, 1, null, null, null, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2", "a3"), config);
    for (String id : List.of("a1", "a2", "a3")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(caseId, "a1", "b-a1",
        new TestObserver("obs", Set.of("temp", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "a2", "b-a2",
        new TestObserver("obs", Set.of("temp", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "a3", "b-a3",
        new TestObserver("obs", Set.of("cooling", "venting")), 20);

    roleTracker.accumulate(caseId);
    var events = roleTracker.detect(caseId, swarmConfig);

    var roles = roleTracker.getDetectedRoles(caseId);
    assertEquals(1, roles.size());
    assertTrue(roles.get(0).memberAgents().containsAll(Set.of("a1", "a2")));
    assertFalse(roles.get(0).memberAgents().contains("a3"));
  }

  @Test
  void detectsRoleEmergenceEvent() {
    var swarmConfig = new SwarmConfig(null, 0.5, 2, 20, 1, null, null, null, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(caseId, "a1", "b-a1",
        new TestObserver("obs", Set.of("temp")), 20);
    observationRegistry.registerObserver(caseId, "a2", "b-a2",
        new TestObserver("obs", Set.of("temp")), 20);

    roleTracker.accumulate(caseId);
    var events = roleTracker.detect(caseId, swarmConfig);

    assertEquals(1, events.size());
    assertEquals(CaseHubEventType.SWARM_ROLE_EMERGED, events.get(0).type());
  }

  @Test
  void evictByCaseRemovesAllState() {
    var swarmConfig = new SwarmConfig(null, 0.5, 2, 20, 1, null, null, null, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1"), config);
    coordinator.agentJoined(caseId, "a1", "b-a1");
    coordinator.agentActivated(caseId, "a1");
    roleTracker.accumulate(caseId);

    roleTracker.evictByCase(caseId);
    assertEquals(BehavioralFingerprint.EMPTY, roleTracker.getFingerprint(caseId, "a1"));
    assertTrue(roleTracker.getDetectedRoles(caseId).isEmpty());
  }
```

- [ ] **Step 9: Run all tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=RoleTrackerTest -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmEvent.java \
  runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/RoleTracker.java \
  runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/RoleTrackerTest.java
git commit -m "feat: add RoleTracker — behavioral fingerprinting and role cluster detection

Refs #1112"
```

---

## Batch 3: TeamDetector + SwarmProgressTracker

### Task 3: TeamDetector — affinity computation and team clustering

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/TeamDetector.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/TeamDetectorTest.java`

**Interfaces:**
- Consumes: `ObservationRegistry.getRegistrationsForAgent(UUID, String) → List<ObserverRegistration>`, `ObservationRegistry.getObservers(UUID) → Map<String, List<EnvironmentObserver>>`, `SignalRegistry.getAllSignals(UUID) → Map<String, Signal>`, `RoleTracker.effectKeys(UUID, String) → Set<String>`, `StigmergyCoordinator.activeAgents(UUID) → List<AgentState>`, `SwarmConfig.effectiveTeamAffinityThreshold()`, `SwarmConfig.effectiveTeamMinSize()`, `CaseHubEventType.SWARM_TEAM_FORMED/DISSOLVED/SHIFT`
- Produces: `TeamDetector.detect(UUID, SwarmConfig) → List<SwarmEvent>` — consumed by handler. `TeamDetector.getDetectedTeams(UUID) → List<DetectedTeam>` — consumed by DefaultMetricsSpace. `TeamDetector.evictByCase(UUID)` — consumed by CaseStatusChangedHandler.

- [ ] **Step 1: Write failing test**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class TeamDetectorTest {

  private TeamDetector teamDetector;
  private RoleTracker roleTracker;
  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private RuleRegistry ruleRegistry;
  private StigmergyCoordinator coordinator;
  private ActivityTracker activityTracker;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    roleTracker = new RoleTracker(observationRegistry, signalRegistry, ruleRegistry, coordinator);
    teamDetector = new TeamDetector(observationRegistry, signalRegistry, roleTracker, coordinator);
    caseId = UUID.randomUUID();
  }

  @Test
  void detectsTeamFromSharedInterests() {
    var swarmConfig = new SwarmConfig(null, null, null, null, null, null, 0.3, 2, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2", "a3"), config);
    for (String id : List.of("a1", "a2", "a3")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(caseId, "a1", "b-a1",
        new TestObserver("obs", Set.of("temp", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "a2", "b-a2",
        new TestObserver("obs", Set.of("temp", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "a3", "b-a3",
        new TestObserver("obs", Set.of("cooling")), 20);

    var events = teamDetector.detect(caseId, swarmConfig);
    var teams = teamDetector.getDetectedTeams(caseId);

    assertEquals(1, teams.size());
    assertTrue(teams.get(0).memberAgents().containsAll(Set.of("a1", "a2")));
    assertEquals(1, events.size());
    assertEquals(CaseHubEventType.SWARM_TEAM_FORMED, events.get(0).type());
  }

  @Test
  void detectsTeamFromSharedSignals() {
    var swarmConfig = new SwarmConfig(null, null, null, null, null, null, 0.3, 2, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "overheating", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "overheating", 0.9, Duration.ofMinutes(5), "a2", 100);

    var events = teamDetector.detect(caseId, swarmConfig);
    assertFalse(teamDetector.getDetectedTeams(caseId).isEmpty());
  }

  @Test
  void evictRemovesState() {
    var swarmConfig = new SwarmConfig(null, null, null, null, null, null, 0.3, 2, null);
    var config = new StigmergyConfig(null, null, swarmConfig);
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }
    observationRegistry.registerObserver(caseId, "a1", "b-a1",
        new TestObserver("obs", Set.of("temp")), 20);
    observationRegistry.registerObserver(caseId, "a2", "b-a2",
        new TestObserver("obs", Set.of("temp")), 20);
    teamDetector.detect(caseId, swarmConfig);
    teamDetector.evictByCase(caseId);
    assertTrue(teamDetector.getDetectedTeams(caseId).isEmpty());
  }

  static class TestObserver implements EnvironmentObserver {
    private final String type;
    private final Set<String> keys;

    TestObserver(String type, Set<String> keys) {
      this.type = type;
      this.keys = keys;
    }

    @Override
    public String observerType() { return type; }

    @Override
    public Set<String> watchedKeys() { return keys; }

    @Override
    public List<Observation> observe(ObservationContext context) { return List.of(); }
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=TeamDetectorTest -q`
Expected: FAIL — `TeamDetector` class does not exist

- [ ] **Step 3: Implement TeamDetector**

Use `ide_create_file` for `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/TeamDetector.java`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.neighbor.NeighborRelation;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

@ApplicationScoped
public class TeamDetector implements Resettable {

  private final ObservationRegistry observationRegistry;
  private final SignalRegistry signalRegistry;
  private final RoleTracker roleTracker;
  private final StigmergyCoordinator coordinator;

  private final ConcurrentHashMap<UUID, CaseTeamState> cases = new ConcurrentHashMap<>();

  @Inject
  public TeamDetector(
      ObservationRegistry observationRegistry,
      SignalRegistry signalRegistry,
      RoleTracker roleTracker,
      StigmergyCoordinator coordinator) {
    this.observationRegistry = observationRegistry;
    this.signalRegistry = signalRegistry;
    this.roleTracker = roleTracker;
    this.coordinator = coordinator;
  }

  public List<SwarmEvent> detect(UUID caseId, SwarmConfig config) {
    var state = cases.computeIfAbsent(caseId, k -> new CaseTeamState());
    var agents = coordinator.activeAgents(caseId);
    if (agents.size() < config.effectiveTeamMinSize()) {
      if (!state.currentTeams.isEmpty()) {
        var events = dissolveAll(state);
        state.currentTeams.clear();
        return events;
      }
      return List.of();
    }

    List<String> agentIds = agents.stream().map(AgentState::agentId).toList();
    int n = agentIds.size();
    double[][] affinity = new double[n][n];

    Map<String, Set<String>> interestKeys = new HashMap<>();
    Map<String, Set<String>> signalNames = new HashMap<>();
    Map<String, Set<String>> effectKeyMap = new HashMap<>();

    for (String agentId : agentIds) {
      interestKeys.put(agentId, extractInterestKeys(caseId, agentId));
      signalNames.put(agentId, extractSignalNames(caseId, agentId));
      effectKeyMap.put(agentId, roleTracker.effectKeys(caseId, agentId));
    }

    for (int i = 0; i < n; i++) {
      for (int j = i + 1; j < n; j++) {
        String ai = agentIds.get(i), aj = agentIds.get(j);
        double sharedInterest = jaccard(interestKeys.get(ai), interestKeys.get(aj));
        double sharedSignal = jaccard(signalNames.get(ai), signalNames.get(aj));
        double comp = Math.max(
            jaccard(effectKeyMap.get(ai), interestKeys.get(aj)),
            jaccard(effectKeyMap.get(aj), interestKeys.get(ai)));
        affinity[i][j] = (sharedInterest + sharedSignal + comp) / 3.0;
        affinity[j][i] = affinity[i][j];
      }
    }

    double threshold = config.effectiveTeamAffinityThreshold();
    int minSize = config.effectiveTeamMinSize();
    List<Set<String>> clusters = cluster(agentIds, affinity, threshold, minSize);

    List<DetectedTeam> newTeams = new ArrayList<>();
    for (var cluster : clusters) {
      String teamId = matchExisting(cluster, state);
      int stability = 1;
      if (teamId != null) {
        var prev = state.currentTeams.stream()
            .filter(t -> t.teamId().equals(teamId)).findFirst();
        stability = prev.map(t -> t.stabilityCount() + 1).orElse(1);
      } else {
        teamId = "team-" + state.teamCounter.incrementAndGet();
      }
      Set<NeighborRelation> dominant = dominantRelations(
          cluster, interestKeys, signalNames, effectKeyMap);
      double avg = averageAffinity(cluster, agentIds, affinity);
      newTeams.add(new DetectedTeam(teamId, cluster, dominant, avg, stability));
    }

    var events = computeEvolution(state, newTeams);
    state.previousTeams = state.currentTeams;
    state.currentTeams = newTeams;
    return events;
  }

  private Set<String> extractInterestKeys(UUID caseId, String agentId) {
    var regs = observationRegistry.getRegistrationsForAgent(caseId, agentId);
    Set<String> keys = new HashSet<>();
    for (var reg : regs) {
      keys.addAll(reg.observer().watchedKeys());
    }
    return keys;
  }

  private Set<String> extractSignalNames(UUID caseId, String agentId) {
    var allSignals = signalRegistry.getAllSignals(caseId);
    Set<String> names = new HashSet<>();
    for (var entry : allSignals.entrySet()) {
      if (entry.getValue().sources().contains(agentId)) {
        names.add(entry.getKey());
      }
    }
    return names;
  }

  private double jaccard(Set<String> a, Set<String> b) {
    if (a.isEmpty() && b.isEmpty()) return 0.0;
    Set<String> intersection = new HashSet<>(a);
    intersection.retainAll(b);
    Set<String> union = new HashSet<>(a);
    union.addAll(b);
    return union.isEmpty() ? 0.0 : (double) intersection.size() / union.size();
  }

  private List<Set<String>> cluster(
      List<String> agentIds, double[][] affinity, double threshold, int minSize) {
    int n = agentIds.size();
    boolean[] visited = new boolean[n];
    List<Set<String>> result = new ArrayList<>();
    for (int i = 0; i < n; i++) {
      if (visited[i]) continue;
      Set<Integer> component = new LinkedHashSet<>();
      Queue<Integer> queue = new ArrayDeque<>();
      queue.add(i);
      visited[i] = true;
      while (!queue.isEmpty()) {
        int curr = queue.poll();
        component.add(curr);
        for (int j = 0; j < n; j++) {
          if (!visited[j] && affinity[curr][j] >= threshold) {
            visited[j] = true;
            queue.add(j);
          }
        }
      }
      if (component.size() >= minSize) {
        double avg = avgInternal(component, affinity);
        if (avg >= threshold) {
          result.add(component.stream().map(agentIds::get).collect(Collectors.toSet()));
        }
      }
    }
    return result;
  }

  private double avgInternal(Set<Integer> component, double[][] affinity) {
    if (component.size() <= 1) return 1.0;
    var list = new ArrayList<>(component);
    double sum = 0;
    int count = 0;
    for (int i = 0; i < list.size(); i++) {
      for (int j = i + 1; j < list.size(); j++) {
        sum += affinity[list.get(i)][list.get(j)];
        count++;
      }
    }
    return count > 0 ? sum / count : 0.0;
  }

  private Set<NeighborRelation> dominantRelations(
      Set<String> cluster, Map<String, Set<String>> interestKeys,
      Map<String, Set<String>> signalNames, Map<String, Set<String>> effectKeys) {
    Set<NeighborRelation> dominant = EnumSet.noneOf(NeighborRelation.class);
    for (String a : cluster) {
      for (String b : cluster) {
        if (a.equals(b)) continue;
        if (!Collections.disjoint(interestKeys.get(a), interestKeys.get(b)))
          dominant.add(NeighborRelation.SHARED_INTEREST);
        if (!Collections.disjoint(signalNames.get(a), signalNames.get(b)))
          dominant.add(NeighborRelation.SHARED_SIGNAL);
        if (!Collections.disjoint(effectKeys.get(a), interestKeys.get(b))
            || !Collections.disjoint(effectKeys.get(b), interestKeys.get(a)))
          dominant.add(NeighborRelation.COMPLEMENTARY);
      }
    }
    return dominant;
  }

  private double averageAffinity(
      Set<String> cluster, List<String> agentIds, double[][] affinity) {
    var indices = cluster.stream().map(agentIds::indexOf).toList();
    double sum = 0;
    int count = 0;
    for (int i = 0; i < indices.size(); i++) {
      for (int j = i + 1; j < indices.size(); j++) {
        sum += affinity[indices.get(i)][indices.get(j)];
        count++;
      }
    }
    return count > 0 ? sum / count : 0.0;
  }

  private String matchExisting(Set<String> cluster, CaseTeamState state) {
    for (var prev : state.currentTeams) {
      long overlap = cluster.stream().filter(prev.memberAgents()::contains).count();
      if (overlap > prev.memberAgents().size() / 2.0 && overlap > cluster.size() / 2.0) {
        return prev.teamId();
      }
    }
    return null;
  }

  private List<SwarmEvent> computeEvolution(CaseTeamState state, List<DetectedTeam> newTeams) {
    List<SwarmEvent> events = new ArrayList<>();
    Set<String> prevIds = state.currentTeams.stream()
        .map(DetectedTeam::teamId).collect(Collectors.toSet());
    Set<String> newIds = newTeams.stream()
        .map(DetectedTeam::teamId).collect(Collectors.toSet());

    for (var team : newTeams) {
      if (!prevIds.contains(team.teamId())) {
        events.add(new SwarmEvent(CaseHubEventType.SWARM_TEAM_FORMED, Map.of(
            "teamId", team.teamId(),
            "memberAgents", team.memberAgents(),
            "dominantRelations", team.dominantRelations(),
            "avgAffinity", team.avgAffinity())));
      }
    }
    for (var prev : state.currentTeams) {
      if (!newIds.contains(prev.teamId())) {
        events.add(new SwarmEvent(CaseHubEventType.SWARM_TEAM_DISSOLVED, Map.of(
            "teamId", prev.teamId(),
            "previousMembers", prev.memberAgents(),
            "lifetimeCycles", prev.stabilityCount())));
      }
    }

    Map<String, String> prevAssignment = new HashMap<>();
    for (var prev : state.currentTeams) {
      for (String agent : prev.memberAgents()) prevAssignment.put(agent, prev.teamId());
    }
    for (var team : newTeams) {
      for (String agent : team.memberAgents()) {
        String old = prevAssignment.get(agent);
        if (old != null && !old.equals(team.teamId())) {
          events.add(new SwarmEvent(CaseHubEventType.SWARM_TEAM_SHIFT, Map.of(
              "agentId", agent, "fromTeamId", old, "toTeamId", team.teamId(),
              "affinityToNewTeam", team.avgAffinity())));
        }
      }
    }
    return events;
  }

  private List<SwarmEvent> dissolveAll(CaseTeamState state) {
    return state.currentTeams.stream()
        .map(t -> new SwarmEvent(CaseHubEventType.SWARM_TEAM_DISSOLVED, Map.of(
            "teamId", t.teamId(), "previousMembers", t.memberAgents(),
            "lifetimeCycles", t.stabilityCount())))
        .toList();
  }

  public List<DetectedTeam> getDetectedTeams(UUID caseId) {
    var state = cases.get(caseId);
    return state != null ? List.copyOf(state.currentTeams) : List.of();
  }

  public void evictByCase(UUID caseId) {
    cases.remove(caseId);
  }

  @Override
  public void reset() {
    cases.clear();
  }

  private static class CaseTeamState {
    final AtomicInteger teamCounter = new AtomicInteger();
    List<DetectedTeam> currentTeams = List.of();
    List<DetectedTeam> previousTeams = List.of();
  }
}
```

- [ ] **Step 4: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=TeamDetectorTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/TeamDetector.java \
  runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/TeamDetectorTest.java
git commit -m "feat: add TeamDetector — signal-based affinity clustering

Refs #1112"
```

### Task 4: SwarmProgressTracker — exploration, consensus, stability

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProgressTracker.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmProgressTrackerTest.java`

**Interfaces:**
- Consumes: `SignalRegistry.consensusSignals(UUID, int, double) → Map<String, Signal>`, `SignalRegistry.perceive(UUID, double) → Map<String, PerceivedSignal>`, `RoleTracker.getDetectedRoles(UUID) → List<DetectedRole>`, `RoleTracker.effectKeys(UUID, String) → Set<String>`, `StigmergyCoordinator.activeAgents(UUID) → List<AgentState>`, `StigmergyConfig.coordination().consensusThreshold()`, `StigmergyConfig.defaults().effectiveZeroThreshold()`, `SwarmConfig.effectiveProgressChangeThreshold()`, `CaseHubEventType.SWARM_PROGRESS`
- Produces: `SwarmProgressTracker.evaluate(UUID, StigmergyConfig) → List<SwarmEvent>`. `SwarmProgressTracker.getProgress(UUID) → SwarmProgress` — consumed by DefaultMetricsSpace. `SwarmProgressTracker.evictByCase(UUID)` — consumed by CaseStatusChangedHandler.

- [ ] **Step 1: Write failing test**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SwarmProgressTrackerTest {

  private SwarmProgressTracker progressTracker;
  private SignalRegistry signalRegistry;
  private RoleTracker roleTracker;
  private StigmergyCoordinator coordinator;
  private ObservationRegistry observationRegistry;
  private RuleRegistry ruleRegistry;
  private ActivityTracker activityTracker;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    signalRegistry = new SignalRegistry();
    observationRegistry = new ObservationRegistry();
    ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    roleTracker = new RoleTracker(observationRegistry, signalRegistry, ruleRegistry, coordinator);
    progressTracker = new SwarmProgressTracker(signalRegistry, roleTracker, coordinator);
    caseId = UUID.randomUUID();
  }

  @Test
  void emptyProgressOnNewCase() {
    var progress = progressTracker.getProgress(caseId);
    assertEquals(SwarmProgress.EMPTY, progress);
  }

  @Test
  void consensusScoreReflectsSignalConsensus() {
    var config = new StigmergyConfig(
        new StigmergyDefaults(null, 0.01, null, null, null, null, null, null, null),
        new CoordinationConfig(2, null, null),
        new SwarmConfig(null, null, null, null, null, null, null, null, 0.05));
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    signalRegistry.deposit(caseId, "signal-A", 0.8, Duration.ofMinutes(5), "a1", 100);
    signalRegistry.deposit(caseId, "signal-A", 0.8, Duration.ofMinutes(5), "a2", 100);
    signalRegistry.deposit(caseId, "signal-B", 0.5, Duration.ofMinutes(5), "a1", 100);

    var events = progressTracker.evaluate(caseId, config);
    var progress = progressTracker.getProgress(caseId);

    assertEquals(0.5, progress.consensusScore(), 0.01);
    assertFalse(events.isEmpty());
    assertEquals(CaseHubEventType.SWARM_PROGRESS, events.get(0).type());
  }

  @Test
  void evictRemovesState() {
    var config = new StigmergyConfig(null, new CoordinationConfig(2, null, null),
        new SwarmConfig(null, null, null, null, null, null, null, null, null));
    coordinator.initializeCase(caseId, List.of("a1"), config);
    coordinator.agentJoined(caseId, "a1", "b");
    coordinator.agentActivated(caseId, "a1");

    progressTracker.evaluate(caseId, config);
    progressTracker.evictByCase(caseId);
    assertEquals(SwarmProgress.EMPTY, progressTracker.getProgress(caseId));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SwarmProgressTrackerTest -q`
Expected: FAIL — `SwarmProgressTracker` class does not exist

- [ ] **Step 3: Implement SwarmProgressTracker**

Use `ide_create_file` for `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProgressTracker.java`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class SwarmProgressTracker implements Resettable {

  private final SignalRegistry signalRegistry;
  private final RoleTracker roleTracker;
  private final StigmergyCoordinator coordinator;

  private final ConcurrentHashMap<UUID, CaseProgressState> cases = new ConcurrentHashMap<>();

  @Inject
  public SwarmProgressTracker(
      SignalRegistry signalRegistry,
      RoleTracker roleTracker,
      StigmergyCoordinator coordinator) {
    this.signalRegistry = signalRegistry;
    this.roleTracker = roleTracker;
    this.coordinator = coordinator;
  }

  public List<SwarmEvent> evaluate(UUID caseId, StigmergyConfig config) {
    var state = cases.computeIfAbsent(caseId, k -> new CaseProgressState());
    var swarmConfig = config.swarm();

    double exploration = computeExplorationPace(caseId, state);
    double consensus = computeConsensusScore(caseId, config);
    double stability = computeStabilityScore(caseId, state);

    var now = Instant.now();
    var newProgress = new SwarmProgress(exploration, consensus, stability, now);

    double threshold = swarmConfig != null
        ? swarmConfig.effectiveProgressChangeThreshold() : 0.1;
    boolean changed = Math.abs(newProgress.explorationPace() - state.lastProgress.explorationPace()) > threshold
        || Math.abs(newProgress.consensusScore() - state.lastProgress.consensusScore()) > threshold
        || Math.abs(newProgress.stabilityScore() - state.lastProgress.stabilityScore()) > threshold;

    state.lastProgress = newProgress;

    if (changed) {
      return List.of(new SwarmEvent(CaseHubEventType.SWARM_PROGRESS, Map.of(
          "explorationPace", exploration,
          "consensusScore", consensus,
          "stabilityScore", stability,
          "cycle", state.evaluationCount)));
    }
    state.evaluationCount++;
    return List.of();
  }

  private double computeExplorationPace(UUID caseId, CaseProgressState state) {
    var agents = coordinator.activeAgents(caseId);
    Set<String> currentFeatures = new HashSet<>();
    for (var agent : agents) {
      currentFeatures.addAll(roleTracker.effectKeys(caseId, agent.agentId()));
      var allSignals = signalRegistry.getAllSignals(caseId);
      for (var entry : allSignals.entrySet()) {
        if (entry.getValue().sources().contains(agent.agentId())) {
          currentFeatures.add("signal:" + entry.getKey());
        }
      }
    }
    int currentCount = currentFeatures.size();
    state.explorationHistory.add(currentCount);
    int window = 20;
    while (state.explorationHistory.size() > window) state.explorationHistory.removeFirst();

    int max = state.explorationHistory.stream().mapToInt(Integer::intValue).max().orElse(1);
    return max > 0 ? (double) currentCount / max : 0.0;
  }

  private double computeConsensusScore(UUID caseId, StigmergyConfig config) {
    int consensusThreshold = 2;
    double ezThreshold = 0.01;
    if (config.coordination() != null && config.coordination().consensusThreshold() != null) {
      consensusThreshold = config.coordination().consensusThreshold();
    }
    if (config.defaults() != null && config.defaults().effectiveZeroThreshold() != null) {
      ezThreshold = config.defaults().effectiveZeroThreshold();
    }

    var consensus = signalRegistry.consensusSignals(caseId, consensusThreshold, ezThreshold);
    var allSignals = signalRegistry.perceive(caseId, ezThreshold);

    if (allSignals.isEmpty()) return 0.0;
    return (double) consensus.size() / allSignals.size();
  }

  private double computeStabilityScore(UUID caseId, CaseProgressState state) {
    var roles = roleTracker.getDetectedRoles(caseId);
    if (roles.isEmpty()) return 0.0;

    var agents = coordinator.activeAgents(caseId);
    if (agents.isEmpty()) return 0.0;

    Map<String, String> currentAssignment = new HashMap<>();
    for (var role : roles) {
      for (String agent : role.memberAgents()) {
        currentAssignment.put(agent, role.roleId());
      }
    }

    if (state.previousRoleAssignment.isEmpty()) {
      state.previousRoleAssignment = currentAssignment;
      return 0.0;
    }

    int stable = 0;
    for (var agent : agents) {
      String current = currentAssignment.get(agent.agentId());
      String previous = state.previousRoleAssignment.get(agent.agentId());
      if (current != null && current.equals(previous)) {
        stable++;
      }
    }
    state.previousRoleAssignment = currentAssignment;
    return (double) stable / agents.size();
  }

  public SwarmProgress getProgress(UUID caseId) {
    var state = cases.get(caseId);
    return state != null ? state.lastProgress : SwarmProgress.EMPTY;
  }

  public void evictByCase(UUID caseId) {
    cases.remove(caseId);
  }

  @Override
  public void reset() {
    cases.clear();
  }

  private static class CaseProgressState {
    SwarmProgress lastProgress = SwarmProgress.EMPTY;
    int evaluationCount;
    final Deque<Integer> explorationHistory = new ArrayDeque<>();
    Map<String, String> previousRoleAssignment = new HashMap<>();
  }
}
```

- [ ] **Step 4: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SwarmProgressTrackerTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/SwarmProgressTracker.java \
  runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmProgressTrackerTest.java
git commit -m "feat: add SwarmProgressTracker — exploration, consensus, stability metrics

Refs #1112"
```

---

## Batch 4: Wiring + Integration

### Task 5: DefaultMetricsSpace, pipeline wiring, lifecycle integration

Wire the trackers into the runtime: DefaultMetricsSpace facet, WorkerRuntimeFactory wiring, CaseContextChangedEventHandler pipeline integration, CaseStatusChangedHandler lifecycle eviction, RuntimeBeans CDI producers.

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/DefaultMetricsSpace.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java:45-66` — add tracker dependencies
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java:43` — implement `metrics()` method
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — add swarm detection in convergenceDetection()
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — add evictByCase for 3 trackers
- Modify: `runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java` — add producers and update WorkerRuntimeFactory wiring

**Interfaces:**
- Consumes: `RoleTracker`, `TeamDetector`, `SwarmProgressTracker` (all from Batch 2-3), `ActivityTracker` (existing), `StigmergyCoordinator` (existing), `StigmergyConfig` (existing)
- Produces: `DefaultMetricsSpace` — wired into `DefaultWorkerRuntime.metrics()`. Pipeline integration calls `roleTracker.accumulate()`, `roleTracker.detect()`, `teamDetector.detect()`, `progressTracker.evaluate()` in `convergenceDetection()`.

- [ ] **Step 1: Create DefaultMetricsSpace**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import io.casehub.api.engine.MetricsSpace;
import io.casehub.api.model.stigmergy.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public class DefaultMetricsSpace implements MetricsSpace {

  private static final Duration DEFAULT_RATE_WINDOW = Duration.ofSeconds(30);

  private final UUID caseId;
  private final String agentId;
  private final ActivityTracker activityTracker;
  private final RoleTracker roleTracker;
  private final TeamDetector teamDetector;
  private final SwarmProgressTracker progressTracker;

  public DefaultMetricsSpace(
      UUID caseId,
      String agentId,
      ActivityTracker activityTracker,
      RoleTracker roleTracker,
      TeamDetector teamDetector,
      SwarmProgressTracker progressTracker) {
    this.caseId = caseId;
    this.agentId = agentId;
    this.activityTracker = activityTracker;
    this.roleTracker = roleTracker;
    this.teamDetector = teamDetector;
    this.progressTracker = progressTracker;
  }

  @Override
  public Map<String, Double> activityRates() {
    var state = activityTracker.getState(caseId);
    if (state == null) return Map.of();
    var now = Instant.now();
    return Map.of(
        "dispatches", state.dispatchRate(DEFAULT_RATE_WINDOW, now),
        "signalDeposits", state.signalDepositRate(DEFAULT_RATE_WINDOW, now),
        "contextMutations", state.contextMutationRate(DEFAULT_RATE_WINDOW, now),
        "evaluationCycles", state.evaluationRate(DEFAULT_RATE_WINDOW, now));
  }

  @Override
  public Map<String, Long> budgetUsage() {
    var state = activityTracker.getState(caseId);
    if (state == null) return Map.of();
    return Map.of(
        "dispatches", state.totalDispatches(),
        "signalDeposits", state.totalSignalDeposits(),
        "contextMutations", state.totalContextMutations(),
        "evaluationCycles", state.totalEvaluationCycles());
  }

  @Override
  public BehavioralFingerprint myFingerprint() {
    return roleTracker.getFingerprint(caseId, agentId);
  }

  @Override
  public SwarmProgress swarmProgress() {
    return progressTracker.getProgress(caseId);
  }

  @Override
  public List<DetectedRole> detectedRoles() {
    return roleTracker.getDetectedRoles(caseId);
  }

  @Override
  public List<DetectedTeam> detectedTeams() {
    return teamDetector.getDetectedTeams(caseId);
  }
}
```

- [ ] **Step 2: Wire WorkerRuntimeFactory — add tracker dependencies**

Read `WorkerRuntimeFactory.java` constructor, then use `ide_replace_text_in_file` to add the three tracker fields and constructor parameters. Add a `createMetricsSpace(UUID caseId, String agentId)` method that returns `DefaultMetricsSpace` when trackers are available, `MetricsSpace.NOOP` otherwise.

- [ ] **Step 3: Wire DefaultWorkerRuntime — implement metrics()**

Use `ide_replace_text_in_file` to add a `MetricsSpace metricsSpace` field set by the factory, and override `metrics()` to return it.

- [ ] **Step 4: Wire CaseContextChangedEventHandler — swarm detection in convergenceDetection()**

Add `Instance<RoleTracker>`, `Instance<TeamDetector>`, `Instance<SwarmProgressTracker>` constructor parameters. In `convergenceDetection()`, after `coordinator.detectPatterns()`, add the swarm detection block:

```java
if (roleTrackerInstance.isResolvable()) {
    var rt = roleTrackerInstance.get();
    var td = teamDetectorInstance.get();
    var spt = progressTrackerInstance.get();
    var stigConfig = caseDefinition.stigmergyConfig();
    var swarmConfig = stigConfig != null ? stigConfig.swarm() : null;
    if (swarmConfig != null) {
        rt.accumulate(caseId);
        if (rt.shouldDetect(caseId)) {
            dispatchSwarmEvents(caseId, rt.detect(caseId, swarmConfig));
            dispatchSwarmEvents(caseId, td.detect(caseId, swarmConfig));
        }
        dispatchSwarmEvents(caseId, spt.evaluate(caseId, stigConfig));
    }
}
```

- [ ] **Step 5: Wire CaseStatusChangedHandler — lifecycle eviction**

Add `Instance<RoleTracker>`, `Instance<TeamDetector>`, `Instance<SwarmProgressTracker>` constructor parameters. In the terminal status block, call `evictByCase(caseId)` on each (guarded by `isResolvable()`).

- [ ] **Step 6: Wire RuntimeBeans — CDI producers and constructor updates**

No new producers needed for the 3 trackers — they are `@ApplicationScoped` CDI beans with `@Inject` constructors. CDI auto-discovers them.

Update `WorkerRuntimeFactory` producer to pass the 3 trackers. Update `CaseContextChangedEventHandler` and `CaseStatusChangedHandler` producers to pass the `Instance<>` parameters.

- [ ] **Step 7: Compile runtime module**

Run: `/opt/homebrew/bin/mvn compile -pl runtime -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/stigmergy/DefaultMetricsSpace.java \
  runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java \
  runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java \
  runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java
git commit -m "feat: wire swarm trackers into runtime — MetricsSpace, pipeline, lifecycle

Refs #1112"
```

### Task 6: Swarm integration test

End-to-end test verifying the full swarm lifecycle: agents register interests/rules, deposit signals, role detection finds clusters, team detection finds affinity groups, progress tracks metrics, MetricsSpace exposes everything.

**Files:**
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmIntegrationTest.java`

**Interfaces:**
- Consumes: all types from Batch 1-4

- [ ] **Step 1: Write integration test**

Use `ide_create_file`:

```java
package io.casehub.engine.internal.stigmergy;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.stigmergy.*;
import io.casehub.api.spi.observation.*;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.observation.ObservationRegistry;
import io.casehub.engine.common.internal.observation.RuleRegistry;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SwarmIntegrationTest {

  private ObservationRegistry observationRegistry;
  private SignalRegistry signalRegistry;
  private RuleRegistry ruleRegistry;
  private ActivityTracker activityTracker;
  private StigmergyCoordinator coordinator;
  private RoleTracker roleTracker;
  private TeamDetector teamDetector;
  private SwarmProgressTracker progressTracker;
  private UUID caseId;
  private StigmergyConfig config;
  private SwarmConfig swarmConfig;

  @BeforeEach
  void setUp() {
    observationRegistry = new ObservationRegistry();
    signalRegistry = new SignalRegistry();
    ruleRegistry = new RuleRegistry();
    activityTracker = new ActivityTracker();
    coordinator = new StigmergyCoordinator(signalRegistry, observationRegistry, activityTracker);
    roleTracker = new RoleTracker(observationRegistry, signalRegistry, ruleRegistry, coordinator);
    teamDetector = new TeamDetector(observationRegistry, signalRegistry, roleTracker, coordinator);
    progressTracker = new SwarmProgressTracker(signalRegistry, roleTracker, coordinator);

    caseId = UUID.randomUUID();
    swarmConfig = new SwarmConfig(null, 0.5, 2, 20, 1, null, 0.3, 2, 0.05);
    config = new StigmergyConfig(
        new StigmergyDefaults(null, 0.01, null, null, null, null, null, null, null),
        new CoordinationConfig(2, 10.0, 0.6),
        swarmConfig);
  }

  @Test
  void fullSwarmLifecycle() {
    coordinator.initializeCase(caseId, List.of("monitor-1", "monitor-2", "controller-1"), config);
    for (String id : List.of("monitor-1", "monitor-2", "controller-1")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(caseId, "monitor-1", "b-monitor-1",
        new TestObserver("temp-obs", Set.of("tempReading", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "monitor-2", "b-monitor-2",
        new TestObserver("pressure-obs", Set.of("tempReading", "pressure")), 20);
    observationRegistry.registerObserver(caseId, "controller-1", "b-controller-1",
        new TestObserver("cooling-obs", Set.of("coolingAction")), 20);

    signalRegistry.deposit(caseId, "overheating", 0.8, Duration.ofMinutes(5), "monitor-1", 100);
    signalRegistry.deposit(caseId, "overheating", 0.9, Duration.ofMinutes(5), "monitor-2", 100);

    roleTracker.accumulate(caseId);
    var roleEvents = roleTracker.detect(caseId, swarmConfig);
    var teamEvents = teamDetector.detect(caseId, swarmConfig);
    var progressEvents = progressTracker.evaluate(caseId, config);

    var roles = roleTracker.getDetectedRoles(caseId);
    assertEquals(1, roles.size(), "monitors should cluster into one role");
    assertTrue(roles.get(0).memberAgents().containsAll(Set.of("monitor-1", "monitor-2")));

    var teams = teamDetector.getDetectedTeams(caseId);
    assertFalse(teams.isEmpty(), "monitors should form a team via shared interests+signals");

    var progress = progressTracker.getProgress(caseId);
    assertTrue(progress.consensusScore() > 0, "consensus should be non-zero (overheating has 2 sources)");

    assertFalse(roleEvents.isEmpty());
    assertTrue(roleEvents.stream().anyMatch(e -> e.type() == CaseHubEventType.SWARM_ROLE_EMERGED));
  }

  @Test
  void metricsSpaceExposesAllData() {
    coordinator.initializeCase(caseId, List.of("a1", "a2"), config);
    for (String id : List.of("a1", "a2")) {
      coordinator.agentJoined(caseId, id, "b-" + id);
      coordinator.agentActivated(caseId, id);
    }

    observationRegistry.registerObserver(caseId, "a1", "b-a1",
        new TestObserver("obs", Set.of("key1")), 20);

    roleTracker.accumulate(caseId);
    progressTracker.evaluate(caseId, config);

    var metrics = new DefaultMetricsSpace(
        caseId, "a1", activityTracker, roleTracker, teamDetector, progressTracker);

    assertNotNull(metrics.myFingerprint());
    assertFalse(metrics.myFingerprint().perception().isEmpty());
    assertNotNull(metrics.swarmProgress());
    assertNotNull(metrics.detectedRoles());
    assertNotNull(metrics.detectedTeams());
    assertNotNull(metrics.activityRates());
    assertNotNull(metrics.budgetUsage());
  }

  @Test
  void evictionCleansAllTrackers() {
    coordinator.initializeCase(caseId, List.of("a1"), config);
    coordinator.agentJoined(caseId, "a1", "b-a1");
    coordinator.agentActivated(caseId, "a1");
    roleTracker.accumulate(caseId);
    roleTracker.detect(caseId, swarmConfig);
    teamDetector.detect(caseId, swarmConfig);
    progressTracker.evaluate(caseId, config);

    roleTracker.evictByCase(caseId);
    teamDetector.evictByCase(caseId);
    progressTracker.evictByCase(caseId);
    coordinator.evictByCase(caseId);

    assertTrue(roleTracker.getDetectedRoles(caseId).isEmpty());
    assertTrue(teamDetector.getDetectedTeams(caseId).isEmpty());
    assertEquals(SwarmProgress.EMPTY, progressTracker.getProgress(caseId));
  }

  static class TestObserver implements EnvironmentObserver {
    private final String type;
    private final Set<String> keys;

    TestObserver(String type, Set<String> keys) {
      this.type = type;
      this.keys = keys;
    }

    @Override
    public String observerType() { return type; }

    @Override
    public Set<String> watchedKeys() { return keys; }

    @Override
    public List<Observation> observe(ObservationContext context) { return List.of(); }
  }
}
```

- [ ] **Step 2: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=SwarmIntegrationTest -q`
Expected: PASS

- [ ] **Step 3: Run full module test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -q`
Expected: PASS — no regressions

- [ ] **Step 4: Commit**

```bash
git add runtime-core/src/test/java/io/casehub/engine/internal/stigmergy/SwarmIntegrationTest.java
git commit -m "test: add swarm integration test — full lifecycle verification

Refs #1112"
```

## References

- [2026-09-18-swarm-execution-model-design.md] — design spec this plan implements
- [StigmergyCoordinator.java:32] — agent lifecycle tracking, pattern detection
- [StigmergyConfig.java:20] — existing config record (2 fields → 3 fields)
- [WorkerRuntime.java:24] — api coordination surface (facets)
- [ObservationRegistry.java:29] — observer storage, getRegistrationsForAgent()
- [SignalRegistry.java:37] — signal storage, sources tracking, consensusSignals()
- [RuleRegistry.java:32] — rule storage, getFirings()
- [ActivityTracker.java:25] — sliding-window rate computation
- [CaseHubEventType.java:18] — event enum (add 7 values)
- [WorkerRuntimeFactory.java:25] — factory wiring
- [RuntimeBeans.java:96] — CDI producers
- [StigmergyCoordinatorTest.java:30] — test pattern reference
- [InterestDeclaration.java:22] — sealed hierarchy (5 variants for key extraction)
- [RuleFiring.java:21] — record(ruleId, executedActions, firedAt)
- [RuleAction.java:22] — sealed hierarchy (WriteContext for effect extraction)
- [GitHub #1112] — focal issue
- [GitHub #1104] — Hive Mind epic
- [D73-D83] — design decisions
