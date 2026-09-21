# Continuous Evolution Loop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1115 — feat: Continuous evolution loop — self-directed growth with quality gates
**Issue group:** #1104, #1105, #1106, #1107, #1108, #1109, #1110, #1111, #1112, #1113, #1114, #1115

**Goal:** Add the continuous improvement loop to the engine: standing directive, outcome feedback, regression detection with confidence-tiered rollback, health-score circuit breaker, conflict avoidance, capability area taxonomy, and research pipeline SPIs.

**Architecture:** `EvolutionTicker` is the single entry point for all improvement proposals, routing through a gate pipeline: opt-in → health refresh → regression monitoring → circuit breaker → consensus → category suppression → anti-oscillation → budget → conflict → `GoalFormationService.propose()`. Outcomes feed back through `ImprovementCategoryTracker` (priority modulation), `RegressionDetector` (confidence-tiered rollback), and `HealthScoreTracker` (circuit breaker). Growth direction uses `CapabilityArea` SPI with 4 research pipeline SPIs.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, Maven

## Global Constraints

- Records use `@Nullable` fields with `effective*()` default methods
- Workers return `WorkerResult`, `WorkerScope` is a parameter — cast to `WorkerRuntime` for engine methods
- Test classes must be `*Test.java`, never `*IT.java`
- `@ObservesAsync` unreliable in `@QuarkusTest` — inject bean and call observer method directly
- All `@ApplicationScoped` beans with mutable state implement `Resettable`
- Build: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
- Tests: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dcheckstyle.skip=true -Dspotless.check.skip=true`
- `GoalFormationService.propose(String agentId, String tenancyId, GoalFormationProposal proposal)` — returns `GoalFormationResult`
- Structural deny list: new safety-critical components MUST be added to `ImprovementBudgetEnforcer.STRUCTURAL_DENIED_PATTERNS`

---

## Batch 1: Configuration Records

Safe wrap point: all new API types compile. No runtime behaviour yet.

### Task 1: T1-RollbackPolicy-HealthPolicy-ResearchMethodology

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/RollbackPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchMethodology.java`
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/RollbackPolicyTest.java`

**Interfaces:**
- Consumes: nothing (leaf types)
- Produces: `RollbackPolicy` record (6 fields: autoRevertThreshold, pauseThreshold, requireReviewForRevert, pauseCategoryOnRegression, regressionWindowMinutes, sustainedFailureCount), `HealthPolicy` record (6 fields: healthThreshold, healthDeltaThreshold, healthWindowMinutes, recoveryWindowMinutes, halfOpenMaxImprovements, weights), `ResearchMethodology` record (6 fields: horizonScanIntervalDays, areaRefreshStalenessThresholdDays, maxSearchResultsPerTier, searchChannels, minTriangulationSources, strategyDwellTimeDays)

- [ ] **Step 1: Write test for RollbackPolicy defaults**

```java
package io.casehub.api.model.stigmergy;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RollbackPolicyTest {

  @Test
  void defaultsAreConservative() {
    var policy = new RollbackPolicy(null, null, null, null, null, null);
    assertEquals(0.9, policy.effectiveAutoRevertThreshold());
    assertEquals(0.5, policy.effectivePauseThreshold());
    assertTrue(policy.effectiveRequireReviewForRevert());
    assertTrue(policy.effectivePauseCategoryOnRegression());
    assertEquals(60, policy.effectiveRegressionWindowMinutes());
    assertEquals(2, policy.effectiveSustainedFailureCount());
  }

  @Test
  void overridesApply() {
    var policy = new RollbackPolicy(0.8, 0.3, false, false, 120, 5);
    assertEquals(0.8, policy.effectiveAutoRevertThreshold());
    assertEquals(0.3, policy.effectivePauseThreshold());
    assertFalse(policy.effectiveRequireReviewForRevert());
    assertFalse(policy.effectivePauseCategoryOnRegression());
    assertEquals(120, policy.effectiveRegressionWindowMinutes());
    assertEquals(5, policy.effectiveSustainedFailureCount());
  }
}
```

- [ ] **Step 2: Run test — verify FAIL** (class not found)

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=RollbackPolicyTest -Dcheckstyle.skip=true -Dspotless.check.skip=true`

- [ ] **Step 3: Create RollbackPolicy record**

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;

public record RollbackPolicy(
    @Nullable Double autoRevertThreshold,
    @Nullable Double pauseThreshold,
    @Nullable Boolean requireReviewForRevert,
    @Nullable Boolean pauseCategoryOnRegression,
    @Nullable Integer regressionWindowMinutes,
    @Nullable Integer sustainedFailureCount) {

  public double effectiveAutoRevertThreshold() {
    return autoRevertThreshold != null ? autoRevertThreshold : 0.9;
  }

  public double effectivePauseThreshold() {
    return pauseThreshold != null ? pauseThreshold : 0.5;
  }

  public boolean effectiveRequireReviewForRevert() {
    return requireReviewForRevert != null ? requireReviewForRevert : true;
  }

  public boolean effectivePauseCategoryOnRegression() {
    return pauseCategoryOnRegression != null ? pauseCategoryOnRegression : true;
  }

  public int effectiveRegressionWindowMinutes() {
    return regressionWindowMinutes != null ? regressionWindowMinutes : 60;
  }

  public int effectiveSustainedFailureCount() {
    return sustainedFailureCount != null ? sustainedFailureCount : 2;
  }
}
```

- [ ] **Step 4: Create HealthPolicy record**

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.util.Map;

public record HealthPolicy(
    @Nullable Double healthThreshold,
    @Nullable Double healthDeltaThreshold,
    @Nullable Integer healthWindowMinutes,
    @Nullable Integer recoveryWindowMinutes,
    @Nullable Integer halfOpenMaxImprovements,
    @Nullable Map<String, Double> weights) {

  public double effectiveHealthThreshold() {
    return healthThreshold != null ? healthThreshold : 0.6;
  }

  public double effectiveHealthDeltaThreshold() {
    return healthDeltaThreshold != null ? healthDeltaThreshold : 0.15;
  }

  public int effectiveHealthWindowMinutes() {
    return healthWindowMinutes != null ? healthWindowMinutes : 1440;
  }

  public int effectiveRecoveryWindowMinutes() {
    return recoveryWindowMinutes != null ? recoveryWindowMinutes : 120;
  }

  public int effectiveHalfOpenMaxImprovements() {
    return halfOpenMaxImprovements != null ? halfOpenMaxImprovements : 2;
  }

  public Map<String, Double> effectiveWeights() {
    if (weights != null) return weights;
    return Map.of(
        "stability", 0.2,
        "performance", 0.15,
        "execution", 0.1,
        "safety", 0.15,
        "integration", 0.1,
        "autonomy", 0.1,
        "cognitive-reasoning", 0.1,
        "coordination", 0.1);
  }
}
```

- [ ] **Step 5: Create ResearchMethodology record**

```java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.util.List;

public record ResearchMethodology(
    @Nullable Integer horizonScanIntervalDays,
    @Nullable Integer areaRefreshStalenessThresholdDays,
    @Nullable Integer maxSearchResultsPerTier,
    @Nullable List<String> searchChannels,
    @Nullable Integer minTriangulationSources,
    @Nullable Integer strategyDwellTimeDays) {

  public int effectiveHorizonScanIntervalDays() {
    return horizonScanIntervalDays != null ? horizonScanIntervalDays : 90;
  }

  public int effectiveAreaRefreshStalenessThresholdDays() {
    return areaRefreshStalenessThresholdDays != null
        ? areaRefreshStalenessThresholdDays : 30;
  }

  public int effectiveMinTriangulationSources() {
    return minTriangulationSources != null ? minTriangulationSources : 3;
  }

  public int effectiveStrategyDwellTimeDays() {
    return strategyDwellTimeDays != null ? strategyDwellTimeDays : 14;
  }
}
```

- [ ] **Step 6: Run test — verify PASS**

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=RollbackPolicyTest -Dcheckstyle.skip=true -Dspotless.check.skip=true`

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/RollbackPolicy.java api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java api/src/main/java/io/casehub/api/model/stigmergy/ResearchMethodology.java api/src/test/java/io/casehub/api/model/stigmergy/RollbackPolicyTest.java
git commit -m "feat(#1115): add RollbackPolicy, HealthPolicy, ResearchMethodology records Refs #1115"
```

### Task 2: T2-ImprovementConfig-Expansion

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementConfig.java`
- Modify: `api/src/test/java/io/casehub/api/model/stigmergy/StigmergyConfigTest.java` (if exists, add backward compat test)
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/ImprovementConfigTest.java`

**Interfaces:**
- Consumes: `RollbackPolicy`, `HealthPolicy`, `ResearchMethodology` (from T1)
- Produces: `ImprovementConfig` with 11 fields (5 existing + 6 new: evolutionEnabled, evolutionTickIntervalMinutes, rollbackPolicy, healthPolicy, researchMethodology, conflictTrivialThreshold), 5-arg backward-compatible constructor

- [ ] **Step 1: Write test for new defaults and backward compatibility**

```java
package io.casehub.api.model.stigmergy;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ImprovementConfigTest {

  @Test
  void backwardsCompatibleConstructor() {
    var config = new ImprovementConfig("improvement", 2, null, null, null);
    assertFalse(config.effectiveEvolutionEnabled());
    assertEquals(60, config.effectiveEvolutionTickIntervalMinutes());
    assertNotNull(config.effectiveRollbackPolicy());
    assertNotNull(config.effectiveHealthPolicy());
    assertEquals(10, config.effectiveConflictTrivialThreshold());
  }

  @Test
  void evolutionIsOptIn() {
    var config = new ImprovementConfig(null, null, null, null, null);
    assertFalse(config.effectiveEvolutionEnabled());
  }

  @Test
  void fullConstructor() {
    var config = new ImprovementConfig(
        "improvement", 2, null, null, null,
        true, 30, new RollbackPolicy(0.8, null, null, null, null, null),
        new HealthPolicy(null, null, null, null, null, null),
        new ResearchMethodology(null, null, null, null, null, null), 5);
    assertTrue(config.effectiveEvolutionEnabled());
    assertEquals(30, config.effectiveEvolutionTickIntervalMinutes());
    assertEquals(0.8, config.effectiveRollbackPolicy().effectiveAutoRevertThreshold());
    assertEquals(5, config.effectiveConflictTrivialThreshold());
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Expand ImprovementConfig record**

Add 6 new fields to the canonical constructor. Add a 5-arg backward-compatible constructor delegating to the canonical with nulls. Add `effective*()` methods for each new field. See spec §10 for exact code.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Compile full project to check backward compatibility**

Run: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`

Fix any compilation errors from existing call sites using the old constructor.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/ImprovementConfig.java api/src/test/java/io/casehub/api/model/stigmergy/ImprovementConfigTest.java
git commit -m "feat(#1115): expand ImprovementConfig with evolution, rollback, health, research fields Refs #1115"
```

### Task 3: T3-CaseHubEventType-CapabilityArea-SPI

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/CapabilityAreaAssessment.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/GapMap.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ResearchDepth.java`
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/CapabilityAreaAssessmentTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `CapabilityArea` SPI (id, name, description, assess), `CapabilityAreaAssessment` record, `CapabilityAreaAssessment.LandscapePosition` enum, `GapMap` record, `ResearchDepth` enum, new `CaseHubEventType` values (CIRCUIT_BREAKER_TRIPPED, CIRCUIT_BREAKER_RECOVERING, CIRCUIT_BREAKER_RESET, CAPABILITY_AREA_CHANGED, REGRESSION_DETECTED, ROLLBACK_STARTED, IMPROVEMENT_CONFLICT_DETECTED)

- [ ] **Step 1: Write test for GapMap ordering**

```java
package io.casehub.api.model.stigmergy;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class CapabilityAreaAssessmentTest {

  @Test
  void gapMapSortsByRoiDescending() {
    var ahead = new CapabilityAreaAssessment("stability", 0.9,
        CapabilityAreaAssessment.LandscapePosition.AHEAD, 0.1, 0.5, 0.2, Instant.now());
    var behind = new CapabilityAreaAssessment("performance", 0.4,
        CapabilityAreaAssessment.LandscapePosition.BEHIND, 0.8, 0.3, 2.67, Instant.now());
    var absent = new CapabilityAreaAssessment("autonomy", 0.2,
        CapabilityAreaAssessment.LandscapePosition.ABSENT, 0.9, 0.5, 1.8, Instant.now());
    var map = new GapMap(List.of(ahead, behind, absent), Instant.now());
    var gaps = map.gapsByRoi();
    assertEquals(2, gaps.size());
    assertEquals("performance", gaps.get(0).areaId());
    assertEquals("autonomy", gaps.get(1).areaId());
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Create CapabilityArea SPI**

```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import java.util.UUID;

public interface CapabilityArea {
  String id();
  String name();
  String description();
  CapabilityAreaAssessment assess(UUID caseId);
}
```

- [ ] **Step 4: Create CapabilityAreaAssessment, GapMap, ResearchDepth**

See spec §6 for `CapabilityAreaAssessment` record with `LandscapePosition` enum. See spec §6 for `GapMap` with `gapsByRoi()`. See spec §7 for `ResearchDepth` enum (HORIZON_SCAN, TECHNOLOGY_SCOUTING, DEEP_DIVE).

- [ ] **Step 5: Add new CaseHubEventType values**

Add at the end of the enum:
```java
CIRCUIT_BREAKER_TRIPPED,
CIRCUIT_BREAKER_RECOVERING,
CIRCUIT_BREAKER_RESET,
CAPABILITY_AREA_CHANGED,
REGRESSION_DETECTED,
ROLLBACK_STARTED,
IMPROVEMENT_CONFLICT_DETECTED
```

- [ ] **Step 6: Run test — verify PASS**

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/improvement/ api/src/main/java/io/casehub/api/model/stigmergy/CapabilityAreaAssessment.java api/src/main/java/io/casehub/api/model/stigmergy/GapMap.java api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java api/src/test/java/io/casehub/api/model/stigmergy/CapabilityAreaAssessmentTest.java
git commit -m "feat(#1115): add CapabilityArea SPI, GapMap, ResearchDepth, new event types Refs #1115"
```

## Batch 2: Research API Types + SPIs

Safe wrap point: all API types and SPI contracts compile. No runtime behaviour.

### Task 4: T4-Research-Records-SPIs

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchScope.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchCandidate.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchAnalysis.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchFinding.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementHypothesis.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/TechnologyBlip.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/HilQueueEntry.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ResearchScoper.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ResearchSearcher.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ResearchAnalyzer.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/HypothesisFormer.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ResearchCorpus.java`
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/ImprovementHypothesisTest.java`

**Interfaces:**
- Consumes: `CapabilityAreaAssessment`, `ResearchDepth` (from T3)
- Produces: all research records and 5 SPIs. See spec §7-§8 for exact signatures.

- [ ] **Step 1: Write test for ImprovementHypothesis radar recommendation**

```java
package io.casehub.api.model.stigmergy;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ImprovementHypothesisTest {

  @Test
  void radarRecommendationValues() {
    assertEquals(3, ImprovementHypothesis.RadarRecommendation.values().length);
    assertNotNull(ImprovementHypothesis.RadarRecommendation.ADOPT);
    assertNotNull(ImprovementHypothesis.RadarRecommendation.TRIAL);
    assertNotNull(ImprovementHypothesis.RadarRecommendation.ASSESS);
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Create all research records**

Create each record exactly as specified in spec §7 and §8. Each record is a simple data carrier with no behaviour. `ResearchScope(question, keywords, channels, area)`, `ResearchCandidate(sourceUrl, title, abstractText, sourceType, metadata)`, `ResearchAnalysis(findings, themes, contradictions, gaps)`, `ResearchFinding(technique, claimedBenefits, limitations, applicability, evidenceQuality, capabilityArea, sourceUrl)`, `ImprovementHypothesis(technique, targetComponent, expectedImprovement, evidence, risk, capabilityArea, radarRecommendation)` with `RadarRecommendation` enum, `TechnologyBlip(id, name, description, ring, capabilityArea, lastAssessed, rationale)` with `RadarRing` enum, `HilQueueEntry(sourceUrl, citation, reason, capabilityArea, priority, blockingHypotheses, queuedAt)`.

- [ ] **Step 4: Create research SPIs**

Create `ResearchScoper`, `ResearchSearcher`, `ResearchAnalyzer`, `HypothesisFormer`, `ResearchCorpus` interfaces exactly as specified in spec §7-§8.

- [ ] **Step 5: Run test — verify PASS**

- [ ] **Step 6: Full project compile check**

Run: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/Research*.java api/src/main/java/io/casehub/api/model/stigmergy/ImprovementHypothesis.java api/src/main/java/io/casehub/api/model/stigmergy/TechnologyBlip.java api/src/main/java/io/casehub/api/model/stigmergy/HilQueueEntry.java api/src/main/java/io/casehub/api/spi/improvement/Research*.java api/src/main/java/io/casehub/api/spi/improvement/HypothesisFormer.java api/src/test/java/io/casehub/api/model/stigmergy/ImprovementHypothesisTest.java
git commit -m "feat(#1115): add research records, Technology Radar model, and 5 research SPIs Refs #1115"
```

## Batch 3: Category Tracking + Budget Enhancement

Safe wrap point: outcome feedback infrastructure works. Budget enforcer enhanced with conflict support.

### Task 5: T5-ImprovementCategoryTracker

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCategoryTracker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCategoryTrackerTest.java`

**Interfaces:**
- Consumes: `ImprovementOutcome.OutcomeStatus`
- Produces: `ImprovementCategoryTracker` — `recordOutcome(UUID caseId, String category, OutcomeStatus status)`, `isSuppressed(UUID caseId, String category): boolean`, `pauseCategory(UUID caseId, String category, Duration duration)`, `unpauseCategory(UUID caseId, String category)`, inner record `CategoryState(successCount, failureCount, rejectionCount, lastOutcome, paused, pausedUntil)`

- [ ] **Step 1: Write tests**

```java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.ImprovementOutcome;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class ImprovementCategoryTrackerTest {

  private ImprovementCategoryTracker tracker;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    tracker = new ImprovementCategoryTracker();
    caseId = UUID.randomUUID();
  }

  @Test
  void notSuppressedByDefault() {
    assertFalse(tracker.isSuppressed(caseId, "dependency-update"));
  }

  @Test
  void threeFailuresSuppresses() {
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    assertFalse(tracker.isSuppressed(caseId, "lint-fix"));
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    assertTrue(tracker.isSuppressed(caseId, "lint-fix"));
  }

  @Test
  void threeRejectionsSuppresses() {
    tracker.recordOutcome(caseId, "recipe", ImprovementOutcome.OutcomeStatus.REJECTED);
    tracker.recordOutcome(caseId, "recipe", ImprovementOutcome.OutcomeStatus.REJECTED);
    tracker.recordOutcome(caseId, "recipe", ImprovementOutcome.OutcomeStatus.REJECTED);
    assertTrue(tracker.isSuppressed(caseId, "recipe"));
  }

  @Test
  void pauseAndUnpause() {
    tracker.pauseCategory(caseId, "ci-triage", Duration.ofMinutes(60));
    assertTrue(tracker.isSuppressed(caseId, "ci-triage"));
    tracker.unpauseCategory(caseId, "ci-triage");
    assertFalse(tracker.isSuppressed(caseId, "ci-triage"));
  }

  @Test
  void successResetsFailureCount() {
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.MERGED);
    tracker.recordOutcome(caseId, "lint-fix", ImprovementOutcome.OutcomeStatus.FAILED);
    assertFalse(tracker.isSuppressed(caseId, "lint-fix"));
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement ImprovementCategoryTracker**

`@ApplicationScoped`, implements `Resettable`. Uses `ConcurrentHashMap<UUID, ConcurrentHashMap<String, CategoryState>>`. See spec §2 for full code sketch. Key suppression rules: 3+ consecutive FAILED → suppress. 3 REJECTED → suppress. MERGED resets failure count. Pause checks `pausedUntil` against `Instant.now()`.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCategoryTracker.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCategoryTrackerTest.java
git commit -m "feat(#1115): add ImprovementCategoryTracker — outcome-driven category suppression Refs #1115"
```

### Task 6: T6-RollbackHistory-BudgetEnforcer-Enhancement

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RollbackHistory.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/RollbackHistoryTest.java`

**Interfaces:**
- Consumes: `ImprovementRequest` (existing)
- Produces: `RollbackHistory` — `record(UUID caseId, UUID improvementCaseId, String category, String target)`, `wasRecentlyRolledBack(UUID caseId, String category, String target, Duration window): boolean`. Enhanced `ImprovementBudgetEnforcer` — `recordStart(UUID improvementCaseId, ImprovementRequest request)`, `activeImprovementRequests(UUID caseId): Map<UUID, ImprovementRequest>`, updated `STRUCTURAL_DENIED_PATTERNS` with all #1115 safety components.

- [ ] **Step 1: Write tests for RollbackHistory**

```java
package io.casehub.engine.internal.improvement;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class RollbackHistoryTest {

  private RollbackHistory history;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    history = new RollbackHistory();
    caseId = UUID.randomUUID();
  }

  @Test
  void emptyHistoryReturnsFalse() {
    assertFalse(history.wasRecentlyRolledBack(
        caseId, "lint-fix", "checkstyle", Duration.ofMinutes(60)));
  }

  @Test
  void recentRollbackDetected() {
    history.record(caseId, UUID.randomUUID(), "lint-fix", "checkstyle");
    assertTrue(history.wasRecentlyRolledBack(
        caseId, "lint-fix", "checkstyle", Duration.ofMinutes(60)));
  }

  @Test
  void differentCategoryNotDetected() {
    history.record(caseId, UUID.randomUUID(), "lint-fix", "checkstyle");
    assertFalse(history.wasRecentlyRolledBack(
        caseId, "dependency-update", "checkstyle", Duration.ofMinutes(60)));
  }

  @Test
  void differentTargetNotDetected() {
    history.record(caseId, UUID.randomUUID(), "lint-fix", "checkstyle");
    assertFalse(history.wasRecentlyRolledBack(
        caseId, "lint-fix", "spotbugs", Duration.ofMinutes(60)));
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement RollbackHistory**

`@ApplicationScoped`, implements `Resettable`. Inner record `RollbackRecord(UUID improvementCaseId, String category, String target, Instant rolledBackAt)`. `ConcurrentHashMap<UUID, List<RollbackRecord>>` keyed by caseId. `wasRecentlyRolledBack` checks category + target match within window.

- [ ] **Step 4: Enhance ImprovementBudgetEnforcer**

Three changes:
1. Change `activeImprovements` from `ConcurrentHashMap<UUID, Instant>` to `ConcurrentHashMap<UUID, ImprovementRequest>`
2. Update `recordStart` signature to accept `ImprovementRequest`: `recordStart(UUID improvementCaseId, ImprovementRequest request)`
3. Add `activeImprovementRequests(UUID caseId)` returning `Map.copyOf(activeImprovements)`
4. Update `STRUCTURAL_DENIED_PATTERNS` to add: `"EvolutionTicker"`, `"ImprovementCircuitBreaker"`, `"RegressionDetector"`, `"ConfidenceScorer"`, `"HealthScoreTracker"`, `"HealthPolicy"`, `"RollbackPolicy"`, `"ConflictDetector"`, `"ImprovementCategoryTracker"`, `"RollbackHistory"`, `"self-improvement-rollback"`
5. Fix `recordCompletion` to work with the new value type (it removes by key, so the change is minimal)

- [ ] **Step 5: Run tests — verify PASS (including existing tests)**

Run: `/opt/homebrew/bin/mvn test -pl runtime-core -Dtest="RollbackHistoryTest,ImprovementBudgetEnforcerTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true`

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/RollbackHistory.java runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/RollbackHistoryTest.java
git commit -m "feat(#1115): add RollbackHistory, enhance BudgetEnforcer with request tracking + deny list Refs #1115"
```

## Batch 4: Conflict Detection + Health Infrastructure

Safe wrap point: conflict detection and health scoring work independently.

### Task 7: T7-ConflictDetector

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConflictDetector.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConflictDetectorTest.java`

**Interfaces:**
- Consumes: `ImprovementRequest.targetPaths()`, `ImprovementRequest.estimatedSize()`
- Produces: `ConflictDetector` — `check(ImprovementRequest request, Map<UUID, ImprovementRequest> active, int trivialThreshold): ConflictCheck`, sealed interface `ConflictCheck` with `Clear` and `Conflicting(UUID blockingId, String conflictPath)` permits

- [ ] **Step 1: Write tests**

```java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.ImprovementRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ConflictDetectorTest {

  private ConflictDetector detector;
  private UUID activeId;

  @BeforeEach
  void setUp() {
    detector = new ConflictDetector();
    activeId = UUID.randomUUID();
  }

  @Test
  void noOverlapIsClear() {
    var request = request("module-a/src/Foo.java", 50);
    var active = Map.of(activeId, request("module-b/src/Bar.java", 50));
    assertInstanceOf(ConflictDetector.ConflictCheck.Clear.class,
        detector.check(request, active, 10));
  }

  @Test
  void fileOverlapIsConflicting() {
    var request = request("module-a/src/Foo.java", 50);
    var active = Map.of(activeId, request("module-a/src/Foo.java", 50));
    assertInstanceOf(ConflictDetector.ConflictCheck.Conflicting.class,
        detector.check(request, active, 10));
  }

  @Test
  void directoryOverlapIsConflicting() {
    var request = request("module-a/src/Foo.java", 50);
    var active = Map.of(activeId, request("module-a/src/Bar.java", 50));
    assertInstanceOf(ConflictDetector.ConflictCheck.Conflicting.class,
        detector.check(request, active, 10));
  }

  @Test
  void trivialExemptFromDirectoryOverlap() {
    var request = request("module-a/src/Foo.java", 5);
    var active = Map.of(activeId, request("module-a/src/Bar.java", 50));
    assertInstanceOf(ConflictDetector.ConflictCheck.Clear.class,
        detector.check(request, active, 10));
  }

  @Test
  void trivialNotExemptFromFileOverlap() {
    var request = request("module-a/src/Foo.java", 5);
    var active = Map.of(activeId, request("module-a/src/Foo.java", 50));
    assertInstanceOf(ConflictDetector.ConflictCheck.Conflicting.class,
        detector.check(request, active, 10));
  }

  private ImprovementRequest request(String path, int size) {
    return new ImprovementRequest("operational", "lint-fix", "target",
        "casehubio/engine", List.of(path), size, Map.of());
  }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement ConflictDetector**

`@ApplicationScoped`, implements `Resettable`. See spec §5 for exact code. Sealed interface `ConflictCheck` with `Clear` and `Conflicting` records. `check()` iterates active improvements, compares paths at file level (exact match) and directory level (same parent, unless trivial exemption applies). Trivial = `estimatedSize <= trivialThreshold && targetPaths.size() == 1`.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConflictDetector.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConflictDetectorTest.java
git commit -m "feat(#1115): add ConflictDetector — path-based conflict avoidance for concurrent improvements Refs #1115"
```

### Task 8: T8-CapabilityAreaRegistry-HealthScoreTracker

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CapabilityAreaRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/CapabilityAreaRegistryTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/HealthScoreTrackerTest.java`

**Interfaces:**
- Consumes: `CapabilityArea` SPI (from T3), `HealthPolicy` (from T1)
- Produces: `CapabilityAreaRegistry` — `register(CapabilityArea)`, `deprecate(String areaId)`, `active(): List<CapabilityArea>`, `get(String areaId): Optional<CapabilityArea>`. `HealthScoreTracker` — `computeScore(UUID caseId, HealthPolicy policy): double`, `refresh(UUID caseId, HealthPolicy policy)`, `latestSnapshot(UUID caseId): HealthSnapshot`, `delta(UUID caseId, int windowMinutes): double`, inner record `HealthSnapshot(score, timestamp, componentScores)`.

- [ ] **Step 1: Write tests for CapabilityAreaRegistry**

```java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.api.spi.improvement.CapabilityArea;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class CapabilityAreaRegistryTest {

  private CapabilityAreaRegistry registry;

  @BeforeEach
  void setUp() {
    registry = new CapabilityAreaRegistry();
  }

  @Test
  void registerAndRetrieve() {
    registry.register(stubArea("stability"));
    assertEquals(1, registry.active().size());
    assertTrue(registry.get("stability").isPresent());
  }

  @Test
  void deprecateRemovesFromActive() {
    registry.register(stubArea("stability"));
    registry.deprecate("stability");
    assertTrue(registry.active().isEmpty());
  }

  private CapabilityArea stubArea(String id) {
    return new CapabilityArea() {
      public String id() { return id; }
      public String name() { return id; }
      public String description() { return "test"; }
      public CapabilityAreaAssessment assess(UUID caseId) {
        return new CapabilityAreaAssessment(id, 0.8,
            CapabilityAreaAssessment.LandscapePosition.AT_PARITY,
            0.5, 0.3, 1.67, Instant.now());
      }
    };
  }
}
```

- [ ] **Step 2: Write tests for HealthScoreTracker**

```java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.CapabilityAreaAssessment;
import io.casehub.api.model.stigmergy.HealthPolicy;
import io.casehub.api.spi.improvement.CapabilityArea;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class HealthScoreTrackerTest {

  private HealthScoreTracker tracker;
  private CapabilityAreaRegistry registry;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    registry = new CapabilityAreaRegistry();
    tracker = new HealthScoreTracker(registry);
    caseId = UUID.randomUUID();
  }

  @Test
  void scoreNormalisesCustomWeights() {
    registry.register(area("stability", 0.8));
    registry.register(area("performance", 0.4));
    var policy = new HealthPolicy(null, null, null, null, null,
        Map.of("stability", 3.0, "performance", 1.0));
    double score = tracker.computeScore(caseId, policy);
    double expected = (3.0 * 0.8 + 1.0 * 0.4) / (3.0 + 1.0);
    assertEquals(expected, score, 0.001);
  }

  @Test
  void snapshotIsNullBeforeRefresh() {
    assertNull(tracker.latestSnapshot(caseId));
  }

  @Test
  void refreshCreatesSnapshot() {
    registry.register(area("stability", 0.8));
    var policy = new HealthPolicy(null, null, null, null, null, null);
    tracker.refresh(caseId, policy);
    assertNotNull(tracker.latestSnapshot(caseId));
  }

  private CapabilityArea area(String id, double health) {
    return new CapabilityArea() {
      public String id() { return id; }
      public String name() { return id; }
      public String description() { return "test"; }
      public CapabilityAreaAssessment assess(UUID caseId) {
        return new CapabilityAreaAssessment(id, health,
            CapabilityAreaAssessment.LandscapePosition.AT_PARITY,
            0.5, 0.3, 1.67, Instant.now());
      }
    };
  }
}
```

- [ ] **Step 3: Run tests — verify FAIL**

- [ ] **Step 4: Implement CapabilityAreaRegistry**

`@ApplicationScoped`, implements `Resettable`. `ConcurrentHashMap<String, CapabilityArea>`. See spec §6.

- [ ] **Step 5: Implement HealthScoreTracker**

`@ApplicationScoped`, implements `Resettable`. See spec §4 for full code. Constructor takes `CapabilityAreaRegistry`. `computeScore` aggregates `area.assess()` with weight normalisation. `refresh` stores `HealthSnapshot` in a `ConcurrentHashMap<UUID, Deque<HealthSnapshot>>`. `delta` computes current minus historical score.

- [ ] **Step 6: Run tests — verify PASS**

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/CapabilityAreaRegistry.java runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/CapabilityAreaRegistryTest.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/HealthScoreTrackerTest.java
git commit -m "feat(#1115): add CapabilityAreaRegistry and HealthScoreTracker Refs #1115"
```

## Batch 5: Circuit Breaker + Regression Detection

Safe wrap point: safety infrastructure works independently.

### Task 9: T9-ImprovementCircuitBreaker

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreaker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreakerTest.java`

**Interfaces:**
- Consumes: `HealthScoreTracker` (from T8), `HealthPolicy` (from T1)
- Produces: `ImprovementCircuitBreaker` — `state(UUID caseId): CircuitBreakerState`, `evaluate(UUID caseId, HealthScoreTracker tracker, HealthPolicy policy)`, `recordImprovementInHalfOpen(UUID caseId)`, `manualReset(UUID caseId)`, `restoreFromEventLog(UUID caseId, EventLogRepository repo)`, inner enum `CircuitBreakerState(CLOSED, OPEN, HALF_OPEN)`

- [ ] **Step 1: Write tests for all state transitions**

Test: CLOSED→OPEN (health below threshold), CLOSED→OPEN (delta below threshold), OPEN stays OPEN (health still bad), OPEN→HALF_OPEN (sustained recovery), HALF_OPEN→CLOSED (completed improvements), HALF_OPEN→OPEN (health drops again), manual reset.

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement ImprovementCircuitBreaker**

`@ApplicationScoped`, implements `Resettable`. See spec §4 for complete code. State stored in `ConcurrentHashMap<UUID, CircuitBreakerState>`. `evaluate()` implements the state machine. `restoreFromEventLog()` queries for the most recent `CIRCUIT_BREAKER_*` event type.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreaker.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreakerTest.java
git commit -m "feat(#1115): add ImprovementCircuitBreaker — CLOSED/OPEN/HALF_OPEN state machine Refs #1115"
```

### Task 10: T10-ConfidenceScorer-RegressionDetector

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConfidenceScorer.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConfidenceScorerTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/RegressionDetectorTest.java`

**Interfaces:**
- Consumes: `HealthScoreTracker.HealthSnapshot` (from T8), `ImprovementCategoryTracker` (from T5), `RollbackHistory` (from T6), `ImprovementOutcome` (existing), `RollbackPolicy` (from T1), `EventLogRepository` (existing)
- Produces: `ConfidenceScorer` — `score(UUID caseId, UUID improvementCaseId, HealthSnapshot before, HealthSnapshot after): double`. `RegressionDetector` — `onOutcome(UUID caseId, ImprovementOutcome outcome)`, `checkActiveMonitors(UUID caseId, HealthScoreTracker tracker, RollbackPolicy policy)`, inner record `MonitoredImprovement(improvementCaseId, category, target, baseline, mergedAt, checksRemaining)`

- [ ] **Step 1: Write ConfidenceScorer tests**

Test: base case (no signals = 0.0), CI build failure adds 0.5, multiple degraded areas adds 0.1, regression in unrelated area subtracts 0.2, clamped to [0, 1].

- [ ] **Step 2: Write RegressionDetector tests**

Test: MERGED outcome starts monitoring, non-MERGED is ignored, high confidence triggers rollback + pause, medium confidence triggers signal + pause, low confidence triggers signal only, window expiry removes monitor, anti-flaky (sustained failure count).

- [ ] **Step 3: Run tests — verify FAIL**

- [ ] **Step 4: Implement ConfidenceScorer**

`@ApplicationScoped`. See spec §3 for full code. Each scoring method checks specific conditions and adds/subtracts confidence. Result clamped to [0, 1].

- [ ] **Step 5: Implement RegressionDetector**

`@ApplicationScoped`, implements `Resettable`. See spec §3 for full code. `onOutcome` registers `MonitoredImprovement` for MERGED outcomes. `checkActiveMonitors` iterates monitors, checks window, compares baseline to current health, calls `onMetricsDegraded` which delegates to `ConfidenceScorer` and executes the tiered response. Constructor takes `ConfidenceScorer`, `ImprovementCategoryTracker`, `RollbackHistory`, `SignalRegistry`, `HealthScoreTracker`.

- [ ] **Step 6: Run tests — verify PASS**

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConfidenceScorer.java runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConfidenceScorerTest.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/RegressionDetectorTest.java
git commit -m "feat(#1115): add ConfidenceScorer and RegressionDetector — confidence-tiered rollback Refs #1115"
```

## Batch 6: Evolution Pipeline Wiring

Safe wrap point: the continuous loop is wired end-to-end.

### Task 11: T11-EvolutionTicker

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionTickerTest.java`

**Interfaces:**
- Consumes: `ImprovementGoalFormationStrategy`, `ImprovementCircuitBreaker`, `HealthScoreTracker`, `RegressionDetector`, `GoalFormationService`
- Produces: `EvolutionTicker` — `tick(UUID caseId, String tenancyId, ImprovementConfig config)`

- [ ] **Step 1: Write tests for gate pipeline**

Test: evolutionEnabled=false → returns immediately, circuit breaker OPEN → blocks, all gates pass → calls GoalFormationService.propose(), health refresh called before circuit breaker evaluate, regression monitoring called between health refresh and circuit breaker.

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement EvolutionTicker**

`@ApplicationScoped`, implements `Resettable`. See spec §1 for the complete gate pipeline code. Constructor takes the 5 dependencies.

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionTickerTest.java
git commit -m "feat(#1115): add EvolutionTicker — unified gate pipeline for continuous evolution Refs #1115"
```

### Task 12: T12-GoalFormation-Enhancement-OutcomeCapture-Wiring

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategy.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementOutcomeEventCapture.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategyTest.java` (add new gate tests)

**Interfaces:**
- Consumes: `ImprovementCategoryTracker` (T5), `RollbackHistory` (T6), `ConflictDetector` (T7), `RegressionDetector` (T10), `EvolutionTicker` (T11)
- Produces: enhanced `proposeImprovements()` with category suppression, anti-oscillation, and conflict detection gates. Enhanced `onImprovementComplete()` with category tracker and regression detector layers. `CaseContextChangedEventHandler` routes through `EvolutionTicker.tick()` instead of calling `proposeImprovements()` directly.

- [ ] **Step 1: Write tests for new gates in proposeImprovements()**

Test: suppressed category is skipped, recently rolled-back target is skipped, conflicting paths are skipped. Add to existing `ImprovementGoalFormationStrategyTest` or create a new focused test.

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Enhance ImprovementGoalFormationStrategy**

Add 3 new constructor dependencies: `ImprovementCategoryTracker`, `RollbackHistory`, `ConflictDetector`. Add category suppression, anti-oscillation, and conflict detection gates inside `proposeImprovements()` loop — see spec §2 for the exact code.

- [ ] **Step 4: Enhance ImprovementOutcomeEventCapture**

Add 2 new constructor dependencies: `ImprovementCategoryTracker`, `RegressionDetector`. Add Layer 5 (`categoryTracker.recordOutcome()`) and Layer 6 (`regressionDetector.onOutcome()`) to `onImprovementComplete()` — see spec §2.

- [ ] **Step 5: Modify CaseContextChangedEventHandler**

Replace direct `proposeImprovements()` call with `evolutionTickerInstance.get().tick()`. Add `Instance<EvolutionTicker>` as a constructor parameter. See spec §1 for the before/after code.

- [ ] **Step 6: Update RuntimeBeans**

Add `EvolutionTicker` wiring to the `CaseContextChangedEventHandler` producer. Wire all new dependencies for `ImprovementGoalFormationStrategy` and `ImprovementOutcomeEventCapture`.

- [ ] **Step 7: Compile and run existing tests**

Run: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
Then: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest="ImprovementGoalFormationStrategyTest,ImprovementOutcomeEventCaptureTest,SelfImprovementIntegrationTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true`

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategy.java runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementOutcomeEventCapture.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java
git commit -m "feat(#1115): wire evolution pipeline — EvolutionTicker routes through all safety gates Refs #1115"
```

## Batch 7: Research Pipeline + Rollback Case

Safe wrap point: research pipeline operational (with skeleton defaults). Rollback case template present.

### Task 13: T13-Research-Pipeline-Corpus

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryResearchCorpus.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/research/DefaultResearchScoper.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/research/DefaultResearchSearcher.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/research/DefaultResearchAnalyzer.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/research/DefaultHypothesisFormer.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryResearchCorpusTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestratorTest.java`

**Interfaces:**
- Consumes: all research SPIs (from T4), `CapabilityAreaAssessment` (from T3)
- Produces: `InMemoryResearchCorpus` implementing `ResearchCorpus` SPI, 4 default SPI implementations (skeleton — return empty/passthrough), `ResearchPipelineOrchestrator` — `execute(ResearchDepth depth, CapabilityAreaAssessment area, Map<String, String> driveContext): List<ImprovementHypothesis>`

- [ ] **Step 1: Write tests for InMemoryResearchCorpus**

Test: store and search, HIL queue add/resolve, empty search returns empty.

- [ ] **Step 2: Write test for pipeline orchestration**

Test: orchestrator calls all 4 SPIs in order, stores results in corpus.

- [ ] **Step 3: Run tests — verify FAIL**

- [ ] **Step 4: Implement InMemoryResearchCorpus**

`@ApplicationScoped`, implements `ResearchCorpus`, `Resettable`. See spec §8 for structure.

- [ ] **Step 5: Implement 4 default SPI implementations**

`DefaultResearchScoper` — constructs scope from area keywords. `DefaultResearchSearcher` — returns empty list. `DefaultResearchAnalyzer` — passes through candidates as findings. `DefaultHypothesisFormer` — converts findings to hypotheses using area metrics. All `@ApplicationScoped`.

- [ ] **Step 6: Implement ResearchPipelineOrchestrator**

`@ApplicationScoped`. See spec §7 for the `execute()` method. Calls scoper → searcher → analyzer → hypothesisFormer, stores in corpus.

- [ ] **Step 7: Run tests — verify PASS**

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/InMemoryResearchCorpus.java runtime-core/src/main/java/io/casehub/engine/internal/improvement/research/ runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/InMemoryResearchCorpusTest.java runtime-core/src/test/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestratorTest.java
git commit -m "feat(#1115): add research pipeline — 4 SPIs with defaults, InMemoryResearchCorpus Refs #1115"
```

### Task 14: T14-RevertWorker-RollbackTemplate

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/worker/ImprovementRevertWorker.java`
- Create: `runtime/src/main/resources/case-templates/self-improvement-rollback.yaml`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/worker/ImprovementRevertWorkerTest.java`

**Interfaces:**
- Consumes: `WorkerScope`, `WorkerRuntime`
- Produces: `ImprovementRevertWorker` with `improvement-revert` capability. Rollback case template YAML.

- [ ] **Step 1: Write test for revert worker**

Test: worker returns WorkerResult with revert commit data, worker handles conflict by returning FAILED.

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Implement ImprovementRevertWorker**

`@ApplicationScoped`. Skeleton implementation (same pattern as other 5 workers). Creates git revert commit via REST/GraphQL API surface. On conflict, returns `WorkerResult` with FAILED status and `revert-conflict` reason.

- [ ] **Step 4: Create rollback case template YAML**

```yaml
id: self-improvement-rollback
name: Improvement Rollback
description: Revert a regressed improvement

bindings:
  - name: confirm-regression
    trigger:
      type: on-create
    capability: improvement-introspect

  - name: revert
    trigger:
      type: on-complete
      source: confirm-regression
    when: "context.layer('WORKING').get('regressionConfirmed') == 'true'"
    capability: improvement-revert

  - name: submit-pr
    trigger:
      type: on-complete
      source: revert
    capability: improvement-submit-pr

  - name: fast-track-review
    trigger:
      type: on-signal
      signal: "improvement:pr-submitted"
    capability: code-review

  - name: integrate
    trigger:
      type: on-complete
      source: fast-track-review
    when: "context.layer('WORKING').get('reviewOutcome') == 'approved'"
    capability: improvement-integrate

  - name: record-outcome
    trigger:
      type: on-complete
      source: integrate
    capability: improvement-outcome
```

- [ ] **Step 5: Run test — verify PASS**

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/worker/ImprovementRevertWorker.java runtime/src/main/resources/case-templates/self-improvement-rollback.yaml runtime-core/src/test/java/io/casehub/engine/internal/improvement/worker/ImprovementRevertWorkerTest.java
git commit -m "feat(#1115): add ImprovementRevertWorker and rollback case template Refs #1115"
```

## Batch 8: Integration Test

Safe wrap point: full continuous evolution loop verified.

### Task 15: T15-ContinuousEvolutionIntegrationTest

**Files:**
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ContinuousEvolutionIntegrationTest.java`

**Interfaces:**
- Consumes: all components from T1–T14

- [ ] **Step 1: Write integration test covering the full cycle**

Test the loop closes: deposit detection signals → `EvolutionTicker.tick()` → goal formation → outcome recording → category tracker updated → next tick evaluates with updated state.

Test critical scenarios:
1. **Evolution is opt-in:** default config → ticker does nothing
2. **Circuit breaker blocks:** low health → OPEN → tick returns without proposals
3. **Category suppression:** 3 failures → category suppressed → no proposals for that category
4. **Anti-oscillation:** rollback recorded → same improvement re-proposed → suppressed
5. **Conflict avoidance:** two improvements with overlapping paths → second skipped
6. **Outcome feedback closes loop:** MERGED outcome → category tracker records success → priority adjusted

Follow the test pattern from `SelfImprovementIntegrationTest` — direct instantiation, recording repos, manual method invocation (no `@ObservesAsync`).

- [ ] **Step 2: Run test — verify PASS**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dtest=ContinuousEvolutionIntegrationTest -Dcheckstyle.skip=true -Dspotless.check.skip=true`

- [ ] **Step 3: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime-core -Dcheckstyle.skip=true -Dspotless.check.skip=true`

Verify no regressions in existing tests.

- [ ] **Step 4: Commit**

```bash
git add runtime-core/src/test/java/io/casehub/engine/internal/improvement/ContinuousEvolutionIntegrationTest.java
git commit -m "test(#1115): add continuous evolution integration test — full lifecycle verification Refs #1115"
```

---

## References

- [2026-09-20-continuous-evolution-loop-design.md] — design spec this plan implements
- [2026-09-20-continuous-improvement-methodology.md] — PRISMA, Technology Radar, Wardley Mapping methodology
- [2026-09-20-autonomous-self-improvement-engine-foundation.md] — #1114 engine foundation (§9 extension points)
- [2026-09-20-cognitive-self-improvement-vision.md] — parent vision (§Epic 5)
- [decisions.md D106–D115] — all design decisions
- [ImprovementGoalFormationStrategy.java:56] — existing proposeImprovements() to enhance
- [ImprovementBudgetEnforcer.java:32] — existing budget enforcer to enhance
- [ImprovementOutcomeEventCapture.java:23] — existing outcome capture to enhance
- [CaseContextChangedEventHandler.java:99] — convergence detection path to reroute
- [RuntimeBeans.java:898] — CDI wiring for handler constructor
- [GoalFormationService.java:18] — SPI for goal lifecycle
- [CaseHubEventType.java:18] — existing event type enum
- [SelfImprovementIntegrationTest.java:37] — test pattern to follow
- [GitHub #1115] — issue
- [GitHub #1104] — parent epic
