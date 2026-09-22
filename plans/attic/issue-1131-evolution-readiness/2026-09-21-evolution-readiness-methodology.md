# Evolution Readiness Methodology Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1131 — feat: Evolution readiness methodology — L0-L3 compliance levels
**Issue group:** #1131, #1132

**Goal:** Make the evolution loop operational by providing 10 concrete CapabilityArea implementations, L0–L3 compliance levels, a readiness validator, and CDI bootstrap wiring.

**Architecture:** Each CapabilityArea directly queries EventLog via `findByCaseAndTypes(caseId, types, tenancyId)` — no intermediate MetricSource SPI. Areas are plain `@ApplicationScoped` beans (not `@DefaultBean`) auto-registered by a CDI bootstrap observer. Override happens via `CapabilityAreaRegistry.register()` at higher `@Priority`. Per-area ComplianceLevel with global rollup as `min(area levels where level > L0)`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Quarkus Arc), JUnit 5, AssertJ

## Global Constraints

- Build: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
- Test: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime-core`
- CapabilityArea implementations are `@ApplicationScoped` — NOT `@DefaultBean` (multi-instance SPI, see spec §2)
- EventLog queries always require `tenancyId` — use `findByCaseAndTypes(caseId, types, tenancyId)`
- Test classes must be `*Test.java`, never `*IT.java`
- Inject by SPI interface, never by concrete class
- All new CaseHubEventType values must be added to the existing enum (not a new file)
- `Map.of()` supports max 10 entries — exactly 10 areas, so use `Map.of()` not `Map.ofEntries()`

---

## Batch 1: SPI Surface + tenancyId Threading

After this batch: the API surface is stable, `CapabilityArea.assess()` takes tenancyId, `HealthScoreTracker` excludes ABSENT areas, all existing tests pass with updated signatures.

### Task 1: API Model Types and SPI Changes

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ComplianceLevel.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ComplianceChecklist.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ReadinessReport.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ComplianceChecklistProvider.java`
- Modify: `api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java` — add tenancyId to assess()
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add 2 new values
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java` — update weights 8→10

**Interfaces:**
- Produces: `ComplianceLevel` enum (L0_INERT, L1_OBSERVE, L2_PROPOSE, L3_AUTONOMOUS)
- Produces: `ComplianceChecklist(String areaId, ComplianceLevel level, List<CheckRequirement> requirements)`
- Produces: `ReadinessReport(ComplianceLevel targetLevel, ComplianceLevel projectLevel, List<AreaCompliance> areas, boolean passed, Instant evaluatedAt)`
- Produces: `ComplianceChecklistProvider.checklistFor(String areaId, ComplianceLevel level)`
- Produces: `CapabilityArea.assess(UUID caseId, String tenancyId)` — updated signature

- [ ] **Step 1: Create ComplianceLevel enum**

```java
// api/src/main/java/io/casehub/api/model/stigmergy/ComplianceLevel.java
package io.casehub.api.model.stigmergy;

public enum ComplianceLevel {
  L0_INERT,
  L1_OBSERVE,
  L2_PROPOSE,
  L3_AUTONOMOUS
}
```

- [ ] **Step 2: Create ComplianceChecklist record**

```java
// api/src/main/java/io/casehub/api/model/stigmergy/ComplianceChecklist.java
package io.casehub.api.model.stigmergy;

import java.util.List;

public record ComplianceChecklist(
    String areaId,
    ComplianceLevel level,
    List<CheckRequirement> requirements) {

  public record CheckRequirement(
      String name,
      String description,
      CheckType type) {

    public enum CheckType {
      DATA_FLOW,
      CONFIGURATION,
      INFRASTRUCTURE
    }
  }
}
```

- [ ] **Step 3: Create ReadinessReport record**

```java
// api/src/main/java/io/casehub/api/model/stigmergy/ReadinessReport.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.time.Instant;
import java.util.List;

public record ReadinessReport(
    ComplianceLevel targetLevel,
    ComplianceLevel projectLevel,
    List<AreaCompliance> areas,
    boolean passed,
    Instant evaluatedAt) {

  public record AreaCompliance(
      String areaId,
      ComplianceLevel areaLevel,
      List<CheckResult> checks) {}

  public record CheckResult(
      String name,
      boolean satisfied,
      String expected,
      String actual,
      @Nullable String remediation) {}
}
```

- [ ] **Step 4: Create ComplianceChecklistProvider SPI**

```java
// api/src/main/java/io/casehub/api/spi/improvement/ComplianceChecklistProvider.java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.ComplianceChecklist;
import io.casehub.api.model.stigmergy.ComplianceLevel;

public interface ComplianceChecklistProvider {
  ComplianceChecklist checklistFor(String areaId, ComplianceLevel level);
}
```

- [ ] **Step 5: Update CapabilityArea SPI — add tenancyId**

Change `assess(UUID caseId)` to `assess(UUID caseId, String tenancyId)`:

```java
CapabilityAreaAssessment assess(UUID caseId, String tenancyId);
```

- [ ] **Step 6: Add CaseHubEventType values**

Add after `IMPROVEMENT_CONFLICT_DETECTED`:

```java
COMPLIANCE_LEVEL_CHANGED,
READINESS_EVALUATED
```

- [ ] **Step 7: Update HealthPolicy.effectiveWeights() — 8→10 areas**

Replace the existing `Map.of(...)` in `effectiveWeights()`:

```java
public Map<String, Double> effectiveWeights() {
  if (weights != null) return weights;
  return Map.of(
      "stability", 0.15,
      "performance", 0.12,
      "execution", 0.10,
      "safety", 0.12,
      "integration", 0.08,
      "autonomy", 0.08,
      "cognitive-reasoning", 0.10,
      "cognitive-memory", 0.08,
      "coordination", 0.08,
      "perception", 0.09);
}
```

- [ ] **Step 8: Build api module to verify compilation**

Run: `/opt/homebrew/bin/mvn install -pl api -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests -q`
Expected: BUILD SUCCESS (api module compiles; downstream modules will fail until Task 2 threads tenancyId)

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/ComplianceLevel.java \
       api/src/main/java/io/casehub/api/model/stigmergy/ComplianceChecklist.java \
       api/src/main/java/io/casehub/api/model/stigmergy/ReadinessReport.java \
       api/src/main/java/io/casehub/api/spi/improvement/ComplianceChecklistProvider.java \
       api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java \
       api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java \
       api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java
git commit -m "feat(#1131): add compliance model types, update CapabilityArea SPI with tenancyId, expand HealthPolicy to 10 areas

Refs #1131"
```

### Task 2: Thread tenancyId + ABSENT Exclusion

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreaker.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/HealthScoreTrackerTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionTickerTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreakerTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/RegressionDetectorTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ContinuousEvolutionIntegrationTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConfidenceScorerTest.java`

**Interfaces:**
- Consumes: `CapabilityArea.assess(UUID caseId, String tenancyId)` (from Task 1)
- Produces: `HealthScoreTracker.computeScore(UUID caseId, String tenancyId, HealthPolicy policy)`
- Produces: `HealthScoreTracker.refresh(UUID caseId, String tenancyId, HealthPolicy policy)`

- [ ] **Step 1: Update HealthScoreTracker — add tenancyId + ABSENT exclusion**

Add `tenancyId` parameter to `computeScore()` and `refresh()`. In both methods, skip areas where `assessment.landscapePosition() == LandscapePosition.ABSENT`:

```java
public double computeScore(UUID caseId, String tenancyId, HealthPolicy policy) {
  var weights = policy.effectiveWeights();
  double weightedSum = 0.0;
  double totalWeight = 0.0;
  for (CapabilityArea area : areaRegistry.active()) {
    var assessment = area.assess(caseId, tenancyId);
    if (assessment.landscapePosition() == CapabilityAreaAssessment.LandscapePosition.ABSENT) {
      continue;
    }
    double weight = weights.getOrDefault(area.id(), 0.1);
    weightedSum += weight * assessment.healthScore();
    totalWeight += weight;
  }
  return totalWeight > 0 ? weightedSum / totalWeight : 0.0;
}

public void refresh(UUID caseId, String tenancyId, HealthPolicy policy) {
  var weights = policy.effectiveWeights();
  Map<String, Double> components = new LinkedHashMap<>();
  double weightedSum = 0.0;
  double totalWeight = 0.0;
  for (CapabilityArea area : areaRegistry.active()) {
    var assessment = area.assess(caseId, tenancyId);
    if (assessment.landscapePosition() == CapabilityAreaAssessment.LandscapePosition.ABSENT) {
      continue;
    }
    double healthScore = assessment.healthScore();
    components.put(area.id(), healthScore);
    double weight = weights.getOrDefault(area.id(), 0.1);
    weightedSum += weight * healthScore;
    totalWeight += weight;
  }
  double score = totalWeight > 0 ? weightedSum / totalWeight : 0.0;
  var snapshot = new HealthSnapshot(score, Instant.now(), components);
  var deque = history.computeIfAbsent(caseId, k -> new ArrayDeque<>());
  deque.addLast(snapshot);
  while (deque.size() > MAX_HISTORY_SIZE) {
    deque.removeFirst();
  }
}
```

- [ ] **Step 2: Update ImprovementCircuitBreaker.evaluate() — add tenancyId**

Thread tenancyId through to `tracker.computeScore()`. Find the `evaluate` method and add `tenancyId` as second parameter, passing it to `tracker.computeScore(caseId, tenancyId, policy)`.

- [ ] **Step 3: Update EvolutionTicker.tick() — pass tenancyId through**

Update calls inside `tick()`:
- `healthTracker.refresh(caseId, tenancyId, config.effectiveHealthPolicy())`
- `circuitBreaker.evaluate(caseId, tenancyId, healthTracker, config.effectiveHealthPolicy())`

- [ ] **Step 4: Update RegressionDetector — thread tenancyId if needed**

Check `checkActiveMonitors` — it uses `healthTracker.latestSnapshot()` and `healthTracker.delta()` which don't need tenancyId (they read from the in-memory history). No change needed unless `checkActiveMonitors` calls `computeScore` directly.

- [ ] **Step 5: Fix test helper — update area() method in all test classes**

The anonymous `CapabilityArea` created by the `area()` helper in `ContinuousEvolutionIntegrationTest` and other test files must match the new signature:

```java
private CapabilityArea area(String id, double health) {
  return new CapabilityArea() {
    public String id() { return id; }
    public String name() { return id; }
    public String description() { return "test"; }
    public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
      return new CapabilityAreaAssessment(
          id, health,
          CapabilityAreaAssessment.LandscapePosition.AT_PARITY,
          0.5, 0.3, 1.67, Instant.now());
    }
  };
}
```

Update all test call sites that pass `caseId` to `computeScore`/`refresh` to also pass `"test-tenant"`.

- [ ] **Step 6: Write ABSENT exclusion test**

Add to `HealthScoreTrackerTest`:

```java
@Test
void absentAreasExcludedFromScore() {
  var absentArea = new CapabilityArea() {
    public String id() { return "absent-area"; }
    public String name() { return "Absent"; }
    public String description() { return "test"; }
    public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
      return new CapabilityAreaAssessment("absent-area", 0.5,
          CapabilityAreaAssessment.LandscapePosition.ABSENT,
          0.0, 0.0, 0.0, Instant.now());
    }
  };
  areaRegistry.register(absentArea);
  areaRegistry.register(area("stability", 0.9));

  var policy = new HealthPolicy(null, null, null, null, null, null);
  double score = healthTracker.computeScore(caseId, "test-tenant", policy);

  // Only the non-ABSENT area contributes — score should be 0.9, not (0.5+0.9)/2
  assertThat(score).isCloseTo(0.9, within(0.01));
}
```

- [ ] **Step 7: Run all existing improvement tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest="io.casehub.engine.internal.improvement.*" -Dcheckstyle.skip=true -Dspotless.check.skip=true`
Expected: All tests pass with updated signatures

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java \
       runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreaker.java \
       runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java \
       runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java \
       runtime-core/src/test/
git commit -m "feat(#1131): thread tenancyId through evolution pipeline, add ABSENT exclusion to HealthScoreTracker

Refs #1131"
```

---

## Batch 2: Core Capability Areas (stability, performance, execution, safety)

After this batch: 4 areas produce real health data from EventLog events. Tests verify health score computation, neutral fallback, and landscape position derivation.

### Task 3: StabilityCapabilityArea + PerformanceCapabilityArea

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/AbstractCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/StabilityCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/PerformanceCapabilityArea.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/StabilityCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/PerformanceCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `EventLogRepository.findByCaseAndTypes(UUID, Collection<CaseHubEventType>, String)`
- Produces: `StabilityCapabilityArea implements CapabilityArea` (id="stability")
- Produces: `PerformanceCapabilityArea implements CapabilityArea` (id="performance")

- [ ] **Step 1: Create AbstractCapabilityArea — shared utilities**

```java
// runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/AbstractCapabilityArea.java
package io.casehub.engine.internal.improvement.area;

import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.api.model.stigmergy.CapabilityAreaAssessment.LandscapePosition;
import io.casehub.api.spi.improvement.CapabilityArea;
import java.time.Instant;

public abstract class AbstractCapabilityArea implements CapabilityArea {

  protected CapabilityAreaAssessment neutralAssessment() {
    return new CapabilityAreaAssessment(
        id(), 0.5, LandscapePosition.ABSENT, 0.0, 0.0, 0.0, Instant.now());
  }

  protected LandscapePosition positionFromScore(double score) {
    if (score < 0.4) return LandscapePosition.BEHIND;
    if (score < 0.7) return LandscapePosition.AT_PARITY;
    return LandscapePosition.AHEAD;
  }

  protected double impactEstimate(double healthScore) {
    return 1.0 - healthScore;
  }

  protected double costEstimate(double healthScore) {
    return healthScore < 0.5 ? 0.7 : 0.3;
  }

  protected double roi(double healthScore) {
    double cost = costEstimate(healthScore);
    return cost > 0 ? impactEstimate(healthScore) / cost : 0.0;
  }
}
```

- [ ] **Step 2: Write StabilityCapabilityAreaTest — failing tests**

```java
@Test
void noEvents_returnsNeutralAssessment() {
  var area = new StabilityCapabilityArea(eventLog);
  var assessment = area.assess(caseId, tenancyId);
  assertThat(assessment.healthScore()).isEqualTo(0.5);
  assertThat(assessment.landscapePosition()).isEqualTo(LandscapePosition.ABSENT);
}

@Test
void allCompleted_returnsHealthScore1() {
  appendEvent(CaseHubEventType.CASE_COMPLETED);
  appendEvent(CaseHubEventType.CASE_COMPLETED);
  var assessment = area.assess(caseId, tenancyId);
  assertThat(assessment.healthScore()).isEqualTo(1.0);
  assertThat(assessment.landscapePosition()).isEqualTo(LandscapePosition.AHEAD);
}

@Test
void mixedOutcomes_returnsRatio() {
  appendEvent(CaseHubEventType.CASE_COMPLETED);
  appendEvent(CaseHubEventType.CASE_FAULTED);
  var assessment = area.assess(caseId, tenancyId);
  assertThat(assessment.healthScore()).isEqualTo(0.5);
  assertThat(assessment.landscapePosition()).isEqualTo(LandscapePosition.AT_PARITY);
}
```

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement StabilityCapabilityArea**

Per spec §2 — queries `CASE_COMPLETED`, `CASE_FAULTED`, `CASE_CANCELLED`. Health = completed / total.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Write PerformanceCapabilityAreaTest — failing tests**

Test cases: no events → neutral, all fast cases → high score, slow cases → low score. The area queries `CASE_STARTED` + `CASE_COMPLETED` and computes the ratio completing within a duration threshold.

- [ ] **Step 7: Run tests to verify they fail**

- [ ] **Step 8: Implement PerformanceCapabilityArea**

Queries `CASE_STARTED` and `CASE_COMPLETED` events. For each COMPLETED event, finds the matching STARTED event by caseId and computes the duration. Health = ratio of cases under threshold (default 60 seconds — configurable via a constant).

- [ ] **Step 9: Run tests to verify they pass**

- [ ] **Step 10: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/
git commit -m "feat(#1131): add StabilityCapabilityArea and PerformanceCapabilityArea with tests

Refs #1131"
```

### Task 4: ExecutionCapabilityArea + SafetyCapabilityArea

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ExecutionCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/SafetyCapabilityArea.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/ExecutionCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/SafetyCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `AbstractCapabilityArea` (from Task 3)
- Produces: `ExecutionCapabilityArea implements CapabilityArea` (id="execution")
- Produces: `SafetyCapabilityArea implements CapabilityArea` (id="safety")

- [ ] **Step 1: Write ExecutionCapabilityAreaTest**

Test: no events → neutral. Worker events: COMPLETED / (COMPLETED + FAILED + DECLINED + OUTCOME_FAILED). Mixed outcomes produce correct ratio.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement ExecutionCapabilityArea**

Queries `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_DECLINED`, `WORKER_OUTCOME_FAILED`. Health = COMPLETED / total.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Write SafetyCapabilityAreaTest**

Test: no events → neutral. Safety violation events (`BUDGET_EXHAUSTED`, `CIRCUIT_BREAKER_TRIPPED`, `ACTION_GATE_REJECTED`): 0 violations = 1.0, N violations drives score toward 0. Use formula: `1.0 / (1.0 + violations)`.

- [ ] **Step 6: Run tests to verify they fail**

- [ ] **Step 7: Implement SafetyCapabilityArea**

Queries safety event types. Health = `1.0 / (1.0 + count)` — asymptotic decay. 0 violations = 1.0, 1 violation = 0.5, 4 violations = 0.2.

- [ ] **Step 8: Run tests to verify they pass**

- [ ] **Step 9: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/
git commit -m "feat(#1131): add ExecutionCapabilityArea and SafetyCapabilityArea with tests

Refs #1131"
```

---

## Batch 3: Extended Capability Areas (integration, coordination, heuristics)

After this batch: all 10 areas exist. Measurable areas query EventLog; heuristic areas return neutral until the cognitive layer ships.

### Task 5: IntegrationCapabilityArea + CoordinationCapabilityArea

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/IntegrationCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/CoordinationCapabilityArea.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/IntegrationCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/CoordinationCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `AbstractCapabilityArea` (from Task 3)
- Produces: `IntegrationCapabilityArea implements CapabilityArea` (id="integration")
- Produces: `CoordinationCapabilityArea implements CapabilityArea` (id="coordination")

- [ ] **Step 1: Write IntegrationCapabilityAreaTest**

Queries `ORCHESTRATION_COMPLETED`, `ORCHESTRATION_ESCALATED`, `WORKFLOW_STEP_COMPLETED`, `WORKFLOW_STEP_FAILED`. Health = successful / total.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement IntegrationCapabilityArea**

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Write CoordinationCapabilityAreaTest**

Queries stigmergy/swarm events. Success events: `CONVERGENCE_DETECTED`, `SWARM_TEAM_FORMED`, `SWARM_PROVISION_COMPLETED`. Failure events: `COORDINATION_STORM_DETECTED`, `SWARM_PROVISION_FAILED`. Health = success / total.

- [ ] **Step 6: Run tests to verify they fail**

- [ ] **Step 7: Implement CoordinationCapabilityArea**

- [ ] **Step 8: Run tests to verify they pass**

- [ ] **Step 9: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/
git commit -m "feat(#1131): add IntegrationCapabilityArea and CoordinationCapabilityArea with tests

Refs #1131"
```

### Task 6: Heuristic Areas (perception, autonomy, cognitive-reasoning, cognitive-memory)

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/PerceptionCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/AutonomyCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/CognitiveReasoningCapabilityArea.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/CognitiveMemoryCapabilityArea.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/PerceptionCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/AutonomyCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/CognitiveReasoningCapabilityAreaTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/CognitiveMemoryCapabilityAreaTest.java`

**Interfaces:**
- Consumes: `AbstractCapabilityArea` (from Task 3)
- Produces: 4 CapabilityArea implementations (ids: "perception", "autonomy", "cognitive-reasoning", "cognitive-memory")

- [ ] **Step 1: Write PerceptionCapabilityAreaTest**

Queries `OBSERVATION_DETECTED`, `PHEROMONE_DEPOSITED`, `SIGNAL_RECEIVED`. Health = activity rate normalised to [0, 1]. No events → neutral.

- [ ] **Step 2: Implement PerceptionCapabilityArea**

- [ ] **Step 3: Write AutonomyCapabilityAreaTest**

Queries `IMPROVEMENT_GOAL_FORMED`, `IMPROVEMENT_OUTCOME`, `SWARM_PROVISION_REQUESTED`, `SWARM_PROVISION_COMPLETED`. Health = autonomous actions / total case activity events.

- [ ] **Step 4: Implement AutonomyCapabilityArea**

- [ ] **Step 5: Write CognitiveReasoningCapabilityAreaTest**

Queries `GOAL_FORMED`, `GOAL_REVISED`, `GOAL_REACHED`, `PLAN_ADAPTED`, `PLAN_CONCEDED`. Health = `GOAL_REACHED / GOAL_FORMED`. No events → neutral.

- [ ] **Step 6: Implement CognitiveReasoningCapabilityArea**

- [ ] **Step 7: Write CognitiveMemoryCapabilityAreaTest**

Returns neutral assessment (0.5, ABSENT) — CBR traces don't emit EventLog entries. Test that it always returns neutral regardless of events.

- [ ] **Step 8: Implement CognitiveMemoryCapabilityArea**

Always returns `neutralAssessment()`. Placeholder until cognitive layer provides real data.

- [ ] **Step 9: Run all area tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest="io.casehub.engine.internal.improvement.area.*" -Dcheckstyle.skip=true -Dspotless.check.skip=true`
Expected: All area tests pass

- [ ] **Step 10: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/area/ \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/area/
git commit -m "feat(#1131): add heuristic capability areas (perception, autonomy, cognitive-reasoning, cognitive-memory)

Refs #1131"
```

---

## Batch 4: Bootstrap + Compliance Validation

After this batch: areas auto-register at startup, ReadinessValidator checks compliance, integration tests verify L0→L3 progression.

### Task 7: CapabilityAreaBootstrap + DefaultComplianceChecklistProvider

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CapabilityAreaBootstrap.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultComplianceChecklistProvider.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/CapabilityAreaBootstrapTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/DefaultComplianceChecklistProviderTest.java`

**Interfaces:**
- Consumes: `CapabilityAreaRegistry.register(CapabilityArea)` (existing)
- Consumes: `ComplianceChecklistProvider` SPI (from Task 1)
- Produces: `CapabilityAreaBootstrap` — CDI startup observer
- Produces: `DefaultComplianceChecklistProvider implements ComplianceChecklistProvider`

- [ ] **Step 1: Write CapabilityAreaBootstrapTest**

Test that `onStartup` registers all injected areas with the registry. Use a list of mock CapabilityAreas, call `onStartup()`, verify `registry.active()` contains them all.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement CapabilityAreaBootstrap**

```java
@ApplicationScoped
public class CapabilityAreaBootstrap {

  @Inject CapabilityAreaRegistry registry;
  @Inject @Any Instance<CapabilityArea> areaInstances;

  void onStartup(@Observes StartupEvent event) {
    for (CapabilityArea area : areaInstances) {
      registry.register(area);
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write DefaultComplianceChecklistProviderTest**

Test all combinations: each level (L0–L3) returns the correct checklist with the right requirement names and types. L0 has no requirements. L1 requires area registration + non-neutral data. L2 adds evolution-enabled + signals. L3 adds rollback policy + health threshold.

- [ ] **Step 6: Run test to verify it fails**

- [ ] **Step 7: Implement DefaultComplianceChecklistProvider**

`@DefaultBean @ApplicationScoped` — 1:1 SPI, so @DefaultBean is correct here. Returns checklists per area per level. L0: empty list. L1: `<areaId>-area-registered` (INFRASTRUCTURE), `<areaId>-has-data` (DATA_FLOW), `<areaId>-non-neutral-score` (DATA_FLOW). L2: all L1 + `evolution-enabled` (CONFIGURATION), `consensus-threshold-met` (CONFIGURATION). L3: all L2 + `rollback-policy-configured` (CONFIGURATION), `health-threshold-configured` (CONFIGURATION).

- [ ] **Step 8: Run test to verify it passes**

- [ ] **Step 9: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/CapabilityAreaBootstrap.java \
       runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultComplianceChecklistProvider.java \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/
git commit -m "feat(#1131): add CapabilityAreaBootstrap and DefaultComplianceChecklistProvider

Refs #1131"
```

### Task 8: ReadinessValidator + Integration Tests

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ReadinessValidator.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ReadinessValidatorTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ReadinessProgressionIntegrationTest.java`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/HealthScoreWithRealAreasTest.java`

**Interfaces:**
- Consumes: `CapabilityAreaRegistry` (existing), `ComplianceChecklistProvider` (from Task 7), `EventLogRepository` (existing), `SignalRegistry` (existing)
- Produces: `ReadinessValidator.validate(UUID caseId, String tenancyId, ComplianceLevel targetLevel, ImprovementConfig config) → ReadinessReport`

- [ ] **Step 1: Write ReadinessValidatorTest — failing tests**

Test cases:
1. No areas registered, target L1 → projectLevel=L0, passed=false
2. Stability registered with data, target L1 → projectLevel=L1, passed=true
3. Target L2 without evolutionEnabled → passed=false, remediation hint
4. Target L3 without rollback policy → passed=false
5. Per-area breakdown shows individual area levels
6. Project level = min of non-L0 area levels

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement ReadinessValidator**

```java
@ApplicationScoped
public class ReadinessValidator {

  private final CapabilityAreaRegistry areaRegistry;
  private final ComplianceChecklistProvider checklistProvider;
  private final EventLogRepository eventLogRepository;

  @Inject
  public ReadinessValidator(
      CapabilityAreaRegistry areaRegistry,
      ComplianceChecklistProvider checklistProvider,
      EventLogRepository eventLogRepository) {
    this.areaRegistry = areaRegistry;
    this.checklistProvider = checklistProvider;
    this.eventLogRepository = eventLogRepository;
  }

  public ReadinessReport validate(UUID caseId, String tenancyId,
      ComplianceLevel targetLevel, ImprovementConfig config) {
    var areas = new ArrayList<ReadinessReport.AreaCompliance>();
    for (CapabilityArea area : areaRegistry.active()) {
      var areaLevel = computeAreaLevel(area, caseId, tenancyId, config);
      var checks = evaluateChecks(area, caseId, tenancyId, targetLevel, config);
      areas.add(new ReadinessReport.AreaCompliance(area.id(), areaLevel, checks));
    }
    var projectLevel = computeProjectLevel(areas);
    boolean passed = projectLevel.compareTo(targetLevel) >= 0;
    var report = new ReadinessReport(targetLevel, projectLevel, areas, passed, Instant.now());
    emitEvents(caseId, tenancyId, report);
    return report;
  }

  private ComplianceLevel computeAreaLevel(CapabilityArea area, UUID caseId,
      String tenancyId, ImprovementConfig config) {
    // Try each level from L3 down to L0
    // Return the highest level where all checks pass
    for (var level : List.of(ComplianceLevel.L3_AUTONOMOUS, ComplianceLevel.L2_PROPOSE,
        ComplianceLevel.L1_OBSERVE)) {
      var checks = evaluateChecks(area, caseId, tenancyId, level, config);
      if (checks.stream().allMatch(ReadinessReport.CheckResult::satisfied)) {
        return level;
      }
    }
    return ComplianceLevel.L0_INERT;
  }

  private ComplianceLevel computeProjectLevel(List<ReadinessReport.AreaCompliance> areas) {
    return areas.stream()
        .map(ReadinessReport.AreaCompliance::areaLevel)
        .filter(l -> l != ComplianceLevel.L0_INERT)
        .min(Comparator.naturalOrder())
        .orElse(ComplianceLevel.L0_INERT);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Write HealthScoreWithRealAreasTest**

Register all 10 areas with an InMemoryEventLogRepository. Append a mix of events. Call `healthTracker.refresh()`. Verify the composite score reflects only non-ABSENT areas with correct weight normalization.

- [ ] **Step 6: Run test to verify it passes**

- [ ] **Step 7: Write ReadinessProgressionIntegrationTest**

Full L0→L1→L2→L3 progression:
1. Start with no config → validate L1 → FAIL (no events)
2. Append CASE_COMPLETED events → validate L1 → PASS
3. Set evolutionEnabled=true, add signal sources → validate L2 → PASS
4. Add rollback policy, health threshold → validate L3 → PASS

- [ ] **Step 8: Run all improvement tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest="io.casehub.engine.internal.improvement.*" -Dcheckstyle.skip=true -Dspotless.check.skip=true`
Expected: All tests pass — existing + new

- [ ] **Step 9: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ReadinessValidator.java \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/
git commit -m "feat(#1131): add ReadinessValidator with L0-L3 progression and integration tests

Closes #1131"
```

---

## References

- [2026-09-21-evolution-readiness-methodology-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java] — SPI interface (modified)
- [api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java] — weights (modified 8→10)
- [api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java] — event types
- [runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java] — score aggregation (modified)
- [runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java] — tick pipeline (modified)
- [runtime-core/src/test/java/io/casehub/engine/internal/improvement/ContinuousEvolutionIntegrationTest.java] — existing tests (modified)
- [common-core/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java] — EventLog query API
- [common-core/src/main/java/io/casehub/engine/common/internal/history/EventLog.java] — event domain object
- [docs/guides/contributor-guide.md] — SPI patterns, CDI conventions, test conventions
- [specs/issue-1104-hive-mind/decisions.md] D109 — 10 bootstrap areas
- [GitHub #1131] — focal issue
- [GitHub #1132] — command centre conductor (consumes ReadinessReport)
