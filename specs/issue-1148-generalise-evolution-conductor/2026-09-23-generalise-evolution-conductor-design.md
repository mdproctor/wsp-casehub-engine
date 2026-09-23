# Generalise Evolution Conductor — Design Spec

**Issue:** casehubio/engine#1148
**Epic:** casehubio/engine#1149
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

**Scope:** Extract 5 new SPIs, generalise `ImprovementRequest` target model, migrate `ImprovementStage` from enum to string, and refactor `ImprovementGoalFormationStrategy` into a coordinator. Code-evolution implementations stay in runtime-core as defaults (D5). The 5 SPIs cover the three orchestration concerns (categories, proposals, regression evaluation) plus the two filtering-pipeline concerns (conflict detection, structural deny patterns) identified in the issue's predecessor audit (#1148 Comment 1).

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

                  ┌────────────────┐   ┌──────────────────┐
                  │ConflictStrategy│   │DenyPatternProvider│
                  │  (NEW SPI)     │   │   (NEW SPI)       │
                  │ domain-aware   │   │ domain-specific   │
                  │ conflict check │   │ structural guards │
                  └────────────────┘   └──────────────────┘
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

Sources return raw `ImprovementRequest` lists. The coordinator (`ImprovementGoalFormationStrategy`) applies shared filtering — budget, conflict, suppression, anti-oscillation — to all proposals regardless of source (D7). Domain-specific filtering (deny patterns, conflict detection) is delegated to the corresponding domain's `DenyPatternProvider` and `ConflictStrategy`.

#### ImprovementRequest generalisation

`ImprovementRequest` is currently code-evolution-shaped (`targetRepo`, `targetPaths`, `estimatedSize`). These fields have no meaning for trading, AML, or clinical domains. The record becomes domain-agnostic:

```java
// api, io.casehub.api.model.stigmergy
public record ImprovementRequest(
    String improvementType,
    String category,
    String target,
    String domainId,
    int estimatedSize,
    Map<String, String> metadata) {}
```

- `targetRepo` and `targetPaths` removed from the record — code-evolution stores these in `metadata` (keys: `"target-repo"`, `"target-paths"` as comma-separated)
- `domainId` added — each proposal identifies its domain
- `estimatedSize` retained — all domains have a notion of change magnitude (line count for code, parameter count for trading, rule count for AML)
- `metadata` retained — domain-specific data that doesn't warrant dedicated fields

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
      UUID caseId,
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

Wraps the existing `ConfidenceScorer` which computes regression confidence from before/after snapshots. The evaluator produces a confidence score; the `RegressionDetector` applies the two-tier threshold policy (auto-revert vs pause) from `RollbackPolicy`.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class HealthScoreDeltaRegressionEvaluator implements RegressionEvaluator {

  private final ConfidenceScorer scorer;

  @Override
  public String evaluatorId() { return "health-score-delta"; }

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public RegressionVerdict evaluate(
      UUID caseId,
      HealthScoreSnapshot baseline, HealthScoreSnapshot current,
      String category) {
    double confidence = scorer.score(caseId, null, baseline, current);
    if (confidence > 0.0) {
      return new RegressionVerdict.Detected(confidence,
          "Health score regression detected");
    }
    return new RegressionVerdict.NoRegression();
  }
}
```

### 1.4 ConflictStrategy

Multi-instance SPI. Each domain defines what "conflict" means for concurrent improvements.

```java
// api, io.casehub.api.spi.improvement
public interface ConflictStrategy {

  String domainId();

  ConflictResult check(
      ImprovementRequest request,
      Map<UUID, ImprovementRequest> activeImprovements,
      int trivialThreshold);

  sealed interface ConflictResult
      permits ConflictResult.Clear, ConflictResult.Conflicting {

    record Clear() implements ConflictResult {}

    record Conflicting(UUID blockingImprovementId, String reason)
        implements ConflictResult {}
  }
}
```

**Code-evolution default (runtime-core):** Moves the existing `ConflictDetector` file-path logic. Extracts `target-paths` from `ImprovementRequest.metadata()` and checks for same-file or same-directory overlap between the request and active improvements.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class FilePathConflictStrategy implements ConflictStrategy {

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public ConflictResult check(
      ImprovementRequest request,
      Map<UUID, ImprovementRequest> activeImprovements,
      int trivialThreshold) {
    List<String> requestPaths = extractPaths(request);
    boolean isTrivial =
        request.estimatedSize() <= trivialThreshold && requestPaths.size() == 1;

    for (var entry : activeImprovements.entrySet()) {
      List<String> activePaths = extractPaths(entry.getValue());
      for (String rp : requestPaths) {
        for (String ap : activePaths) {
          if (rp.equals(ap)) {
            return new ConflictResult.Conflicting(entry.getKey(), rp);
          }
          if (!isTrivial && sameDirectory(rp, ap)) {
            return new ConflictResult.Conflicting(entry.getKey(), rp);
          }
        }
      }
    }
    return new ConflictResult.Clear();
  }

  private List<String> extractPaths(ImprovementRequest request) {
    String paths = request.metadata().getOrDefault("target-paths", "");
    return paths.isEmpty() ? List.of() : List.of(paths.split(","));
  }

  private boolean sameDirectory(String a, String b) {
    return parentDir(a).equals(parentDir(b));
  }

  private String parentDir(String path) {
    int last = path.lastIndexOf('/');
    return last > 0 ? path.substring(0, last) : "";
  }
}
```

The existing `ConflictDetector` is deleted. Its file-path logic moves to `FilePathConflictStrategy`. The `ConflictDetector.ConflictCheck` sealed interface is superseded by `ConflictStrategy.ConflictResult`.

### 1.5 DenyPatternProvider

Multi-instance SPI. Each domain defines structural invariants that must never be self-modified by the improvement loop.

```java
// api, io.casehub.api.spi.improvement
public interface DenyPatternProvider {

  String domainId();

  boolean isDenied(UUID caseId, String tenancyId, ImprovementRequest request);
}
```

**Code-evolution default (runtime-core):** Moves the existing `STRUCTURAL_DENIED_PATTERNS` set from `ImprovementBudgetEnforcer`. Also checks dynamic deny patterns via `DenyPatternStore` and config-driven denied paths and allowed repos from `ImprovementBudget`.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class CodeEvolutionDenyPatternProvider implements DenyPatternProvider {

  private static final Set<String> STRUCTURAL_DENIED_PATTERNS =
      Set.of(
          "ImprovementBudget", "ImprovementBudgetEnforcer", "ImprovementConfig",
          "SafetyConfig", "improvement-case-template", "EvolutionTicker",
          "ImprovementCircuitBreaker", "RegressionDetector", "ConfidenceScorer",
          "HealthScoreTracker", "HealthPolicy", "RollbackPolicy",
          "ConflictDetector", "ImprovementCategoryTracker",
          "RollbackHistory", "self-improvement-rollback");

  private final DenyPatternStore denyPatternStore;

  @Override
  public String domainId() { return "code-evolution"; }

  @Override
  public boolean isDenied(UUID caseId, String tenancyId, ImprovementRequest request) {
    List<String> paths = extractPaths(request);
    for (String path : paths) {
      for (String pattern : STRUCTURAL_DENIED_PATTERNS) {
        if (path.contains(pattern)) return true;
      }
      for (String pattern : denyPatternStore.findAll(caseId, tenancyId)) {
        if (path.contains(pattern)) return true;
      }
    }
    return false;
  }

  private List<String> extractPaths(ImprovementRequest request) {
    String paths = request.metadata().getOrDefault("target-paths", "");
    return paths.isEmpty() ? List.of() : List.of(paths.split(","));
  }
}
```

A trading domain would provide its own `TradingDenyPatternProvider` that protects risk limit parameters, regulatory thresholds, and circuit breaker configuration from self-modification — using domain-specific matching logic, not file paths.

## 2. Registries and Bootstrap

Following the established `CapabilityAreaRegistry` / `CapabilityAreaBootstrap` pattern (CDI discovery via `Instance<T>` at startup → register into `ConcurrentHashMap`-backed registry). Per protocol PP-20260921-b7c277: NO `@DefaultBean` on multi-instance SPIs.

### ImprovementCategoryRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementCategoryRegistry implements Resettable {

  private final ConcurrentHashMap<String, CategoryDescriptor> categories =
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
  }

  public List<CategoryDescriptor> allCategories() {
    return List.copyOf(categories.values());
  }

  public Optional<CategoryDescriptor> getCategory(String categoryId) {
    return Optional.ofNullable(categories.get(categoryId));
  }

  public boolean isGateCheckpoint(String domainId, String stageId) {
    var domainStages = stagesByDomain.get(domainId);
    if (domainStages == null) return false;
    return domainStages.stream()
        .anyMatch(s -> s.id().equals(stageId) && s.gateCheckpoint());
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
    stagesByDomain.clear();
  }
}
```

Stage IDs are scoped per domain — different domains may reuse stage IDs (e.g., both "code-evolution" and "trading" can have an "analyze" stage) without collision. The flat `stages` map is removed; lookups always go through `stagesByDomain`. `isGateCheckpoint(domainId, stageId)` requires the caller to provide domain context, which is available from the case's category via `domainForCategory()`.

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

### ConflictStrategyRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ConflictStrategyRegistry implements Resettable {

  private final ConcurrentHashMap<String, ConflictStrategy> strategies =
      new ConcurrentHashMap<>();

  public void register(ConflictStrategy strategy) {
    strategies.put(strategy.domainId(), strategy);
  }

  public Optional<ConflictStrategy> forDomain(String domainId) {
    return Optional.ofNullable(strategies.get(domainId));
  }

  @Override
  public void reset() {
    strategies.clear();
  }
}
```

### DenyPatternProviderRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class DenyPatternProviderRegistry implements Resettable {

  private final ConcurrentHashMap<String, DenyPatternProvider> providers =
      new ConcurrentHashMap<>();

  public void register(DenyPatternProvider provider) {
    providers.put(provider.domainId(), provider);
  }

  public Optional<DenyPatternProvider> forDomain(String domainId) {
    return Optional.ofNullable(providers.get(domainId));
  }

  @Override
  public void reset() {
    providers.clear();
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
  @Inject ConflictStrategyRegistry conflictStrategyRegistry;
  @Inject DenyPatternProviderRegistry denyPatternProviderRegistry;

  @Inject @Any Instance<ImprovementCategoryProvider> categoryProviders;
  @Inject @Any Instance<ImprovementProposalSource> proposalSources;
  @Inject @Any Instance<RegressionEvaluator> regressionEvaluators;
  @Inject @Any Instance<ConflictStrategy> conflictStrategies;
  @Inject @Any Instance<DenyPatternProvider> denyPatternProviders;

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
    for (var strategy : conflictStrategies) {
      conflictStrategyRegistry.register(strategy);
    }
    for (var provider : denyPatternProviders) {
      denyPatternProviderRegistry.register(provider);
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

  // 3. Domain-agnostic filtering (suppression, anti-oscillation, budget)
  // ... existing suppression, rollback history, budget checks unchanged

  // 4. Domain-aware deny pattern check
  for (var proposal : filtered) {
    var provider = denyPatternProviderRegistry.forDomain(proposal.domainId());
    if (provider.isPresent() && provider.get().isDenied(caseId, tenancyId, proposal)) {
      continue; // denied by domain safety rules
    }
  }

  // 5. Domain-aware conflict check
  for (var proposal : afterDenyFilter) {
    var strategy = conflictStrategyRegistry.forDomain(proposal.domainId());
    if (strategy.isPresent()) {
      var check = strategy.get().check(
          proposal, budgetEnforcer.activeImprovementRequests(),
          config.effectiveConflictTrivialThreshold());
      if (check instanceof ConflictStrategy.ConflictResult.Conflicting) {
        continue; // conflict with active improvement
      }
    }
  }

  // 6. Build goals
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

**Constructor changes:** Remove `SignalRegistry`, `ImprovementSignalContext`, and `ConflictDetector` (moved to `SignalConsensusProposalSource` and `FilePathConflictStrategy`). Add `ImprovementProposalSourceRegistry`, `ImprovementCategoryRegistry`, `ConflictStrategyRegistry`, and `DenyPatternProviderRegistry`.

### 3.2 RegressionDetector → delegates to evaluators

The `onMetricsDegraded()` method changes from using a single `ConfidenceScorer` to calling all registered evaluators. The two-tier threshold response (auto-revert vs pause) from `RollbackPolicy` stays in the detector — evaluators produce confidence, the detector decides the response.

**Before:**
```java
private void onMetricsDegraded(UUID caseId, MonitoredImprovement monitor,
    RollbackPolicy policy, HealthSnapshot before, HealthSnapshot after) {
  double confidence = scorer.score(caseId, monitor.improvementCaseId(), before, after);
  regressionDetectedEvent.fireAsync(...);
  if (confidence >= policy.effectiveAutoRevertThreshold()) { /* revert + pause */ }
  else if (confidence >= policy.effectivePauseThreshold()) { /* pause only */ }
}
```

**After:**
```java
private void onMetricsDegraded(UUID caseId, MonitoredImprovement monitor,
    RollbackPolicy policy, HealthScoreSnapshot before, HealthScoreSnapshot after) {
  String domainId = categoryRegistry.domainForCategory(monitor.category())
      .orElse("unknown");
  double maxConfidence = 0.0;
  String reason = null;
  for (var evaluator : regressionEvaluatorRegistry.all()) {
    if (!evaluator.domainId().equals(domainId)) continue;
    var verdict = evaluator.evaluate(caseId, before, after, monitor.category());
    if (verdict instanceof RegressionVerdict.Detected detected
        && detected.confidence() > maxConfidence) {
      maxConfidence = detected.confidence();
      reason = detected.reason();
    }
  }
  if (maxConfidence > 0.0) {
    regressionDetectedEvent.fireAsync(...);
    if (maxConfidence >= policy.effectiveAutoRevertThreshold()) { /* revert + pause */ }
    else if (maxConfidence >= policy.effectivePauseThreshold()) { /* pause only */ }
  }
}
```

Evaluators are filtered by domain: only the evaluators whose `domainId()` matches the improvement's domain (resolved from the improvement's category via `categoryRegistry.domainForCategory()`) are called. This ensures a trading improvement is evaluated by trading-domain evaluators, not code-evolution evaluators. If multiple evaluators exist for the same domain, max-confidence within the domain drives the revert/pause decision.

**Constructor changes:** Add `RegressionEvaluatorRegistry` and `ImprovementCategoryRegistry`.

### 3.3 HealthScoreTracker.HealthSnapshot → HealthScoreSnapshot

`HealthScoreTracker.HealthSnapshot` (inner record) moves to `api` module as `HealthScoreSnapshot` (top-level record). This avoids leaking runtime-core types into SPI interfaces.

`HealthScoreTracker` continues to use `HealthScoreSnapshot` internally — the move is a type relocation, not a redesign.

**Full chain of type migrations for HealthSnapshot → HealthScoreSnapshot:**
- `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot` (inner record → top-level record in api)
- `ConfidenceScorer.score()` parameter types: `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot`
- `RegressionDetector.MonitoredImprovement.baseline` field type: `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot`
- `RegressionDetector.checkActiveMonitors()` — `healthTracker.latestSnapshot()` return type
- `RegressionDetector.onMetricsDegraded()` parameter types

### 3.4 ImprovementBudgetEnforcer refactoring

`ImprovementBudgetEnforcer.check()` is refactored to remove domain-specific logic:

**Removed from the enforcer (moved to domain SPIs):**
- Structural deny pattern check (`STRUCTURAL_DENIED_PATTERNS`) → `DenyPatternProvider`
- Dynamic deny pattern check (path-based matching) → `DenyPatternProvider`
- Config denied paths check (`budget.effectiveDeniedPaths()`) → `DenyPatternProvider`
- Allowed repos check (`budget.effectiveAllowedRepos()`) → `CodeEvolutionDenyPatternProvider`

**Retained in the enforcer (domain-agnostic budget limits):**
- Concurrent improvement limit (`effectiveMaxConcurrent()`)
- Daily improvement limit (`effectiveMaxPerDay()`)
- Cooldown between improvements (`effectiveCooldownMinutes()`)
- Max change size (`effectiveMaxChangeSize()` — renamed from `effectiveMaxPRSize()`)

The coordinator calls deny pattern checking (§3.1 step 4) BEFORE budget checking, so the enforcer no longer needs to duplicate deny logic.

### 3.5 TickTrace.SignalFilteringSummary → ProposalFilteringSummary

`TickTrace.SignalFilteringSummary` has signal-consensus-specific fields (`consensusSignals`, `afterNamespaceFilter`) that don't apply to non-signal proposal sources. After refactoring, the coordinator collects from multiple sources, only one of which uses the signal-consensus model.

**Before:**
```java
public record SignalFilteringSummary(
    int consensusSignals,
    int afterNamespaceFilter,
    int afterCategoryFilter,
    int afterSuppressionFilter,
    int afterAntiOscillationFilter,
    int afterBudgetFilter,
    int afterConflictFilter,
    int proposed) {}
```

**After:**
```java
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

- `consensusSignals` and `afterNamespaceFilter` replaced by `proposalsBySource` — a map from source ID to proposal count, generalising per-source observability
- `afterDenyFilter` added — tracks deny pattern filtering (previously bundled into budget check)
- `TickOutcome.ProposalGenerated` updated to use `ProposalFilteringSummary`

## 4. ImprovementStage Migration

`ImprovementStage` changes from an enum to a `String`. The `ImprovementStage` enum class is deleted. All references change to `String`.

### Affected production types

| Type | Field/Method | Change |
|------|-------------|--------|
| `ImprovementRequest` | `targetRepo`, `targetPaths` fields | Removed — code-evolution stores in `metadata` |
| `ImprovementRequest` | (new field) | `domainId` added |
| `GatePolicy` | `Map<ImprovementStage, GateMode> modes` | → `Map<String, GateMode> modes` |
| `GatePolicy` | `effectiveMode(ImprovementStage)` | → `effectiveMode(String stageId)` |
| `GatePolicy` | constructor validation (`isGateCheckpoint()`) | → validation via `ImprovementCategoryRegistry.isGateCheckpoint(domainId, stageId)`. Deferred to runtime — the record constructor no longer validates because it has no registry reference. Validation moves to `DefaultEngineEvolutionApi.setGatePolicy()`. |
| `ConductorInboxEntry` | `ImprovementStage stage` | → `String stage` |
| `ArtifactEntry` | `ImprovementStage stage` | → `String stage` |
| `ResearchPipelineResult.AwaitingGate` | `ImprovementStage stage` | → `String stage` |
| `EscalationProvider` | `evaluate(..., ImprovementStage stage, ...)` | → `evaluate(..., String stage, ...)` |
| `EvolutionStateSnapshot.ImprovementStreamView` | `ImprovementStage currentStage` | → `String currentStage` |
| `EvolutionStateSnapshot.StageProgress` | `ImprovementStage stage` | → `String stage` |
| `ResearchPipelineOrchestrator` | Stage references (`ImprovementStage.RESEARCH_SCOPE`) | → String constants from `CodeEvolutionStages` |
| `ConfidenceScorer` | `score(..., HealthScoreTracker.HealthSnapshot, HealthScoreTracker.HealthSnapshot)` | → `score(..., HealthScoreSnapshot, HealthScoreSnapshot)` |
| `RegressionDetector.MonitoredImprovement` | `HealthScoreTracker.HealthSnapshot baseline` | → `HealthScoreSnapshot baseline` |
| `RegressionDetector` | `checkActiveMonitors()`, `onMetricsDegraded()` parameter types | `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot` |
| `ConflictDetector` | entire class | Deleted — logic moves to `FilePathConflictStrategy` |
| `ImprovementBudgetEnforcer` | `STRUCTURAL_DENIED_PATTERNS`, path/repo/deny checks | Moved to `CodeEvolutionDenyPatternProvider` |
| `ImprovementBudget` | `maxPRSize`, `effectiveMaxPRSize()` | → `maxChangeSize`, `effectiveMaxChangeSize()` |
| `TickTrace.SignalFilteringSummary` | Signal-consensus-specific fields | → `ProposalFilteringSummary` with per-source counts (see §3.5) |

### Well-known stage constants

Code-evolution stage constants are a companion to `CodeEvolutionCategoryProvider`, placed in runtime-core alongside the provider. They do NOT belong in the api module — the api module defines interfaces and shared model types; domain-specific constants live in domain implementations.

```java
// runtime-core, io.casehub.engine.internal.improvement
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

### GatePolicy validation migration

`GatePolicy`'s constructor currently validates that only gate checkpoints appear in the `modes` map. This validation used `ImprovementStage.isGateCheckpoint()` which is available at construction time for an enum. With strings, the validation needs access to the `ImprovementCategoryRegistry` which is a runtime bean.

**Resolution:** Remove the constructor validation from `GatePolicy`. `GatePolicy` becomes a pure data record with no invariant enforcement. Validation moves to a single point: `DefaultEngineEvolutionApi.setGatePolicy()`, which has access to the `ImprovementCategoryRegistry` and the case's domain context.

Validation is NOT added to `GatePolicyStore.save()` — that would couple the store SPI (in common-core) to the improvement registry (in runtime-core), which is an architectural regression. The store is a persistence abstraction; validation belongs at the API layer.

**YAML deserialization path:** `YamlGatePolicy` constructs a `GatePolicy` via the canonical constructor. After migration, a YAML config with a non-checkpoint stage would be accepted at deserialization time. Validation occurs when the gate policy is applied to a case — all case configuration changes route through `DefaultEngineEvolutionApi`, which validates before persisting. The template application path (case creation from a case template) must also validate via the API layer.

**`isGateCheckpoint` call site:** The validation at `setGatePolicy()` resolves the case's domain from its category configuration, then calls `categoryRegistry.isGateCheckpoint(domainId, stageId)` for each stage in the `modes` map.

The `effectiveMode()` default also changes — the code-evolution-specific "PR_REVIEW defaults to GATED" moves to a per-domain default. `GatePolicy.effectiveMode()` returns `GateMode.AUTO` for any unknown stage. The code-evolution provider can set its own defaults:

```java
public GateMode effectiveMode(String stageId) {
  if (modes != null && modes.containsKey(stageId)) {
    return modes.get(stageId);
  }
  return GateMode.AUTO;
}
```

The "PR_REVIEW defaults to GATED" semantic moves to the initial `ImprovementConfig` on the code-evolution case template (`caseTemplateId: "self-improvement"`). The template's `gatePolicy` field sets `{"pr-review": "GATED"}` as the starting configuration. This is case-level config, not provider-level — different cases in the same domain can have different gate defaults.

### YAML codegen impact

`YamlGatePolicy` (generated) currently has `Map<ImprovementStage, GateMode> modes`. After migration, this becomes `Map<String, GateMode> modes`. The codegen entry in `yaml-record-mappings.yaml` needs updating to reflect the key type change from enum to String.

## 5. ImprovementConfig and ImprovementBudget Changes

### ImprovementBudget field rename

`ImprovementBudget.maxPRSize` → `maxChangeSize`, `effectiveMaxPRSize()` → `effectiveMaxChangeSize()`. "PR size" is code-evolution terminology; "change size" is domain-neutral (line count for code, parameter count for trading, rule count for AML).

The `allowedRepos` and `deniedPaths` fields remain on the record (nullable, defaulting to empty) but their enforcement moves from `ImprovementBudgetEnforcer` to `CodeEvolutionDenyPatternProvider`. Non-code domains leave them null — the fields are inert when no code-evolution deny pattern provider is registered.

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
| `ImprovementCategoryRegistryTest` | Provider registration, category lookup, domain-scoped stage lookup, domain resolution, gate checkpoint check with domain context, multi-provider coexistence, stage ID collision across domains resolved correctly, reset |
| `ImprovementProposalSourceRegistryTest` | Source registration, retrieval, reset |
| `RegressionEvaluatorRegistryTest` | Evaluator registration, retrieval, reset |
| `ConflictStrategyRegistryTest` | Strategy registration by domain, domain lookup, reset |
| `DenyPatternProviderRegistryTest` | Provider registration by domain, domain lookup, reset |
| `CodeEvolutionCategoryProviderTest` | Default categories (5), default stages (11), stage ordering, gate checkpoints match expected set |
| `SignalConsensusProposalSourceTest` | Signal-consensus proposal generation (extracted from ImprovementGoalFormationStrategyTest), namespace filtering |
| `HealthScoreDeltaRegressionEvaluatorTest` | Delta threshold detection, no-regression case, boundary values |
| `FilePathConflictStrategyTest` | File-path-based conflict detection (extracted from ConflictDetectorTest), trivial threshold, same-directory detection, metadata path extraction |
| `CodeEvolutionDenyPatternProviderTest` | Structural deny patterns, dynamic deny patterns, path extraction from metadata |
| `EvolutionBootstrapTest` | Multi-provider discovery, all 5 registries populated |

### Modified existing tests

| Test class | Change |
|------------|--------|
| `ImprovementGoalFormationStrategyTest` | Refactor: inject mock ImprovementProposalSourceRegistry, ConflictStrategyRegistry, DenyPatternProviderRegistry instead of SignalRegistry and ConflictDetector. Test filtering pipeline with proposals from multiple sources, domain-aware deny and conflict checks. |
| `RegressionDetectorTest` | Refactor: inject RegressionEvaluatorRegistry and ImprovementCategoryRegistry. Test that evaluators are filtered by domain and first Detected verdict triggers regression handling. |
| `GatePolicyTest` | Change ImprovementStage enum → String constants. Remove constructor validation tests (moved to API layer). |
| `ConductorInboxManagerTest` | Change ImprovementStage enum → String constants. |
| `ConductorInboxRepositoryContractTest` | Change ImprovementStage enum → String constants (makeEntry helper and all entry construction sites). |
| `InMemoryConductorInboxRepositoryContractTest` | Change ImprovementStage enum → String constants (entry construction at line 46). |
| `DefaultEscalationProviderTest` | Change ImprovementStage enum → String constants. |
| `EvolutionApiTest` | Change ImprovementStage enum → String constants. Add gate policy validation tests with domain-scoped `isGateCheckpoint(domainId, stageId)` (moved from GatePolicy constructor). |
| `ResearchPipelineCheckpointTest` | Change ImprovementStage enum → String constants. |
| `TickTraceTest` | Remove `improvementStageGateCheckpoints` test (gate checkpoint logic moves to registry). Update `SignalFilteringSummary` → `ProposalFilteringSummary`. |
| `ConflictDetectorTest` | Renamed to `FilePathConflictStrategyTest`. Update to test `ConflictStrategy` SPI with metadata-based path extraction. |
| `ImprovementBudgetEnforcerTest` | Remove deny pattern and path-based tests (moved to `CodeEvolutionDenyPatternProviderTest`). Retain domain-agnostic budget tests. Rename `maxPRSize` → `maxChangeSize`. |
| `ConfidenceScorerTest` | Update `HealthScoreTracker.HealthSnapshot` → `HealthScoreSnapshot` in all test fixtures. |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `MultiDomainEvolutionTest` | Two domains registered (code-evolution + trading-stub), categories from both discovered, proposals from both sources collected, domain-specific deny patterns and conflict strategies applied per-domain. Trading regression evaluator detects trading-specific regression while code-evolution evaluator sees no regression — domain-filtered evaluation routes correctly. |
| `CategoryFilteringIntegrationTest` | ImprovementConfig.enabledCategories filters across domain providers. Null enables all. Explicit list filters to subset. |

### Critical test scenarios

1. **Multi-provider coexistence:** Two category providers register — both domains' categories appear in registry. Category IDs must be globally unique across domains (e.g., "dependency-update" for code-evolution, "parameter-tuning" for trading). If two providers register the same ID, the second registration logs a warning and overwrites — this is a configuration error, not a supported use case.
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
| `ConflictStrategy` SPI | `api` | `io.casehub.api.spi.improvement` |
| `DenyPatternProvider` SPI | `api` | `io.casehub.api.spi.improvement` |
| `CategoryDescriptor` | `api` | `io.casehub.api.model.stigmergy` |
| `StageDescriptor` | `api` | `io.casehub.api.model.stigmergy` |
| `RegressionVerdict` | `api` | `io.casehub.api.model.stigmergy` |
| `HealthScoreSnapshot` | `api` | `io.casehub.api.model.stigmergy` |
| `ProposalFilteringSummary` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementCategoryRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementProposalSourceRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `RegressionEvaluatorRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ConflictStrategyRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `DenyPatternProviderRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `EvolutionBootstrap` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `CodeEvolutionCategoryProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `CodeEvolutionStages` (constants) | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `SignalConsensusProposalSource` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `HealthScoreDeltaRegressionEvaluator` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `FilePathConflictStrategy` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `CodeEvolutionDenyPatternProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |

## 8. Migration Sequence

The changes have dependencies that determine ordering:

1. **API types first:** `CategoryDescriptor`, `StageDescriptor`, `RegressionVerdict`, `HealthScoreSnapshot`, `ProposalFilteringSummary`, `ImprovementRequest` generalisation (remove `targetRepo`/`targetPaths`, add `domainId`)
2. **SPI interfaces:** `ImprovementCategoryProvider`, `ImprovementProposalSource`, `RegressionEvaluator`, `ConflictStrategy`, `DenyPatternProvider`
3. **ImprovementStage migration:** Delete enum, update all references to `String`, update `GatePolicy`, update YAML codegen
4. **ImprovementBudget rename:** `maxPRSize` → `maxChangeSize`
5. **Registries:** `ImprovementCategoryRegistry`, `ImprovementProposalSourceRegistry`, `RegressionEvaluatorRegistry`, `ConflictStrategyRegistry`, `DenyPatternProviderRegistry`
6. **Default implementations:** `CodeEvolutionCategoryProvider`, `CodeEvolutionStages` (runtime-core), `SignalConsensusProposalSource`, `HealthScoreDeltaRegressionEvaluator`, `FilePathConflictStrategy`, `CodeEvolutionDenyPatternProvider`
7. **Bootstrap:** `EvolutionBootstrap` (discovers all 5 SPI types)
8. **Backbone refactoring:** `ImprovementGoalFormationStrategy` (coordinator with domain-aware filtering), `RegressionDetector` (domain-filtered evaluation), `HealthScoreTracker` (HealthSnapshot → HealthScoreSnapshot), `ConfidenceScorer` (type migration), `ImprovementBudgetEnforcer` (remove domain-specific logic), `TickTrace` (SignalFilteringSummary → ProposalFilteringSummary)
9. **ImprovementConfig:** Remove `effectiveEnabledCategories()` hardcoded default
10. **Delete:** `ConflictDetector` (replaced by `FilePathConflictStrategy`), `ImprovementStage` enum, `ImprovementStages` constants class (if created — now `CodeEvolutionStages` in runtime-core instead)

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
- `ConflictDetector.java:25-68` — conflict logic to extract to ConflictStrategy SPI
- `ImprovementBudgetEnforcer.java:37-53` — STRUCTURAL_DENIED_PATTERNS to extract to DenyPatternProvider SPI
- `ImprovementRequest.java:21-28` — target model to generalise
- `ConfidenceScorer.java:22-58` — type migration for HealthSnapshot
- `TickTrace.java:55-63` — SignalFilteringSummary to generalise
- `EvolutionTicker.java:36-131` — orchestration backbone (unchanged)
- PP-20260921-b7c277 — never use @DefaultBean on multi-instance SPIs
- PP-20260915-aa504e — API surface parity (YAML, DSL, annotations) — applies to GatePolicy codegen change; new SPI/model types are not CaseDefinition fields and do not require YAML codegen entries
- Decisions D1–D8 in `decisions.md`
- Issue casehubio/engine#1148 (Comment 1 predecessor audit: ImprovementRequest target model, ConflictStrategy SPI, DenyPatternProvider SPI)
- Epic casehubio/engine#1149
- Parent spec: `2026-09-21-command-centre-conductor-design.md`
