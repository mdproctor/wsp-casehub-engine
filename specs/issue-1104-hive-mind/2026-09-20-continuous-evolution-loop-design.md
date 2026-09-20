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
│  │  EvolutionTicker (single entry point — event + timer)              │  │
│  │       │                                                            │  │
│  │       ▼                                                            │  │
│  │  ImprovementGoalFormationStrategy                                 │  │
│  │       │ ◄── ImprovementCategoryTracker (outcome feedback §2)      │  │
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
│  │       └─→ ImprovementCategoryTracker ────────────► (back to top)   │  │
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

1. `EvolutionTicker.tick()` fires — either from `CaseContextChangedEventHandler` (event-driven) or periodic timer (backstop)
2. Ticker checks: opt-in → health refresh → circuit breaker evaluate → block if OPEN
3. `ImprovementGoalFormationStrategy.proposeImprovements()` scans consensus, checks category suppression, anti-oscillation, budget, and conflict detection
4. Ticker calls `GoalFormationService.propose()` → standard goal lifecycle → improvement case spawned
5. Improvement case executes through existing lifecycle
6. `ImprovementOutcomeEventCapture` records outcome (5 layers: EventLog, signals, CBR, budget, category tracker)
7. `RegressionDetector` evaluates whether the outcome caused regression
8. `HealthScoreTracker` updates the rolling health score
9. `ImprovementCategoryTracker` modulates category priority → next evaluation cycle's `proposeImprovements()` consults `isSuppressed()`

## 1. Standing Directive — Hybrid Trigger Model

The continuous evolution loop uses two complementary trigger mechanisms that together ensure the improvement cycle runs without manual intervention.

### Event-driven re-entry (primary)

The existing `CaseContextChangedEventHandler` calls improvement proposals during the convergence detection phase. **This call must route through `EvolutionTicker.tick()`** — not call `proposeImprovements()` directly — so that all safety gates apply on every invocation path.

The convergence detection path changes from:

```java
// BEFORE (existing code, line 1496 — bypasses all safety gates)
var proposal = improvementStrategy.proposeImprovements(caseInstance.getUuid(), improvementConfig);
if (proposal != null && !proposal.goals().isEmpty() && goalFormationServiceInstance.isResolvable()) {
  goalFormationServiceInstance.get().propose(agentId, caseInstance.tenancyId, proposal);
}
```

to:

```java
// AFTER (#1115 — routes through the complete gate pipeline)
if (evolutionTickerInstance.isResolvable()) {
  evolutionTickerInstance.get().tick(
      caseInstance.getUuid(), caseInstance.tenancyId, improvementConfig);
}
```

Outcome signals projected by `ImprovementOutcomeEventCapture` (existing, D98) feed back into the signal registry and modulate category priority via `ImprovementCategoryTracker` (§2). They are **not** improvement requests and are not consumed by the consensus scan. The feedback path is: outcome → category priority updated → next evaluation cycle → `EvolutionTicker.tick()` → detection signals evaluated with adjusted priority.

### Timer backstop (secondary)

The ticker is also invoked periodically (configurable interval, default 60 minutes) as a backstop for cases where context changes are infrequent.

### EvolutionTicker — unified gate pipeline

`EvolutionTicker` is the **single entry point** for all improvement proposals. Both the event-driven path (convergence detection) and the timer backstop call `tick()`. This ensures every invocation passes through all safety gates.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class EvolutionTicker implements Resettable {

  private final ImprovementGoalFormationStrategy goalFormation;
  private final ImprovementCircuitBreaker circuitBreaker;
  private final HealthScoreTracker healthTracker;
  private final GoalFormationService goalFormationService;

  public void tick(UUID caseId, String tenancyId, ImprovementConfig config) {
    // Gate 1: opt-in check
    if (!config.effectiveEvolutionEnabled()) {
      return;
    }

    // Gate 2: refresh health score
    healthTracker.refresh(caseId, config.effectiveHealthPolicy());

    // Gate 3: evaluate circuit breaker (may trip OPEN based on health)
    circuitBreaker.evaluate(caseId, healthTracker, config.effectiveHealthPolicy());

    // Gate 4: block if circuit breaker is OPEN
    if (circuitBreaker.state(caseId) == CircuitBreakerState.OPEN) {
      return;
    }

    // Gate 5: propose improvements
    // (internally: consensus → category suppression → anti-oscillation
    //  → budget check → conflict check)
    var proposal = goalFormation.proposeImprovements(caseId, config);
    if (proposal == null || proposal.goals().isEmpty()) {
      return;
    }

    // Gate 6: enter the standard goal lifecycle
    goalFormationService.propose("improvement-system", tenancyId, proposal);
  }
}
```

**Gate pipeline (complete and canonical):**

```
evolutionEnabled → health refresh → circuit breaker evaluate → circuit breaker check
  → consensus scan → category suppression → anti-oscillation → budget check
  → conflict check → GoalFormationService.propose()
```

Gates 1–4 live in `EvolutionTicker`. Gates 5–9 are internal to `proposeImprovements()`. Gate 10 uses `GoalFormationService.propose()` — the same standard goal lifecycle path used by the existing convergence detection code. This produces `GOAL_FORMED` and `GOAL_PROPOSED` EventLog entries and enters the standard case template binding via `SubCaseBinding`.

**No parallel spawning mechanism.** The ticker does NOT call `spawnImprovementCase()` or any custom case-spawning method. It produces a `GoalFormationProposal` and delegates to `GoalFormationService.propose()` — the same API the convergence detection handler already uses.

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

### Two signal types — detection vs outcome

**Detection signals** (e.g., `improvement:dependency:staleness:*`) are deposited by agents that observe quality issues. They carry `ImprovementRequest` context via `ImprovementSignalContext` and are consumed by `proposeImprovements()` through the consensus scan → `signalContext.get()` lookup path.

**Outcome signals** (e.g., `improvement:outcome:positive:pr-merged`) are deposited by `ImprovementSignalProjector` after an improvement completes. They do NOT carry `ImprovementRequest` context and are NOT consumed by the consensus scan. They serve a different purpose: modulating the priority of future detection signals in the same category.

The existing `proposeImprovements()` code correctly skips outcome signals — `signalContext.get()` returns empty for them. This is by design, not a gap. Outcome signals close the loop through category priority modulation, not through direct re-proposal.

### Feedback paths

| Outcome status | Signal projected | Effect on next cycle |
|---------------|-----------------|---------------------|
| MERGED | `improvement:outcome:positive:pr-merged` | Category priority boost — `ImprovementCategoryTracker` raises category weight |
| REJECTED | `improvement:outcome:rejected` | Category priority reduction — repeated rejections suppress the category |
| REGRESSION | `improvement:outcome:regression:*` | Triggers `RegressionDetector` (§3) — may pause category |
| FAILED | `improvement:outcome:failed` | Enriches CBR trace — "what went wrong?" informs future attempts |
| ABANDONED | `improvement:outcome:abandoned` | Clears related detection signals — direction was abandoned |

### ImprovementCategoryTracker

Tracks per-category outcome history and computes priority modulation:

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementCategoryTracker implements Resettable {

  public record CategoryState(
      int successCount, int failureCount, int rejectionCount,
      Instant lastOutcome, boolean paused, @Nullable Instant pausedUntil) {}

  private final ConcurrentHashMap<String, ConcurrentHashMap<String, CategoryState>> states =
      new ConcurrentHashMap<>();  // caseId → category → state

  public void recordOutcome(UUID caseId, String category, ImprovementOutcome.OutcomeStatus status) {
    // Update counts based on outcome status
  }

  public boolean isSuppressed(UUID caseId, String category) {
    // 3+ consecutive failures → suppress
    // 3 rejections in window → suppress
    // Category paused by RegressionDetector → suppress
  }

  public void pauseCategory(UUID caseId, String category, Duration duration) {
    // Pauses new improvements in this category for the specified duration
    // Safety-critical: used by RegressionDetector on medium/high confidence regression
  }

  public void unpauseCategory(UUID caseId, String category) {
    // Manual unpause — also called when pause duration expires
  }
}
```

### Enhanced proposeImprovements() — complete internal gate pipeline

`proposeImprovements()` now owns all proposal-level gates. It needs new dependencies: `ImprovementCategoryTracker`, `RollbackHistory`, and `ConflictDetector`. The full internal pipeline:

```java
public GoalFormationProposal proposeImprovements(UUID caseId, ImprovementConfig config) {
  // ... consensus scan (existing) ...
  for (var entry : consensus.entrySet()) {
    // ... namespace filter, signalContext lookup (existing) ...
    ImprovementRequest request = ctxOpt.get();

    // Gate: category suppression (NEW — from ImprovementCategoryTracker)
    if (categoryTracker.isSuppressed(caseId, request.category())) {
      continue;
    }

    // Gate: anti-oscillation (NEW — from RollbackHistory, then CBR)
    if (rollbackHistory.wasRecentlyRolledBack(caseId, request.category(),
        request.target(), Duration.ofMinutes(
            config.effectiveRollbackPolicy().effectiveRegressionWindowMinutes()))) {
      continue;
    }

    // Gate: budget check (existing)
    var budgetCheck = budgetEnforcer.check(caseId, config.effectiveBudget(), request);
    if (budgetCheck instanceof BudgetCheck.Denied) { continue; }

    // Gate: conflict detection (NEW — from ConflictDetector)
    var conflictCheck = conflictDetector.check(
        request, budgetEnforcer.activeImprovementRequests(caseId),
        config.effectiveConflictTrivialThreshold());
    if (conflictCheck instanceof ConflictCheck.Conflicting) { continue; }

    // ... build ProposedGoal (existing) ...
  }
}
```

**Key design decisions:**

- **Conflict detection is inside `proposeImprovements()`**, not in the ticker. This is where the `ImprovementRequest` object is available — the ticker never sees request objects, only `GoalFormationProposal` and `ProposedGoal`.
- **Conflicting proposals are skipped, not queued.** They will be re-evaluated on the next tick when the blocking improvement completes and the `budgetEnforcer.activeImprovementRequests()` no longer contains it. This is simpler than maintaining a separate queue with dequeue triggers.
- **`ImprovementBudgetEnforcer.activeImprovementRequests()`** returns `Map<UUID, ImprovementRequest>` — the enforcer is enhanced to store `ImprovementRequest` alongside timestamps in `recordStart()`:

```java
// Enhanced recordStart — stores request for conflict detection
public void recordStart(UUID improvementCaseId, ImprovementRequest request) {
  activeImprovements.put(improvementCaseId, request);
  dailyCounts.computeIfAbsent(LocalDate.now(ZoneOffset.UTC), k -> new AtomicInteger(0))
      .incrementAndGet();
}

public Map<UUID, ImprovementRequest> activeImprovementRequests(UUID caseId) {
  return Map.copyOf(activeImprovements);
}
```

### Outcome-driven suppression rules

`ImprovementCategoryTracker.isSuppressed()` applies:

- **Repeated failures:** 3+ FAILED outcomes in the same category within the health window → category suppressed until manual reset or health window expires.
- **Repeated rejections:** 3 REJECTED PRs in the same improvement direction → category suppressed and emit `improvement:outcome:abandoned`.
- **Regression:** Any REGRESSION outcome → `RegressionDetector` (§3) handles via `ImprovementCategoryTracker.pauseCategory()`.

### Outcome capture wiring

`ImprovementCategoryTracker` is wired into the existing `ImprovementOutcomeEventCapture` as a fifth layer:

```java
// ImprovementOutcomeEventCapture — #1115 addition
// Add ImprovementCategoryTracker as a constructor dependency

public void onImprovementComplete(@ObservesAsync ImprovementCaseCompleted event) {
  var outcome = event.outcome();
  outcomeRecorder.record(event.caseId(), event.tenancyId(), outcome);     // Layer 1: EventLog
  signalProjector.project(event.caseId(), outcome);                       // Layer 2: Signals
  cbrProjector.project(event.tenancyId(), outcome);                       // Layer 3: CBR
  budgetEnforcer.recordCompletion(outcome.improvementCaseId());            // Layer 4: Budget
  categoryTracker.recordOutcome(event.caseId(),                           // Layer 5: Category
      outcome.category(), outcome.status());
}
```

Without this wiring, `ImprovementCategoryTracker` never sees outcomes — it never increments success/failure/rejection counts, never suppresses categories on repeated failures, and the feedback loop remains open for non-regression outcomes.

**Note:** `GoalRevisionEvaluator` operates within the eidos agent goal system (`AgentDescriptor`, `AgentGoal`, `GoalEvolution`) and has no concept of `GoalKind` or improvement-specific logic. Improvement outcome processing lives entirely in `ImprovementCategoryTracker` and `ImprovementGoalFormationStrategy`.

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
  private final ImprovementCategoryTracker categoryTracker;
  private final RollbackHistory rollbackHistory;
  private final SignalRegistry signalRegistry;

  public void evaluate(UUID caseId, ImprovementOutcome outcome) {
    if (outcome.status() != ImprovementOutcome.OutcomeStatus.MERGED) {
      return;
    }

    // Monitor metrics within the regression window
    // (invoked periodically after merge, not just once)
  }

  public void onMetricsDegraded(
      UUID caseId, UUID improvementCaseId, String category,
      RollbackPolicy policy, HealthSnapshot before, HealthSnapshot after) {
    double confidence = scorer.score(caseId, improvementCaseId, before, after);

    if (confidence >= policy.effectiveAutoRevertThreshold()) {
      spawnRollbackCase(caseId, improvementCaseId, confidence);
      categoryTracker.pauseCategory(caseId, category,
          Duration.ofMinutes(policy.effectiveRegressionWindowMinutes()));
      rollbackHistory.record(caseId, improvementCaseId, category);
    } else if (confidence >= policy.effectivePauseThreshold()) {
      emitRegressionSignal(caseId, improvementCaseId, confidence);
      categoryTracker.pauseCategory(caseId, category,
          Duration.ofMinutes(policy.effectiveRegressionWindowMinutes()));
    } else {
      emitRegressionSignal(caseId, improvementCaseId, confidence);
    }
  }
}
```

### ConfidenceScorer

Composable signals with additive weights. All data derives from EventLog and case context — no external CI system query required.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ConfidenceScorer {

  private final EventLogRepository eventLogRepository;
  private final CaseContextReader contextReader;

  public double score(
      UUID caseId, UUID improvementCaseId,
      HealthSnapshot before, HealthSnapshot after) {
    double confidence = 0.0;

    if (improvementCiBuildFailed(improvementCaseId)) {
      confidence += 0.5;
    }
    if (failingTestsTouchModifiedFiles(improvementCaseId, after)) {
      confidence += 0.3;
    }
    if (regressionWithinWindow(before, after)) {
      confidence += 0.2;
    }
    if (cbrShowsSimilarRegressions(improvementCaseId)) {
      confidence += 0.1;
    }
    if (multipleAreasDegraded(before, after)) {
      confidence += 0.1;
    }
    if (regressionInUnrelatedArea(improvementCaseId, before, after)) {
      confidence -= 0.2;
    }
    if (otherImprovementsMergedInWindow(caseId, improvementCaseId)) {
      confidence -= 0.3;
    }

    return Math.max(0.0, Math.min(1.0, confidence));
  }
}
```

**Data sources for each signal:**

| Signal method | Data source | How |
|--------------|------------|-----|
| `improvementCiBuildFailed` | Case context of the improvement case | The `integrate` worker records CI outcome as `context.layer('WORKING').put('ciOutcome', ...)`. Query via `CaseContextReader`. |
| `failingTestsTouchModifiedFiles` | Case context | Correlates `failingTests` (from CI outcome) with `affectedPaths` from `IntrospectionResult` stored in working context by the `introspect` worker. |
| `regressionWithinWindow` | `HealthSnapshot` params | Pure comparison: `after.score() < before.score()` — the before/after snapshots are captured around the merge window by `RegressionDetector`. |
| `cbrShowsSimilarRegressions` | CBR retriever (optional) | Queries neocortex for similar past regression patterns. Returns `false` without neocortex (conservative — doesn't add confidence). |
| `multipleAreasDegraded` | `HealthSnapshot` params | Compares `before.componentScores()` vs `after.componentScores()` — counts areas where score dropped. Pure data comparison, no external query. |
| `regressionInUnrelatedArea` | `HealthSnapshot` + case context | Checks if degraded areas in `after.componentScores()` are unrelated to the improvement's target area (from case context). Reduces confidence — regression in an unrelated area suggests another cause. |
| `otherImprovementsMergedInWindow` | EventLog | Queries for `IMPROVEMENT_OUTCOME` events with MERGED status within the regression window, excluding the current improvement. Other concurrent merges reduce causal confidence. |

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

### Revert worker

The `improvement-revert` capability is provided by `ImprovementRevertWorker`:

```java
// runtime-core, io.casehub.engine.internal.improvement.worker
@ApplicationScoped
public class ImprovementRevertWorker {
  // Creates a git revert commit of the improvement's merge commit
  // via the same REST/GraphQL API surface used by ImprovementIntegrateWorker.
  //
  // If the revert has merge conflicts (target code changed since the
  // improvement landed), the worker returns WorkerOutcome.Failed with
  // reason "revert-conflict" and emits a signal for human review.
  // It does NOT attempt automatic conflict resolution.
}
```

| Aspect | Detail |
|--------|--------|
| Module | `runtime-core` |
| Package | `io.casehub.engine.internal.improvement.worker` |
| Capability | `improvement-revert` |
| Conflict handling | Fails with `revert-conflict` — escalates to human review |

### RollbackHistory (engine-only anti-oscillation fallback)

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class RollbackHistory implements Resettable {

  public record RollbackRecord(UUID improvementCaseId, String category,
      String target, Instant rolledBackAt) {}

  private final ConcurrentHashMap<UUID, List<RollbackRecord>> history =
      new ConcurrentHashMap<>();

  public void record(UUID caseId, UUID improvementCaseId, String category) { ... }

  public boolean wasRecentlyRolledBack(UUID caseId, String category,
      String target, Duration window) {
    // Returns true if the same category+target was rolled back within the window
  }
}
```

This provides the fail-closed anti-oscillation guard when CBR (neocortex) is unavailable. `ImprovementGoalFormationStrategy` checks `RollbackHistory` first (always available), then CBR (when available, provides richer context matching). If either indicates a recent rollback of the same target and context hasn't materially changed, the proposal is suppressed.

### Safety guards

**Anti-oscillation:** Before proposing an improvement, `ImprovementGoalFormationStrategy` checks:
1. `RollbackHistory.wasRecentlyRolledBack()` (always available — engine-only fallback)
2. CBR retrieval for similar past rollbacks (when neocortex is available — richer matching)
If either returns a match and context hasn't materially changed, the proposal is suppressed. The engine-only fallback is fail-closed — without rollback history, the system defaults to allowing (history starts empty, fills on first rollback). With rollback history present, proposals for recently-rolled-back targets are blocked.

**Anti-cascade:** Before spawning a rollback case, `RegressionDetector` checks `ConflictDetector` (§5) for later improvements that depend on the regressed change. If dependencies exist, downgrade to signal + pause — don't auto-revert.

**Anti-flaky:** `RegressionDetector` requires `sustainedFailureCount` (default 2) consecutive failures before acting. A single flaky test run does not trigger rollback.

### Structural deny list update

The following new safety-critical components must be added to `ImprovementBudgetEnforcer.STRUCTURAL_DENIED_PATTERNS`:

```java
private static final Set<String> STRUCTURAL_DENIED_PATTERNS =
    Set.of(
        // Existing (#1114)
        "ImprovementBudget",
        "ImprovementBudgetEnforcer",
        "ImprovementConfig",
        "SafetyConfig",
        "improvement-case-template",
        // New (#1115) — safety-critical components
        "EvolutionTicker",
        "ImprovementCircuitBreaker",
        "RegressionDetector",
        "ConfidenceScorer",
        "HealthScoreTracker",
        "HealthPolicy",
        "RollbackPolicy",
        "ConflictDetector",
        "ImprovementCategoryTracker",
        "RollbackHistory",
        "self-improvement-rollback"
    );
```

The improvement system must not be able to modify its own safety constraints — this is the foundational safety invariant. Any component involved in gating, detecting, or responding to improvement failures is structurally denied.

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
  private final CapabilityAreaRegistry areaRegistry;

  public double computeScore(UUID caseId, HealthPolicy policy) {
    // Aggregate health from capability area assessments
    // Each CapabilityArea.assess() returns healthScore in [0, 1]
    var weights = policy.effectiveWeights();
    double weightedSum = 0.0;
    double totalWeight = 0.0;
    for (CapabilityArea area : areaRegistry.active()) {
      var assessment = area.assess(caseId);
      double weight = weights.getOrDefault(area.id(), 0.1);
      weightedSum += weight * assessment.healthScore();
      totalWeight += weight;
    }
    // Normalise by total weight to guarantee [0, 1] regardless of custom weights
    return totalWeight > 0 ? weightedSum / totalWeight : 0.0;
  }

  public void refresh(UUID caseId, HealthPolicy policy) {
    double score = computeScore(caseId, policy);
    Map<String, Double> components = new LinkedHashMap<>();
    for (CapabilityArea area : areaRegistry.active()) {
      components.put(area.id(), area.assess(caseId).healthScore());
    }
    var snapshot = new HealthSnapshot(score, Instant.now(), components);
    history.computeIfAbsent(caseId, k -> new ArrayDeque<>()).addLast(snapshot);
    // Trim history to bounded window
  }

  public double delta(UUID caseId, int windowMinutes) {
    // Current score minus score at windowMinutes ago
  }
}
```

**No parallel metrics infrastructure.** Health scoring consumes `CapabilityArea.assess()` — the SPI that already exists for each area. Each area's `assess()` implementation derives its `healthScore` from available data (EventLog entries, ActivityTracker state, signal registry). External metrics (CI pass rate, test coverage, build time) are provided by capability area implementations that query external systems — not by a separate metrics pipeline. The `MetricsSnapshot` record is removed.
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
    // Default weights keyed by capability area ID from the bootstrap areas (§6)
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

### Circuit breaker signals

| Signal | When |
|--------|------|
| `improvement:circuit-breaker:tripped` | CLOSED → OPEN |
| `improvement:circuit-breaker:recovering` | OPEN → HALF_OPEN |
| `improvement:circuit-breaker:reset` | HALF_OPEN → CLOSED (or manual reset) |

All transitions produce an `EventLog` entry with the health score, delta, and triggering metrics.

### Restart recovery — event-sourced state reconstruction

The circuit breaker state is safety-critical — it must survive restarts. On startup, `ImprovementCircuitBreaker` reconstructs its state from the most recent `CIRCUIT_BREAKER_*` EventLog entry:

```java
public void restoreFromEventLog(UUID caseId, EventLogRepository repository) {
  // Query for most recent improvement:circuit-breaker:* event for this case
  // Reconstruct state, halfOpenCount, and health baseline from event payload
  // If no events found, defaults to CLOSED (clean start)
}
```

`HealthScoreTracker` history is NOT persisted — it rebuilds naturally as `refresh()` is called. The first few ticks after restart may have insufficient history for accurate delta computation; the circuit breaker treats missing history as "no trend data" (delta = 0), which is conservative (won't trip on delta alone).

`ImprovementCategoryTracker` pause state is reconstructed from EventLog entries for regression events. Active pauses with remaining duration are restored; expired pauses are ignored.

### Trade-off: lagging indicator

The health score is a lagging indicator — by the time it drops below threshold, multiple problematic improvements may have landed. The delta threshold helps (catches trends earlier than the absolute threshold) but still lags. The weights need tuning from real deployment data; initial defaults are heuristic. Over-sensitive thresholds cause frequent false trips that block legitimate improvements. The rollback system (§3) handles individual regressions; the circuit breaker handles aggregate drift that rollback can't catch.

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

### Conflict handling — skip, not queue

Conflicting proposals are **skipped** during `proposeImprovements()`, not queued. When the blocking improvement completes and is removed from `budgetEnforcer.activeImprovementRequests()`, the next evaluation cycle's `proposeImprovements()` will re-evaluate and the conflict check will pass. This is simpler than maintaining a separate queue with dequeue triggers — the consensus model already provides natural re-evaluation.

### Trivial change exemption

Improvements with `estimatedSize <= trivialThreshold` (default 10 lines) touching a single file are exempt from directory-level conflict detection. They still check file-level overlap. This prevents a large refactor from blocking all small fixes in the same module.

### Integration with the evolution pipeline

`ConflictDetector.check()` is called **inside** `proposeImprovements()` — it's the last gate before a proposal is included. This is where the `ImprovementRequest` object is available. The ticker never sees request objects; it works with `GoalFormationProposal` and delegates to `GoalFormationService.propose()`. See §1 for the complete gate pipeline and §2 for the enhanced `proposeImprovements()` code.

### Conflict scope trade-off

The path-based approach (file and directory level) is intentionally simple and build-tool agnostic. It has known trade-offs:

- **False positives (safe):** Two independent lint fixes in the same directory are serialized even though they don't conflict. This is conservative — slower but safe.
- **False negatives (rare):** A dependency update (`pom.xml`) and a lint fix in a deep directory won't trigger directory overlap. However, root-level files like `pom.xml` are typically caught by file-level overlap since multiple dependency updates would target the same file.

Module-level conflict detection (Maven module awareness) would reduce false positives but introduces build-tool coupling. The `IntrospectionResult.affectedPaths()` could be enhanced with module scope information in a future iteration, allowing the conflict detector to operate at module granularity when available and fall back to path-based matching otherwise.

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

### Trigger mechanism

The research pipeline is triggered by `EvolutionTicker` when:
1. A capability area assessment shows `LandscapePosition.BEHIND` or `ABSENT`
2. The area's last assessment is older than `ResearchMethodology.effectiveAreaRefreshStalenessThresholdDays()` (default: 30 days)
3. The circuit breaker is not OPEN

```java
// In EvolutionTicker, after improvement proposals but before return:
if (circuitBreaker.state(caseId) != CircuitBreakerState.OPEN) {
  for (CapabilityArea area : areaRegistry.active()) {
    var assessment = area.assess(caseId);
    if (isStaleOrBehind(assessment, config.effectiveResearchMethodology())) {
      var hypotheses = researchPipeline.execute(
          selectDepth(assessment), assessment, driveContext(caseId));
      for (var hypothesis : hypotheses) {
        depositHypothesisSignal(caseId, hypothesis);
      }
    }
  }
}
```

Research hypotheses are deposited as `improvement:capability:*` signals. They enter the normal consensus → goal formation path — no special wiring. Multiple independent research cycles reinforcing the same hypothesis builds consensus for a capability improvement.

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

### Backwards-compatible constructor

`ImprovementConfig` expands from 5 to 11 fields. A backwards-compatible constructor is provided for existing call sites:

```java
public ImprovementConfig(
    @Nullable String signalNamespace,
    @Nullable Integer consensusMinSources,
    @Nullable List<String> enabledCategories,
    @Nullable ImprovementBudget budget,
    @Nullable String caseTemplateId) {
  this(signalNamespace, consensusMinSources, enabledCategories, budget, caseTemplateId,
       null, null, null, null, null, null);
}
```

This follows the same pattern as `StigmergyConfig` (3-arg constructor that passes `null` for improvement).

### YAML record codegen

The new nested records require YAML codegen entries (following the #1114 pattern for `ImprovementConfig` and `ImprovementBudget`):

| Record | Codegen entry |
|--------|--------------|
| `RollbackPolicy` | `io.casehub.api.model.stigmergy.RollbackPolicy` |
| `HealthPolicy` | `io.casehub.api.model.stigmergy.HealthPolicy` |
| `ResearchMethodology` | `io.casehub.api.model.stigmergy.ResearchMethodology` |

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
| `GapMap` | `api` | `io.casehub.api.model.stigmergy` |
| `ImprovementCategoryTracker` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `RollbackHistory` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementRevertWorker` | `runtime-core` | `io.casehub.engine.internal.improvement.worker` |
| `ResearchDepth` | `api` | `io.casehub.api.spi.improvement` |
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
| `EvolutionTickerTest` | Complete gate pipeline: opt-in guard, health refresh, circuit breaker, goal formation, conflict detection |
| `RegressionDetectorTest` | All three confidence tiers, anti-cascade, anti-flaky guards, category pause via tracker |
| `ConfidenceScorerTest` | Composable weights — each signal contribution, clamping to [0, 1] |
| `HealthScoreTrackerTest` | Capability area-based aggregation, weight normalisation, delta computation over window |
| `ImprovementCircuitBreakerTest` | State transitions: CLOSED→OPEN, OPEN→HALF_OPEN, HALF_OPEN→CLOSED, manual reset, event-sourced recovery |
| `ConflictDetectorTest` | File overlap, directory overlap, no overlap, trivial exemption |
| `CapabilityAreaRegistryTest` | Register, deprecate, active list, bootstrap |
| `ResearchPipelineOrchestratorTest` | Pipeline execution with mock SPIs at each depth tier |
| `InMemoryResearchCorpusTest` | Store, search, HIL queue lifecycle |
| `ImprovementCategoryTrackerTest` | Outcome tracking, suppression logic (3+ failures, 3 rejections), pause/unpause |
| `RollbackHistoryTest` | Record, lookup, window-based expiry |
| `ImprovementRevertWorkerTest` | Revert commit creation, conflict detection/escalation |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `ContinuousEvolutionIntegrationTest` | Full cycle: signal → goal → case → outcome → feedback → next goal. Verifies the loop closes. |
| `RegressionRollbackIntegrationTest` | Improvement causes regression → detector fires → rollback case spawned → outcome recorded |
| `CircuitBreakerIntegrationTest` | Health degrades → breaker opens → improvements blocked → health recovers → HALF_OPEN → CLOSED |
| `ConflictSerializationIntegrationTest` | Two improvements with overlapping paths → first executes, second queued → second re-introspects after first completes |

### Critical test scenarios

1. **Loop closure:** Outcome from improvement A → `ImprovementCategoryTracker` boosts category → detection signals reach consensus → improvement B proposed
2. **Circuit breaker blocks improvements:** Health score drops → OPEN → `EvolutionTicker.tick()` returns without proposing goals
3. **High-confidence auto-revert:** CI failure after merge → confidence ≥ 0.9 → rollback case spawned → category paused
4. **Anti-oscillation (engine-only):** Improvement reverted → `RollbackHistory.record()` → same improvement re-proposed → suppressed by `wasRecentlyRolledBack()`
5. **Anti-oscillation (with CBR):** Same as above but CBR provides richer context matching for similar (not exact) targets
6. **Conflict queueing:** Two dependency updates in the same module → second queued → first completes → second re-introspects
7. **Trivial exemption:** Small fix (5 lines, 1 file) runs concurrently with broad refactor in same directory
8. **Evolution is opt-in:** Default `ImprovementConfig` → `evolutionEnabled == false` → ticker does nothing
9. **Category suppression:** 3 failures in dependency-update → category suppressed → no proposals until window expires or manual reset
10. **Circuit breaker restart recovery:** Process restarts with OPEN circuit breaker → EventLog entry → state reconstructed → improvements still blocked

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
