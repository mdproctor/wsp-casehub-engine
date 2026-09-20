# Continuous Evolution Loop — Design Spec

**Issue:** casehubio/engine#1115
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `2026-09-20-cognitive-self-improvement-vision.md`
**Engine foundation:** `2026-09-20-autonomous-self-improvement-engine-foundation.md`
**Methodology:** `2026-09-20-continuous-improvement-methodology.md`
**Date:** 2026-09-20
**Decisions:** D106–D115

## Problem

Issue #1114 delivers the complete single-shot improvement cycle: signal consensus → goal formation → budget check → improvement case → worker execution → review → integration → outcome recording. An agent detects a stale dependency, the system proposes a goal, spawns a case, executes the improvement, submits for review, integrates on approval, and records the outcome across three layers (EventLog, signal projection, CBR trace).

What's missing is the word "continuously." The single-shot cycle must be triggered manually or by coincidence — an agent happens to observe the right signals at the right time. There is no standing directive, no feedback loop, no mechanism for outcomes to influence the next sensing cycle, no detection of regression, no circuit breaker for runaway degradation, no strategic framework for deciding what to improve, and no structured research methodology for discovering how.

| Gap | What's missing |
|-----|---------------|
| Standing directive | No continuous trigger — improvements happen only when signals coincidentally reach consensus |
| Outcome feedback | Outcomes don't feed back into detection — the loop is open, not closed |
| Regression detection | No mechanism to detect when an improvement makes things worse |
| Data autophagy prevention | No circuit breaker for gradual, multi-improvement quality degradation |
| Concurrent scheduling | No conflict detection — concurrent improvements can produce merge conflicts |
| Growth direction | No strategic framework for deciding what to improve next |
| Research methodology | No structured process for discovering improvement opportunities beyond pre-programmed categories |
| Research persistence | No corpus — every research cycle starts from scratch |

**Scope boundary:** This issue delivers the continuous loop infrastructure in engine, with SPI contracts for blocks integration. The cognitive agent (CognitionCore, PAD, narrative) is Epic 2. LLM-powered research implementations are Epic 3. Full cognitive memory is Epic 4. This spec provides the pipeline, the contracts, and the rule-based defaults — blocks and neocortex provide the intelligence.

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Continuous Evolution Loop                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Trigger Layer                                    │  │
│  │                                                                    │  │
│  │  EvolutionTicker (timer backstop)                                 │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  ImprovementGoalFormationStrategy ◄── outcome signals (feedback)   │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  ImprovementCircuitBreaker ── OPEN? → block                       │  │
│  │       │ CLOSED/HALF_OPEN                                          │  │
│  │       ▼                                                            │  │
│  │  ConflictDetector ── overlap? → queue                             │  │
│  │       │ clear                                                     │  │
│  │       ▼                                                            │  │
│  │  ImprovementBudgetEnforcer (existing)                             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                          │                                               │
│                          ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │              Improvement Case (existing from #1114)                │  │
│  │                                                                    │  │
│  │  introspect → [research → analyse] → implement → submit-pr        │  │
│  │                                        → review → integrate        │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                          │                                               │
│                          ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │              Outcome Processing                                    │  │
│  │                                                                    │  │
│  │  ImprovementOutcomeEventCapture (existing — 3 layers)             │  │
│  │       │                                                            │  │
│  │       ├─→ RegressionDetector → ConfidenceScorer                   │  │
│  │       │       │                                                    │  │
│  │       │       ├─ high confidence → spawn rollback case             │  │
│  │       │       ├─ medium confidence → signal + pause category       │  │
│  │       │       └─ low confidence → signal only, enrich CBR          │  │
│  │       │                                                            │  │
│  │       ├─→ HealthScoreTracker → ImprovementCircuitBreaker           │  │
│  │       │                                                            │  │
│  │       └─→ outcome signals ──────────────────────► (back to top)    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │              Growth Direction                                      │  │
│  │                                                                    │  │
│  │  CapabilityAreaRegistry                                           │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  ResearchPipeline (4 SPIs)                                        │  │
│  │  ResearchScoper → ResearchSearcher → ResearchAnalyzer             │  │
│  │       → HypothesisFormer                                          │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  TechnologyRadar (persistent artifact)                            │  │
│  │  ResearchCorpus (Living Systematic Review)                        │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  improvement hypotheses → signal deposit → consensus → goal       │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

**Data flow — continuous cycle:**

1. `EvolutionTicker` fires periodically (timer backstop) OR outcome signals arrive (event-driven re-entry)
2. `ImprovementGoalFormationStrategy` scans for improvement signal consensus
3. `ImprovementCircuitBreaker` checks health score — blocks new improvements if OPEN
4. `ConflictDetector` checks target paths against active improvements — queues if overlap
5. `ImprovementBudgetEnforcer` validates budget constraints (existing)
6. Improvement case executes through existing lifecycle
7. `ImprovementOutcomeEventCapture` records outcome (existing — 3 layers)
8. `RegressionDetector` evaluates whether the outcome caused regression
9. `HealthScoreTracker` updates the rolling health score
10. Outcome signals feed back into the signal registry → next cycle

## 1. Standing Directive — Hybrid Trigger Model

The continuous evolution loop uses two complementary trigger mechanisms that together ensure the improvement cycle runs without manual intervention.

### Event-driven re-entry (primary)

Outcome signals projected by `ImprovementOutcomeEventCapture` (existing, D98) feed back into the signal registry. These signals — `improvement:outcome:positive:*`, `improvement:outcome:regression:*`, `improvement:outcome:rejected:*` — are visible to `ImprovementGoalFormationStrategy` during the next evaluation cycle. When outcome signals reinforce existing improvement signals (e.g., a positive outcome in dependency updates reinforces `improvement:dependency:staleness:*` signals for similar dependencies), the consensus threshold is reached and new goals are proposed.

This path requires no new infrastructure — the signal projection and goal formation strategy already exist. The loop closes by wiring outcome signals into the same consensus model that triggers new improvements.

### Timer backstop (secondary)

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class EvolutionTicker implements Resettable {

  private final ImprovementGoalFormationStrategy goalFormation;
  private final ImprovementCircuitBreaker circuitBreaker;
  private final HealthScoreTracker healthTracker;

  public void tick(UUID caseId, ImprovementConfig config) {
    healthTracker.refresh(caseId);

    if (circuitBreaker.state(caseId) == CircuitBreakerState.OPEN) {
      return;
    }

    goalFormation.proposeImprovements(caseId, config);
  }
}
```

The ticker is invoked periodically by the case evaluation pipeline. The interval is configurable via `ImprovementConfig.evolutionTickIntervalMinutes()` (default: 60). The tick is lightweight — it refreshes the health score and invokes the existing goal formation strategy, which already handles consensus detection and budget checking.

### Idempotency

Both triggers can fire within the same evaluation window. The existing consensus model provides natural deduplication — if the same improvement signals are already at consensus, `proposeImprovements` returns the same proposal. The budget enforcer's concurrent limit prevents duplicate improvement cases from being spawned.

### Configuration

```java
// api/model/stigmergy — new fields on ImprovementConfig
@Nullable Integer evolutionTickIntervalMinutes,  // default 60
@Nullable Boolean evolutionEnabled               // default false — opt-in
```

The loop is **opt-in**. `evolutionEnabled` must be explicitly set to `true`. This prevents existing cases from acquiring autonomous improvement behaviour without configuration.

## 2. Outcome Feedback Loop

The feedback loop connects improvement outcomes back to the sensing layer, creating a closed cycle where results influence future detection.

### Feedback paths

| Outcome status | Signal projected | Effect on next cycle |
|---------------|-----------------|---------------------|
| MERGED | `improvement:outcome:positive:pr-merged` | Reinforces similar improvement signals — "this category works, keep going" |
| REJECTED | `improvement:outcome:rejected` | Weakens signals in this category — goal formation reduces priority |
| REGRESSION | `improvement:outcome:regression:*` | Triggers `RegressionDetector` (§3) — may pause category |
| FAILED | `improvement:outcome:failed` | Enriches CBR trace — "what went wrong?" informs future attempts |
| ABANDONED | `improvement:outcome:abandoned` | Clears related signals — direction was abandoned |

### Goal revision from outcomes

`GoalRevisionEvaluator` applies improvement-specific logic when `goal.kind() == StandardGoalKind.SELF_IMPROVEMENT`:

- **Repeated failures:** 3+ FAILED outcomes in the same category within the health window → deprioritise. The drive source for COMPETENCE lowers intensity for that category.
- **Repeated rejections:** 3 REJECTED PRs in the same improvement direction → abandon the goal and emit `improvement:outcome:abandoned`.
- **Regression:** Any REGRESSION outcome → trigger `RegressionDetector` (§3) for confidence-tiered response.

### CBR-informed goal formation

When `ImprovementGoalFormationStrategy` proposes a new improvement, it queries CBR for similar past outcomes:

```java
var similar = cbrRetriever.retrieve(tenantId, "improvement-outcomes",
    FeatureVectorCbrCase.builder()
        .feature("category", FeatureValue.categorical(request.category()))
        .feature("target", FeatureValue.text(request.target()))
        .build(),
    5);
```

If similar past improvements have a high failure rate, the strategy reduces priority. If similar improvements have a high success rate, priority is maintained. This is the "learning from experience" mechanism — no LLM needed, purely structural CBR.

## 3. Regression Detection and Rollback

When an improvement lands and makes things worse, the system detects and responds proportionally to its confidence in causal attribution.

### RegressionDetector

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class RegressionDetector {

  private final ConfidenceScorer scorer;
  private final ImprovementBudgetEnforcer budgetEnforcer;
  private final SignalRegistry signalRegistry;

  public void evaluate(UUID caseId, ImprovementOutcome outcome) {
    if (outcome.status() != ImprovementOutcome.OutcomeStatus.MERGED) {
      return;
    }

    // Monitor metrics within the regression window
    // (invoked periodically after merge, not just once)
  }

  public void onMetricsDegraded(
      UUID caseId, UUID improvementCaseId,
      RollbackPolicy policy, MetricsSnapshot before, MetricsSnapshot after) {
    double confidence = scorer.score(caseId, improvementCaseId, before, after);

    if (confidence >= policy.effectiveAutoRevertThreshold()) {
      spawnRollbackCase(caseId, improvementCaseId, confidence);
      budgetEnforcer.pauseCategory(caseId, outcome.category());
    } else if (confidence >= policy.effectivePauseThreshold()) {
      emitRegressionSignal(caseId, improvementCaseId, confidence);
      budgetEnforcer.pauseCategory(caseId, outcome.category());
    } else {
      emitRegressionSignal(caseId, improvementCaseId, confidence);
    }
  }
}
```

### ConfidenceScorer

Composable signals with additive weights:

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ConfidenceScorer {

  public double score(
      UUID caseId, UUID improvementCaseId,
      MetricsSnapshot before, MetricsSnapshot after) {
    double confidence = 0.0;

    if (improvementCiBuildFailed(improvementCaseId)) {
      confidence += 0.5;
    }
    if (failingTestsTouchModifiedFiles(improvementCaseId, after)) {
      confidence += 0.3;
    }
    if (regressionWithinWindow(improvementCaseId, after)) {
      confidence += 0.2;
    }
    if (cbrShowsSimilarRegressions(improvementCaseId)) {
      confidence += 0.1;
    }
    if (multipleMetricsDegraded(before, after)) {
      confidence += 0.1;
    }
    if (regressionInUnrelatedModule(improvementCaseId, after)) {
      confidence -= 0.2;
    }
    if (otherChangesMergedInWindow(improvementCaseId)) {
      confidence -= 0.3;
    }

    return Math.max(0.0, Math.min(1.0, confidence));
  }
}
```

### RollbackPolicy

```java
// api/model/stigmergy
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

### Rollback case template

A rollback is itself an improvement case with a shortened lifecycle:

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

### Safety guards

**Anti-oscillation:** Before proposing an improvement, `ImprovementGoalFormationStrategy` queries CBR for past rollbacks of the same target. If found and context hasn't materially changed, the proposal is suppressed.

**Anti-cascade:** Before spawning a rollback case, `RegressionDetector` checks `ConflictDetector` (§5) for later improvements that depend on the regressed change. If dependencies exist, downgrade to signal + pause — don't auto-revert.

**Anti-flaky:** `RegressionDetector` requires `sustainedFailureCount` (default 2) consecutive failures before acting. A single flaky test run does not trigger rollback.

## 4. Health Score and Circuit Breaker

Proactive defense against gradual, multi-improvement quality degradation — the "data autophagy" risk.

### HealthScoreTracker

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class HealthScoreTracker implements Resettable {

  public record HealthSnapshot(
      double score,
      Instant timestamp,
      Map<String, Double> componentScores) {}

  private final ConcurrentHashMap<UUID, Deque<HealthSnapshot>> history =
      new ConcurrentHashMap<>();

  public double computeScore(UUID caseId, HealthPolicy policy) {
    // Aggregate normalised metrics weighted by policy
    // Each metric normalised to [0, 1]
    double score = 0.0;
    var weights = policy.effectiveWeights();
    score += weights.getOrDefault("ciPassRate", 0.2) * normaliseCiPassRate(caseId);
    score += weights.getOrDefault("testCoverage", 0.15) * normaliseTestCoverage(caseId);
    score += weights.getOrDefault("lintViolations", 0.15) * normaliseLintViolations(caseId);
    score += weights.getOrDefault("dependencyFreshness", 0.15) * normaliseDependencyFreshness(caseId);
    score += weights.getOrDefault("buildTime", 0.1) * normaliseBuildTime(caseId);
    score += weights.getOrDefault("flakyTestRate", 0.15) * normaliseFlakyTestRate(caseId);
    score += weights.getOrDefault("improvementSuccessRate", 0.1) * normaliseImprovementSuccessRate(caseId);
    return score;
  }

  public void refresh(UUID caseId) {
    // Compute and store in history ring buffer
  }

  public double delta(UUID caseId, int windowMinutes) {
    // Current score minus score at windowMinutes ago
  }
}
```

### ImprovementCircuitBreaker

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementCircuitBreaker implements Resettable {

  public enum CircuitBreakerState { CLOSED, OPEN, HALF_OPEN }

  private final ConcurrentHashMap<UUID, CircuitBreakerState> states =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, Integer> halfOpenCount =
      new ConcurrentHashMap<>();

  public CircuitBreakerState state(UUID caseId) {
    return states.getOrDefault(caseId, CircuitBreakerState.CLOSED);
  }

  public void evaluate(UUID caseId, HealthScoreTracker tracker, HealthPolicy policy) {
    double score = tracker.computeScore(caseId, policy);
    double delta = tracker.delta(caseId, policy.effectiveHealthWindowMinutes());
    var current = state(caseId);

    switch (current) {
      case CLOSED -> {
        if (score < policy.effectiveHealthThreshold()
            || delta < -policy.effectiveHealthDeltaThreshold()) {
          transition(caseId, CircuitBreakerState.OPEN);
          emitCircuitBreakerSignal(caseId, "tripped", score, delta);
        }
      }
      case OPEN -> {
        if (score >= policy.effectiveHealthThreshold()
            && sustainedRecovery(caseId, policy)) {
          transition(caseId, CircuitBreakerState.HALF_OPEN);
          halfOpenCount.put(caseId, 0);
        }
      }
      case HALF_OPEN -> {
        int completed = halfOpenCount.getOrDefault(caseId, 0);
        if (score < policy.effectiveHealthThreshold()) {
          transition(caseId, CircuitBreakerState.OPEN);
        } else if (completed >= policy.effectiveHalfOpenMaxImprovements()) {
          transition(caseId, CircuitBreakerState.CLOSED);
        }
      }
    }
  }

  public void recordImprovementInHalfOpen(UUID caseId) {
    halfOpenCount.computeIfPresent(caseId, (k, v) -> v + 1);
  }

  public void manualReset(UUID caseId) {
    transition(caseId, CircuitBreakerState.CLOSED);
  }
}
```

### HealthPolicy

```java
// api/model/stigmergy
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
        "ciPassRate", 0.2,
        "testCoverage", 0.15,
        "lintViolations", 0.15,
        "dependencyFreshness", 0.15,
        "buildTime", 0.1,
        "flakyTestRate", 0.15,
        "improvementSuccessRate", 0.1);
  }
}
```

### Circuit breaker signals

| Signal | When |
|--------|------|
| `improvement:circuit-breaker:tripped` | CLOSED → OPEN |
| `improvement:circuit-breaker:recovering` | OPEN → HALF_OPEN |
| `improvement:circuit-breaker:reset` | HALF_OPEN → CLOSED (or manual reset) |

All transitions produce an `EventLog` entry with the health score, delta, and triggering metrics.

## 5. Concurrent Improvement Scheduling

Conflict avoidance for concurrent improvements, preventing the merge conflicts and stale-context failures that LLMs handle poorly.

### ConflictDetector

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ConflictDetector implements Resettable {

  public sealed interface ConflictCheck
      permits ConflictCheck.Clear, ConflictCheck.Conflicting {
    record Clear() implements ConflictCheck {}
    record Conflicting(UUID blockingImprovementId, String conflictPath)
        implements ConflictCheck {}
  }

  public ConflictCheck check(
      ImprovementRequest request,
      Map<UUID, ImprovementRequest> activeImprovements,
      int trivialThreshold) {
    boolean isTrivial = request.estimatedSize() <= trivialThreshold
        && request.targetPaths().size() == 1;

    for (var entry : activeImprovements.entrySet()) {
      var active = entry.getValue();
      for (String requestPath : request.targetPaths()) {
        for (String activePath : active.targetPaths()) {
          if (requestPath.equals(activePath)) {
            return new ConflictCheck.Conflicting(entry.getKey(), requestPath);
          }
          if (!isTrivial && sameDirectory(requestPath, activePath)) {
            return new ConflictCheck.Conflicting(entry.getKey(), requestPath);
          }
        }
      }
    }
    return new ConflictCheck.Clear();
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

### Conflict resolution

| Overlap type | Action |
|-------------|--------|
| No path overlap | Run concurrently (within `maxConcurrent` budget) |
| File-level overlap | Serialize — queue the later improvement |
| Directory-level overlap (broad change) | Serialize — queue the later improvement |
| Same module, different files | Run concurrently (low conflict risk) |

### Queueing model

Conflicting improvements are queued, not rejected. When the blocking improvement completes, the queued improvement's `introspect` phase re-runs against the updated codebase, ensuring it sees the post-change state.

### Trivial change exemption

Improvements with `estimatedSize <= trivialThreshold` (default 10 lines) touching a single file are exempt from directory-level conflict detection. They still check file-level overlap. This prevents a large refactor from blocking all small fixes in the same module.

### Integration with budget enforcer

`ConflictDetector.check()` is called after budget enforcement passes, before the improvement case is spawned. The check is an additional gate in the goal formation pipeline:

```
consensus → budget check → circuit breaker check → conflict check → spawn case
```

## 6. Growth Direction — Capability Area Taxonomy

The strategic framework for deciding what to improve. The system's understanding of its own capability areas is a living model that evolves through self-assessment and landscape analysis.

### CapabilityArea SPI

```java
// api/spi/improvement
public interface CapabilityArea {

  String id();

  String name();

  String description();

  CapabilityAreaAssessment assess(UUID caseId);
}
```

```java
// api/model/stigmergy
public record CapabilityAreaAssessment(
    String areaId,
    double healthScore,
    LandscapePosition landscapePosition,
    double impactEstimate,
    double costEstimate,
    double roi,
    Instant assessedAt) {

  public enum LandscapePosition {
    AHEAD, AT_PARITY, BEHIND, ABSENT
  }
}
```

### CapabilityAreaRegistry

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class CapabilityAreaRegistry implements Resettable {

  private final ConcurrentHashMap<String, CapabilityArea> areas = new ConcurrentHashMap<>();

  public void register(CapabilityArea area) { ... }
  public void deprecate(String areaId) { ... }

  public List<CapabilityArea> active() {
    return List.copyOf(areas.values());
  }

  public Optional<CapabilityArea> get(String areaId) {
    return Optional.ofNullable(areas.get(areaId));
  }
}
```

### Bootstrap areas

Engine registers 10 default areas at startup (see D109 for the full table): Stability, Performance, Execution, Coordination, Perception, Autonomy, Cognitive reasoning, Cognitive memory, Safety, Integration.

Each default area provides a rule-based `assess()` implementation that computes health from available engine metrics (EventLog entries, ActivityTracker data, signal registry state). Blocks can replace or enhance any area's assessment with LLM-powered evaluation.

### Taxonomy evolution

The registry supports: `register` (add new area), `deprecate` (mark area inactive), merge (register new combined area + deprecate old), split (register new sub-areas + deprecate parent). All operations emit `CaseHubEventType.CAPABILITY_AREA_CHANGED` EventLog entries.

### Selection strategies

| Strategy | Innovation type | Drive alignment |
|----------|----------------|-----------------|
| Incremental | Incremental innovation | High COMPETENCE |
| Sustaining | Sustaining innovation (Christensen) | Moderate COMPETENCE |
| Architectural | Architectural innovation (Henderson & Clark) | High AUTONOMY |
| Radical | Radical/breakthrough innovation | High CURIOSITY |
| Pivot | Strategic pivot (Ries) | Negative outcome patterns |

Strategy emerges from the Drive profile. Engine provides the mapping from dominant drive to strategy bias. Blocks provides the full DriveOrchestrator integration. In engine-only mode, the strategy defaults to Incremental (operational improvements).

### Fit-gap model

The gap map is computed by assessing all active capability areas:

```java
public record GapMap(
    List<CapabilityAreaAssessment> assessments,
    Instant computedAt) {

  public List<CapabilityAreaAssessment> gapsByRoi() {
    return assessments.stream()
        .filter(a -> a.landscapePosition() != LandscapePosition.AHEAD)
        .sorted(Comparator.comparingDouble(CapabilityAreaAssessment::roi).reversed())
        .toList();
  }
}
```

## 7. Research Pipeline

The structured research methodology (PRISMA protocol + Technology Radar + Wardley Mapping), implemented as four core SPIs. The methodology document (`2026-09-20-continuous-improvement-methodology.md`) is the authoritative reference.

### Research depth

```java
// api/spi/improvement
public enum ResearchDepth {
  HORIZON_SCAN,
  TECHNOLOGY_SCOUTING,
  DEEP_DIVE
}
```

### Four core SPIs

```java
// api/spi/improvement
public interface ResearchScoper {
  ResearchScope scope(ResearchDepth depth, CapabilityAreaAssessment area,
      Map<String, String> driveContext);
}

public interface ResearchSearcher {
  List<ResearchCandidate> search(ResearchScope scope, ResearchDepth depth);
}

public interface ResearchAnalyzer {
  ResearchAnalysis analyze(List<ResearchCandidate> candidates,
      ResearchScope scope, ResearchDepth depth);
}

public interface HypothesisFormer {
  List<ImprovementHypothesis> form(ResearchAnalysis analysis,
      CapabilityAreaAssessment area);
}
```

### Supporting records

```java
// api/model/stigmergy
public record ResearchScope(
    String question,
    List<String> keywords,
    List<String> channels,
    CapabilityAreaAssessment area) {}

public record ResearchCandidate(
    String sourceUrl,
    String title,
    String abstractText,
    String sourceType,
    Map<String, String> metadata) {}

public record ResearchAnalysis(
    List<ResearchFinding> findings,
    List<String> themes,
    List<String> contradictions,
    List<String> gaps) {}

public record ResearchFinding(
    String technique,
    String claimedBenefits,
    String limitations,
    String applicability,
    String evidenceQuality,
    String capabilityArea,
    String sourceUrl) {}

public record ImprovementHypothesis(
    String technique,
    String targetComponent,
    String expectedImprovement,
    String evidence,
    String risk,
    String capabilityArea,
    RadarRecommendation radarRecommendation) {

  public enum RadarRecommendation { ADOPT, TRIAL, ASSESS }
}
```

### ResearchPipelineOrchestrator

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ResearchPipelineOrchestrator {

  private final ResearchScoper scoper;
  private final ResearchSearcher searcher;
  private final ResearchAnalyzer analyzer;
  private final HypothesisFormer hypothesisFormer;
  private final ResearchCorpus corpus;

  public List<ImprovementHypothesis> execute(
      ResearchDepth depth, CapabilityAreaAssessment area,
      Map<String, String> driveContext) {
    var scope = scoper.scope(depth, area, driveContext);
    var candidates = searcher.search(scope, depth);
    var analysis = analyzer.analyze(candidates, scope, depth);

    corpus.store(candidates, analysis);

    return hypothesisFormer.form(analysis, area);
  }
}
```

### Engine defaults

Engine provides rule-based skeleton implementations:
- `DefaultResearchScoper` — constructs scope from drive context keywords and area metrics
- `DefaultResearchSearcher` — returns empty list (no external search without blocks)
- `DefaultResearchAnalyzer` — passes through candidates as findings (no synthesis without LLM)
- `DefaultHypothesisFormer` — converts findings to hypotheses using area assessment metrics

These defaults are functional but shallow. The real intelligence comes from blocks LLM-powered implementations (Epic 3).

### Technology Radar as persistent artifact

The Technology Radar tracks every technology blip the system has evaluated, following the ThoughtWorks model with Adopt/Trial/Assess/Hold rings. See methodology document §5 for structure.

```java
// api/model/stigmergy
public record TechnologyBlip(
    String id,
    String name,
    String description,
    RadarRing ring,
    String capabilityArea,
    Instant lastAssessed,
    String rationale) {

  public enum RadarRing { ADOPT, TRIAL, ASSESS, HOLD }
}
```

The radar is stored as case context (key `improvement:technology-radar`) and updated by `ResearchPipelineOrchestrator` after each execution.

## 8. Research Corpus — Living Systematic Review

The persistent, searchable store of all research findings. Named after the Living Systematic Review methodology — continuously updated rather than point-in-time.

### Corpus SPI

```java
// api/spi/improvement
public interface ResearchCorpus {

  void store(List<ResearchCandidate> candidates, ResearchAnalysis analysis);

  List<ResearchFinding> search(String query, String capabilityArea, int limit);

  Optional<ResearchFinding> get(String sourceUrl);

  List<HilQueueEntry> pendingHilEntries();

  void addToHilQueue(HilQueueEntry entry);

  void resolveHilEntry(String sourceUrl, ResearchCandidate retrieved);
}
```

### HIL queue entry

```java
// api/model/stigmergy
public record HilQueueEntry(
    String sourceUrl,
    String citation,
    String reason,
    String capabilityArea,
    int priority,
    List<String> blockingHypotheses,
    Instant queuedAt) {}
```

### In-memory default implementation

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class InMemoryResearchCorpus implements ResearchCorpus, Resettable {

  private final ConcurrentHashMap<String, ResearchFinding> findings =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<String, HilQueueEntry> hilQueue =
      new ConcurrentHashMap<>();

  // ... implementation
}
```

The in-memory implementation is functional for testing and engine-only mode. Neocortex provides a persistent implementation backed by MindMap nodes in Epic 3/4.

### Freshness model

| Source type | Staleness threshold |
|-------------|---------------------|
| Academic papers | 12 months |
| Competitor docs | 3 months |
| Open-source repos | 6 months |
| Industry reports | 12 months |
| Internal CBR traces | Never stale |

Stale entries are flagged for re-check during the next technology scouting pass for their capability area.

## 9. Continuous Improvement Methodology Reference

The methodology document at `2026-09-20-continuous-improvement-methodology.md` governs the entire research-to-implementation pipeline. It composes four established frameworks:

| Framework | What it solves |
|-----------|---------------|
| **Horizon Scanning** (OECD) | Detecting signals of change in the external landscape |
| **PRISMA Protocol** (systematic review) | Structured, reproducible research pipeline |
| **Technology Radar** (ThoughtWorks) | Tracking technology maturity and adoption decisions |
| **Wardley Mapping** (Simon Wardley) | Positioning components by evolution stage |

The methodology defines standard terminology (§12 of the methodology document) used throughout this spec and the implementation. The methodology is a reference — it informs the SPI contracts and pipeline design but is not itself an engine component.

**Future:** The methodology could become a case template (the improvement system follows its own methodology as a case) or a skill (Claude follows the methodology when doing research work).

## 10. Configuration Model

New fields on `ImprovementConfig`:

```java
// api/model/stigmergy — extended ImprovementConfig
public record ImprovementConfig(
    @Nullable String signalNamespace,
    @Nullable Integer consensusMinSources,
    @Nullable List<String> enabledCategories,
    @Nullable ImprovementBudget budget,
    @Nullable String caseTemplateId,
    // --- New fields for #1115 ---
    @Nullable Boolean evolutionEnabled,
    @Nullable Integer evolutionTickIntervalMinutes,
    @Nullable RollbackPolicy rollbackPolicy,
    @Nullable HealthPolicy healthPolicy,
    @Nullable ResearchMethodology researchMethodology,
    @Nullable Integer conflictTrivialThreshold) {

  // ... existing effective* methods ...

  public boolean effectiveEvolutionEnabled() {
    return evolutionEnabled != null ? evolutionEnabled : false;
  }

  public int effectiveEvolutionTickIntervalMinutes() {
    return evolutionTickIntervalMinutes != null ? evolutionTickIntervalMinutes : 60;
  }

  public RollbackPolicy effectiveRollbackPolicy() {
    return rollbackPolicy != null ? rollbackPolicy
        : new RollbackPolicy(null, null, null, null, null, null);
  }

  public HealthPolicy effectiveHealthPolicy() {
    return healthPolicy != null ? healthPolicy
        : new HealthPolicy(null, null, null, null, null, null);
  }

  public int effectiveConflictTrivialThreshold() {
    return conflictTrivialThreshold != null ? conflictTrivialThreshold : 10;
  }
}
```

### ResearchMethodology

```java
// api/model/stigmergy
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

## 11. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `RollbackPolicy` | `api` | `io.casehub.api.model.stigmergy` |
| `HealthPolicy` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchMethodology` | `api` | `io.casehub.api.model.stigmergy` |
| `CapabilityAreaAssessment` | `api` | `io.casehub.api.model.stigmergy` |
| `TechnologyBlip` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchScope` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchCandidate` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchAnalysis` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchFinding` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementHypothesis` | `api` | `io.casehub.api.model.stigmergy` |
| `HilQueueEntry` | `api` | `io.casehub.api.model.stigmergy` |
| `CapabilityArea` SPI | `api` | `io.casehub.api.spi.improvement` |
| `ResearchScoper` SPI | `api` | `io.casehub.api.spi.improvement` |
| `ResearchSearcher` SPI | `api` | `io.casehub.api.spi.improvement` |
| `ResearchAnalyzer` SPI | `api` | `io.casehub.api.spi.improvement` |
| `HypothesisFormer` SPI | `api` | `io.casehub.api.spi.improvement` |
| `ResearchCorpus` SPI | `api` | `io.casehub.api.spi.improvement` |
| `EvolutionTicker` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `RegressionDetector` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ConfidenceScorer` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `HealthScoreTracker` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementCircuitBreaker` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ConflictDetector` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `CapabilityAreaRegistry` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ResearchPipelineOrchestrator` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `InMemoryResearchCorpus` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| Default capability area impls | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| Default research SPI impls | `runtime-core` | `io.casehub.engine.internal.improvement.research` |
| Rollback case template YAML | `runtime` | `case-templates/self-improvement-rollback.yaml` |

## 12. Extension Points for Blocks and Neocortex

### For blocks (Epic 2 — Cognitive Agent)

| Extension point | How blocks uses it |
|----------------|-------------------|
| `CapabilityArea` SPI | LLM-powered area assessment — richer health scoring than rule-based metrics |
| `ResearchAnalyzer` SPI | LLM-powered synthesis, triangulation, and prioritisation |
| `HypothesisFormer` SPI | LLM-powered hypothesis formation from research analysis |
| `DriveSource` SPI | Feed capability area health into drive intensity |
| `EvolutionTicker.tick()` | Wire into `CognitionCore.tick()` for cognitive timing |
| Selection strategy | Drive profile → strategy mapping with personality modulation |
| `improvement:circuit-breaker:*` signals | Feed PAD mood signals from circuit breaker state |

### For blocks (Epic 3 — Research Loop)

| Extension point | How blocks uses it |
|----------------|-------------------|
| `ResearchScoper` SPI | LLM-powered scoping from drive context and curiosity signals |
| `ResearchSearcher` SPI | Structured API calls to arXiv, Scholar, GitHub |
| `ResearchAnalyzer` SPI | Full PRISMA screening → extraction → synthesis → triangulation |
| `ResearchCorpus` SPI | Persistent corpus backed by document store |

### For neocortex (Epic 4 — Cognitive Memory)

| Extension point | How neocortex uses it |
|----------------|----------------------|
| `ResearchCorpus` SPI | Persistent implementation backed by MindMap nodes |
| `CapabilityArea.assess()` | Memory-augmented assessment using accumulated experience |
| CBR traces from regression detection | Episodic memory nodes for improvement failures |
| Technology Radar changes | Semantic knowledge nodes for technique evaluations |

## 13. Test Strategy

### Unit tests

| Test class | What it covers |
|------------|---------------|
| `EvolutionTickerTest` | Timer tick invokes goal formation, respects circuit breaker state, opt-in guard |
| `RegressionDetectorTest` | All three confidence tiers, anti-oscillation, anti-cascade, anti-flaky guards |
| `ConfidenceScorerTest` | Composable weights — each signal contribution, clamping to [0, 1] |
| `HealthScoreTrackerTest` | Metric normalisation, weighted aggregation, delta computation over window |
| `ImprovementCircuitBreakerTest` | State transitions: CLOSED→OPEN, OPEN→HALF_OPEN, HALF_OPEN→CLOSED, manual reset |
| `ConflictDetectorTest` | File overlap, directory overlap, no overlap, trivial exemption |
| `CapabilityAreaRegistryTest` | Register, deprecate, active list, bootstrap |
| `ResearchPipelineOrchestratorTest` | Pipeline execution with mock SPIs at each depth tier |
| `InMemoryResearchCorpusTest` | Store, search, HIL queue lifecycle |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `ContinuousEvolutionIntegrationTest` | Full cycle: signal → goal → case → outcome → feedback → next goal. Verifies the loop closes. |
| `RegressionRollbackIntegrationTest` | Improvement causes regression → detector fires → rollback case spawned → outcome recorded |
| `CircuitBreakerIntegrationTest` | Health degrades → breaker opens → improvements blocked → health recovers → HALF_OPEN → CLOSED |
| `ConflictSerializationIntegrationTest` | Two improvements with overlapping paths → first executes, second queued → second re-introspects after first completes |

### Critical test scenarios

1. **Loop closure:** Outcome signals from improvement A feed back and contribute to consensus for improvement B
2. **Circuit breaker blocks improvements:** Health score drops → OPEN → `EvolutionTicker.tick()` returns without proposing goals
3. **High-confidence auto-revert:** CI failure after merge → confidence ≥ 0.9 → rollback case spawned
4. **Anti-oscillation:** Improvement reverted → same improvement re-proposed → CBR suppresses re-attempt
5. **Conflict queueing:** Two dependency updates in the same module → second queued → first completes → second re-introspects
6. **Trivial exemption:** Small fix (5 lines, 1 file) runs concurrently with broad refactor in same directory
7. **Evolution is opt-in:** Default `ImprovementConfig` → `evolutionEnabled == false` → ticker does nothing

## 14. References

- `2026-09-20-cognitive-self-improvement-vision.md` — parent vision spec, §Epic 5
- `2026-09-20-autonomous-self-improvement-engine-foundation.md` — #1114 engine foundation, §9 Extension Points
- `2026-09-20-continuous-improvement-methodology.md` — PRISMA, Technology Radar, Wardley Mapping, Horizon Scanning methodology
- Decisions D106–D115 in `decisions.md`
- `ImprovementGoalFormationStrategy.java` — existing goal formation
- `ImprovementBudgetEnforcer.java` — existing budget enforcement
- `ImprovementOutcomeEventCapture.java` — existing outcome tracking (3 layers)
- `ImprovementSignalProjector.java` — existing signal projection
- `ImprovementCbrProjector.java` — existing CBR projection
- `self-improvement.yaml` — existing case template
- `ActivityTracker` (D46) — per-case metrics infrastructure
- `SignalRegistry.consensusSignals()` — consensus detection
- Kitchenham & Charters (2007) — Systematic Literature Review guidelines
- PRISMA 2020 statement — systematic review reporting
- ThoughtWorks Technology Radar — technology maturity tracking
- Wardley, S. (2016) — Wardley Maps
- Christensen (1997) — The Innovator's Dilemma (sustaining vs disruptive innovation)
- Henderson & Clark (1990) — Architectural Innovation
- Ries (2011) — The Lean Startup (strategic pivot)
- Resilience4j — circuit breaker pattern
- arXiv:2507.21046 — Self-Evolving Agents Survey (data autophagy risk)
