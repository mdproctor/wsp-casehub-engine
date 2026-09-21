# Evolution Readiness Methodology — Design Spec

**Issue:** casehubio/engine#1131
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `2026-09-20-continuous-evolution-loop-design.md` (#1115)
**Decisions:** D1–D9 in `decisions.md`
**Date:** 2026-09-21

## Problem

The evolution loop infrastructure (#1115) is complete: `EvolutionTicker`, `HealthScoreTracker`, `ImprovementCircuitBreaker`, `RegressionDetector`, `ImprovementCategoryTracker`, `ConflictDetector`, and the full research pipeline are all implemented with 296 passing tests. But `CapabilityArea.assess()` is an empty SPI — no concrete implementations exist. Without real health data flowing:

- `HealthScoreTracker.computeScore()` returns 0.0 (no registered areas)
- `ImprovementCircuitBreaker` never trips (no health signal)
- `RegressionDetector` never detects regression (no baseline to compare)
- The entire feedback loop is inert

Projects need a methodology to become evolution-capable — a progression from "nothing configured" to "fully autonomous improvement."

## Scope

This spec covers:

1. Ten concrete `CapabilityArea` implementations (all bootstrap areas from D109)
2. `ComplianceLevel` enum (L0–L3) and per-area `ComplianceChecklist`
3. `ReadinessValidator` producing `ReadinessReport`
4. CDI bootstrap observer for auto-registering capability areas
5. `HealthPolicy.effectiveWeights()` update (8 → 10 areas)
6. EventLog persistence for compliance state transitions

Out of scope: command centre UI (#1132), evolution-from-zero bootstrapping, LLM-powered area assessment (Epics 2-3).

## 1. Compliance Levels

Four progressive levels define the evolution readiness journey. Each level is a superset of the previous.

```java
// api, io.casehub.api.model.stigmergy
public enum ComplianceLevel {
  L0_INERT,       // nothing configured — evolution disabled
  L1_OBSERVE,     // health data flows — monitoring without action
  L2_PROPOSE,     // system proposes improvements — HIL approval required
  L3_AUTONOMOUS   // system executes improvements — HIL oversight
}
```

### Level requirements

| Level | What's needed | Behaviour |
|-------|--------------|-----------|
| L0_INERT | Nothing — default state | `evolutionEnabled: false`. No areas registered, no health tracking. |
| L1_OBSERVE | ≥1 CapabilityArea with non-trivial `assess()`, `evolutionEnabled: false` | Health scores computed and tracked. HealthScoreTracker records snapshots. Circuit breaker evaluated but moot (evolution disabled). Dashboard-ready. |
| L2_PROPOSE | L1 + `evolutionEnabled: true`, signal sources configured, consensus threshold met | EvolutionTicker.tick() runs the full gate pipeline. Proposals generated. GoalFormationService.propose() fires. Improvement cases spawned but require HIL review (existing `submit-pr → review → integrate` lifecycle). |
| L3_AUTONOMOUS | L2 + auto-integration configured, rollback policy configured, health threshold configured | Same as L2 but improvement cases can auto-integrate without HIL review. Rollback policy active. Circuit breaker and regression detection protect against degradation. |

### Per-area compliance

Each `CapabilityArea` can be at a different compliance level. A project can be L3 for stability (CI data flows, auto-fix works) while remaining L0 for cognitive-reasoning (no cognitive layer yet). The **project-level compliance** is `min(area levels where level > L0)`, falling back to L0 when no areas are above L0. Areas at L0 are not yet participating — they don't constrain the project level. This avoids the cliff where an unconfigured area (cognitive-reasoning at L0 before Epics 2-3 ship) blocks the entire project from progressing beyond L0.

```java
// api, io.casehub.api.model.stigmergy
public record ComplianceChecklist(
    String areaId,
    ComplianceLevel level,
    List<CheckRequirement> requirements) {

  public record CheckRequirement(
      String name,
      String description,
      CheckType type) {

    public enum CheckType {
      DATA_FLOW,      // area receives real data (not stub defaults)
      CONFIGURATION,  // required ImprovementConfig fields set
      INFRASTRUCTURE  // supporting infrastructure present (e.g. signal sources)
    }
  }
}
```

## 2. Capability Area Implementations

### Architecture

Each `CapabilityArea` implementation:
- Is plain `@ApplicationScoped` — no `@DefaultBean` (see below)
- Directly queries its data sources (EventLog, signal registry, CDI-injected services) — no intermediate MetricSource SPI (D2)
- Returns a `CapabilityAreaAssessment` with computed `healthScore` (0.0–1.0), `landscapePosition`, and cost/impact/ROI estimates
- Lives in `runtime-core`, package `io.casehub.engine.internal.improvement.area`

### Why not `@DefaultBean`

`@DefaultBean` is the correct pattern for 1:1 SPI replacement (e.g. `InMemoryResearchCorpus` → custom `ResearchCorpus`) but wrong for multi-instance SPIs. In Quarkus Arc, `@DefaultBean` suppresses ALL default beans of the same type when ANY non-default bean exists. If a consumer provides one custom `CapabilityArea` without `@DefaultBean`, all 10 default areas are silently removed from CDI resolution — the `Instance<CapabilityArea>` in the bootstrap observer would discover only the custom one.

Override instead happens via the `CapabilityAreaRegistry` runtime API. The `CapabilityAreaBootstrap` (§3) registers all CDI-discovered areas at startup. A consumer overrides a specific area by registering their custom implementation in a higher-priority startup observer — `registry.register(customArea)` overwrites by ID.

### SPI change: tenancyId parameter

The existing `CapabilityArea.assess(UUID caseId)` signature lacks the `tenancyId` parameter required by `EventLogRepository.findByCaseAndTypes()`. Since no external consumers implement this SPI yet (the entire point of this issue), we add it now:

```java
// api/spi/improvement — updated signature
CapabilityAreaAssessment assess(UUID caseId, String tenancyId);
```

This threads through the complete evolution pipeline:

```
EvolutionTicker.tick(caseId, tenancyId, config)
  ├── healthTracker.refresh(caseId, tenancyId, policy)
  │     └── area.assess(caseId, tenancyId)
  ├── circuitBreaker.evaluate(caseId, tenancyId, tracker, policy)
  │     └── tracker.computeScore(caseId, tenancyId, policy)
  │           └── area.assess(caseId, tenancyId)
  └── regressionDetector.checkActiveMonitors(...)  ← uses latestSnapshot, OK
```

The `computeScore()`, `refresh()`, and `delta()` methods on `HealthScoreTracker` gain `tenancyId`. `ImprovementCircuitBreaker.evaluate()` also gains `tenancyId` to pass through to `tracker.computeScore()`.

### Data access pattern

```
CapabilityArea.assess(caseId, tenancyId)
    └── queries EventLog via findByCaseAndTypes(caseId, types, tenancyId)
    └── queries signal registry for detection signals
    └── queries CDI-injected services (ActivityTracker, etc.)
    └── computes healthScore from available data
    └── returns CapabilityAreaAssessment
```

When no data is available (no events, no injected services), the default `assess()` returns a neutral assessment: `healthScore=0.5`, `landscapePosition=ABSENT`, `impactEstimate=0.0`, `costEstimate=0.0`, `roi=0.0`. This is the L0_INERT baseline — present but providing no signal.

**ABSENT exclusion:** `HealthScoreTracker.computeScore()` and `refresh()` skip areas where `assessment.landscapePosition() == ABSENT` from the weighted average. Only areas with real data contribute to the composite health score. Without this exclusion, 7 neutral areas at 0.5 would drag a healthy composite (3 real areas scoring 0.9+) down to ~0.66 — barely above the circuit breaker's 0.6 threshold — despite every measured area being healthy.

### The 10 bootstrap areas

| # | Area ID | assess() event types | Rule-based health metric |
|---|---------|---------------------|--------------------------|
| 1 | `stability` | `CASE_COMPLETED`, `CASE_FAULTED`, `CASE_CANCELLED` | Ratio of CASE_COMPLETED to total terminal case events |
| 2 | `performance` | `CASE_STARTED` + `CASE_COMPLETED` (timestamp delta) | Ratio of cases completing within configurable duration threshold |
| 3 | `execution` | `WORKER_EXECUTION_STARTED`, `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_DECLINED`, `WORKER_OUTCOME_FAILED` | Worker success rate: COMPLETED / (COMPLETED + FAILED + DECLINED + OUTCOME_FAILED) |
| 4 | `safety` | `BUDGET_EXHAUSTED`, `CIRCUIT_BREAKER_TRIPPED`, `ACTION_GATE_REJECTED` | Inverse of safety violation events in window (0 violations = 1.0) |
| 5 | `integration` | `ORCHESTRATION_COMPLETED`, `ORCHESTRATION_ESCALATED`, `WORKFLOW_STEP_COMPLETED`, `WORKFLOW_STEP_FAILED` | Ratio of successful orchestrations/workflows to total |
| 6 | `coordination` | `STIGMERGY_*`, `SWARM_*`, `CONVERGENCE_DETECTED`, `COORDINATION_STORM_DETECTED` | Convergence rate minus storm events; team formation success rate |
| 7 | `perception` | `OBSERVATION_DETECTED`, `PHEROMONE_DEPOSITED`, `SIGNAL_RECEIVED` | Signal activity rate: observation/signal events per time window (normalised) |
| 8 | `autonomy` | `IMPROVEMENT_GOAL_FORMED`, `IMPROVEMENT_OUTCOME`, `SWARM_PROVISION_REQUESTED`, `SWARM_PROVISION_COMPLETED` | Ratio of autonomous actions (improvements, provisions) to total case activity |
| 9 | `cognitive-reasoning` | `GOAL_FORMED`, `GOAL_REVISED`, `PLAN_ADAPTED`, `PLAN_DEEPENED`, `PLAN_CONCEDED` | Goal-to-completion ratio: completed goals / formed goals |
| 10 | `cognitive-memory` | `IMPROVEMENT_OUTCOME` (for CBR reuse patterns) | Heuristic: returns 0.5 (neutral) until cognitive layer provides real assessment. CBR traces don't emit EventLog entries directly. |

### Health score computation

Each area computes its health score as a ratio in [0.0, 1.0]:

```java
// Example: StabilityCapabilityArea
@ApplicationScoped
public class StabilityCapabilityArea implements CapabilityArea {

  private static final Collection<CaseHubEventType> TERMINAL_TYPES = List.of(
      CaseHubEventType.CASE_COMPLETED,
      CaseHubEventType.CASE_FAULTED,
      CaseHubEventType.CASE_CANCELLED);

  private final EventLogRepository eventLog;

  @Inject
  public StabilityCapabilityArea(EventLogRepository eventLog) {
    this.eventLog = eventLog;
  }

  @Override
  public String id() { return "stability"; }

  @Override
  public String name() { return "Stability"; }

  @Override
  public String description() {
    return "Case completion rate — ratio of successfully completed cases to total terminal events";
  }

  @Override
  public CapabilityAreaAssessment assess(UUID caseId, String tenancyId) {
    var events = eventLog.findByCaseAndTypes(caseId, TERMINAL_TYPES, tenancyId);
    if (events.isEmpty()) {
      return neutralAssessment(); // L0: no data
    }
    long successes = events.stream()
        .filter(e -> e.type() == CaseHubEventType.CASE_COMPLETED)
        .count();
    double healthScore = (double) successes / events.size();

    return new CapabilityAreaAssessment(
        "stability", healthScore,
        positionFromScore(healthScore),
        impactEstimate(healthScore),
        costEstimate(healthScore),
        roi(healthScore),
        Instant.now());
  }
}
```

Areas that lack real data sources (perception, cognitive-memory, cognitive-reasoning) provide heuristic defaults. When the EventLog has no relevant events, they return the neutral assessment. This is architecturally correct — consumers (blocks/neocortex) replace them with LLM-powered assessment via the registry override mechanism (§3) when the cognitive layer ships.

### Landscape position derivation

Each area derives its `LandscapePosition` from its health score:

| Health score range | Position | Meaning |
|-------------------|----------|---------|
| No data (0 events) | `ABSENT` | Area not active — no signal to assess |
| 0.0 – 0.4 | `BEHIND` | Significantly below acceptable health |
| 0.4 – 0.7 | `AT_PARITY` | Functional but room for improvement |
| 0.7 – 1.0 | `AHEAD` | Strong health — focus elsewhere |

These thresholds are in the area implementation, not configurable — the compliance level model (not landscape position) governs progression decisions.

## 3. CDI Bootstrap Observer

A startup observer discovers all `@ApplicationScoped` `CapabilityArea` beans and registers them with the `CapabilityAreaRegistry`. This preserves the registry's runtime mutability (register/deprecate) for taxonomy evolution while automating initial population.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class CapabilityAreaBootstrap {

  @Inject
  CapabilityAreaRegistry registry;

  @Inject
  @Any
  Instance<CapabilityArea> areaInstances;

  void onStartup(@Observes StartupEvent event) {
    for (CapabilityArea area : areaInstances) {
      registry.register(area);
    }
  }
}
```

This runs once at startup. Runtime taxonomy evolution (merge, split, deprecate) continues to use `CapabilityAreaRegistry` directly.

### Selective override

A consumer overrides a specific area by providing a custom `@ApplicationScoped CapabilityArea` bean with the same `id()` and registering it after the bootstrap:

```java
@ApplicationScoped
public class CustomStabilityOverride {

  @Inject CapabilityAreaRegistry registry;
  @Inject CustomStabilityArea customStability;

  void onStartup(@Observes @Priority(APPLICATION + 10) StartupEvent event) {
    registry.register(customStability);
  }
}
```

`CapabilityAreaRegistry.register()` does `areas.put(area.id(), area)` — the custom area overwrites the default by ID. The `@Priority(APPLICATION + 10)` ensures the consumer's observer runs after the `CapabilityAreaBootstrap`.

## 4. Readiness Validator

The `ReadinessValidator` checks all registered `CapabilityArea` implementations against the compliance checklist for a target level and produces a `ReadinessReport`.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ReadinessValidator {

  private final CapabilityAreaRegistry areaRegistry;
  private final ComplianceChecklistProvider checklistProvider;
  private final EventLogRepository eventLogRepository;

  public ReadinessReport validate(UUID caseId, String tenancyId,
      ComplianceLevel targetLevel, ImprovementConfig config) {
    // For each registered area:
    //   1. Get the area's ComplianceChecklist for the target level
    //   2. Evaluate each CheckRequirement by dispatching on CheckType:
    //      - DATA_FLOW: call area.assess(caseId, tenancyId), check
    //        landscapePosition != ABSENT; query eventLogRepository
    //      - CONFIGURATION: check ImprovementConfig fields
    //        (evolutionEnabled, consensusMinSources, rollbackPolicy)
    //      - INFRASTRUCTURE: check areaRegistry.get(areaId), signal config
    //   3. Collect results into AreaCompliance
    // Compute project-level compliance as min(area levels where level > L0)
    // Compare against last persisted level; emit COMPLIANCE_LEVEL_CHANGED if changed
    // Emit READINESS_EVALUATED with full report
    // Return ReadinessReport with per-area breakdown
  }
}
```

### ReadinessReport

```java
// api, io.casehub.api.model.stigmergy
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

### ComplianceChecklistProvider

Provides the `ComplianceChecklist` for each area at each level. Implemented as a `@DefaultBean @ApplicationScoped` bean — the 1:1 SPI replacement pattern is correct here because there is exactly one `ComplianceChecklistProvider`, not multiple instances.

```java
// runtime-core, io.casehub.engine.internal.improvement
@DefaultBean
@ApplicationScoped
public class DefaultComplianceChecklistProvider implements ComplianceChecklistProvider {

  public ComplianceChecklist checklistFor(String areaId, ComplianceLevel level) {
    // Returns the built-in checklist for the area at the level.
    // L0: no requirements (always satisfied)
    // L1: area must be registered + assess() returns non-neutral data
    // L2: L1 + evolutionEnabled=true + signal sources configured
    // L3: L2 + rollback policy + health threshold + auto-integration
  }
}
```

### Example checklist: stability at L1

| Requirement | Type | Check |
|-------------|------|-------|
| `stability-area-registered` | INFRASTRUCTURE | CapabilityAreaRegistry.get("stability") present |
| `stability-has-terminal-events` | DATA_FLOW | EventLog has CASE_COMPLETED/CASE_FAULTED/CASE_CANCELLED events for the case |
| `stability-non-neutral-score` | DATA_FLOW | assess(caseId, tenancyId).landscapePosition != ABSENT |

### Example checklist: stability at L2

| Requirement | Type | Check |
|-------------|------|-------|
| All L1 requirements | — | — |
| `evolution-enabled` | CONFIGURATION | config.effectiveEvolutionEnabled() == true |
| `stability-signals-configured` | INFRASTRUCTURE | SignalRegistry has `improvement:quality:*` signals |
| `consensus-threshold-met` | CONFIGURATION | config.effectiveConsensusMinSources() >= 1 |

## 5. HealthPolicy Update

`HealthPolicy.effectiveWeights()` must be updated from 8 to 10 areas to match the D109 bootstrap taxonomy:

```java
// Current (8 areas):
Map.of(
    "stability", 0.2,
    "performance", 0.15,
    "execution", 0.1,
    "safety", 0.15,
    "integration", 0.1,
    "autonomy", 0.1,
    "cognitive-reasoning", 0.1,
    "coordination", 0.1);

// Updated (10 areas — adds perception and cognitive-memory):
Map.of(
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
```

Weights are re-normalised to sum to 1.0. Stability and safety remain the highest-weighted areas. The two new areas (perception, cognitive-memory) receive lower weights reflecting their initially heuristic assessment quality.

## 6. EventLog Persistence

Compliance state transitions are recorded as EventLog entries. The #1115 spec designed event-sourced restart recovery for the circuit breaker (`restoreFromEventLog`), but this was not yet implemented — circuit breaker state is currently in-memory only. The compliance level persistence below is the first implementation of this pattern, establishing the precedent.

- New `CaseHubEventType` values: `COMPLIANCE_LEVEL_CHANGED`, `READINESS_EVALUATED`
- `COMPLIANCE_LEVEL_CHANGED` — emitted by `ReadinessValidator.validate()` when the computed project-level compliance differs from the last persisted level. The validator is the single owner of compliance level transitions — no other component emits this event.
- `READINESS_EVALUATED` — emitted by `ReadinessValidator.validate()` on every evaluation, capturing the full `ReadinessReport` as event data

On restart, the current compliance level for each case reconstructs from the most recent `COMPLIANCE_LEVEL_CHANGED` EventLog entry.

## 7. Module Placement

Following the #1115 placement pattern (D5):

| Component | Module | Package |
|-----------|--------|---------|
| `ComplianceLevel` enum | `api` | `io.casehub.api.model.stigmergy` |
| `ComplianceChecklist` record | `api` | `io.casehub.api.model.stigmergy` |
| `ReadinessReport` record | `api` | `io.casehub.api.model.stigmergy` |
| `ComplianceChecklistProvider` SPI | `api` | `io.casehub.api.spi.improvement` |
| `StabilityCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `PerformanceCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `ExecutionCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `SafetyCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `IntegrationCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `CoordinationCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `PerceptionCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `AutonomyCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `CognitiveReasoningCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `CognitiveMemoryCapabilityArea` | `runtime-core` | `io.casehub.engine.internal.improvement.area` |
| `CapabilityAreaBootstrap` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ReadinessValidator` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `DefaultComplianceChecklistProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |

## 8. Test Strategy

### Unit tests

| Test class | What it covers |
|------------|---------------|
| `StabilityCapabilityAreaTest` | Health score from terminal case events, neutral on no data, landscape position thresholds |
| `PerformanceCapabilityAreaTest` | Health score from case lifecycle timing, SLA adherence |
| `ExecutionCapabilityAreaTest` | Worker success rate, dispatch efficiency |
| `SafetyCapabilityAreaTest` | Trust violations, budget overruns, circuit breaker state |
| `IntegrationCapabilityAreaTest` | API/contract event ratios |
| `CoordinationCapabilityAreaTest` | Stigmergy/swarm event success rates |
| `PerceptionCapabilityAreaTest` | Signal freshness ratios (heuristic) |
| `AutonomyCapabilityAreaTest` | Autonomous vs HIL action ratios (heuristic) |
| `CognitiveReasoningCapabilityAreaTest` | Goal achievement rate (heuristic) |
| `CognitiveMemoryCapabilityAreaTest` | CBR retrieval hit rate (heuristic) |
| `CapabilityAreaBootstrapTest` | CDI discovery and registration at startup |
| `ReadinessValidatorTest` | All 4 levels validated, per-area breakdown, project-level min |
| `DefaultComplianceChecklistProviderTest` | Correct checklist for each area at each level |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `ReadinessProgressionIntegrationTest` | Full L0 → L1 → L2 → L3 progression: start with no config, add areas, enable evolution, configure auto-integration |
| `HealthScoreWithRealAreasTest` | HealthScoreTracker aggregation with all 10 registered areas, weight normalisation |

## 9. Project Template — Getting to L1

A minimal starter configuration to reach L1 (OBSERVE) for the core areas:

### Step 1: Ensure a project case exists

The evolution loop presupposes a CaseHub case instance (D1). The project must have a running case with EventLog entries flowing.

### Step 2: Verify area registration

On application startup, `CapabilityAreaBootstrap` auto-registers all 10 default areas. Verify with `ReadinessValidator`:

```java
var report = validator.validate(caseId, tenancyId, ComplianceLevel.L1_OBSERVE, config);
// Check report.areas() — each should show stability-area-registered: satisfied
```

### Step 3: Generate terminal events

L1 requires non-neutral assessment data. For stability, this means `CASE_COMPLETED`, `CASE_FAULTED`, or `CASE_CANCELLED` events in the EventLog. Run at least one case to completion. For execution, this means worker execution events. For performance, cases with both `CASE_STARTED` and `CASE_COMPLETED` timestamps.

### Step 4: Validate L1 readiness

```java
var report = validator.validate(caseId, tenancyId, ComplianceLevel.L1_OBSERVE, config);
// report.passed() == true when ≥1 area has non-neutral data
// Areas without data remain at L0 — they don't block the project level
```

At L1, health scores are computed and tracked. The dashboard shows per-area health and landscape position. No improvements are proposed (evolution remains disabled).

### Progression to L2

Set `ImprovementConfig.evolutionEnabled = true`, configure signal sources (`improvement:quality:*`), and set `consensusMinSources >= 1`. Run the validator against `ComplianceLevel.L2_PROPOSE` to check remaining gaps.

## References

- `api/src/main/java/io/casehub/api/spi/improvement/CapabilityArea.java` — existing SPI interface
- `api/src/main/java/io/casehub/api/model/stigmergy/CapabilityAreaAssessment.java` — assessment record
- `api/src/main/java/io/casehub/api/model/stigmergy/HealthPolicy.java` — weights (needs update 8→10)
- `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementConfig.java` — evolution config
- `runtime-core/src/main/java/io/casehub/engine/internal/improvement/CapabilityAreaRegistry.java` — area registry
- `runtime-core/src/main/java/io/casehub/engine/internal/improvement/HealthScoreTracker.java` — score aggregation
- `runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java` — tick pipeline
- `specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` §6 (capability areas), §11 (module placement), D109 (bootstrap taxonomy)
- `specs/issue-1104-hive-mind/decisions.md` D109 — 10 bootstrap areas with coverage table
- `docs/guides/contributor-guide.md` §SPI Architecture — SPI placement rules
- `docs/guides/contributor-guide.md` §CDI Conventions — @DefaultBean pattern
- issue #1131 — problem statement and scope
- issue #1132 — command centre conductor (consumes ReadinessReport)
- Memory: `evolution-readiness-methodology` — priority ordering
- Memory: `evolution-from-zero` — bootstrap from zero with HIL briefing
- Memory: `command-centre-conductor` — observability and HIL intervention
