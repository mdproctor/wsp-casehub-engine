# Generalise Evolution Conductor — Design Spec

**Issue:** casehubio/engine#1148
**Epic:** casehubio/engine#1139
**Parent spec:** `2026-09-21-command-centre-conductor-design.md` (#1132)
**Decisions:** D1–D8 in `decisions.md`
**Date:** 2026-09-23

## Problem

The evolution conductor (EvolutionTicker, HealthScoreTracker, RegressionDetector, ImprovementGoalFormationStrategy) is domain-agnostic in its orchestration but hardcodes code-evolution assumptions:

| Hardcoding | Location | What it assumes |
|------------|----------|----------------|
| Category strings | `ImprovementConfig.effectiveEnabledCategories()` | `dependency-update`, `lint-fix`, `coverage-gap`, `ci-triage`, `recipe` |
| Proposal source | `ImprovementGoalFormationStrategy.proposeImprovements()` | Signal-consensus from `SignalRegistry` is the only proposal mechanism |
| Regression meaning | `RegressionDetector.onMetricsDegraded()` | Health score delta against a single threshold defines regression |
| Lifecycle stages | `ImprovementStage` enum | 11 code-evolution stages (INTROSPECT through OUTCOME_RECORDING) |

The orchestration backbone (ticker, circuit breaker, budget enforcer, conflict detector) is already domain-agnostic. Health scoring via `CapabilityArea` is already a multi-instance SPI (D1). Execution via workers/case lifecycle is already pluggable (D6). Gate policies are already configurable (verified generic).

**Scope:** Extract 3 new SPIs, migrate `ImprovementStage` from enum to string, and refactor `ImprovementGoalFormationStrategy` into a coordinator. Code-evolution implementations stay in runtime-core as defaults (D5).

## Design Overview

Three new SPIs replace hardcoded domain assumptions. The orchestration backbone stays concrete, delegating to pluggable components:

```
                          ┌─────────────────────────┐
                          │    EvolutionTicker       │
                          │    (orchestrator)        │
                          └──────────┬──────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
  ┌──────────────────┐   ┌────────────────────────┐   ┌──────────────────┐
  │ HealthScoreTracker│   │ ImprovementGoalFormation│  │ RegressionDetector│
  │                  │   │ Strategy (coordinator)  │   │ (orchestrator)   │
  └──────┬───────────┘   └──────────┬─────────────┘   └──────┬───────────┘
         │                          │                         │
         ▼                          ▼                         ▼
  ┌──────────────┐        ┌──────────────────┐       ┌──────────────────┐
  │CapabilityArea│        │ImprovementProposal│      │RegressionEvaluator│
  │  (existing   │        │  Source (NEW SPI) │      │    (NEW SPI)      │
  │   SPI, D1)   │        └──────────────────┘       └──────────────────┘
  └──────────────┘
                          ┌──────────────────────┐
                          │ImprovementCategory    │
                          │ Provider (NEW SPI)    │
                          │ contributes categories│
                          │ + stage definitions   │
                          └──────────────────────┘
```

## 1. New SPI Interfaces

All interfaces in `api/src/main/java/io/casehub/api/spi/improvement/`.

### 1.1 ImprovementCategoryProvider

Multi-instance SPI. Each domain registers a provider contributing categories and stage definitions.

```java
// api, io.casehub.api.spi.improvement
public interface ImprovementCategoryProvider {

  String domainId();

  List<CategoryDescriptor> categories();

  List<StageDescriptor> stages();
}
```

```java
// api, io.casehub.api.model.stigmergy
public record CategoryDescriptor(
    String id,
    String name,
    String description,
    String domainId) {}
```

```java
// api, io.casehub.api.model.stigmergy
public record StageDescriptor(
    String id,
    String name,
    int ordinal,
    boolean gateCheckpoint,
    String domainId) {}
```

**Code-evolution default provider (runtime-core):**

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class CodeEvolutionCategoryProvider implements ImprovementCategoryProvider {

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public List<CategoryDescriptor> categories() {
    return List.of(
        new CategoryDescriptor("dependency-update", "Dependency Update",
            "Bump outdated dependencies", "code-evolution"),
        new CategoryDescriptor("lint-fix", "Lint Fix",
            "Fix linting and style violations", "code-evolution"),
        new CategoryDescriptor("coverage-gap", "Coverage Gap",
            "Add tests for uncovered code", "code-evolution"),
        new CategoryDescriptor("ci-triage", "CI Triage",
            "Fix CI pipeline failures", "code-evolution"),
        new CategoryDescriptor("recipe", "Recipe",
            "Apply automated code transformation recipes", "code-evolution"));
  }

  @Override
  public List<StageDescriptor> stages() {
    return List.of(
        new StageDescriptor("introspect", "Introspect", 0, false, "code-evolution"),
        new StageDescriptor("research-scope", "Research Scope", 1, true, "code-evolution"),
        new StageDescriptor("search", "Search", 2, false, "code-evolution"),
        new StageDescriptor("analyze", "Analyze", 3, false, "code-evolution"),
        new StageDescriptor("hypothesis-approval", "Hypothesis Approval", 4, true, "code-evolution"),
        new StageDescriptor("implementation-plan", "Implementation Plan", 5, true, "code-evolution"),
        new StageDescriptor("implement", "Implement", 6, false, "code-evolution"),
        new StageDescriptor("submit-pr", "Submit PR", 7, false, "code-evolution"),
        new StageDescriptor("pr-review", "PR Review", 8, true, "code-evolution"),
        new StageDescriptor("integrate", "Integrate", 9, false, "code-evolution"),
        new StageDescriptor("outcome-recording", "Outcome Recording", 10, false, "code-evolution"));
  }
}
```

### 1.2 ImprovementProposalSource

Multi-instance SPI. Each domain registers sources that generate improvement proposals.

```java
// api, io.casehub.api.spi.improvement
public interface ImprovementProposalSource {

  String sourceId();

  String domainId();

  List<ImprovementRequest> propose(UUID caseId, String tenancyId, ImprovementConfig config);
}
```

Sources return raw `ImprovementRequest` lists. The coordinator (`ImprovementGoalFormationStrategy`) applies shared filtering — budget, conflict, suppression, anti-oscillation — to all proposals regardless of source (D7).

**Code-evolution default source (runtime-core):**

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class SignalConsensusProposalSource implements ImprovementProposalSource {

  private final SignalRegistry signalRegistry;
  private final ImprovementSignalContext signalContext;

  @Override
  public String sourceId() { return "signal-consensus"; }

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public List<ImprovementRequest> propose(
      UUID caseId, String tenancyId, ImprovementConfig config) {
    String namespace = config.effectiveSignalNamespace();
    int minSources = config.effectiveConsensusMinSources();
    var consensus = signalRegistry.consensusSignals(caseId, minSources, 0.01);

    List<ImprovementRequest> proposals = new ArrayList<>();
    for (var entry : consensus.entrySet()) {
      String signalName = entry.getKey();
      if (!signalName.startsWith(namespace + ":")) {
        continue;
      }
      signalContext.get(caseId, signalName).ifPresent(proposals::add);
    }
    return proposals;
  }
}
```

### 1.3 RegressionEvaluator

Multi-instance SPI. Each domain defines what "regression" means for its health metrics.

```java
// api, io.casehub.api.spi.improvement
public interface RegressionEvaluator {

  String evaluatorId();

  String domainId();

  RegressionVerdict evaluate(
      UUID caseId, String tenancyId,
      HealthScoreSnapshot baseline,
      HealthScoreSnapshot current,
      String category);
}
```

```java
// api, io.casehub.api.model.stigmergy
public sealed interface RegressionVerdict
    permits RegressionVerdict.NoRegression, RegressionVerdict.Detected {

  record NoRegression() implements RegressionVerdict {}

  record Detected(double confidence, String reason) implements RegressionVerdict {}
}
```

```java
// api, io.casehub.api.model.stigmergy
public record HealthScoreSnapshot(
    double score, Instant timestamp, Map<String, Double> componentScores) {}
```

`HealthScoreSnapshot` is the API-module equivalent of `HealthScoreTracker.HealthSnapshot`. The existing inner record moves to the api module as a top-level record to avoid leaking runtime-core types into the SPI interface.

**Code-evolution default evaluator (runtime-core):**

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class HealthScoreDeltaRegressionEvaluator implements RegressionEvaluator {

  @Override
  public String evaluatorId() { return "health-score-delta"; }

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public RegressionVerdict evaluate(
      UUID caseId, String tenancyId,
      HealthScoreSnapshot baseline, HealthScoreSnapshot current,
      String category) {
    double delta = current.score() - baseline.score();
    if (delta < -0.1) {
      return new RegressionVerdict.Detected(
          Math.min(1.0, Math.abs(delta)),
          "Health score dropped by " + String.format("%.2f", Math.abs(delta)));
    }
    return new RegressionVerdict.NoRegression();
  }
}
```

## 2. Registries and Bootstrap

Following the established `CapabilityAreaRegistry` / `CapabilityAreaBootstrap` pattern (CDI discovery via `Instance<T>` at startup → register into `ConcurrentHashMap`-backed registry). Per protocol PP-20260921-b7c277: NO `@DefaultBean` on multi-instance SPIs.

### ImprovementCategoryRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementCategoryRegistry implements Resettable {

  private final ConcurrentHashMap<String, CategoryDescriptor> categories =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<String, StageDescriptor> stages =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<String, List<StageDescriptor>> stagesByDomain =
      new ConcurrentHashMap<>();

  public void registerProvider(ImprovementCategoryProvider provider) {
    for (var cat : provider.categories()) {
      categories.put(cat.id(), cat);
    }
    var sortedStages = provider.stages().stream()
        .sorted(Comparator.comparingInt(StageDescriptor::ordinal))
        .toList();
    stagesByDomain.put(provider.domainId(), sortedStages);
    for (var stage : sortedStages) {
      stages.put(stage.id(), stage);
    }
  }

  public List<CategoryDescriptor> allCategories() {
    return List.copyOf(categories.values());
  }

  public Optional<CategoryDescriptor> getCategory(String categoryId) {
    return Optional.ofNullable(categories.get(categoryId));
  }

  public boolean isGateCheckpoint(String stageId) {
    var stage = stages.get(stageId);
    return stage != null && stage.gateCheckpoint();
  }

  public List<StageDescriptor> stagesForDomain(String domainId) {
    return stagesByDomain.getOrDefault(domainId, List.of());
  }

  public Optional<String> domainForCategory(String categoryId) {
    var cat = categories.get(categoryId);
    return cat != null ? Optional.of(cat.domainId()) : Optional.empty();
  }

  @Override
  public void reset() {
    categories.clear();
    stages.clear();
    stagesByDomain.clear();
  }
}
```

### ImprovementProposalSourceRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementProposalSourceRegistry implements Resettable {

  private final List<ImprovementProposalSource> sources = new CopyOnWriteArrayList<>();

  public void register(ImprovementProposalSource source) {
    sources.add(source);
  }

  public List<ImprovementProposalSource> all() {
    return List.copyOf(sources);
  }

  @Override
  public void reset() {
    sources.clear();
  }
}
```

### RegressionEvaluatorRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class RegressionEvaluatorRegistry implements Resettable {

  private final List<RegressionEvaluator> evaluators = new CopyOnWriteArrayList<>();

  public void register(RegressionEvaluator evaluator) {
    evaluators.add(evaluator);
  }

  public List<RegressionEvaluator> all() {
    return List.copyOf(evaluators);
  }

  @Override
  public void reset() {
    evaluators.clear();
  }
}
```

### EvolutionBootstrap (unified)

A single bootstrap class that discovers and registers all evolution SPIs:

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class EvolutionBootstrap {

  @Inject ImprovementCategoryRegistry categoryRegistry;
  @Inject ImprovementProposalSourceRegistry proposalSourceRegistry;
  @Inject RegressionEvaluatorRegistry regressionEvaluatorRegistry;

  @Inject @Any Instance<ImprovementCategoryProvider> categoryProviders;
  @Inject @Any Instance<ImprovementProposalSource> proposalSources;
  @Inject @Any Instance<RegressionEvaluator> regressionEvaluators;

  void onStartup(@Observes StartupEvent event) {
    for (var provider : categoryProviders) {
      categoryRegistry.registerProvider(provider);
    }
    for (var source : proposalSources) {
      proposalSourceRegistry.register(source);
    }
    for (var evaluator : regressionEvaluators) {
      regressionEvaluatorRegistry.register(evaluator);
    }
  }
}
```

`CapabilityAreaBootstrap` remains separate — it already exists and works. No need to merge into `EvolutionBootstrap`.

## 3. Changes to Orchestration Backbone

### 3.1 ImprovementGoalFormationStrategy → coordinator

The `proposeImprovements()` method changes from reading signals directly to collecting proposals from all registered sources:

**Before:**
```java
public GoalFormationProposal proposeImprovements(UUID caseId, String tenancyId, ImprovementConfig config) {
  var consensus = signalRegistry.consensusSignals(caseId, minSources, 0.01);
  // filter by namespace, category, suppression, rollback, budget, conflict
  // build goals
}
```

**After:**
```java
public GoalFormationProposal proposeImprovements(UUID caseId, String tenancyId, ImprovementConfig config) {
  // 1. Collect from all sources
  List<ImprovementRequest> allProposals = new ArrayList<>();
  for (var source : proposalSourceRegistry.all()) {
    allProposals.addAll(source.propose(caseId, tenancyId, config));
  }

  // 2. Filter by enabled categories (ImprovementConfig)
  var enabledCategories = effectiveEnabledCategories(config);
  // ... existing filtering pipeline unchanged (suppression, rollback, budget, conflict)

  // 3. Build goals
}
```

The `effectiveEnabledCategories()` resolution changes:

```java
private Set<String> effectiveEnabledCategories(ImprovementConfig config) {
  if (config.enabledCategories() != null) {
    return Set.copyOf(config.enabledCategories());
  }
  return categoryRegistry.allCategories().stream()
      .map(CategoryDescriptor::id)
      .collect(Collectors.toSet());
}
```

When `ImprovementConfig.enabledCategories` is null, ALL categories from all providers are enabled. When specified, it acts as a filter.

**Constructor changes:** Remove `SignalRegistry` and `ImprovementSignalContext` (moved to `SignalConsensusProposalSource`). Add `ImprovementProposalSourceRegistry` and `ImprovementCategoryRegistry`.

### 3.2 RegressionDetector → delegates to evaluators

The `onMetricsDegraded()` method changes from inline health-score comparison to calling registered evaluators:

**Before:**
```java
private void onMetricsDegraded(UUID caseId, MonitoredImprovement monitor,
    RollbackPolicy policy, HealthSnapshot before, HealthSnapshot after) {
  // inline health score comparison
  double delta = after.score() - before.score();
  if (delta < -policy.effectiveRegressionThreshold()) { ... }
}
```

**After:**
```java
private void onMetricsDegraded(UUID caseId, MonitoredImprovement monitor,
    RollbackPolicy policy, HealthScoreSnapshot before, HealthScoreSnapshot after) {
  for (var evaluator : regressionEvaluatorRegistry.all()) {
    var verdict = evaluator.evaluate(
        caseId, monitor.tenancyId(), before, after, monitor.category());
    if (verdict instanceof RegressionVerdict.Detected detected) {
      // existing regression handling: fire event, record in rollback history
      handleRegression(caseId, monitor, detected);
      return;
    }
  }
}
```

**Constructor changes:** Add `RegressionEvaluatorRegistry`.

### 3.3 HealthScoreTracker.HealthSnapshot → HealthScoreSnapshot

`HealthScoreTracker.HealthSnapshot` (inner record) moves to `api` module as `HealthScoreSnapshot` (top-level record). This avoids leaking runtime-core types into SPI interfaces.

`HealthScoreTracker` continues to use `HealthScoreSnapshot` internally — the move is a type relocation, not a redesign.

## 4. ImprovementStage Migration

`ImprovementStage` changes from an enum to a `String`. The `ImprovementStage` enum class is deleted. All references change to `String`.

### Affected production types

| Type | Field/Method | Change |
|------|-------------|--------|
| `GatePolicy` | `Map<ImprovementStage, GateMode> modes` | → `Map<String, GateMode> modes` |
| `GatePolicy` | `effectiveMode(ImprovementStage)` | → `effectiveMode(String stageId)` |
| `GatePolicy` | constructor validation (`isGateCheckpoint()`) | → validation via `ImprovementCategoryRegistry.isGateCheckpoint(stageId)`. Deferred to runtime — the record constructor no longer validates because it has no registry reference. Validation moves to `DefaultEngineEvolutionApi.setGatePolicy()`. |
| `ConductorInboxEntry` | `ImprovementStage stage` | → `String stage` |
| `ArtifactEntry` | `ImprovementStage stage` | → `String stage` |
| `ResearchPipelineResult.AwaitingGate` | `ImprovementStage stage` | → `String stage` |
| `EscalationProvider` | `evaluate(..., ImprovementStage stage, ...)` | → `evaluate(..., String stage, ...)` |
| `EvolutionStateSnapshot.ImprovementStreamView` | `ImprovementStage currentStage` | → `String currentStage` |
| `EvolutionStateSnapshot.StageProgress` | `ImprovementStage stage` | → `String stage` |
| `ResearchPipelineOrchestrator` | Stage references (`ImprovementStage.RESEARCH_SCOPE`) | → String constants from `CodeEvolutionStages` |

### Well-known stage constants

```java
// api, io.casehub.api.model.stigmergy
public final class ImprovementStages {
  // Code-evolution stages (backward compat)
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

  private ImprovementStages() {}
}
```

### GatePolicy validation migration

`GatePolicy`'s constructor currently validates that only gate checkpoints appear in the `modes` map. This validation used `ImprovementStage.isGateCheckpoint()` which is available at construction time for an enum. With strings, the validation needs access to the `ImprovementCategoryRegistry` which is a runtime bean.

**Resolution:** Remove the constructor validation from `GatePolicy`. Move validation to `DefaultEngineEvolutionApi.setGatePolicy()` and `InMemoryGatePolicyStore.store()` where the registry is available. `GatePolicy` becomes a pure data record.

The `effectiveMode()` default also changes — the code-evolution-specific "PR_REVIEW defaults to GATED" moves to a per-domain default. `GatePolicy.effectiveMode()` returns `GateMode.AUTO` for any unknown stage. The code-evolution provider can set its own defaults:

```java
public GateMode effectiveMode(String stageId) {
  if (modes != null && modes.containsKey(stageId)) {
    return modes.get(stageId);
  }
  return GateMode.AUTO;
}
```

The "PR_REVIEW defaults to GATED" semantic moves to the `CodeEvolutionCategoryProvider`'s stage definition or to the initial `ImprovementConfig` that a code-evolution case starts with.

### YAML codegen impact

`YamlGatePolicy` (generated) currently has `Map<ImprovementStage, GateMode> modes`. After migration, this becomes `Map<String, GateMode> modes`. The codegen entry in `yaml-record-mappings.yaml` needs updating to reflect the key type change from enum to String.

## 5. ImprovementConfig Changes

### effectiveEnabledCategories()

The hardcoded default list is removed. When `enabledCategories` is null, ALL categories from all providers are enabled (resolved at runtime via the registry):

**Before:**
```java
public List<String> effectiveEnabledCategories() {
  return enabledCategories != null
      ? enabledCategories
      : List.of("dependency-update", "lint-fix", "coverage-gap", "ci-triage", "recipe");
}
```

**After:** `effectiveEnabledCategories()` is removed from `ImprovementConfig`. The resolution logic moves to `ImprovementGoalFormationStrategy` which has access to the `ImprovementCategoryRegistry`. `ImprovementConfig.enabledCategories()` remains as a nullable field — when non-null, it filters; when null, all registered categories are active.

## 6. Test Strategy

### New unit tests

| Test class | What it covers |
|------------|---------------|
| `ImprovementCategoryRegistryTest` | Provider registration, category lookup, stage lookup, domain resolution, gate checkpoint check, multi-provider coexistence, reset |
| `ImprovementProposalSourceRegistryTest` | Source registration, retrieval, reset |
| `RegressionEvaluatorRegistryTest` | Evaluator registration, retrieval, reset |
| `CodeEvolutionCategoryProviderTest` | Default categories (5), default stages (11), stage ordering, gate checkpoints match expected set |
| `SignalConsensusProposalSourceTest` | Signal-consensus proposal generation (extracted from ImprovementGoalFormationStrategyTest), namespace filtering |
| `HealthScoreDeltaRegressionEvaluatorTest` | Delta threshold detection, no-regression case, boundary values |
| `EvolutionBootstrapTest` | Multi-provider discovery, all registries populated |

### Modified existing tests

| Test class | Change |
|------------|--------|
| `ImprovementGoalFormationStrategyTest` | Refactor: inject mock ImprovementProposalSourceRegistry instead of SignalRegistry. Test filtering pipeline with proposals from multiple sources. |
| `RegressionDetectorTest` | Refactor: inject RegressionEvaluatorRegistry. Test that evaluators are called and first Detected verdict triggers regression handling. |
| `GatePolicyTest` | Change ImprovementStage enum → String constants. Remove constructor validation tests (moved to API layer). |
| `ConductorInboxManagerTest` | Change ImprovementStage enum → String constants. |
| `DefaultEscalationProviderTest` | Change ImprovementStage enum → String constants. |
| `EvolutionApiTest` | Change ImprovementStage enum → String constants. Add gate policy validation tests (moved from GatePolicy constructor). |
| `ResearchPipelineCheckpointTest` | Change ImprovementStage enum → String constants. |
| `TickTraceTest` | Remove `improvementStageGateCheckpoints` test (gate checkpoint logic moves to registry). |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `MultiDomainEvolutionTest` | Two domains registered (code-evolution + trading-stub), categories from both discovered, proposals from both sources collected, filtering applies to all. Trading regression evaluator detects trading-specific regression while code-evolution evaluator sees no regression — correct routing. |
| `CategoryFilteringIntegrationTest` | ImprovementConfig.enabledCategories filters across domain providers. Null enables all. Explicit list filters to subset. |

### Critical test scenarios

1. **Multi-provider coexistence:** Two category providers register — both domains' categories appear in registry, no collision on category IDs (different domains can have same-named categories, distinguished by domainId).
2. **Proposal source aggregation:** Three sources return proposals — coordinator collects all, applies shared filtering pipeline, generates combined goal set.
3. **Evaluator aggregation (any-triggered):** Two evaluators registered — first returns NoRegression, second returns Detected — regression IS triggered (any-trigger semantics).
4. **Category filtering as intersection:** Config.enabledCategories = ["dependency-update", "parameter-tuning"], provider A has "dependency-update", provider B has "parameter-tuning" — both active. Config has "nonexistent-category" — silently ignored (no error, just no matching proposals).
5. **Stage migration backward compat:** GatePolicy with string keys "pr-review" → GateMode.GATED works identically to former enum-based policy.
6. **Gate checkpoint validation at API layer:** setGatePolicy with a non-checkpoint stage → rejected with IllegalArgumentException at the API layer (not in the record constructor).
7. **Empty registry:** No category providers registered → no categories → no proposals → EvolutionTicker tick returns NoProposal. System is inert but not broken.

## 7. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `ImprovementCategoryProvider` SPI | `api` | `io.casehub.api.spi.improvement` |
| `ImprovementProposalSource` SPI | `api` | `io.casehub.api.spi.improvement` |
| `RegressionEvaluator` SPI | `api` | `io.casehub.api.spi.improvement` |
| `CategoryDescriptor` | `api` | `io.casehub.api.model.stigmergy` |
| `StageDescriptor` | `api` | `io.casehub.api.model.stigmergy` |
| `RegressionVerdict` | `api` | `io.casehub.api.model.stigmergy` |
| `HealthScoreSnapshot` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementStages` (constants) | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementCategoryRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementProposalSourceRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `RegressionEvaluatorRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `EvolutionBootstrap` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `CodeEvolutionCategoryProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `SignalConsensusProposalSource` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `HealthScoreDeltaRegressionEvaluator` | `runtime-core` | `io.casehub.engine.internal.improvement` |

## 8. Migration Sequence

The changes have dependencies that determine ordering:

1. **API types first:** `CategoryDescriptor`, `StageDescriptor`, `RegressionVerdict`, `HealthScoreSnapshot`, `ImprovementStages` constants
2. **SPI interfaces:** `ImprovementCategoryProvider`, `ImprovementProposalSource`, `RegressionEvaluator`
3. **ImprovementStage migration:** Delete enum, update all references to `String`, update `GatePolicy`, update YAML codegen
4. **Registries:** `ImprovementCategoryRegistry`, `ImprovementProposalSourceRegistry`, `RegressionEvaluatorRegistry`
5. **Default implementations:** `CodeEvolutionCategoryProvider`, `SignalConsensusProposalSource`, `HealthScoreDeltaRegressionEvaluator`
6. **Bootstrap:** `EvolutionBootstrap`
7. **Backbone refactoring:** `ImprovementGoalFormationStrategy` (coordinator), `RegressionDetector` (delegation), `HealthScoreTracker` (HealthSnapshot → HealthScoreSnapshot)
8. **ImprovementConfig:** Remove `effectiveEnabledCategories()` hardcoded default

## References

- `CapabilityArea.java:21` — existing multi-instance SPI pattern
- `CapabilityAreaRegistry.java` — registry pattern to follow
- `CapabilityAreaBootstrap.java` — CDI discovery pattern to follow
- `ImprovementConfig.java:94-98` — hardcoded categories to extract
- `ImprovementGoalFormationStrategy.java:83-156` — proposal logic to refactor
- `RegressionDetector.java:87-139` — regression logic to extract
- `HealthScoreTracker.java:37-38` — HealthSnapshot to relocate
- `ImprovementStage.java:20-39` — enum to migrate to strings
- `GatePolicy.java:21-50` — Map<ImprovementStage, GateMode> to migrate
- `EvolutionTicker.java:36-131` — orchestration backbone (unchanged)
- PP-20260921-b7c277 — never use @DefaultBean on multi-instance SPIs
- PP-20260915-aa504e — API surface parity (YAML, DSL, annotations)
- Decisions D1–D8 in `decisions.md`
- Issue casehubio/engine#1148
- Parent spec: `2026-09-21-command-centre-conductor-design.md`
