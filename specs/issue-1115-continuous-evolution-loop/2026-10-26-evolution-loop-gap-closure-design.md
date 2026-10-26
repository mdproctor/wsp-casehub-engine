# Evolution Loop Gap Closure — Design Spec

**Issue:** casehubio/engine#1115
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md`
**Gap analysis:** `specs/issue-1115-continuous-evolution-loop/gap-analysis.md`
**Decisions:** D1–D5 in `decisions.md`
**Date:** 2026-10-26
**Revised:** 2026-10-26 (incorporated light design review findings)

## Problem

The 2026-09-20 design spec is comprehensive and sound. All major components are
implemented and pass their unit tests. However, the production CDI wiring is
disconnected — the loop is open. Components work in isolation but are not connected
in the runtime graph.

This spec addresses every gap identified in the gap analysis. It is an addendum to
the parent spec, not a replacement.

### Module boundary convention

Throughout this spec, changes respect the existing `runtime-core` / `runtime` split:

- **`runtime-core`** — framework-agnostic. No CDI annotations (`@ApplicationScoped`,
  `@Observes`, `@Scheduled`). Classes are plain Java with constructor injection.
  CDI wiring is done via `@Produces` methods in `RuntimeBeans.java` (runtime module).
- **`runtime`** — Quarkus-specific. Contains `RuntimeBeans.java` producers, scheduled
  tasks, CDI observers, and framework adapters.

### Event pattern convention

The improvement subsystem uses `Consumer<Event>` callbacks passed via constructor
injection — NOT CDI `@ObservesAsync`. This is consistent with `RegressionDetector`,
`ImprovementCircuitBreaker`, and `EvolutionTicker`, which all receive event callbacks
through `RuntimeBeans.java` producers. New components follow this pattern.

---

## 1. Wire EvolutionTicker into Convergence Detection (C1)

### Current state

`CaseContextChangedEventHandler.convergenceDetection()` calls
`improvementStrategy.proposeImprovements()` directly (line ~1498), bypassing the
EvolutionTicker's gate pipeline.

### Change

Replace the direct call with a ticker invocation. In `RuntimeBeans.java`, add
`EvolutionTicker` to the `CaseContextChangedEventHandler` producer. In the handler,
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

### Per-case tick serialization

Both event-driven and timer paths can fire concurrently for the same case. Add a
per-case lock in `EvolutionTicker` to prevent concurrent ticks:

```java
private final ConcurrentHashMap<UUID, ReentrantLock> tickLocks =
    new ConcurrentHashMap<>();

public TickTrace tick(UUID caseId, String tenancyId,
    ImprovementConfig config, TickTrigger trigger) {
  var lock = tickLocks.computeIfAbsent(caseId, k -> new ReentrantLock());
  if (!lock.tryLock()) {
    return TickTrace.skipped(caseId, trigger, "concurrent tick in progress");
  }
  try {
    return doTick(caseId, tenancyId, config, trigger);
  } finally {
    lock.unlock();
  }
}
```

### Injection change

`CaseContextChangedEventHandler` gains `Instance<EvolutionTicker>` via its producer
in `RuntimeBeans.java`. The handler is a runtime-core class that receives the ticker
as a constructor parameter; the producer resolves it.

`ImprovementGoalFormationStrategy` and `GoalFormationService` are still needed for
non-evolution improvement paths (manual goal proposals). The handler only routes
through the ticker when evolution is configured.

---

## 2. Timer Backstop (C2)

### New class in runtime module

The timer backstop lives in the `runtime` module (not runtime-core) because it uses
Quarkus `@Scheduled`:

```java
// runtime, io.casehub.engine.quarkus.improvement
@ApplicationScoped
public class EvolutionTimerBackstop {

  private final EvolutionTicker ticker;
  private final CaseInstanceRepository caseRepository;
  private final CaseDefinitionRegistry definitionRegistry;

  @Scheduled(every = "{casehub.evolution.timer-interval:60m}")
  void tick() {
    for (var caseInstance : caseRepository.findActive()) {
      var definition = definitionRegistry.get(caseInstance.caseDefinitionId());
      if (definition == null) continue;
      var stigConfig = definition.stigmergyConfig();
      if (stigConfig == null || stigConfig.improvement() == null) continue;
      var improvementConfig = stigConfig.improvement();
      if (!improvementConfig.effectiveEvolutionEnabled()) continue;

      ticker.tick(caseInstance.getUuid(), caseInstance.tenancyId(),
          improvementConfig, TickTrace.TickTrigger.TIMER);
    }
  }
}
```

**Key changes from prior draft:**
- Uses `caseRepository.findActive()` (exists) instead of `findWithEvolutionEnabled()` (doesn't exist)
- Reads `ImprovementConfig` via `CaseDefinitionRegistry` → `StigmergyConfig.improvement()` — no `StigmergyConfigReader` needed
- Filters by `effectiveEvolutionEnabled()` in the loop
- Each case is ticked with its own tenancy context
- Located in `runtime` module to use `@Scheduled`

**Guard:** The per-case lock in `EvolutionTicker` (§1) prevents concurrent ticks
from timer and event-driven paths. The budget enforcer's concurrent limit prevents
duplicate improvement cases.

---

## 3. Wire ImprovementOutcomeEventCapture (C3)

### Current state

The class exists with all 6 layers wired internally but is not a CDI bean.
`ImprovementCaseCompleted` events are never fired.

### Changes

**3a. Wire ImprovementOutcomeEventCapture via producer:**

`ImprovementOutcomeEventCapture` is a runtime-core class (no CDI annotations).
It receives its dependencies via constructor injection. A producer in `RuntimeBeans.java`
creates it and wires it as a `CaseOutcomeObserver` (the existing SPI for case
completion notifications):

```java
// In RuntimeBeans.java
@Produces @ApplicationScoped
ImprovementOutcomeEventCapture improvementOutcomeEventCapture(
    ImprovementOutcomeRecorder outcomeRecorder,
    ImprovementSignalProjector signalProjector,
    ImprovementCbrProjector cbrProjector,
    ImprovementBudgetEnforcer budgetEnforcer,
    ImprovementCategoryTracker categoryTracker,
    RegressionDetector regressionDetector) {
  return new ImprovementOutcomeEventCapture(
      outcomeRecorder, signalProjector, cbrProjector,
      budgetEnforcer, categoryTracker, regressionDetector);
}
```

**3b. Fire from case outcome:**

Rather than introducing a new `ImprovementCaseCompleted` CDI event, use the existing
`CaseOutcomeObserver` SPI. `ImprovementOutcomeEventCapture` implements
`CaseOutcomeObserver`. Its `onOutcome(CaseOutcomeEvent)` method checks whether the
completed case is an improvement case (by template ID) and extracts the
`ImprovementOutcome` from case context:

```java
public class ImprovementOutcomeEventCapture implements CaseOutcomeObserver {

  @Override
  public void onOutcome(CaseOutcomeEvent event) {
    if (!isImprovementCase(event)) return;
    var outcome = extractOutcome(event);
    if (outcome == null) return;

    outcomeRecorder.record(event.caseId(), event.tenancyId(), outcome);
    signalProjector.project(event.caseId(), outcome);
    cbrProjector.project(event.tenancyId(), outcome);
    budgetEnforcer.recordCompletion(outcome.improvementCaseId());
    categoryTracker.recordOutcome(event.caseId(),
        outcome.category(), outcome.status());
    regressionDetector.onOutcome(event.caseId(), event.tenancyId(), outcome);
  }
}
```

This avoids introducing a new event type and reuses the existing case lifecycle
notification mechanism.

**3c. Fix the CBR projector (part of M1):**

`ImprovementCbrProjector.project()` currently just logs. Change to write to CBR
when available. The CBR types (`CbrRetrieverService`) are provided by the neocortex
module as an optional dependency — they are NOT engine types. The projector uses
`Instance<>` optional injection:

```java
// runtime-core
public class ImprovementCbrProjector {

  private final Instance<CbrRetrieverService> cbrService;

  public ImprovementCbrProjector(Instance<CbrRetrieverService> cbrService) {
    this.cbrService = cbrService;
  }

  public void project(String tenancyId, ImprovementOutcome outcome) {
    if (!cbrService.isResolvable()) {
      return;  // No CBR available — graceful no-op
    }
    cbrService.get().store(tenancyId, "improvement-outcomes",
        buildCbrCase(outcome));
  }
}
```

`CbrRetrieverService` is defined in the neocortex API module. Engine declares it
as an optional Maven dependency (`<optional>true</optional>`). When neocortex is
on the classpath, the CBR projector stores outcomes. When absent, it no-ops.

---

## 4. Rollback Case Spawning (H1)

### Relationship to existing events

`RegressionDetector` already fires `RegressionDetectedEvent` (in common-core
`spi.event`) unconditionally when health degrades. This existing event is consumed
by `EvolutionStreamBroadcaster` for SSE streaming to the UI.

The new `RollbackRequested` event **supplements** `RegressionDetectedEvent`. It fires
only when confidence exceeds the auto-revert threshold — a subset of regressions.
Both events are independent: `RegressionDetectedEvent` is always fired (for
observability); `RollbackRequested` is conditionally fired (for automated response).

### New event

```java
// common-core, io.casehub.engine.common.spi.event
public record RollbackRequested(
    UUID caseId,
    String tenancyId,
    UUID improvementCaseId,
    String category,
    String target,
    List<String> targetPaths,
    double confidence) {}
```

Located in `common-core` alongside other internal events (`RegressionDetectedEvent`,
`PheromoneStateChangedEvent`), not in the `api` module.

### MonitoredImprovement change

`MonitoredImprovement` must store target paths so they're available when regression
is detected (potentially many ticks later). Add `targetPaths` to the record:

```java
public record MonitoredImprovement(
    UUID improvementCaseId, String category, String target,
    List<String> targetPaths,
    HealthScoreSnapshot baseline, Instant mergedAt, int checksRemaining) {}
```

Target paths are extracted from the improvement case's context at the time
`onOutcome()` registers the monitor — the `ImprovementOutcome` carries
`affectedPaths` from the introspection stage.

### RegressionDetector change

In `onMetricsDegraded()`, when confidence >= autoRevertThreshold, fire the rollback
request callback in addition to recording and pausing:

```java
if (confidence >= policy.effectiveAutoRevertThreshold()) {
  rollbackHistory.record(caseId, monitor.improvementCaseId(),
      monitor.category(), monitor.target());
  categoryTracker.pauseCategory(caseId, monitor.category(),
      Duration.ofMinutes(policy.effectiveRegressionWindowMinutes()));
  rollbackRequestedCallback.accept(new RollbackRequested(
      caseId, tenancyId, monitor.improvementCaseId(),
      monitor.category(), monitor.target(),
      monitor.targetPaths(), confidence));
}
```

`rollbackRequestedCallback` is a `Consumer<RollbackRequested>` passed via
constructor (following the existing event callback pattern). The producer in
`RuntimeBeans.java` wires it to the `RollbackCaseSpawner`.

### New handler

```java
// runtime-core, io.casehub.engine.runtime.improvement
public class RollbackCaseSpawner {

  private final GoalFormationService goalFormationService;
  private final ImprovementBudgetEnforcer budgetEnforcer;

  public void onRollbackRequested(RollbackRequested event) {
    // Anti-cascade check (M2): query active improvements for path overlap
    var activeRequests = budgetEnforcer.activeImprovementRequests();
    if (hasOverlappingPaths(event.targetPaths(), activeRequests)) {
      return;  // Downgrade: regression signal + category pause only
    }

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

  private boolean hasOverlappingPaths(
      List<String> targetPaths,
      Map<UUID, ImprovementRequest> activeRequests) {
    for (var active : activeRequests.values()) {
      for (String targetPath : targetPaths) {
        for (String activePath : active.targetPaths()) {
          if (targetPath.equals(activePath)
              || sameDirectory(targetPath, activePath)) {
            return true;
          }
        }
      }
    }
    return false;
  }
}
```

Wired via `RuntimeBeans.java` producer. The producer creates `RollbackCaseSpawner`
and passes `spawner::onRollbackRequested` as the `Consumer<RollbackRequested>` to
`RegressionDetector`.

---

## 5. Restart Recovery (H2 + systemic)

### Circuit breaker recovery

`EvolutionBootstrap` currently has no startup handler — it's constructor-only.
Add a `@Observes @Priority(200) StartupEvent` handler in the runtime module:

```java
// runtime, io.casehub.engine.quarkus.improvement
@ApplicationScoped
public class EvolutionStartupRecovery {

  private final ImprovementCircuitBreaker circuitBreaker;
  private final RegressionDetector regressionDetector;
  private final ImprovementCategoryTracker categoryTracker;
  private final CaseInstanceRepository caseRepository;
  private final CaseDefinitionRegistry definitionRegistry;
  private final EventLogRepository eventLogRepository;

  void onStartup(@Observes @Priority(200) StartupEvent event) {
    for (var caseInstance : caseRepository.findActive()) {
      var definition = definitionRegistry.get(caseInstance.caseDefinitionId());
      if (definition == null) continue;
      var stigConfig = definition.stigmergyConfig();
      if (stigConfig == null || stigConfig.improvement() == null) continue;
      if (!stigConfig.improvement().effectiveEvolutionEnabled()) continue;

      var caseId = caseInstance.getUuid();
      var tenancyId = caseInstance.tenancyId();

      // Recover circuit breaker state
      circuitBreaker.restoreFromEventLog(caseId, tenancyId);

      // Recover category pause state from regression events
      categoryTracker.restoreFromEventLog(caseId, tenancyId, eventLogRepository);

      // Recover active regression monitors from MERGED improvement events
      regressionDetector.restoreFromEventLog(caseId, tenancyId, eventLogRepository);
    }
  }
}
```

**Systemic recovery:** The spec extends recovery beyond circuit breaker to include:
- `ImprovementCategoryTracker` — reconstruct pause state from `REGRESSION_DETECTED`
  EventLog entries with active pause durations
- `RegressionDetector` — reconstruct active monitors from recent `IMPROVEMENT_OUTCOME`
  events with MERGED status within the regression window

`RollbackHistory` is NOT recovered — it's an engine-only anti-oscillation fallback.
CBR provides the persistent path. After restart, `RollbackHistory` starts empty,
which is conservative (allows proposals). The category tracker's pause state
(recovered from EventLog) provides the primary post-restart defense.

`TickTraceBuffer` is NOT recovered — diagnostic only, rebuilds naturally on next tick.

**Startup ordering:** Priority 200 runs after `CaseHubClassRegistrar` (150) but
before the first timer tick (`@Scheduled` fires after startup completes).

---

## 6. CBR-Informed Goal Formation (M1 — read side)

In `ImprovementGoalFormationStrategy.proposeImprovements()`, after the
anti-oscillation filter and before the deny filter, add an optional CBR check.

`CbrRetrieverService` is an optional dependency from the neocortex module. The
goal formation strategy receives `Instance<CbrRetrieverService>` via its constructor
(resolved by the producer in RuntimeBeans.java):

```java
if (cbrService.isResolvable()) {
  var similar = cbrService.get().findSimilar(tenancyId,
      "improvement-outcomes", request.category(), request.target(), 5);
  if (computeFailureRate(similar) > 0.6) {
    continue;  // Skip — similar improvements have high failure rate
  }
}
```

The specific `CbrRetrieverService` API (`findSimilar`) is defined by neocortex.
If the API signature differs from the above, the implementation adapts at the
call site. The engine defines no CBR types — it consumes the neocortex SPI.

When CBR is unavailable, this step is skipped entirely. The engine-only fallback
(RollbackHistory + CategoryTracker) remains sufficient.

---

## 7. ProposalFilteringSummary (M3)

Change `ImprovementGoalFormationStrategy.proposeImprovements()` to track filter
counts and return a `ProposalFilteringSummary` alongside the proposal. No
backwards-compatibility shim — change the return type directly:

```java
public record ProposalResult(
    GoalFormationProposal proposal,
    ProposalFilteringSummary filtering) {}

public ProposalResult proposeImprovements(
    UUID caseId, String tenancyId, ImprovementConfig config) {
  // ... filter pipeline tracking counts at each stage ...
  var summary = new ProposalFilteringSummary(
      proposalsBySource, afterCategoryFilter, afterSuppressionFilter,
      afterAntiOscillationFilter, afterDenyFilter, afterBudgetFilter,
      afterConflictFilter, proposed);
  return new ProposalResult(proposal, summary);
}
```

The ticker uses the summary when building the TickTrace:

```java
var result = goalFormation.proposeImprovements(caseId, tenancyId, config);
// Build TickTrace with result.filtering()
var outcome = new TickTrace.TickOutcome.ProposalGenerated(
    result.proposal().goals().size(), result.filtering());
```

All callers of `proposeImprovements()` are updated to handle the new return type.
There is currently one caller (the ticker) plus test code.

---

## 8. YAML Codegen Entries (M4)

Add to `yaml-record-mappings.yaml`:

```yaml
io.casehub.api.model.improvement.RollbackPolicy:
  yamlName: YamlRollbackPolicy

io.casehub.api.model.improvement.HealthPolicy:
  yamlName: YamlHealthPolicy

io.casehub.api.model.stigmergy.ResearchMethodology:
  yamlName: YamlResearchMethodology
```

Note: `RollbackPolicy` and `HealthPolicy` are in `io.casehub.api.model.improvement`
(not `model.stigmergy`). `ResearchMethodology` is in `model.stigmergy`.

---

## 9. Circuit Breaker Hysteresis (robustness)

The circuit breaker can thrash between OPEN and CLOSED when health oscillates near
the threshold. Add hysteresis: require a recovery margin above the trip threshold
before transitioning OPEN → HALF_OPEN:

```java
case OPEN -> {
  double recoveryThreshold = policy.effectiveHealthThreshold()
      + policy.effectiveHysteresisMargin();  // default 0.05
  if (score >= recoveryThreshold && sustainedRecovery(caseId, policy)) {
    transition(caseId, CircuitBreakerState.HALF_OPEN);
    halfOpenCount.put(caseId, 0);
  }
}
```

Add `hysteresisMargin` field to `HealthPolicy`:

```java
public double effectiveHysteresisMargin() {
  return hysteresisMargin != null ? hysteresisMargin : 0.05;
}
```

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
RegressionDetector fires → RollbackCaseSpawner creates rollback case → rollback
executes → outcome recorded.

### CircuitBreakerIntegrationTest

Full lifecycle: health degrades → circuit breaker trips OPEN → ticker blocks all
proposals → health recovers past hysteresis margin → HALF_OPEN → improvement
succeeds → CLOSED.

Restart scenario: circuit breaker is OPEN → `EvolutionStartupRecovery` reconstructs
OPEN state → proposals still blocked.

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
| `EvolutionTimerBackstop` | runtime | `io.casehub.engine.quarkus.improvement` |
| `EvolutionStartupRecovery` | runtime | `io.casehub.engine.quarkus.improvement` |
| `RollbackCaseSpawner` | runtime-core | `io.casehub.engine.runtime.improvement` |
| `RollbackRequested` | common-core | `io.casehub.engine.common.spi.event` |
| DevTown workers (5) | devtown/app | `io.casehub.devtown.app.evolution.worker` |
| DevTown services (5) | devtown/app | `io.casehub.devtown.app.evolution.service` |

---

## 14. Cross-Repo Scope

| Repo | Changes |
|------|---------|
| casehubio/engine | C1-C3, H1-H2, M1-M4, L1-L2, robustness fixes (all core wiring and safety) |
| casehubio/devtown | M5 (improvement workers and supporting services) |

---

## References

- `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` — parent spec
- `specs/issue-1115-continuous-evolution-loop/gap-analysis.md` — gap analysis
- `specs/issue-1115-continuous-evolution-loop/decisions.md` — D1–D5
- `CaseContextChangedEventHandler.java` — convergence detection (line ~1498)
- `RuntimeBeans.java` — CDI producer wiring (runtime module)
- `EvolutionTicker.java` — gate pipeline (runtime-core)
- `ImprovementGoalFormationStrategy.java` — filter pipeline (runtime-core)
- `ImprovementOutcomeEventCapture.java` — 6-layer outcome capture (runtime-core)
- `RegressionDetector.java` — regression detection and confidence scoring (runtime-core)
- `RegressionDetectedEvent.java` — existing regression event (common-core)
- `ImprovementCircuitBreaker.java` — circuit breaker with event-sourced recovery (runtime-core)
- `CodeEvolutionDenyPatternProvider.java` — structural deny list (runtime-core)
- `self-improvement-rollback.yaml` — rollback case template (runtime)
- `CaseOutcomeObserver.java` — case completion SPI (api)
- `DevtownProposalSource.java` — DevTown proposal generation (devtown)
- `DevtownCategoryProvider.java` — DevTown 5-stage lifecycle (devtown)
- neocortex `CbrRetrieverService` — optional CBR dependency (external)
