# Evolution Loop Gap Closure — Design Spec

**Issue:** casehubio/engine#1115
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md`
**Gap analysis:** `specs/issue-1115-continuous-evolution-loop/gap-analysis.md`
**Decisions:** D1–D5 in `decisions.md`
**Date:** 2026-10-26

## Problem

The 2026-09-20 design spec is comprehensive and sound. All major components are
implemented and pass their unit tests. However, the production CDI wiring is
disconnected — the loop is open. Components work in isolation but are not connected
in the runtime graph.

This spec addresses every gap identified in the gap analysis. It is an addendum to
the parent spec, not a replacement.

---

## 1. Wire EvolutionTicker into Convergence Detection (C1)

### Current state

`CaseContextChangedEventHandler.convergenceDetection()` calls
`improvementStrategy.proposeImprovements()` directly (line ~1498), bypassing the
EvolutionTicker's gate pipeline.

### Change

Replace the direct call with a ticker invocation. In `RuntimeBeans.java`, inject
`EvolutionTicker` into the `CaseContextChangedEventHandler` producer. In the handler,
replace:

```java
// BEFORE — bypasses all safety gates
var proposal = improvementStrategy.proposeImprovements(
    caseInstance.getUuid(), caseInstance.tenancyId(), improvementConfig);
if (proposal != null && !proposal.goals().isEmpty()
    && goalFormationServiceInstance.isResolvable()) {
  goalFormationServiceInstance.get().propose(
      agentId, caseInstance.tenancyId(), proposal);
}
```

with:

```java
// AFTER — routes through complete gate pipeline
if (evolutionTickerInstance.isResolvable()) {
  evolutionTickerInstance.get().tick(
      caseInstance.getUuid(), caseInstance.tenancyId(), improvementConfig,
      TickTrace.TickTrigger.EVENT_DRIVEN);
}
```

The `tick()` method signature adds a `TickTrigger` parameter so traces distinguish
event-driven from timer-triggered ticks:

```java
public TickTrace tick(UUID caseId, String tenancyId,
    ImprovementConfig config, TickTrigger trigger)
```

### Injection change

`CaseContextChangedEventHandler` gains a new constructor parameter:

```java
Instance<EvolutionTicker> evolutionTickerInstance
```

`ImprovementGoalFormationStrategy` and `GoalFormationService` are still needed for
non-evolution improvement paths (manual goal proposals). The handler only routes
through the ticker when evolution is configured.

---

## 2. Timer Backstop (C2)

### New class

```java
// runtime-core, io.casehub.engine.runtime.improvement
@ApplicationScoped
public class EvolutionTimerBackstop {

  private final CaseInstanceRepository caseRepository;
  private final EvolutionTicker ticker;
  private final StigmergyConfigReader configReader;

  @Scheduled(every = "{casehub.evolution.timer-interval:60m}")
  void tick() {
    // Find all cases with evolutionEnabled=true
    // For each, call ticker.tick(caseId, tenancyId, config, TIMER)
  }
}
```

The interval is configurable via Quarkus config property
`casehub.evolution.timer-interval` (default 60 minutes). The timer iterates over
active cases with evolution enabled and ticks each one.

**Guard:** The ticker's own `evolutionEnabled` gate (Gate 1) prevents double-processing
if the timer fires during an event-driven tick. The budget enforcer's concurrent limit
prevents duplicate improvement cases.

---

## 3. Wire ImprovementOutcomeEventCapture (C3)

### Current state

The class exists with all 6 layers wired internally but is not a CDI bean.
`ImprovementCaseCompleted` events are never fired.

### Changes

**3a. Make ImprovementOutcomeEventCapture a CDI observer:**

Add `@ApplicationScoped` to the class. Change `onImprovementComplete` to observe
the CDI event:

```java
@ApplicationScoped
public class ImprovementOutcomeEventCapture {

  public void onImprovementComplete(
      @ObservesAsync ImprovementCaseCompleted event) {
    // ... existing 6-layer capture logic ...
  }
}
```

**3b. Fire ImprovementCaseCompleted from the outcome worker:**

The `improvement-outcome` capability worker (`ImprovementOutcomeRecorder`) fires
the event after recording the outcome:

```java
// In the worker's execute() method, after recording:
event.fireAsync(new ImprovementCaseCompleted(
    caseId, tenancyId, outcome));
```

This requires injecting `Event<ImprovementCaseCompleted>` into the outcome worker.

**3c. Fix the CBR projector (part of M1):**

`ImprovementCbrProjector.project()` currently just logs. Change to write to CBR
RetrieverService when available:

```java
@ApplicationScoped
public class ImprovementCbrProjector {

  private final Instance<CbrStorageService> cbrStorage;

  public void project(String tenancyId, ImprovementOutcome outcome) {
    if (!cbrStorage.isResolvable()) {
      return;  // CBR unavailable — graceful no-op
    }
    cbrStorage.get().store(tenancyId, "improvement-outcomes",
        buildCbrCase(outcome));
  }
}
```

---

## 4. Rollback Case Spawning (H1)

### New event

```java
// api/model/stigmergy
public record RollbackRequested(
    UUID caseId,
    String tenancyId,
    UUID improvementCaseId,
    String category,
    String target,
    List<String> targetPaths,
    double confidence) {}
```

### RegressionDetector change

In `onMetricsDegraded()`, when confidence >= autoRevertThreshold, fire the event
in addition to recording and pausing:

```java
if (confidence >= policy.effectiveAutoRevertThreshold()) {
  rollbackHistory.record(caseId, monitor.improvementCaseId(),
      monitor.category(), monitor.target());
  categoryTracker.pauseCategory(caseId, monitor.category(),
      Duration.ofMinutes(policy.effectiveRegressionWindowMinutes()));
  // NEW — fire rollback request
  rollbackRequestedEvent.fireAsync(new RollbackRequested(
      caseId, tenancyId, monitor.improvementCaseId(),
      monitor.category(), monitor.target(),
      monitor.targetPaths(), confidence));
}
```

### New handler

```java
// runtime-core, io.casehub.engine.runtime.improvement
@ApplicationScoped
public class RollbackCaseSpawner {

  private final GoalFormationService goalFormationService;
  private final BudgetEnforcer budgetEnforcer;

  public void onRollbackRequested(
      @ObservesAsync RollbackRequested event) {
    // Anti-cascade check (M2): query active improvements for path overlap
    // event.targetPaths() carries the original improvement's file paths
    var activeRequests = budgetEnforcer.activeImprovementRequests();
    if (hasOverlappingActiveImprovement(event.targetPaths(), activeRequests)) {
      // Downgrade: signal + pause only, don't auto-revert
      return;
    }

    // Create rollback case from template
    var proposal = GoalFormationProposal.of(
        List.of(ProposedGoal.builder()
            .name("rollback-" + event.improvementCaseId())
            .description("Revert regressed improvement")
            .caseTemplateId("self-improvement-rollback")
            .context(Map.of(
                "originalImprovementCaseId",
                    event.improvementCaseId().toString(),
                "category", event.category(),
                "target", event.target(),
                "confidence", String.valueOf(event.confidence())))
            .build()));
    goalFormationService.propose(
        "regression-detector", event.tenancyId(), proposal);
  }

  private boolean hasOverlappingActiveImprovement(
      RollbackRequested event,
      Map<UUID, ImprovementRequest> activeRequests) {
    // Reuse FilePathConflictStrategy path-overlap logic
    // Check if any active improvement targets overlap with
    // the improvement being rolled back
  }
}
```

---

## 5. Circuit Breaker Restart Recovery (H2)

### Change

In `EvolutionBootstrap`, after registering SPI providers, restore circuit breaker
state for all active evolution cases:

```java
// In EvolutionBootstrap startup handler, after SPI registration:
for (var caseInstance : caseRepository.findWithEvolutionEnabled()) {
  circuitBreaker.restoreFromEventLog(
      caseInstance.getUuid(), caseInstance.tenancyId());
}
```

This requires `EvolutionBootstrap` to receive `ImprovementCircuitBreaker` and
`CaseInstanceRepository` as constructor parameters.

**Startup ordering:** EvolutionBootstrap runs at StartupEvent priority 200
(after CaseHubClassRegistrar at 150). Circuit breaker restoration must complete
before the first ticker invocation. Since the timer backstop uses `@Scheduled`
(which fires after startup completes), ordering is safe.

---

## 6. Structural Deny List Update (H3)

Add `RegressionDetector` to `CodeEvolutionDenyPatternProvider.STRUCTURAL_DENIED_PATTERNS`:

```java
private static final Set<String> STRUCTURAL_DENIED_PATTERNS = Set.of(
    // ... existing entries ...
    "RegressionDetector"
);
```

---

## 7. CBR-Informed Goal Formation (M1 — read side)

In `ImprovementGoalFormationStrategy.proposeImprovements()`, after the
anti-oscillation filter and before the deny filter, add an optional CBR check:

```java
// Optional CBR-informed priority reduction
if (cbrRetriever.isResolvable()) {
  var similar = cbrRetriever.get().retrieve(tenancyId,
      "improvement-outcomes",
      FeatureVectorCbrCase.builder()
          .feature("category", FeatureValue.categorical(request.category()))
          .feature("target", FeatureValue.text(request.target()))
          .build(),
      5);
  double failureRate = computeFailureRate(similar);
  if (failureRate > 0.6) {
    continue;  // Skip — similar improvements have high failure rate
  }
}
```

When CBR is unavailable (`!cbrRetriever.isResolvable()`), this step is skipped
entirely. The engine-only fallback (RollbackHistory + CategoryTracker) remains
sufficient.

---

## 8. ProposalFilteringSummary (M3)

In `ImprovementGoalFormationStrategy.proposeImprovements()`, track counts at each
filter stage and return them alongside the proposal:

```java
public record ProposalResult(
    GoalFormationProposal proposal,
    ProposalFilteringSummary filtering) {}
```

The ticker uses the summary when building the TickTrace:

```java
var result = goalFormation.proposeImprovementsWithSummary(
    caseId, tenancyId, config);
// ... build TickTrace with result.filtering() ...
```

This requires changing the return type of the proposal method. The existing
`proposeImprovements()` signature is preserved for backwards compatibility;
`proposeImprovementsWithSummary()` is the new method that returns the richer type.

---

## 9. YAML Codegen Entries (M4)

Add to `yaml-record-mappings.yaml`:

```yaml
io.casehub.api.model.stigmergy.RollbackPolicy:
  yamlName: YamlRollbackPolicy

io.casehub.api.model.stigmergy.HealthPolicy:
  yamlName: YamlHealthPolicy

io.casehub.api.model.stigmergy.ResearchMethodology:
  yamlName: YamlResearchMethodology
```

Fields match the record constructors directly — no aliases or type overrides needed.

---

## 10. DevTown Improvement Workers (M5)

Five workers in the devtown repo, one per category. Each is a CDI bean implementing
the improvement worker capability for the "apply" stage.

| Worker | Category | Service Call |
|--------|----------|-------------|
| `DevtownReviewerCalibrationWorker` | reviewer-calibration | `ReviewerTrustService.adjustWeight(reviewerId, delta)` |
| `DevtownRoutingAdjustmentWorker` | routing-adjustment | `RoutingConfigService.updateRule(ruleId, changes)` |
| `DevtownSlaWorker` | sla-tuning | `SlaConfigService.updateThreshold(slaId, newThreshold)` |
| `DevtownGateTighteningWorker` | gate-tightening | `GatePolicyService.adjustStrictness(gateId, level)` |
| `DevtownQueueOptimizationWorker` | queue-optimization | `QueueConfigService.updatePriority(queueId, weights)` |

Each worker:
1. Reads the `ImprovementRequest` metadata for the specific change parameters
2. Calls the appropriate internal service
3. Records the applied change in case context for audit
4. Returns `WorkerResult.completed()` with the change summary

**Service availability:** If the target services don't exist in devtown yet, they
need to be created. Each service is a thin CDI bean that modifies the relevant
operational configuration. The configuration storage location (case context, database,
or config file) is a devtown implementation detail — the worker just calls the service.

**Testing:** Each worker is tested with a mock service verifying the correct call
is made with the correct parameters extracted from the improvement request.

---

## 11. Missing Integration Tests (L1)

### RegressionRollbackIntegrationTest

Full workflow: signal consensus → improvement case → outcome MERGED → health drops →
RegressionDetector fires → RollbackRequested event → RollbackCaseSpawner creates
rollback case → rollback executes → outcome recorded.

### CircuitBreakerIntegrationTest

Full lifecycle: health degrades → circuit breaker trips OPEN → ticker blocks all
proposals → health recovers → HALF_OPEN → improvement succeeds → CLOSED.

Restart scenario: circuit breaker is OPEN → process restarts →
`restoreFromEventLog()` reconstructs OPEN state → proposals still blocked.

### ConflictSerializationIntegrationTest

Two concurrent improvements with overlapping file paths → first proceeds → second
blocked by ConflictStrategy → first completes → next tick re-evaluates → second
proceeds.

---

## 12. ResearchPipelineOrchestratorTest Enhancement (L2)

Expand from 1 to 5+ tests:
- Pipeline executes all steps at each depth tier
- Corpus stores candidates and analysis
- HIL queue populated for inaccessible sources
- Empty search results handled gracefully
- Scoper receives correct drive context

---

## 13. Module Placement — New Components

| Component | Module | Package |
|-----------|--------|---------|
| `EvolutionTimerBackstop` | runtime-core | `io.casehub.engine.runtime.improvement` |
| `RollbackCaseSpawner` | runtime-core | `io.casehub.engine.runtime.improvement` |
| `RollbackRequested` | api | `io.casehub.api.model.stigmergy` |
| DevTown workers (5) | devtown/app | `io.casehub.devtown.app.evolution.worker` |
| DevTown services (5) | devtown/app | `io.casehub.devtown.app.evolution.service` |

---

## 14. Cross-Repo Scope

| Repo | Changes |
|------|---------|
| casehubio/engine | C1-C3, H1-H3, M1-M4, L1-L2 (all core wiring and safety) |
| casehubio/devtown | M5 (improvement workers and supporting services) |

---

## References

- `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` — parent spec
- `specs/issue-1115-continuous-evolution-loop/gap-analysis.md` — gap analysis
- `specs/issue-1115-continuous-evolution-loop/decisions.md` — D1–D5
- `CaseContextChangedEventHandler.java` — convergence detection (line ~1498)
- `RuntimeBeans.java` — CDI producer wiring
- `EvolutionTicker.java` — gate pipeline
- `ImprovementGoalFormationStrategy.java` — filter pipeline
- `ImprovementOutcomeEventCapture.java` — 6-layer outcome capture
- `RegressionDetector.java` — regression detection and confidence scoring
- `ImprovementCircuitBreaker.java` — circuit breaker with event-sourced recovery
- `CodeEvolutionDenyPatternProvider.java` — structural deny list
- `self-improvement-rollback.yaml` — rollback case template
- `DevtownProposalSource.java` — DevTown proposal generation
- `DevtownCategoryProvider.java` — DevTown 5-stage lifecycle
