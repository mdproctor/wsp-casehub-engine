# CBR Ensemble Consensus Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1051 — CbrRetrievalService: integrate PlanEnsembleAnalyzer for consensus synthesis
**Issue group:** #1051

**Goal:** Invoke `PlanEnsembleAnalyzer` inside `CbrRetrievalService` after retrieval and adaptation, map the result to engine-owned types, and surface consensus classification in the CaseContext working layer.

**Architecture:** New engine-owned types (`AgreementLevel`, `StepConsensusEntry`, `ConsensusScope`, `EnsembleConsensus`, `CbrRetrievalResult`) in `api/spi/routing/`. `CbrRetrievalService.retrieve()` returns `CbrRetrievalResult` wrapping experiences + nullable ensemble. Plan-type results (2+, homogeneous caseType) get step-level consensus via `PlanEnsembleAnalyzer`; non-plan and heterogeneous types get outcome-only consensus. `CaseStartedEventHandler` writes `cbrEnsemble` to the working layer.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex-memory-api (PlanEnsembleAnalyzer SPI)

## Global Constraints

- `casehub-neocortex-memory-api` is a compile dependency — types are available but must not leak into the engine API layer. All neocortex types map to engine-owned equivalents.
- `PlanEnsembleAnalyzer.analyze()` bounded by `casehub.engine.cbr.ensemble-timeout-ms` (default 5000ms).
- `retrieveForSelection()` return type unchanged — no ensemble support in v1.
- Ensemble is null when fewer than 2 results or analyzer reports `inputCount < 2`.

---

## Batch 1: Engine-Owned Types + ExperienceAnalyser consolidation

### Task 1: Create engine-owned ensemble types

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/routing/AgreementLevel.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/ConsensusScope.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/StepConsensusEntry.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/EnsembleConsensus.java`
- Create: `api/src/main/java/io/casehub/api/spi/routing/CbrRetrievalResult.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/EnsembleConsensusTest.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/StepConsensusEntryTest.java`
- Test: `api/src/test/java/io/casehub/api/spi/routing/CbrRetrievalResultTest.java`

**Interfaces:**
- Produces: `AgreementLevel` enum (UNANIMOUS, CONSENSUS, CONTESTED, MINORITY, UNIQUE)
- Produces: `ConsensusScope` enum (STEP_LEVEL, OUTCOME_ONLY)
- Produces: `StepConsensusEntry(String bindingName, @Nullable String capabilityName, int occurrenceCount, int totalPlans, Map<String, Integer> workerDistribution, Map<String, Integer> outcomeDistribution, Map<Integer, Integer> priorityDistribution, List<String> contributingCaseIds, AgreementLevel agreement)`
- Produces: `EnsembleConsensus(ConsensusScope scope, List<StepConsensusEntry> stepAnalysis, double ensembleConfidence, int inputCount, List<String> sourceCaseIds)`
- Produces: `CbrRetrievalResult(List<RetrievedExperience> experiences, @Nullable EnsembleConsensus ensemble)` with `static CbrRetrievalResult empty()`

- [ ] **Step 1: Write validation tests for StepConsensusEntry**

```java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class StepConsensusEntryTest {

    @Test
    void valid_construction() {
        var entry = new StepConsensusEntry(
            "reduce-exposure", "risk-mitigation", 3, 5,
            Map.of("analyst-1", 2, "analyst-2", 1),
            Map.of("SUCCESS", 3),
            Map.of(1, 3),
            List.of("case-a", "case-b", "case-d"),
            AgreementLevel.CONSENSUS);
        assertEquals("reduce-exposure", entry.bindingName());
        assertEquals(3, entry.occurrenceCount());
        assertEquals(5, entry.totalPlans());
        assertEquals(AgreementLevel.CONSENSUS, entry.agreement());
        assertEquals(List.of("case-a", "case-b", "case-d"), entry.contributingCaseIds());
    }

    @Test
    void null_bindingName_rejected() {
        assertThrows(NullPointerException.class, () ->
            new StepConsensusEntry(null, null, 1, 1, Map.of(), Map.of(), Map.of(), List.of(), AgreementLevel.UNANIMOUS));
    }

    @Test
    void occurrenceCount_below_one_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new StepConsensusEntry("b", null, 0, 1, Map.of(), Map.of(), Map.of(), List.of(), AgreementLevel.UNANIMOUS));
    }

    @Test
    void totalPlans_below_one_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new StepConsensusEntry("b", null, 1, 0, Map.of(), Map.of(), Map.of(), List.of(), AgreementLevel.UNANIMOUS));
    }

    @Test
    void maps_are_defensively_copied() {
        var workers = new java.util.HashMap<>(Map.of("a", 1));
        var entry = new StepConsensusEntry("b", null, 1, 1, workers, Map.of(), Map.of(), List.of(), AgreementLevel.UNANIMOUS);
        workers.put("b", 2);
        assertEquals(1, entry.workerDistribution().size());
    }

    @Test
    void nullable_capabilityName_accepted() {
        var entry = new StepConsensusEntry("b", null, 1, 1, Map.of(), Map.of(), Map.of(), List.of(), AgreementLevel.UNIQUE);
        assertNull(entry.capabilityName());
    }
}
```

- [ ] **Step 2: Write validation tests for EnsembleConsensus**

```java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import org.junit.jupiter.api.Test;

class EnsembleConsensusTest {

    @Test
    void valid_step_level_construction() {
        var consensus = new EnsembleConsensus(
            ConsensusScope.STEP_LEVEL, List.of(), 0.72, 5, List.of("a", "b"));
        assertEquals(ConsensusScope.STEP_LEVEL, consensus.scope());
        assertEquals(0.72, consensus.ensembleConfidence(), 0.001);
        assertEquals(5, consensus.inputCount());
    }

    @Test
    void valid_outcome_only_construction() {
        var consensus = new EnsembleConsensus(
            ConsensusScope.OUTCOME_ONLY, List.of(), 0.8, 3, List.of());
        assertEquals(ConsensusScope.OUTCOME_ONLY, consensus.scope());
    }

    @Test
    void confidence_below_zero_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new EnsembleConsensus(ConsensusScope.STEP_LEVEL, List.of(), -0.1, 1, List.of()));
    }

    @Test
    void confidence_above_one_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new EnsembleConsensus(ConsensusScope.STEP_LEVEL, List.of(), 1.1, 1, List.of()));
    }

    @Test
    void inputCount_negative_rejected() {
        assertThrows(IllegalArgumentException.class, () ->
            new EnsembleConsensus(ConsensusScope.STEP_LEVEL, List.of(), 0.5, -1, List.of()));
    }

    @Test
    void lists_are_defensively_copied() {
        var ids = new java.util.ArrayList<>(List.of("a"));
        var consensus = new EnsembleConsensus(ConsensusScope.OUTCOME_ONLY, List.of(), 0.5, 1, ids);
        ids.add("b");
        assertEquals(1, consensus.sourceCaseIds().size());
    }
}
```

- [ ] **Step 3: Write validation tests for CbrRetrievalResult**

```java
package io.casehub.api.spi.routing;

import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import org.junit.jupiter.api.Test;

class CbrRetrievalResultTest {

    @Test
    void empty_factory() {
        var result = CbrRetrievalResult.empty();
        assertTrue(result.experiences().isEmpty());
        assertNull(result.ensemble());
    }

    @Test
    void null_experiences_rejected() {
        assertThrows(NullPointerException.class, () ->
            new CbrRetrievalResult(null, null));
    }

    @Test
    void null_ensemble_accepted() {
        var result = new CbrRetrievalResult(List.of(), null);
        assertNull(result.ensemble());
    }

    @Test
    void experiences_defensively_copied() {
        var list = new java.util.ArrayList<RetrievedExperience>();
        var result = new CbrRetrievalResult(list, null);
        assertThrows(UnsupportedOperationException.class, () -> result.experiences().add(null));
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest="StepConsensusEntryTest,EnsembleConsensusTest,CbrRetrievalResultTest" -DfailIfNoTests=false -q`
Expected: compilation failure (types don't exist yet)

- [ ] **Step 5: Create AgreementLevel enum**

```java
package io.casehub.api.spi.routing;

public enum AgreementLevel {
    UNANIMOUS,
    CONSENSUS,
    CONTESTED,
    MINORITY,
    UNIQUE
}
```

- [ ] **Step 6: Create ConsensusScope enum**

```java
package io.casehub.api.spi.routing;

public enum ConsensusScope {
    STEP_LEVEL,
    OUTCOME_ONLY
}
```

- [ ] **Step 7: Create StepConsensusEntry record**

```java
package io.casehub.api.spi.routing;

import jakarta.annotation.Nullable;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record StepConsensusEntry(
    String bindingName,
    @Nullable String capabilityName,
    int occurrenceCount,
    int totalPlans,
    Map<String, Integer> workerDistribution,
    Map<String, Integer> outcomeDistribution,
    Map<Integer, Integer> priorityDistribution,
    List<String> contributingCaseIds,
    AgreementLevel agreement) {

  public StepConsensusEntry {
    Objects.requireNonNull(bindingName, "bindingName");
    if (occurrenceCount < 1) throw new IllegalArgumentException("occurrenceCount must be >= 1");
    if (totalPlans < 1) throw new IllegalArgumentException("totalPlans must be >= 1");
    workerDistribution = workerDistribution != null ? Map.copyOf(workerDistribution) : Map.of();
    outcomeDistribution = outcomeDistribution != null ? Map.copyOf(outcomeDistribution) : Map.of();
    priorityDistribution = priorityDistribution != null ? Map.copyOf(priorityDistribution) : Map.of();
    contributingCaseIds = contributingCaseIds != null ? List.copyOf(contributingCaseIds) : List.of();
    Objects.requireNonNull(agreement, "agreement");
  }
}
```

- [ ] **Step 8: Create EnsembleConsensus record**

```java
package io.casehub.api.spi.routing;

import jakarta.annotation.Nullable;
import java.util.List;
import java.util.Objects;

public record EnsembleConsensus(
    ConsensusScope scope,
    List<StepConsensusEntry> stepAnalysis,
    double ensembleConfidence,
    int inputCount,
    List<String> sourceCaseIds) {

  public EnsembleConsensus {
    Objects.requireNonNull(scope, "scope");
    Objects.requireNonNull(stepAnalysis);
    stepAnalysis = List.copyOf(stepAnalysis);
    Objects.requireNonNull(sourceCaseIds);
    sourceCaseIds = List.copyOf(sourceCaseIds);
    if (ensembleConfidence < 0.0 || ensembleConfidence > 1.0) {
      throw new IllegalArgumentException("ensembleConfidence must be in [0, 1]");
    }
    if (inputCount < 0) {
      throw new IllegalArgumentException("inputCount must be >= 0");
    }
  }
}
```

- [ ] **Step 9: Create CbrRetrievalResult record**

```java
package io.casehub.api.spi.routing;

import jakarta.annotation.Nullable;
import java.util.List;
import java.util.Objects;

public record CbrRetrievalResult(
    List<RetrievedExperience> experiences,
    @Nullable EnsembleConsensus ensemble) {

  public CbrRetrievalResult {
    Objects.requireNonNull(experiences);
    experiences = List.copyOf(experiences);
  }

  public static CbrRetrievalResult empty() {
    return new CbrRetrievalResult(List.of(), null);
  }
}
```

- [ ] **Step 10: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="StepConsensusEntryTest,EnsembleConsensusTest,CbrRetrievalResultTest" -q`
Expected: all PASS

- [ ] **Step 11: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/AgreementLevel.java api/src/main/java/io/casehub/api/spi/routing/ConsensusScope.java api/src/main/java/io/casehub/api/spi/routing/StepConsensusEntry.java api/src/main/java/io/casehub/api/spi/routing/EnsembleConsensus.java api/src/main/java/io/casehub/api/spi/routing/CbrRetrievalResult.java api/src/test/java/io/casehub/api/spi/routing/EnsembleConsensusTest.java api/src/test/java/io/casehub/api/spi/routing/StepConsensusEntryTest.java api/src/test/java/io/casehub/api/spi/routing/CbrRetrievalResultTest.java
git commit -m "feat: add engine-owned ensemble consensus types Refs #1051"
```

### Task 2: Add ExperienceAnalyser.outcomeConsistency() and update CaseStartedEventHandler

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/ExperienceAnalyser.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java:82-95`
- Test: `api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java`

**Interfaces:**
- Consumes: `RetrievedExperience.outcome()` (String)
- Produces: `ExperienceAnalyser.outcomeConsistency(List<RetrievedExperience>)` → `double`

- [ ] **Step 1: Write test for outcomeConsistency**

Add to `ExperienceAnalyserTest.java`:

```java
@Test
void outcomeConsistency_majority_outcome() {
    var experiences = List.of(
        buildExperience("SUCCESS"), buildExperience("SUCCESS"), buildExperience("FAILURE"));
    assertEquals(2.0 / 3.0, ExperienceAnalyser.outcomeConsistency(experiences), 0.001);
}

@Test
void outcomeConsistency_all_same() {
    var experiences = List.of(
        buildExperience("SUCCESS"), buildExperience("SUCCESS"));
    assertEquals(1.0, ExperienceAnalyser.outcomeConsistency(experiences), 0.001);
}

@Test
void outcomeConsistency_all_null_outcomes() {
    var experiences = List.of(buildExperience(null), buildExperience(null));
    assertEquals(0.0, ExperienceAnalyser.outcomeConsistency(experiences), 0.001);
}

@Test
void outcomeConsistency_empty_list() {
    assertEquals(0.0, ExperienceAnalyser.outcomeConsistency(List.of()), 0.001);
}
```

Where `buildExperience(String outcome)` creates a `RetrievedExperience` with the given outcome and default values for all other fields.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest="ExperienceAnalyserTest#outcomeConsistency*" -DfailIfNoTests=false -q`
Expected: compilation failure

- [ ] **Step 3: Add outcomeConsistency to ExperienceAnalyser**

Use `ide_insert_member` to add to `ExperienceAnalyser`:

```java
public static double outcomeConsistency(List<RetrievedExperience> experiences) {
    if (experiences.isEmpty()) return 0.0;
    Map<String, Long> freq = experiences.stream()
        .map(RetrievedExperience::outcome)
        .filter(java.util.Objects::nonNull)
        .collect(java.util.stream.Collectors.groupingBy(
            java.util.function.Function.identity(),
            java.util.stream.Collectors.counting()));
    if (freq.isEmpty()) return 0.0;
    return (double) java.util.Collections.max(freq.values()) / experiences.size();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -Dtest="ExperienceAnalyserTest#outcomeConsistency*" -q`
Expected: all PASS

- [ ] **Step 5: Replace CaseStartedEventHandler.computeOutcomeConsistency with delegate**

In `CaseStartedEventHandler.java`, replace the private `computeOutcomeConsistency` method (lines 82-95) with:

```java
private static double computeOutcomeConsistency(List<RetrievedExperience> experiences) {
    return ExperienceAnalyser.outcomeConsistency(experiences);
}
```

- [ ] **Step 6: Run existing CaseStartedEventHandler tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseStartedEventHandlerTest" -q`
Expected: all PASS (behavior identical)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/routing/ExperienceAnalyser.java api/src/test/java/io/casehub/api/spi/routing/ExperienceAnalyserTest.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java
git commit -m "refactor: consolidate outcomeConsistency into ExperienceAnalyser Refs #1051"
```

## Batch 2: CbrRetrievalService integration + CDI wiring

### Task 3: Refactor CbrRetrievalService to return CbrRetrievalResult with ensemble analysis

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java:584-597`
- Modify: `runtime-spring/src/main/java/io/casehub/engine/internal/spring/RuntimeManualConfig.java:327-334`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalCachingTest.java`

**Interfaces:**
- Consumes: `AgreementLevel`, `ConsensusScope`, `StepConsensusEntry`, `EnsembleConsensus`, `CbrRetrievalResult` (from Task 1)
- Consumes: `ExperienceAnalyser.outcomeConsistency()` (from Task 2)
- Consumes: `PlanEnsembleAnalyzer.analyze(String, List<ScoredCbrCase<ResolvedCase>>, List<AdaptedPlan>, Map<String, FeatureValue>)` (neocortex SPI)
- Produces: `CbrRetrievalService.retrieve(CaseDefinition, CaseInstance) → CbrRetrievalResult`
- Produces: `CbrRetrievalService.retrieve(CaseDefinition, CaseInstance, Class<C>) → CbrRetrievalResult`

- [ ] **Step 1: Write ensemble invocation test**

Add to `CbrRetrievalServiceTest.java`:

```java
@Test
void ensemble_invoked_for_plan_type_with_multiple_results() {
    // Given: 2 ResolvedCase results, mock analyzer returns EnsemblePlan with 2 steps
    CbrConfig config = buildConfig("plan", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(ResolvedCase.class)))
        .thenReturn(List.of(scoredCase1, scoredCase2));
    when(planAdapter.adapt(anyString(), any(), any()))
        .thenReturn(adaptedPlan1, adaptedPlan2);
    EnsemblePlan ensemblePlan = new EnsemblePlan(
        adaptedPlan1,
        List.of(new StepConsensus("binding-a", "cap-a", 2, 2,
            Map.of("w1", 2), Map.of("SUCCESS", 2), Map.of(1, 2),
            List.of("c1", "c2"), StepAgreement.UNANIMOUS)),
        List.of("c1", "c2"), 0.9, 2);
    when(ensembleAnalyzer.analyze(anyString(), anyList(), anyList(), anyMap()))
        .thenReturn(ensemblePlan);

    // When
    CbrRetrievalResult result = service.retrieve(def, buildInstance());

    // Then
    assertNotNull(result.ensemble());
    assertEquals(ConsensusScope.STEP_LEVEL, result.ensemble().scope());
    assertEquals(1, result.ensemble().stepAnalysis().size());
    assertEquals(AgreementLevel.UNANIMOUS, result.ensemble().stepAnalysis().get(0).agreement());
    assertEquals(0.9, result.ensemble().ensembleConfidence(), 0.001);
    verify(ensembleAnalyzer).analyze(anyString(), anyList(), anyList(), anyMap());
}
```

- [ ] **Step 2: Write test for single result (no ensemble)**

```java
@Test
void ensemble_skipped_for_single_result() {
    CbrConfig config = buildConfig("plan", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(ResolvedCase.class)))
        .thenReturn(List.of(scoredCase1));
    when(planAdapter.adapt(anyString(), any(), any())).thenReturn(adaptedPlan1);

    CbrRetrievalResult result = service.retrieve(def, buildInstance());

    assertFalse(result.experiences().isEmpty());
    assertNull(result.ensemble());
    verifyNoInteractions(ensembleAnalyzer);
}
```

- [ ] **Step 3: Write test for non-plan type (outcome-only)**

```java
@Test
void non_plan_type_gets_outcome_only_consensus() {
    CbrConfig config = buildConfig("feature-vector", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(FeatureVectorCbrCase.class)))
        .thenReturn(List.of(fvCase1, fvCase2));

    CbrRetrievalResult result = service.retrieve(def, buildInstance());

    assertNotNull(result.ensemble());
    assertEquals(ConsensusScope.OUTCOME_ONLY, result.ensemble().scope());
    assertTrue(result.ensemble().stepAnalysis().isEmpty());
    verifyNoInteractions(ensembleAnalyzer);
}
```

- [ ] **Step 4: Write test for analyzer timeout**

```java
@Test
void ensemble_timeout_returns_null_ensemble() {
    CbrConfig config = buildConfig("plan", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(ResolvedCase.class)))
        .thenReturn(List.of(scoredCase1, scoredCase2));
    when(planAdapter.adapt(anyString(), any(), any()))
        .thenReturn(adaptedPlan1, adaptedPlan2);
    when(ensembleAnalyzer.analyze(anyString(), anyList(), anyList(), anyMap()))
        .thenAnswer(inv -> { Thread.sleep(10_000); return null; });

    // Service constructed with 100ms timeout for test
    var testService = new CbrRetrievalService(
        jqEvaluator, cbrStore, planAdapter, ensembleAnalyzer, List.of(), 100L);

    CbrRetrievalResult result = testService.retrieve(def, buildInstance());

    assertFalse(result.experiences().isEmpty());
    assertNull(result.ensemble());
}
```

- [ ] **Step 5: Write test for analyzer reporting single plan (NoOp behavior)**

```java
@Test
void analyzer_reporting_single_plan_returns_null_ensemble() {
    CbrConfig config = buildConfig("plan", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(ResolvedCase.class)))
        .thenReturn(List.of(scoredCase1, scoredCase2));
    when(planAdapter.adapt(anyString(), any(), any()))
        .thenReturn(adaptedPlan1, adaptedPlan2);
    // NoOp-style: picks best plan, inputPlanCount=1
    EnsemblePlan noopResult = new EnsemblePlan(
        adaptedPlan1, List.of(), List.of("c1"), 0.9, 1);
    when(ensembleAnalyzer.analyze(anyString(), anyList(), anyList(), anyMap()))
        .thenReturn(noopResult);

    CbrRetrievalResult result = service.retrieve(def, buildInstance());

    assertNull(result.ensemble());
}
```

- [ ] **Step 6: Write test for partial adaptation failure**

```java
@Test
void partial_adaptation_failure_preserves_sizing_invariant() {
    CbrConfig config = buildConfig("plan", "domain", false);
    CaseDefinition def = buildDefinition(config);
    when(cbrStore.retrieveSimilar(any(), eq(ResolvedCase.class)))
        .thenReturn(List.of(scoredCase1, scoredCase2));
    when(planAdapter.adapt(anyString(), eq(scoredCase1), any()))
        .thenReturn(adaptedPlan1);
    when(planAdapter.adapt(anyString(), eq(scoredCase2), any()))
        .thenThrow(new RuntimeException("adaptation failed"));
    EnsemblePlan plan = new EnsemblePlan(
        adaptedPlan1, List.of(), List.of("c1", "c2"), 0.8, 2);
    when(ensembleAnalyzer.analyze(anyString(), anyList(), anyList(), anyMap()))
        .thenReturn(plan);

    CbrRetrievalResult result = service.retrieve(def, buildInstance());

    // analyzer was still called — fallback AdaptedPlan kept the lists aligned
    verify(ensembleAnalyzer).analyze(anyString(), argThat(l -> l.size() == 2),
        argThat(l -> l.size() == 2), anyMap());
    assertEquals(2, result.experiences().size());
}
```

- [ ] **Step 7: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest="CbrRetrievalServiceTest#ensemble*,CbrRetrievalServiceTest#non_plan*,CbrRetrievalServiceTest#partial*,CbrRetrievalServiceTest#analyzer*" -DfailIfNoTests=false -q`
Expected: compilation failure

- [ ] **Step 8: Implement CbrRetrievalService changes**

Modify `CbrRetrievalService.java`:

1. Add `PlanEnsembleAnalyzer ensembleAnalyzer` and `long ensembleTimeoutMs` fields
2. Update both constructors to accept `PlanEnsembleAnalyzer` and timeout
3. Add internal record `AdaptationResult(AdaptedPlan adaptedPlan, List<ExperiencePlanStep> steps)`
4. Refactor `adaptAndMapResolutionStep()` to return `AdaptationResult` (keeping fallback behavior)
5. Change `retrieveInternal()` to:
   - Collect `AdaptationResult` per scored case
   - Build experiences from adaptation results
   - Invoke ensemble analysis (with partitioning, min-count guard, timeout)
   - Return `CbrRetrievalResult`
6. Change `retrieve()` and generic `retrieve()` return types to `CbrRetrievalResult`
7. Change cache from `ConcurrentHashMap<UUID, List<RetrievedExperience>>` to `ConcurrentHashMap<UUID, CbrRetrievalResult>`
8. Add `mapEnsemblePlan(EnsemblePlan)` and `buildOutcomeOnlyConsensus(List<RetrievedExperience>, List<? extends ScoredCbrCase<?>>)` methods

See spec §CbrRetrievalService Changes for the full implementation details.

- [ ] **Step 9: Update RuntimeBeans CDI producer**

In `RuntimeBeans.java:584-597`, add `PlanEnsembleAnalyzer ensembleAnalyzer` and `@ConfigProperty(name = "casehub.engine.cbr.ensemble-timeout-ms", defaultValue = "5000") long ensembleTimeoutMs` parameters. Pass them to the `CbrRetrievalService` constructor.

- [ ] **Step 10: Update RuntimeManualConfig Spring bean**

In `RuntimeManualConfig.java:327-334`, add `PlanEnsembleAnalyzer ensembleAnalyzer` and `@Value("${casehub.engine.cbr.ensemble-timeout-ms:5000}") long ensembleTimeoutMs` parameters. Pass them to the constructor.

- [ ] **Step 11: Update existing CbrRetrievalServiceTest tests**

All existing tests call `service.retrieve()` and expect `List<RetrievedExperience>`. Update them to expect `CbrRetrievalResult` and use `.experiences()`. The test constructor needs updating to accept the mock `PlanEnsembleAnalyzer`.

- [ ] **Step 12: Update CbrRetrievalCachingTest**

Cache type changed — update test expectations to use `CbrRetrievalResult`.

- [ ] **Step 13: Run all CbrRetrievalService tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CbrRetrievalServiceTest,CbrRetrievalCachingTest" -q`
Expected: all PASS

- [ ] **Step 14: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java runtime-spring/src/main/java/io/casehub/engine/internal/spring/RuntimeManualConfig.java runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalServiceTest.java runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalCachingTest.java
git commit -m "feat: integrate PlanEnsembleAnalyzer into CbrRetrievalService Refs #1051"
```

## Batch 3: Caller updates + context surfacing + CLAUDE.md

### Task 4: Update all retrieve() callers and surface cbrEnsemble in context

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java:175-199`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:286`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java:149`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DeeperDecompositionHandler.java:143`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java:585`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java:164-165`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandlerTest.java`

**Interfaces:**
- Consumes: `CbrRetrievalResult.experiences()`, `CbrRetrievalResult.ensemble()` (from Task 3)

- [ ] **Step 1: Write test for cbrEnsemble context injection**

Add to `CaseStartedEventHandlerTest.java`:

```java
@Test
void cbrEnsemble_written_to_context_when_ensemble_present() {
    // Given: CbrRetrievalService returns result with ensemble
    var ensemble = new EnsembleConsensus(
        ConsensusScope.STEP_LEVEL,
        List.of(new StepConsensusEntry("binding-a", "cap-a", 2, 3,
            Map.of("w1", 2), Map.of("SUCCESS", 2), Map.of(1, 2),
            List.of("c1", "c2"), AgreementLevel.CONSENSUS)),
        0.72, 3, List.of("c1", "c2", "c3"));
    var result = new CbrRetrievalResult(List.of(experience1, experience2), ensemble);
    when(cbrRetrievalService.retrieve(eq(definition), eq(instance))).thenReturn(result);

    // When
    handler.onCaseStarted(new CaseStartedEvent(instance));

    // Then
    var working = instance.getCaseContext().layer(ContextLayer.WORKING);
    assertNotNull(working.get("cbrEnsemble"));
}

@Test
void cbrEnsemble_not_written_when_ensemble_null() {
    var result = new CbrRetrievalResult(List.of(experience1), null);
    when(cbrRetrievalService.retrieve(eq(definition), eq(instance))).thenReturn(result);

    handler.onCaseStarted(new CaseStartedEvent(instance));

    var working = instance.getCaseContext().layer(ContextLayer.WORKING);
    assertNull(working.get("cbrEnsemble"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseStartedEventHandlerTest#cbrEnsemble*" -DfailIfNoTests=false -q`
Expected: compilation failure (return type mismatch)

- [ ] **Step 3: Update CaseStartedEventHandler.injectCbrExperiences()**

Replace the `injectCbrExperiences` method body. Use `CbrRetrievalResult result = cbrRetrievalService.retrieve(definition, instance)`, extract `result.experiences()` for existing logic, and add `cbrEnsemble` injection when `result.ensemble() != null`. See spec §Context Surfacing for the exact code.

- [ ] **Step 4: Update CaseContextChangedEventHandler.rules()**

At line 286, change:
```java
List<RetrievedExperience> experiences = cbrRetrievalService.retrieve(definition, caseInstance);
```
to:
```java
List<RetrievedExperience> experiences = cbrRetrievalService.retrieve(definition, caseInstance).experiences();
```

- [ ] **Step 5: Update DefaultGoalDecomposer.decomposeGoal()**

At line 149, change:
```java
experiences = cbrRetrievalServiceInstance.get().retrieve(definition, instance);
```
to:
```java
experiences = cbrRetrievalServiceInstance.get().retrieve(definition, instance).experiences();
```

- [ ] **Step 6: Update DeeperDecompositionHandler.tryDecompose()**

At line 143, same pattern:
```java
experiences = cbrRetrievalServiceInstance.get().retrieve(definition, instance).experiences();
```

- [ ] **Step 7: Update DefaultPlanAdaptationEvaluator.retrieveExperiences()**

At line 585, change:
```java
return cbrRetrievalService.get().retrieve(definition, instance);
```
to:
```java
return cbrRetrievalService.get().retrieve(definition, instance).experiences();
```

- [ ] **Step 8: Update DefaultWorkOrchestrator.doSubmit()**

At line 164-165, change:
```java
final java.util.List<RetrievedExperience> experiences =
    cbrRetrievalService.retrieve(definition, instance);
```
to:
```java
final java.util.List<RetrievedExperience> experiences =
    cbrRetrievalService.retrieve(definition, instance).experiences();
```

- [ ] **Step 9: Update existing CaseStartedEventHandlerTest mocks**

Existing tests mock `cbrRetrievalService.retrieve()` to return `List.of(...)`. Update all mocks to return `new CbrRetrievalResult(List.of(...), null)`.

- [ ] **Step 10: Update any remaining test mocks across modules**

Check `CaseContextChangedEventHandlerRoutingTest` and `DefaultWorkOrchestratorTest` — update mocks similarly.

- [ ] **Step 11: Build and run full test suite for affected modules**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime,runtime-core,planning-core -q`
Expected: all PASS

- [ ] **Step 11b: Verify planning-core callers compile**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -q`
Expected: all PASS (confirms DefaultGoalDecomposer, DeeperDecompositionHandler, DefaultPlanAdaptationEvaluator callers updated correctly)

- [ ] **Step 12: Update CLAUDE.md**

Add to `## CBR Retrieval Bridge` section the documentation block from the spec §CLAUDE.md Updates.

- [ ] **Step 13: Create follow-up issue for retrieveForSelection ensemble support**

```bash
gh issue create --repo casehubio/engine --title "CBR: add ensemble analysis support to retrieveForSelection overloads" --body "Follow-up from #1051. retrieveForSelection() retains List<RetrievedExperience> return type — no CaseDefinition or CbrConfig available, cannot perform ensemble analysis. Track adding ensemble support when a consumer needs it." --label "scale: S" --label "complexity: Low"
```

- [ ] **Step 14: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java planning-core/src/main/java/io/casehub/engine/planning/decomposition/DefaultGoalDecomposer.java planning-core/src/main/java/io/casehub/engine/planning/adaptation/DeeperDecompositionHandler.java planning-core/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandlerTest.java CLAUDE.md
git commit -m "feat: surface cbrEnsemble in context, update all retrieve() callers Closes #1051"
```

## References

- [2026-09-15-cbr-ensemble-consensus-design.md] — design spec this plan implements
- `runtime-core/.../CbrRetrievalService.java:56` — retrieval service (main integration point)
- `runtime/.../CaseStartedEventHandler.java:82-95,175-199` — outcome consistency + context injection
- `runtime-core/.../CaseContextChangedEventHandler.java:286` — caller site
- `planning-core/.../DefaultGoalDecomposer.java:149` — caller site
- `planning-core/.../DeeperDecompositionHandler.java:143` — caller site
- `planning-core/.../DefaultPlanAdaptationEvaluator.java:585` — caller site
- `runtime/.../DefaultWorkOrchestrator.java:164-165` — caller site
- `runtime/.../RuntimeBeans.java:584-597` — Quarkus CDI producer
- `runtime-spring/.../RuntimeManualConfig.java:327-334` — Spring bean factory
- `api/.../ExperienceAnalyser.java` — outcomeConsistency consolidation target
- GitHub #1051 — focal issue
