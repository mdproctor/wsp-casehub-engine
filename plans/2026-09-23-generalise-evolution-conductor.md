# Generalise Evolution Conductor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1148 — Generalise evolution conductor — domain-agnostic improvement loop with pluggable categories and health sensors
**Issue group:** #1141, #1142, #1143, #1144, #1146, #1148

**Goal:** Extract 5 new SPIs from the evolution conductor so improvement categories, proposal sources, regression evaluation, conflict detection, and deny patterns are pluggable per domain. Code-evolution becomes one domain plugin.

**Architecture:** Three orchestration SPIs (ImprovementCategoryProvider, ImprovementProposalSource, RegressionEvaluator) and two filtering-pipeline SPIs (ConflictStrategy, DenyPatternProvider) replace hardcoded code-evolution assumptions. Each SPI follows the established CapabilityArea registry+bootstrap pattern. The orchestration backbone (EvolutionTicker, circuit breaker, budget enforcer) stays concrete, delegating domain-specific behavior to pluggable components. ImprovementStage migrates from enum to string to support domain-contributed lifecycle stages.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (Jakarta Inject), JUnit 5, AssertJ

## Global Constraints

- NO `@DefaultBean` on multi-instance SPIs (PP-20260921-b7c277)
- All default implementations in `runtime-core` package `io.casehub.engine.internal.improvement`
- All SPI interfaces in `api` package `io.casehub.api.spi.improvement`
- All model records in `api` package `io.casehub.api.model.stigmergy`
- TDD: failing test → verify fail → implement → verify pass → commit
- Use `ide_insert_member` / `ide_replace_member` / `ide_edit_member` for code changes
- Use `ide_refactor_rename` for renames, `ide_refactor_safe_delete` for deletions
- Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime-core -q`

---

## Batch 1: API Contracts and Type Migration

After this batch: all new types and SPI interfaces exist, ImprovementStage enum is gone (replaced by String throughout), ImprovementRequest has `domainId`, ImprovementBudget has `maxChangeSize`. All tests pass with string constants.

### Task 1: Create API model types and SPI interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/CategoryDescriptor.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/StageDescriptor.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/RegressionVerdict.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/HealthScoreSnapshot.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ProposalFilteringSummary.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ImprovementCategoryProvider.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ImprovementProposalSource.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/RegressionEvaluator.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/ConflictStrategy.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/DenyPatternProvider.java`
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/RegressionVerdictTest.java`
- Test: `api/src/test/java/io/casehub/api/model/stigmergy/ProposalFilteringSummaryTest.java`

**Interfaces:**
- Produces: `CategoryDescriptor(String id, String name, String description, String domainId)`, `StageDescriptor(String id, String name, int ordinal, boolean gateCheckpoint, String domainId)`, `RegressionVerdict` sealed interface (`NoRegression`, `Detected(double confidence, String reason)`), `HealthScoreSnapshot(double score, Instant timestamp, Map<String, Double> componentScores)`, `ProposalFilteringSummary(Map<String, Integer> proposalsBySource, int afterCategoryFilter, int afterSuppressionFilter, int afterAntiOscillationFilter, int afterDenyFilter, int afterBudgetFilter, int afterConflictFilter, int proposed)`, `ImprovementCategoryProvider` interface, `ImprovementProposalSource` interface, `RegressionEvaluator` interface, `ConflictStrategy` interface with `ConflictResult` sealed interface, `DenyPatternProvider` interface

- [ ] **Step 1: Write tests for RegressionVerdict sealed interface**

```java
package io.casehub.api.model.stigmergy;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class RegressionVerdictTest {

  @Test
  void noRegressionIsInstanceOfVerdict() {
    RegressionVerdict verdict = new RegressionVerdict.NoRegression();
    assertThat(verdict).isInstanceOf(RegressionVerdict.class);
  }

  @Test
  void detectedCarriesConfidenceAndReason() {
    var detected = new RegressionVerdict.Detected(0.85, "health score dropped");
    assertThat(detected.confidence()).isEqualTo(0.85);
    assertThat(detected.reason()).isEqualTo("health score dropped");
  }

  @Test
  void patternMatchExhaustive() {
    RegressionVerdict verdict = new RegressionVerdict.Detected(0.5, "test");
    String result = switch (verdict) {
      case RegressionVerdict.NoRegression nr -> "none";
      case RegressionVerdict.Detected d -> "detected:" + d.confidence();
    };
    assertThat(result).isEqualTo("detected:0.5");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=RegressionVerdictTest -q`
Expected: FAIL — `RegressionVerdict` class not found

- [ ] **Step 3: Create all API model records**

Create `CategoryDescriptor.java`:
```java
package io.casehub.api.model.stigmergy;

public record CategoryDescriptor(String id, String name, String description, String domainId) {}
```

Create `StageDescriptor.java`:
```java
package io.casehub.api.model.stigmergy;

public record StageDescriptor(String id, String name, int ordinal, boolean gateCheckpoint, String domainId) {}
```

Create `RegressionVerdict.java`:
```java
package io.casehub.api.model.stigmergy;

public sealed interface RegressionVerdict
    permits RegressionVerdict.NoRegression, RegressionVerdict.Detected {
  record NoRegression() implements RegressionVerdict {}
  record Detected(double confidence, String reason) implements RegressionVerdict {}
}
```

Create `HealthScoreSnapshot.java`:
```java
package io.casehub.api.model.stigmergy;

import java.time.Instant;
import java.util.Map;

public record HealthScoreSnapshot(double score, Instant timestamp, Map<String, Double> componentScores) {}
```

Create `ProposalFilteringSummary.java`:
```java
package io.casehub.api.model.stigmergy;

import java.util.Map;

public record ProposalFilteringSummary(
    Map<String, Integer> proposalsBySource,
    int afterCategoryFilter,
    int afterSuppressionFilter,
    int afterAntiOscillationFilter,
    int afterDenyFilter,
    int afterBudgetFilter,
    int afterConflictFilter,
    int proposed) {}
```

- [ ] **Step 4: Create all 5 SPI interfaces**

Create `ImprovementCategoryProvider.java`:
```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.CategoryDescriptor;
import io.casehub.api.model.stigmergy.StageDescriptor;
import java.util.List;

public interface ImprovementCategoryProvider {
  String domainId();
  List<CategoryDescriptor> categories();
  List<StageDescriptor> stages();
}
```

Create `ImprovementProposalSource.java`:
```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.ImprovementConfig;
import io.casehub.api.model.stigmergy.ImprovementRequest;
import java.util.List;
import java.util.UUID;

public interface ImprovementProposalSource {
  String sourceId();
  String domainId();
  List<ImprovementRequest> propose(UUID caseId, String tenancyId, ImprovementConfig config);
}
```

Create `RegressionEvaluator.java`:
```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.HealthScoreSnapshot;
import io.casehub.api.model.stigmergy.RegressionVerdict;
import java.util.UUID;

public interface RegressionEvaluator {
  String evaluatorId();
  String domainId();
  RegressionVerdict evaluate(UUID caseId, HealthScoreSnapshot baseline, HealthScoreSnapshot current, String category);
}
```

Create `ConflictStrategy.java`:
```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.ImprovementRequest;
import java.util.Map;
import java.util.UUID;

public interface ConflictStrategy {
  String domainId();
  ConflictResult check(ImprovementRequest request, Map<UUID, ImprovementRequest> activeImprovements, int trivialThreshold);

  sealed interface ConflictResult permits ConflictResult.Clear, ConflictResult.Conflicting {
    record Clear() implements ConflictResult {}
    record Conflicting(UUID blockingImprovementId, String reason) implements ConflictResult {}
  }
}
```

Create `DenyPatternProvider.java`:
```java
package io.casehub.api.spi.improvement;

import io.casehub.api.model.stigmergy.ImprovementConfig;
import io.casehub.api.model.stigmergy.ImprovementRequest;
import java.util.UUID;

public interface DenyPatternProvider {
  String domainId();
  boolean isDenied(UUID caseId, String tenancyId, ImprovementRequest request, ImprovementConfig config);
}
```

- [ ] **Step 5: Run tests to verify all compile and RegressionVerdictTest passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=RegressionVerdictTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/CategoryDescriptor.java api/src/main/java/io/casehub/api/model/stigmergy/StageDescriptor.java api/src/main/java/io/casehub/api/model/stigmergy/RegressionVerdict.java api/src/main/java/io/casehub/api/model/stigmergy/HealthScoreSnapshot.java api/src/main/java/io/casehub/api/model/stigmergy/ProposalFilteringSummary.java api/src/main/java/io/casehub/api/spi/improvement/ImprovementCategoryProvider.java api/src/main/java/io/casehub/api/spi/improvement/ImprovementProposalSource.java api/src/main/java/io/casehub/api/spi/improvement/RegressionEvaluator.java api/src/main/java/io/casehub/api/spi/improvement/ConflictStrategy.java api/src/main/java/io/casehub/api/spi/improvement/DenyPatternProvider.java api/src/test/java/io/casehub/api/model/stigmergy/RegressionVerdictTest.java
git commit -m "feat(#1148): add API types and SPI interfaces for evolution generalisation"
```

### Task 2: Migrate ImprovementStage enum → String + ImprovementRequest + ImprovementBudget

**Files:**
- Delete: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementStage.java` (use `ide_refactor_safe_delete`)
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CodeEvolutionStages.java`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/GatePolicy.java` — `Map<ImprovementStage, GateMode>` → `Map<String, GateMode>`, remove constructor validation
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ConductorInboxEntry.java` — `ImprovementStage stage` → `String stage`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ArtifactEntry.java` — `ImprovementStage stage` → `String stage`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchPipelineResult.java` — `ImprovementStage stage` → `String stage`
- Modify: `api/src/main/java/io/casehub/api/spi/improvement/EscalationProvider.java` — `ImprovementStage stage` → `String stage`
- Modify: `api/src/main/java/io/casehub/api/view/EvolutionStateSnapshot.java` — `ImprovementStage` → `String` in ImprovementStreamView and StageProgress
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementRequest.java` — add `domainId` field (keep `targetRepo`/`targetPaths` for now)
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementBudget.java` — rename `maxPRSize` → `maxChangeSize`, `effectiveMaxPRSize()` → `effectiveMaxChangeSize()`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/TickTrace.java` — rename `SignalFilteringSummary` → `ProposalFilteringSummary` (use new top-level record), update `ProposalGenerated`
- Modify: `schema/src/main/resources/schema/yaml-record-mappings.yaml` — update GatePolicy key type
- Modify: all test files referencing `ImprovementStage` enum values — use `CodeEvolutionStages` string constants
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/GatePolicyTest.java` — update to use String keys
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceTest.java` — remove `improvementStageGateCheckpoints` test, update SignalFilteringSummary refs

**Interfaces:**
- Consumes: `ProposalFilteringSummary` from Task 1
- Produces: `CodeEvolutionStages` constants class, `ImprovementRequest` with `domainId`, `GatePolicy(Map<String, GateMode>, Integer)`, `ImprovementBudget` with `maxChangeSize`/`effectiveMaxChangeSize()`

- [ ] **Step 1: Create CodeEvolutionStages constants class**

```java
package io.casehub.engine.internal.improvement;

public final class CodeEvolutionStages {
  public static final String INTROSPECT = "introspect";
  public static final String RESEARCH_SCOPE = "research-scope";
  public static final String SEARCH = "search";
  public static final String ANALYZE = "analyze";
  public static final String HYPOTHESIS_APPROVAL = "hypothesis-approval";
  public static final String IMPLEMENTATION_PLAN = "implementation-plan";
  public static final String IMPLEMENT = "implement";
  public static final String SUBMIT_PR = "submit-pr";
  public static final String PR_REVIEW = "pr-review";
  public static final String INTEGRATE = "integrate";
  public static final String OUTCOME_RECORDING = "outcome-recording";
  private CodeEvolutionStages() {}
}
```

- [ ] **Step 2: Migrate GatePolicy — change Map key from ImprovementStage to String, remove constructor validation**

Use `ide_replace_member` on `GatePolicy` to replace the full record body. The new record:
```java
public record GatePolicy(@Nullable Map<String, GateMode> modes, @Nullable Integer gateTimeoutMinutes) {
  public enum GateMode { GATED, AUTO, NOTIFY }

  public GateMode effectiveMode(String stageId) {
    if (modes != null && modes.containsKey(stageId)) {
      return modes.get(stageId);
    }
    return GateMode.AUTO;
  }

  public int effectiveGateTimeoutMinutes() {
    return gateTimeoutMinutes != null ? gateTimeoutMinutes : 1440;
  }
}
```

Note: the old `PR_REVIEW defaults to GATED` behavior is removed from `effectiveMode()`. It will be configured via the case template's `ImprovementConfig.gatePolicy`.

- [ ] **Step 3: Migrate ConductorInboxEntry, ArtifactEntry, ResearchPipelineResult, EscalationProvider, EvolutionStateSnapshot — ImprovementStage → String**

For each file, use `ide_edit_member` to change the `ImprovementStage` field type to `String`. Remove the `import io.casehub.api.model.stigmergy.ImprovementStage;` line.

Files and changes:
- `ConductorInboxEntry.java:26` — `ImprovementStage stage` → `String stage`
- `ArtifactEntry.java:21` — `ImprovementStage stage` → `String stage`
- `ResearchPipelineResult.java:25` — `AwaitingGate` record: `ImprovementStage stage` → `String stage`
- `EscalationProvider.java:31` — `evaluate()` parameter: `ImprovementStage stage` → `String stage`
- `EvolutionStateSnapshot.java:59` — `ImprovementStreamView.currentStage`: `ImprovementStage` → `String`
- `EvolutionStateSnapshot.java:66` — `StageProgress.stage`: `ImprovementStage` → `String`

- [ ] **Step 4: Add domainId to ImprovementRequest**

Use `ide_edit_member` on `ImprovementRequest` to add `@Nullable String domainId` as the last parameter. Keep `targetRepo` and `targetPaths` for now — they will be removed in Batch 3 when `CodeEvolutionMetadata` consumers are in place.

```java
public record ImprovementRequest(
    String improvementType,
    String category,
    String target,
    String targetRepo,
    List<String> targetPaths,
    int estimatedSize,
    Map<String, String> metadata,
    @Nullable String domainId) {}
```

- [ ] **Step 5: Rename ImprovementBudget.maxPRSize → maxChangeSize**

Use `ide_refactor_rename` on the `maxPRSize` field to `maxChangeSize`. Then rename `effectiveMaxPRSize()` to `effectiveMaxChangeSize()`. IntelliJ will update all references.

- [ ] **Step 6: Update TickTrace — replace SignalFilteringSummary with ProposalFilteringSummary**

Use `ide_edit_member` on `TickTrace` to replace `SignalFilteringSummary` with a reference to the new `ProposalFilteringSummary` record from Task 1. Update `ProposalGenerated`:

```java
record ProposalGenerated(int goalCount, @Nullable ProposalFilteringSummary filtering)
    implements TickOutcome {}
```

Delete the `SignalFilteringSummary` inner record from TickTrace.

- [ ] **Step 7: Delete ImprovementStage enum**

Use `ide_refactor_safe_delete` on `ImprovementStage.java`. If safe delete reports usages, update remaining references first.

- [ ] **Step 8: Update ResearchPipelineOrchestrator — use CodeEvolutionStages constants**

Replace all `ImprovementStage.RESEARCH_SCOPE` references with `CodeEvolutionStages.RESEARCH_SCOPE`, etc. There are ~5 references at lines 82, 85, 92, 94, and 151.

- [ ] **Step 9: Update DefaultEscalationProvider — String stage parameter**

Replace `ImprovementStage stage` with `String stage` in `evaluate()` method signature at line 40.

- [ ] **Step 10: Update all test files — replace enum references with CodeEvolutionStages constants**

Files to update (replace `ImprovementStage.RESEARCH_SCOPE` → `CodeEvolutionStages.RESEARCH_SCOPE` etc.):
- `GatePolicyTest.java` — remove constructor validation test (`rejectsNonCheckpointStage`), update all `ImprovementStage.X` to `CodeEvolutionStages.X`
- `ConductorInboxManagerTest.java` — update `makeEntry` and all references
- `ConductorInboxRepositoryContractTest.java` — update `makeEntry` and all references
- `InMemoryConductorInboxRepositoryContractTest.java` — update entry construction
- `DefaultEscalationProviderTest.java` — update all `ImprovementStage.X`
- `EvolutionApiTest.java` — update all `ImprovementStage.X`
- `ResearchPipelineCheckpointTest.java` — update all `ImprovementStage.X`
- `TickTraceTest.java` — remove `improvementStageGateCheckpoints` test, update `SignalFilteringSummary` refs

- [ ] **Step 11: Update yaml-record-mappings.yaml for GatePolicy key type change**

Update the `GatePolicy` entry so the key type is `String` instead of `ImprovementStage`.

- [ ] **Step 12: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime-core -q`
Expected: All tests PASS

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "feat(#1148): migrate ImprovementStage enum→string, add domainId to ImprovementRequest, rename maxPRSize→maxChangeSize"
```

---

## Batch 2: Registries, Default Implementations, and Bootstrap

After this batch: 5 registries exist with their own tests, all default code-evolution implementations are created and tested, EvolutionBootstrap discovers all SPI beans. No backbone rewiring yet — old and new infrastructure coexist.

### Task 3: Create registries, CodeEvolutionMetadata, CodeEvolutionCategoryProvider, and EvolutionBootstrap

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCategoryRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementProposalSourceRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionEvaluatorRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConflictStrategyRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DenyPatternProviderRegistry.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CodeEvolutionMetadata.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CodeEvolutionCategoryProvider.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionBootstrap.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCategoryRegistryTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/CodeEvolutionMetadataTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/CodeEvolutionCategoryProviderTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionBootstrapTest.java`

**Interfaces:**
- Consumes: `CategoryDescriptor`, `StageDescriptor`, `ImprovementCategoryProvider`, `ImprovementProposalSource`, `RegressionEvaluator`, `ConflictStrategy`, `DenyPatternProvider` from Task 1
- Produces: `ImprovementCategoryRegistry` (allCategories, getCategory, isGateCheckpoint, stagesForDomain, domainForCategory), `ImprovementProposalSourceRegistry` (all), `RegressionEvaluatorRegistry` (all), `ConflictStrategyRegistry` (forDomain), `DenyPatternProviderRegistry` (forDomain), `CodeEvolutionMetadata` (extractPaths, extractRepo, encode), `CodeEvolutionCategoryProvider` (5 categories, 11 stages), `EvolutionBootstrap` (discovers and registers all SPIs)

- [ ] **Step 1: Write ImprovementCategoryRegistryTest**

Test: provider registration populates categories and stages; `isGateCheckpoint(domainId, stageId)` returns true for checkpoints, false for non-checkpoints; `domainForCategory()` resolves correctly; multi-provider coexistence; reset clears all.

```java
@Test
void registerProviderPopulatesCategoriesAndStages() {
  var provider = new CodeEvolutionCategoryProvider();
  registry.registerProvider(provider);
  assertThat(registry.allCategories()).hasSize(5);
  assertThat(registry.stagesForDomain("code-evolution")).hasSize(11);
  assertThat(registry.isGateCheckpoint("code-evolution", "research-scope")).isTrue();
  assertThat(registry.isGateCheckpoint("code-evolution", "introspect")).isFalse();
  assertThat(registry.domainForCategory("dependency-update")).hasValue("code-evolution");
}
```

- [ ] **Step 2: Write CodeEvolutionMetadataTest**

Test: `extractPaths()` with empty metadata, single path, multiple comma-separated paths; `extractRepo()` with and without key; `encode()` round-trip.

- [ ] **Step 3: Write CodeEvolutionCategoryProviderTest**

Test: `categories()` returns 5 entries; `stages()` returns 11 entries in ordinal order; gate checkpoints are RESEARCH_SCOPE, HYPOTHESIS_APPROVAL, IMPLEMENTATION_PLAN, PR_REVIEW.

- [ ] **Step 4: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ImprovementCategoryRegistryTest,CodeEvolutionMetadataTest,CodeEvolutionCategoryProviderTest -q`
Expected: FAIL — classes not found

- [ ] **Step 5: Create all 5 registries**

Create each registry following the spec's code in §2 exactly. Each implements `Resettable`. `ImprovementCategoryRegistry` uses `ConcurrentHashMap` keyed by category ID and domain-scoped stage lists. `ConflictStrategyRegistry` and `DenyPatternProviderRegistry` key by `domainId`. `ImprovementProposalSourceRegistry` and `RegressionEvaluatorRegistry` use `CopyOnWriteArrayList`.

- [ ] **Step 6: Create CodeEvolutionMetadata**

Utility class with `TARGET_PATHS` and `TARGET_REPO` constants, `extractPaths(ImprovementRequest)`, `extractRepo(ImprovementRequest)`, `encode(String repo, List<String> paths)`. See spec §1.5 CodeEvolutionMetadata for complete code.

- [ ] **Step 7: Create CodeEvolutionCategoryProvider**

`@ApplicationScoped`, implements `ImprovementCategoryProvider`, `domainId()` returns `"code-evolution"`, `categories()` returns 5 descriptors, `stages()` returns 11 stage descriptors with correct ordinals and gate checkpoint flags. See spec §1.1 for complete code.

- [ ] **Step 8: Create EvolutionBootstrap**

`@ApplicationScoped`, `@Inject @Any Instance<T>` for all 5 SPI types, `@Observes StartupEvent` handler iterates and registers. See spec §2 EvolutionBootstrap for complete code.

- [ ] **Step 9: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ImprovementCategoryRegistryTest,CodeEvolutionMetadataTest,CodeEvolutionCategoryProviderTest -q`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(#1148): add registries, CodeEvolutionMetadata, CodeEvolutionCategoryProvider, EvolutionBootstrap"
```

### Task 4: Create default domain implementations

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/SignalConsensusProposalSource.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreDeltaRegressionEvaluator.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/FilePathConflictStrategy.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CodeEvolutionDenyPatternProvider.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/SignalConsensusProposalSourceTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/HealthScoreDeltaRegressionEvaluatorTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/FilePathConflictStrategyTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/CodeEvolutionDenyPatternProviderTest.java`

**Interfaces:**
- Consumes: `ImprovementProposalSource`, `RegressionEvaluator`, `ConflictStrategy`, `DenyPatternProvider` from Task 1; `CodeEvolutionMetadata`, `CodeEvolutionStages` from Tasks 2-3
- Produces: `SignalConsensusProposalSource` (wraps existing signal-consensus logic), `HealthScoreDeltaRegressionEvaluator` (wraps ConfidenceScorer), `FilePathConflictStrategy` (extracts ConflictDetector logic using CodeEvolutionMetadata), `CodeEvolutionDenyPatternProvider` (extracts ImprovementBudgetEnforcer deny logic)

- [ ] **Step 1: Write SignalConsensusProposalSourceTest**

Test: consensus signals matching namespace produce ImprovementRequests; non-matching namespace filtered; empty consensus returns empty list; missing signal context skipped.

Key test structure:
```java
@Test
void proposesFromConsensusSignals() {
  signalRegistry.recordSignal(caseId, "improvement:dep-update:lodash", "source-a", 1.0);
  signalRegistry.recordSignal(caseId, "improvement:dep-update:lodash", "source-b", 1.0);
  var request = new ImprovementRequest("upgrade", "dependency-update", "lodash", "repo", List.of(), 5, Map.of(), "code-evolution");
  signalContext.put(caseId, "improvement:dep-update:lodash", request);

  var proposals = source.propose(caseId, "t1", config);
  assertThat(proposals).hasSize(1);
  assertThat(proposals.get(0).category()).isEqualTo("dependency-update");
}
```

- [ ] **Step 2: Write HealthScoreDeltaRegressionEvaluatorTest**

Test: positive delta returns NoRegression; negative delta returns Detected with confidence; zero delta returns NoRegression.

- [ ] **Step 3: Write FilePathConflictStrategyTest**

Test: same-file conflict detected; same-directory conflict detected for non-trivial; trivial single-file allows same-directory; no path overlap returns Clear. Tests use `CodeEvolutionMetadata.encode()` to set metadata.

- [ ] **Step 4: Write CodeEvolutionDenyPatternProviderTest**

Test: structural patterns denied; dynamic patterns denied; config denied paths (glob matching); allowed repos check; non-denied path returns false. The structural deny set must include the new infrastructure class names from the spec.

- [ ] **Step 5: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SignalConsensusProposalSourceTest,HealthScoreDeltaRegressionEvaluatorTest,FilePathConflictStrategyTest,CodeEvolutionDenyPatternProviderTest -q`
Expected: FAIL — classes not found

- [ ] **Step 6: Implement all 4 default domain implementations**

Create each following the spec's code:
- `SignalConsensusProposalSource` — spec §1.2 (injects `SignalRegistry`, `ImprovementSignalContext`)
- `HealthScoreDeltaRegressionEvaluator` — spec §1.3 (injects `ConfidenceScorer`)
- `FilePathConflictStrategy` — spec §1.4 (uses `CodeEvolutionMetadata.extractPaths()`)
- `CodeEvolutionDenyPatternProvider` — spec §1.5 (injects `DenyPatternStore`, uses `CodeEvolutionMetadata.extractPaths()` and `extractRepo()`, includes updated `STRUCTURAL_DENIED_PATTERNS` set with new infrastructure class names)

- [ ] **Step 7: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=SignalConsensusProposalSourceTest,HealthScoreDeltaRegressionEvaluatorTest,FilePathConflictStrategyTest,CodeEvolutionDenyPatternProviderTest -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#1148): add default domain implementations — proposal source, regression evaluator, conflict strategy, deny pattern provider"
```

---

## Batch 3: Backbone Rewiring

After this batch: orchestration backbone delegates to registered SPIs, old hardcoded classes deleted, full integration tests pass. Evolution conductor is domain-agnostic.

### Task 5: Refactor ImprovementGoalFormationStrategy into coordinator

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategy.java` — replace signal-based logic with source aggregation + domain-aware filtering
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementGoalFormationStrategyTest.java` — inject registry mocks instead of SignalRegistry
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementConfig.java` via `api` — remove `effectiveEnabledCategories()`

**Interfaces:**
- Consumes: `ImprovementProposalSourceRegistry`, `ImprovementCategoryRegistry`, `ConflictStrategyRegistry`, `DenyPatternProviderRegistry` from Task 3; `ImprovementBudgetEnforcer`, `ImprovementCategoryTracker`, `RollbackHistory` (existing)
- Produces: Coordinator that collects from all sources, applies 6-stage filtering (category → suppression → anti-oscillation → deny → budget → conflict), builds `GoalFormationProposal` with `ProposalFilteringSummary`

- [ ] **Step 1: Write test for coordinator with multiple sources**

Test that proposals from 2+ sources are collected, filtered by enabled categories, and domain-aware deny/conflict checks apply:

```java
@Test
void collectsFromMultipleSourcesAndFilters() {
  var source1 = mockSource("s1", "code-evolution", List.of(codeEvolutionRequest));
  var source2 = mockSource("s2", "trading", List.of(tradingRequest));
  proposalSourceRegistry.register(source1);
  proposalSourceRegistry.register(source2);
  categoryRegistry.registerProvider(codeEvolutionProvider);
  categoryRegistry.registerProvider(tradingProvider);

  var proposal = strategy.proposeImprovements(caseId, "t1", config);
  assertThat(proposal.goals()).hasSize(2);
}
```

- [ ] **Step 2: Write test for domain-aware deny filtering**

```java
@Test
void denyPatternProviderFiltersByDomain() {
  var denyProvider = mock(DenyPatternProvider.class);
  when(denyProvider.domainId()).thenReturn("code-evolution");
  when(denyProvider.isDenied(any(), any(), any(), any())).thenReturn(true);
  denyPatternProviderRegistry.register(denyProvider);

  var proposal = strategy.proposeImprovements(caseId, "t1", config);
  assertThat(proposal).isNull(); // all proposals denied
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ImprovementGoalFormationStrategyTest -q`
Expected: FAIL

- [ ] **Step 4: Refactor ImprovementGoalFormationStrategy**

Replace constructor dependencies: remove `SignalRegistry`, `ImprovementSignalContext`, `ConflictDetector`. Add `ImprovementProposalSourceRegistry`, `ImprovementCategoryRegistry`, `ConflictStrategyRegistry`, `DenyPatternProviderRegistry`.

Replace `proposeImprovements()` body with the coordinator logic from spec §3.1:
1. Collect proposals from all sources
2. Filter by enabled categories (resolved from registry when config is null)
3. Suppression check (existing `categoryTracker.isSuppressed()`)
4. Anti-oscillation check (existing `rollbackHistory.wasRecentlyRolledBack()`)
5. Domain-aware deny check via `denyPatternProviderRegistry.forDomain()`
6. Budget check (existing `budgetEnforcer.check()`)
7. Domain-aware conflict check via `conflictStrategyRegistry.forDomain()`
8. Build goals with `ProposalFilteringSummary`

- [ ] **Step 5: Remove effectiveEnabledCategories() from ImprovementConfig**

Delete the `effectiveEnabledCategories()` method. The `enabledCategories()` accessor remains (nullable field).

- [ ] **Step 6: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime-core -Dtest=ImprovementGoalFormationStrategyTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat(#1148): refactor ImprovementGoalFormationStrategy into domain-aware coordinator"
```

### Task 6: Refactor RegressionDetector, HealthScoreTracker, ImprovementBudgetEnforcer + cleanup + integration test

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java` — delegate to RegressionEvaluatorRegistry, domain-filtered
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java` — `HealthSnapshot` inner record → use `HealthScoreSnapshot`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConfidenceScorer.java` — `HealthSnapshot` → `HealthScoreSnapshot`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java` — remove STRUCTURAL_DENIED_PATTERNS, deny-pattern checks, path/repo checks
- Delete: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConflictDetector.java` (use `ide_refactor_safe_delete`)
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/RegressionDetectorTest.java` — inject registry
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcerTest.java` — remove deny tests, rename maxPRSize
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConfidenceScorerTest.java` — HealthSnapshot → HealthScoreSnapshot
- Delete: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConflictDetectorTest.java` (use `ide_refactor_safe_delete`)
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/MultiDomainEvolutionTest.java`

**Interfaces:**
- Consumes: `RegressionEvaluatorRegistry`, `ImprovementCategoryRegistry` from Task 3; `HealthScoreSnapshot` from Task 1
- Produces: Domain-filtered `RegressionDetector`, domain-agnostic `ImprovementBudgetEnforcer`, `HealthScoreTracker` using `HealthScoreSnapshot`

- [ ] **Step 1: Write test for domain-filtered regression evaluation**

```java
@Test
void evaluatesOnlyMatchingDomainEvaluators() {
  var codeEval = new HealthScoreDeltaRegressionEvaluator(scorer);
  var tradingEval = mock(RegressionEvaluator.class);
  when(tradingEval.domainId()).thenReturn("trading");
  when(tradingEval.evaluate(any(), any(), any(), any()))
      .thenReturn(new RegressionVerdict.Detected(0.9, "sharpe dropped"));
  regressionEvaluatorRegistry.register(codeEval);
  regressionEvaluatorRegistry.register(tradingEval);
  categoryRegistry.registerProvider(codeEvolutionProvider);

  // Monitor a code-evolution improvement
  detector.onOutcome(caseId, codeEvolutionOutcome);
  detector.checkActiveMonitors(caseId, healthTracker, rollbackPolicy);

  // Trading evaluator should NOT be called for code-evolution improvement
  verify(tradingEval, never()).evaluate(any(), any(), any(), any());
}
```

- [ ] **Step 2: Write MultiDomainEvolutionTest**

Integration test with two domains registered. Verify:
- Categories from both domains discovered
- Proposals from both sources collected
- Domain-specific deny and conflict strategies apply per-domain
- Regression evaluator is domain-filtered

- [ ] **Step 3: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=RegressionDetectorTest,MultiDomainEvolutionTest -q`
Expected: FAIL

- [ ] **Step 4: Refactor RegressionDetector**

Add `RegressionEvaluatorRegistry` and `ImprovementCategoryRegistry` to constructor. Replace `onMetricsDegraded()` body with domain-filtered evaluator aggregation from spec §3.2. Remove `ConfidenceScorer` direct dependency (it's now used by `HealthScoreDeltaRegressionEvaluator`).

- [ ] **Step 5: Migrate HealthScoreTracker.HealthSnapshot → HealthScoreSnapshot**

Delete the inner `HealthSnapshot` record. Replace all usages with `HealthScoreSnapshot` from the api module. Update `refresh()`, `latestSnapshot()`, `delta()`, `computeScore()` return/parameter types.

- [ ] **Step 6: Migrate ConfidenceScorer — HealthSnapshot → HealthScoreSnapshot**

Update `score()` parameters from `HealthScoreTracker.HealthSnapshot` to `HealthScoreSnapshot`. Update `regressionWithinWindow()` and `multipleAreasDegraded()` accordingly.

- [ ] **Step 7: Refactor ImprovementBudgetEnforcer — remove domain-specific logic**

Remove: `STRUCTURAL_DENIED_PATTERNS` set, `isDenied()` method, `dynamicDenyPatterns()` method, `staticDenyPatterns()` method, deny-pattern/path/repo checks from `check()` method. Keep: concurrent limit, daily limit, cooldown, max change size. Rename `maxPRSize` references to `maxChangeSize` in `check()`.

- [ ] **Step 8: Delete ConflictDetector and ConflictDetectorTest**

Use `ide_refactor_safe_delete` on both files. If references remain in `ImprovementGoalFormationStrategy` (should have been removed in Task 5), update them first.

- [ ] **Step 9: Update remaining test files**

- `RegressionDetectorTest.java` — inject `RegressionEvaluatorRegistry` and `ImprovementCategoryRegistry`
- `ImprovementBudgetEnforcerTest.java` — remove deny-pattern tests, update `maxPRSize` → `maxChangeSize`
- `ConfidenceScorerTest.java` — `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot`

- [ ] **Step 10: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime-core -q`
Expected: All tests PASS

- [ ] **Step 11: Run full project test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: All tests PASS (verifying no cross-module breakage)

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#1148): complete backbone rewiring — domain-filtered regression, budget cleanup, conflict extraction, multi-domain integration test

Closes #1148"
```

## References

- `2026-09-23-generalise-evolution-conductor-design.md` — design spec this plan implements
- `CapabilityArea.java:21` — existing multi-instance SPI pattern
- `CapabilityAreaRegistry.java` — registry pattern followed
- `CapabilityAreaBootstrap.java` — CDI discovery pattern followed
- `ImprovementConfig.java:94-98` — hardcoded categories being extracted
- `ImprovementGoalFormationStrategy.java:83-156` — proposal logic being refactored
- `RegressionDetector.java:87-139` — regression logic being extracted
- `HealthScoreTracker.java:37-38` — HealthSnapshot being relocated
- `ImprovementStage.java:20-39` — enum being migrated to strings
- `GatePolicy.java:21-50` — Map key type migrating
- `ConflictDetector.java:25-67` — conflict logic being extracted
- `ImprovementBudgetEnforcer.java:34-217` — deny logic being extracted
- `ConfidenceScorer.java:22-57` — type migration
- `TickTrace.java:54-62` — SignalFilteringSummary being replaced
- PP-20260921-b7c277 — no @DefaultBean on multi-instance SPIs
- casehubio/engine#1148 — focal issue
- casehubio/engine#1139 — epic
- D1–D8 in `decisions.md`
