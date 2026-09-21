# Command Centre Conductor — Design Spec

**Issue:** casehubio/engine#1132
**Epic:** casehubio/engine#1104 (Hive Mind)
**Parent spec:** `2026-09-20-continuous-evolution-loop-design.md` (#1115)
**Prerequisite:** `2026-09-21-evolution-readiness-methodology-design.md` (#1131)
**Decisions:** D10–D27 in `decisions.md`
**Date:** 2026-09-21

## Problem

The evolution loop infrastructure is complete: `EvolutionTicker`, `HealthScoreTracker`, `ImprovementCircuitBreaker`, `RegressionDetector`, `ImprovementCategoryTracker`, `ConflictDetector`, and the full research pipeline — 385 tests, 0 failures. The readiness methodology (#1131) provides 10 capability areas, L0–L3 compliance progression, and `ReadinessValidator`.

But the developer has no way to interact with this system:

| Gap | What's missing |
|-----|---------------|
| Observability | No API surface — evolution state is only visible via test assertions |
| Manual controls | `pauseCategory()`, `manualReset()` exist as Java methods — no API exposure |
| Gate pipeline visibility | `EvolutionTicker.tick()` is void — no trace of which gates passed or blocked |
| Research steering | No mechanism for the HIL to influence research direction or approve hypotheses |
| Lifecycle checkpoints | The only HIL gate is PR review — no checkpoints at research, hypothesis, or implementation stages |
| Summarization | No way to get layered summaries across areas, categories, or time windows |
| Artifact trail | On-disk artifacts (analysis, designs, debates) exist but aren't indexed per improvement stream |
| Smart escalation | No mechanism for the system to self-assess confidence and surface items needing human attention |
| Stream progress | No per-improvement progress view — current stage, completed stages, what's pending |
| Manual coordination | ConflictDetector handles automatic file-level conflicts — no manual semaphore for ordering improvement streams |
| Bootstrap | No guided path from L0 (nothing configured) to L3 (autonomous improvement) |

The developer should act as **conductor** — shaping tempo, direction, and focus through an observable command centre, not operating a black box.

## Architecture Overview

The command centre is the conductor's interface to the evolution loop. Five layers:

```
Conductor (HIL)
    │
    ├── 1. OBSERVE ── state snapshot, event stream, tick history
    ├── 2. SUMMARIZE ── layered summaries at any altitude
    ├── 3. CONTROL ── immediate manual overrides
    ├── 4. STEER ── lifecycle gates with smart escalation
    └── 5. REVIEW ── artifact trail per improvement stream
```

**Components:**

| Component | Module | Purpose |
|-----------|--------|---------|
| `DefaultEngineEvolutionApi` | `rest` | Single `@McpDomain("engine/evolution")` — all queries and mutations |
| `EvolutionStreamBroadcaster` | `rest` | Dedicated `BroadcastProcessor<EvolutionEvent>` for SSE |
| `TickTrace` + `TickTraceBuffer` | `runtime-core` | Gate pipeline instrumentation |
| `GatePolicy` | `api` | Configurable lifecycle gate modes per stage (Map-based, extensible) |
| `EscalationPolicy` | `api` | Category rules + confidence threshold for escalation |
| `SummarizationProvider` SPI | `api` | Pluggable summarization (rule-based → LLM) |
| `EscalationProvider` SPI | `api` | Pluggable confidence scoring (heuristic → LLM) |
| `ArtifactManifest` | `api` | Per-improvement artifact index |
| CDI events (4 types) | `engine-common` | Evolution-specific CDI events for broadcaster |

**Data flow:**

```
EvolutionTicker.tick()
    │
    ├── returns TickTrace (per-gate results)
    ├── TickTraceBuffer stores recent traces (ring buffer)
    ├── notable outcomes → EventLog (TICK_EVALUATED)
    ├── periodic → EventLog (TICK_HEARTBEAT, every 24 ticks)
    │
    ├── gate blocked → CDI event → EvolutionStreamBroadcaster
    ├── proposal generated → CDI event → EvolutionStreamBroadcaster
    │
    ├── GatePolicy check → GATED? → ConductorInboxEntry → inbox
    │                     → AUTO? → proceed
    │                     → NOTIFY? → proceed + inbox notification
    │
    └── EscalationPolicy check → any layer triggers?
                                → yes → promote to inbox
                                → no → auto-proceed
```

## 1. Observe — State Snapshot and Event Stream

### EvolutionStateSnapshot

A single composed record returned by `getEvolutionState(caseId)`. Assembled server-side from existing in-memory beans — all reads are cheap (ConcurrentHashMap lookups, ring buffer reads, cached values).

```java
// api, io.casehub.api.view
public record EvolutionStateSnapshot(
    UUID caseId,
    Instant timestamp,
    // Health
    double healthScore,
    Map<String, Double> componentScores,  // areaId → score
    double healthDelta,
    int healthWindowMinutes,
    // Circuit breaker
    CircuitBreakerState circuitBreakerState,
    // Categories
    Map<String, CategoryStateView> categoryStates,
    // Compliance
    ComplianceLevel projectComplianceLevel,
    Instant complianceEvaluatedAt,
    Map<String, ComplianceLevel> areaComplianceLevels,
    // Tick history
    List<TickTrace> recentTicks,
    // Active improvements
    int activeImprovementCount,
    int dailyImprovementCount,
    List<ImprovementStreamView> activeStreams,
    // Evolution config
    boolean evolutionEnabled,
    // Inbox
    int pendingInboxCount) {}

public record CategoryStateView(
    int successCount, int failureCount, int rejectionCount,
    boolean paused, @Nullable Instant pausedUntil,
    boolean suppressed) {}

public record ImprovementStreamView(
    UUID improvementCaseId,
    String category,
    @Nullable String target,
    ImprovementStage currentStage,
    List<StageProgress> stageHistory,
    @Nullable UUID blockedBy,          // manual semaphore
    boolean conflictBlocked,           // automatic ConflictDetector
    Instant startedAt) {}

public record StageProgress(
    ImprovementStage stage,
    StageStatus status,
    Instant enteredAt,
    @Nullable Instant completedAt) {

  public enum StageStatus { PENDING, IN_PROGRESS, COMPLETED, GATED, SKIPPED }
}

public enum ImprovementStage {
  INTROSPECT,
  RESEARCH_SCOPE,
  SEARCH,
  ANALYZE,
  HYPOTHESIS_APPROVAL,
  IMPLEMENTATION_PLAN,
  IMPLEMENT,
  SUBMIT_PR,
  PR_REVIEW,
  INTEGRATE,
  OUTCOME_RECORDING;

  private static final Set<ImprovementStage> GATE_CHECKPOINTS = Set.of(
      RESEARCH_SCOPE, HYPOTHESIS_APPROVAL, IMPLEMENTATION_PLAN, PR_REVIEW);

  public boolean isGateCheckpoint() {
    return GATE_CHECKPOINTS.contains(this);
  }
}
```

**Composition sources:**

| Field | Source | Cost |
|-------|--------|------|
| healthScore, componentScores | `HealthScoreTracker.latestSnapshot()` | ConcurrentHashMap lookup |
| healthDelta | `HealthScoreTracker.delta()` | Deque traversal (bounded) |
| circuitBreakerState | `ImprovementCircuitBreaker.state()` | ConcurrentHashMap lookup |
| categoryStates | `ImprovementCategoryTracker` states map | ConcurrentHashMap scan |
| projectComplianceLevel | Cached from most recent `COMPLIANCE_LEVEL_CHANGED` EventLog entry | In-memory cache read |
| areaComplianceLevels | Cached alongside project level | In-memory cache read |
| recentTicks | `TickTraceBuffer.recent()` | Ring buffer read |
| activeImprovementCount | `ImprovementBudgetEnforcer.activeImprovementRequests().size()` | ConcurrentHashMap size |
| pendingInboxCount | `ConductorInboxManager.pendingCount()` | Queue size |

Compliance level is NOT re-validated on every snapshot query (D15) — `ReadinessValidator.validate()` calls `area.assess()` for each area, triggering database queries. The snapshot uses the cached result. Staleness is visible via `complianceEvaluatedAt`.

### TickTrace — gate pipeline instrumentation (D11)

`EvolutionTicker.tick()` changes from `void` to returning a `TickTrace`:

```java
// api, io.casehub.api.model.stigmergy
public record TickTrace(
    UUID caseId,
    Instant timestamp,
    TickTrigger trigger,         // EVENT_DRIVEN or TIMER
    List<GateResult> gates,
    @Nullable TickOutcome outcome) {

  public enum TickTrigger { EVENT_DRIVEN, TIMER }

  public record GateResult(
      String gateName,
      GateVerdict verdict,
      @Nullable String reason) {

    public enum GateVerdict { PASSED, BLOCKED }
  }

  public sealed interface TickOutcome
      permits TickOutcome.NoProposal, TickOutcome.ProposalGenerated,
              TickOutcome.Heartbeat {
    record NoProposal(String reason) implements TickOutcome {}
    record ProposalGenerated(
        int goalCount,
        SignalFilteringSummary filtering) implements TickOutcome {}
    record Heartbeat() implements TickOutcome {}
  }

  public record SignalFilteringSummary(
      int consensusSignals,
      int afterNamespaceFilter,
      int afterCategoryFilter,
      int afterSuppressionFilter,
      int afterAntiOscillationFilter,
      int afterBudgetFilter,
      int afterConflictFilter,
      int proposed) {}
}
```

**Tick-level gate names (canonical order):**

```
evolution_enabled → health_refresh → regression_monitor →
circuit_breaker_evaluate → circuit_breaker_check
```

These 5 gates are tick-level pass/fail decisions in `EvolutionTicker.tick()`. Each produces a single `GateResult` with PASSED or BLOCKED.

After all tick-level gates pass, `proposeImprovements()` iterates over consensus signals and independently filters each through: namespace matching, enabled category check, category suppression, anti-oscillation (recent rollback), budget check, conflict check. These are **per-signal filters**, not tick-level gates — signal A may pass budget but be blocked by conflict, while signal B passes both. The `SignalFilteringSummary` in `ProposalGenerated` reports aggregate filter survival counts instead of modelling per-signal outcomes as gates.

### TickTraceBuffer

Per-case ring buffer holding recent traces:

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class TickTraceBuffer implements Resettable {

  private static final int DEFAULT_CAPACITY = 100;

  private final ConcurrentHashMap<UUID, ArrayDeque<TickTrace>> buffers =
      new ConcurrentHashMap<>();

  public void record(TickTrace trace) {
    var deque = buffers.computeIfAbsent(trace.caseId(),
        k -> new ArrayDeque<>(DEFAULT_CAPACITY));
    synchronized (deque) {
      if (deque.size() >= DEFAULT_CAPACITY) {
        deque.pollFirst();
      }
      deque.addLast(trace);
    }
  }

  public List<TickTrace> recent(UUID caseId, int limit) {
    var deque = buffers.get(caseId);
    if (deque == null) return List.of();
    synchronized (deque) {
      return deque.stream()
          .sorted(Comparator.comparing(TickTrace::timestamp).reversed())
          .limit(limit)
          .toList();
    }
  }
}
```

### EventLog entries

| Event type | When | Payload |
|-----------|------|---------|
| `TICK_EVALUATED` | Gate blocked, proposal generated, or regression detected | TickTrace summary (gate results, outcome) |
| `TICK_HEARTBEAT` | Every 24 ticks or every 24 hours (whichever first) | Tick count since last heartbeat, current health score |

The heartbeat provides restart-survivable liveness evidence — distinguishes "healthy and quiet" from "broken and silent."

### EvolutionStreamBroadcaster (D13)

Dedicated broadcaster subscribing to 4 new CDI events:

```java
// rest, io.casehub.engine.rest
@ApplicationScoped
public class EvolutionStreamBroadcaster {

  private final BroadcastProcessor<EvolutionEvent> processor =
      BroadcastProcessor.create();

  void onCircuitBreakerChanged(
      @ObservesAsync CircuitBreakerStateChangedEvent event) {
    emit(event.caseId(), "circuit-breaker",
        Map.of("oldState", event.oldState().name(),
               "newState", event.newState().name()));
  }

  void onComplianceLevelChanged(
      @ObservesAsync ComplianceLevelChangedEvent event) {
    emit(event.caseId(), "compliance",
        Map.of("oldLevel", event.oldLevel().name(),
               "newLevel", event.newLevel().name()));
  }

  void onRegressionDetected(
      @ObservesAsync RegressionDetectedEvent event) {
    emit(event.caseId(), "regression",
        Map.of("improvementCaseId", event.improvementCaseId().toString(),
               "confidence", String.valueOf(event.confidence()),
               "category", event.category()));
  }

  void onTickEvaluated(
      @ObservesAsync TickEvaluatedEvent event) {
    emit(event.caseId(), "tick", Map.of(
        "outcome", event.trace().outcome() != null
            ? event.trace().outcome().getClass().getSimpleName() : "none"));
  }

  public Multi<EvolutionEvent> stream(UUID caseId) {
    return processor.toHotStream().filter(e -> caseId.equals(e.caseId()));
  }

  private void emit(UUID caseId, String type, Map<String, String> data) {
    try {
      processor.onNext(new EvolutionEvent(caseId, type, data, Instant.now()));
    } catch (BackPressureFailure e) {
      log.debug("Evolution event dropped due to backpressure: type={}, caseId={}", type, caseId);
    }
  }
}
```

### CDI event types

All in `engine-common`, package `io.casehub.engine.common.spi.event`:

```java
public record CircuitBreakerStateChangedEvent(
    UUID caseId, CircuitBreakerState oldState, CircuitBreakerState newState) {}

public record ComplianceLevelChangedEvent(
    UUID caseId, ComplianceLevel oldLevel, ComplianceLevel newLevel) {}

public record RegressionDetectedEvent(
    UUID caseId, UUID improvementCaseId, double confidence, String category) {}

public record TickEvaluatedEvent(UUID caseId, TickTrace trace) {}
```

Fired from existing components alongside their current EventLog writes:

| CDI event | Fired from | When |
|-----------|-----------|------|
| `CircuitBreakerStateChangedEvent` | `ImprovementCircuitBreaker.evaluate()` | State transitions (CLOSED→OPEN, OPEN→HALF_OPEN, HALF_OPEN→CLOSED) |
| `ComplianceLevelChangedEvent` | `ReadinessValidator.validate()` | Computed level differs from cached level |
| `RegressionDetectedEvent` | `RegressionDetector.onMetricsDegraded()` | Health degradation detected |
| `TickEvaluatedEvent` | `EvolutionTicker.tick()` | Notable outcome (gate blocked, proposal generated) |

## 2. Summarize — Layered Summaries (D21)

### SummarizationProvider SPI

```java
// api, io.casehub.api.spi.improvement
public interface SummarizationProvider {

  EvolutionSummary summarize(
      UUID caseId, String tenancyId, SummaryScope scope);
}
```

### SummaryScope

```java
// api, io.casehub.api.model.stigmergy
public record SummaryScope(
    @Nullable String areaId,
    @Nullable String category,
    @Nullable Integer timeWindowMinutes,
    @Nullable UUID improvementCaseId) {}
```

Scope parameters control altitude:
- No parameters → project-wide summary
- `areaId` → single area's improvement history and health trends
- `category` → category's outcome distribution and research direction
- `timeWindowMinutes` → constrain to recent time window
- `improvementCaseId` → single improvement's reasoning chain (investigation → research → design → outcome)

### EvolutionSummary

```java
// api, io.casehub.api.view
public record EvolutionSummary(
    SummaryScope scope,
    Instant computedAt,
    // Quantitative
    int totalImprovements,
    int successCount,
    int failureCount,
    int rejectionCount,
    int regressionCount,
    int rollbackCount,
    // Trends
    @Nullable Double healthTrend,     // delta over time window
    @Nullable Double successRateTrend,
    // Categories
    List<CategorySummary> categories,
    // Research
    List<ResearchDirectionSummary> researchDirections,
    // Notable events
    List<NotableEvent> notableEvents) {}

public record CategorySummary(
    String category,
    int totalOutcomes,
    double successRate,
    boolean suppressed,
    boolean paused) {}

public record ResearchDirectionSummary(
    String areaId,
    int hypothesesGenerated,
    int hypothesesApproved,
    int hypothesesRejected,
    @Nullable String currentFocus,
    List<String> activeHypotheses) {}

public record NotableEvent(
    CaseHubEventType type,
    Instant timestamp,
    Map<String, String> summary) {}
```

### DefaultSummarizationProvider

```java
// runtime-core, io.casehub.engine.internal.improvement
@DefaultBean
@ApplicationScoped
public class DefaultSummarizationProvider implements SummarizationProvider {

  private final EventLogRepository eventLogRepository;

  public EvolutionSummary summarize(
      UUID caseId, String tenancyId, SummaryScope scope) {
    // Query EventLog for improvement lifecycle events
    // Filter by scope parameters (area, category, time window)
    // Compute counts, ratios, trends from event data
    // Return structured summary
  }
}
```

The default implementation computes structured summaries from EventLog data — counts, ratios, trends. Blocks provides an LLM-powered implementation later that produces narrative summaries, theme extraction, and architecture reasoning chains.

## 3. Control — Manual Overrides

Direct mutations delegating to existing bean methods. Each mutation produces an EventLog entry for audit.

| Mutation | Delegates to | EventLog type |
|----------|-------------|---------------|
| `pauseCategory(caseId, category, durationMinutes)` | `ImprovementCategoryTracker.pauseCategory()` | `CATEGORY_PAUSED` |
| `unpauseCategory(caseId, category)` | `ImprovementCategoryTracker.unpauseCategory()` | `CATEGORY_UNPAUSED` |
| `resetCircuitBreaker(caseId)` | `ImprovementCircuitBreaker.manualReset()` | `CIRCUIT_BREAKER_RESET` |
| `addDenyPattern(caseId, pattern)` | `ImprovementBudgetEnforcer` dynamic deny set | `DENY_PATTERN_ADDED` |
| `removeDenyPattern(caseId, pattern)` | `ImprovementBudgetEnforcer` dynamic deny set | `DENY_PATTERN_REMOVED` |
| `triggerReadinessValidation(caseId, targetLevel)` | `ReadinessValidator.validate()` | `READINESS_EVALUATED` (existing) |

### New CaseHubEventType values

```java
CATEGORY_PAUSED,
CATEGORY_UNPAUSED,
DENY_PATTERN_ADDED,
DENY_PATTERN_REMOVED,
TICK_EVALUATED,
TICK_HEARTBEAT,
GATE_PENDING,       // lifecycle gate awaiting conductor decision
GATE_RESOLVED,      // conductor resolved a lifecycle gate
WATCH_PATTERN_ADDED,
WATCH_PATTERN_REMOVED,
IMPROVEMENT_BLOCKED,
IMPROVEMENT_UNBLOCKED
```

### Two-layer deny list (D16)

```java
// Query response
public record DenyPatternView(
    List<String> staticPatterns,     // read-only, from STRUCTURAL_DENIED_PATTERNS
    List<DynamicDenyEntry> dynamicPatterns) {

  public record DynamicDenyEntry(
      String pattern,
      String addedBy,
      Instant addedAt) {}
}
```

Dynamic deny patterns are EventLog-persisted. On restart, the current set reconstructs by replaying `DENY_PATTERN_ADDED` / `DENY_PATTERN_REMOVED` events in order.

### ImprovementBudgetEnforcer enhancement

```java
// runtime-core — new fields and methods
private final ConcurrentHashMap<UUID, Set<String>> dynamicDenyPatterns =
    new ConcurrentHashMap<>();

public void addDenyPattern(UUID caseId, String pattern) {
  dynamicDenyPatterns.computeIfAbsent(caseId, k -> ConcurrentHashMap.newKeySet())
      .add(pattern);
}

public void removeDenyPattern(UUID caseId, String pattern) {
  var patterns = dynamicDenyPatterns.get(caseId);
  if (patterns != null) patterns.remove(pattern);
  // Static patterns cannot be removed — check is in the API layer
}

public boolean isDenied(UUID caseId, ImprovementRequest request) {
  // Check static patterns (existing)
  if (STRUCTURAL_DENIED_PATTERNS.stream().anyMatch(
      p -> request.target().contains(p))) {
    return true;
  }
  // Check dynamic patterns (new)
  var dynamic = dynamicDenyPatterns.getOrDefault(caseId, Set.of());
  return dynamic.stream().anyMatch(p -> request.target().contains(p));
}
```

## 4. Steer — Lifecycle Gates and Smart Escalation

### GatePolicy (D20)

Configurable per-stage gate modes on `ImprovementConfig`. Gate stages are defined by `ImprovementStage.isGateCheckpoint()` — no separate enum.

```java
// api, io.casehub.api.model.stigmergy
public record GatePolicy(
    Map<ImprovementStage, GateMode> modes,
    @Nullable Integer gateTimeoutMinutes) {

  public enum GateMode {
    GATED,    // blocks until HIL resolves
    AUTO,     // auto-approve, escalation policy may still surface to inbox
    NOTIFY    // auto-approve + always surface in inbox
  }

  public GatePolicy {
    if (modes != null) {
      for (var stage : modes.keySet()) {
        if (!stage.isGateCheckpoint()) {
          throw new IllegalArgumentException(
              stage + " is not a gate checkpoint");
        }
      }
    }
  }

  public GateMode effectiveMode(ImprovementStage stage) {
    if (modes != null && modes.containsKey(stage)) {
      return modes.get(stage);
    }
    return stage == ImprovementStage.PR_REVIEW
        ? GateMode.GATED : GateMode.AUTO;
  }

  public int effectiveGateTimeoutMinutes() {
    return gateTimeoutMinutes != null ? gateTimeoutMinutes : 1440;
  }
}
```

Default: all stages AUTO except PR review (GATED — preserves existing devtown behaviour). The conductor tightens gates as desired via `setGatePolicy()`. Adding a new gate checkpoint requires only adding an `ImprovementStage` enum value and including it in `GATE_CHECKPOINTS` — the `Map`-based design needs no record changes.

GatePolicy is stored on `ImprovementConfig` (case context YAML). `setGatePolicy()` updates the case context. Survives restart via case context persistence — no EventLog entry needed.

### EscalationPolicy (D23)

Three composable layers — any layer triggering promotes the item to the conductor inbox:

```java
// api, io.casehub.api.model.stigmergy
public record EscalationPolicy(
    @Nullable CategoryEscalationRules categoryRules,
    @Nullable Double confidenceThreshold) {

  public double effectiveConfidenceThreshold() {
    return confidenceThreshold != null ? confidenceThreshold : 0.7;
  }
}

public record CategoryEscalationRules(
    List<String> alwaysEscalate,     // e.g. "architecture", "safety"
    List<String> neverEscalate) {}   // e.g. "lint-fix"

public record WatchPattern(
    String id,
    @Nullable String category,      // match on category
    @Nullable String areaId,         // match on capability area
    @Nullable String targetPattern,  // glob match on target paths
    @Nullable Integer minEstimatedSize,  // escalate if size >= N
    Instant createdAt) {}
```

**Persistence model:** `EscalationPolicy` (category rules + confidence threshold) is stored on `ImprovementConfig` — configuration, case context YAML. `WatchPattern` instances are dynamic runtime state, persisted via EventLog (`WATCH_PATTERN_ADDED` / `WATCH_PATTERN_REMOVED`). On restart, active watch patterns reconstruct from EventLog replay — same pattern as dynamic deny patterns (D16). Watch patterns are NOT stored on `EscalationPolicy` or `ImprovementConfig`. The `ConductorInboxManager` manages both watch patterns and inbox entries.

**`neverEscalate` semantics:** `neverEscalate` suppresses the **category rule layer only**. It does NOT suppress watch pattern matching or confidence scoring — those are independent concerns. If `lint-fix` is in `neverEscalate` but matches a watch pattern, the watch pattern still fires. If a category appears in both `alwaysEscalate` and `neverEscalate`, `neverEscalate` wins (explicit suppression overrides implicit escalation).

### EscalationProvider SPI

```java
// api, io.casehub.api.spi.improvement
public interface EscalationProvider {

  EscalationResult evaluate(
      UUID caseId, String tenancyId,
      ImprovementStage stage,
      EscalationContext context,
      EscalationPolicy policy,
      List<WatchPattern> activeWatchPatterns);
}

public record EscalationContext(
    @Nullable String category,
    @Nullable String areaId,
    @Nullable String target,
    @Nullable List<String> targetPaths,
    @Nullable Integer estimatedSize,
    Map<String, String> metadata) {}

public record EscalationResult(
    boolean escalate,
    double confidence,
    List<EscalationTrigger> triggers) {}

public record EscalationTrigger(
    EscalationLayer layer,
    String reason) {

  public enum EscalationLayer {
    CATEGORY_RULE,
    WATCH_PATTERN,
    CONFIDENCE_SCORE
  }
}
```

### DefaultEscalationProvider

```java
// runtime-core, io.casehub.engine.internal.improvement
@DefaultBean
@ApplicationScoped
public class DefaultEscalationProvider implements EscalationProvider {

  public EscalationResult evaluate(
      UUID caseId, String tenancyId,
      ImprovementStage stage,
      EscalationContext context,
      EscalationPolicy policy,
      List<WatchPattern> activeWatchPatterns) {
    var triggers = new ArrayList<EscalationTrigger>();

    // Layer 1: Category rules
    if (policy.categoryRules() != null && context.category() != null) {
      if (policy.categoryRules().neverEscalate().contains(context.category())) {
        // neverEscalate suppresses category rule layer — skip to Layer 2
      } else if (policy.categoryRules().alwaysEscalate()
          .contains(context.category())) {
        triggers.add(new EscalationTrigger(CATEGORY_RULE,
            "Category '" + context.category() + "' always escalates"));
      }
    }

    // Layer 2: Watch patterns (independent of neverEscalate)
    for (var pattern : activeWatchPatterns) {
      if (matches(pattern, context)) {
        triggers.add(new EscalationTrigger(WATCH_PATTERN,
            "Matches watch pattern '" + pattern.id() + "'"));
      }
    }

    // Layer 3: Confidence scoring (heuristic default)
    double confidence = computeConfidence(caseId, tenancyId, context);
    if (confidence < policy.effectiveConfidenceThreshold()) {
      triggers.add(new EscalationTrigger(CONFIDENCE_SCORE,
          "Confidence " + confidence + " below threshold "
          + policy.effectiveConfidenceThreshold()));
    }

    return new EscalationResult(!triggers.isEmpty(), confidence, triggers);
  }

  private double computeConfidence(
      UUID caseId, String tenancyId, EscalationContext context) {
    double confidence = 1.0;
    // CBR novelty: no similar past outcomes → lower confidence
    // Category failure rate: recent failures → lower confidence
    // Scope size: large changes → lower confidence
    return Math.max(0.0, Math.min(1.0, confidence));
  }
}
```

### ConductorInboxManager

Manages the conductor's inbox — pending gate decisions, escalated items, and watch patterns. Reconstructs state from EventLog on startup.

The existing `HilQueueEntry` (in `api/model/stigmergy`) serves the **research corpus** — tracking sources needing human retrieval. The conductor inbox is a separate concept with a separate type (`ConductorInboxEntry`) and separate lifecycle.

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ConductorInboxManager implements Resettable {

  private final ConcurrentHashMap<UUID, List<ConductorInboxEntry>> queues =
      new ConcurrentHashMap<>();
  private final ConcurrentHashMap<UUID, List<WatchPattern>> watchPatterns =
      new ConcurrentHashMap<>();
  private final EventLogRepository eventLogRepository;
  private final Event<GatePendingEvent> gatePendingEvent;

  // Startup reconstruction from EventLog
  void onStartup(@Observes StartupEvent event) {
    // Replay GATE_PENDING events not matched by GATE_RESOLVED
    // → reconstruct pending inbox entries
    // Replay WATCH_PATTERN_ADDED / WATCH_PATTERN_REMOVED
    // → reconstruct active watch patterns
  }

  public String enqueue(UUID caseId, ConductorInboxEntry entry) {
    queues.computeIfAbsent(caseId, k -> new CopyOnWriteArrayList<>())
        .add(entry);
    // Write EventLog entry (GATE_PENDING)
    // Fire CDI event → EvolutionStreamBroadcaster
    return entry.id();
  }

  public List<ConductorInboxEntry> pending(UUID caseId) {
    return queues.getOrDefault(caseId, List.of()).stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .toList();
  }

  public int pendingCount(UUID caseId) {
    return (int) queues.getOrDefault(caseId, List.of()).stream()
        .filter(e -> e.status() == ConductorInboxEntry.Status.PENDING)
        .count();
  }

  public void resolve(UUID caseId, String entryId,
      ConductorDecision decision) {
    // Update entry status → APPROVED/REJECTED/REDIRECTED
    // Write EventLog entry (GATE_RESOLVED)
    // Fire CDI event → case lifecycle resumes from checkpoint
  }

  public List<WatchPattern> activeWatchPatterns(UUID caseId) {
    return watchPatterns.getOrDefault(caseId, List.of());
  }

  public void addWatchPattern(UUID caseId, WatchPattern pattern) {
    watchPatterns.computeIfAbsent(caseId, k -> new CopyOnWriteArrayList<>())
        .add(pattern);
    // Write EventLog entry (WATCH_PATTERN_ADDED)
  }

  public void removeWatchPattern(UUID caseId, String patternId) {
    var patterns = watchPatterns.get(caseId);
    if (patterns != null) patterns.removeIf(p -> p.id().equals(patternId));
    // Write EventLog entry (WATCH_PATTERN_REMOVED)
  }
}
```

### ConductorInboxEntry

A NEW type for the conductor's gate pipeline — distinct from the research corpus `HilQueueEntry`:

```java
// api, io.casehub.api.model.stigmergy
public record ConductorInboxEntry(
    String id,
    ImprovementStage stage,
    Status status,
    // Context
    @Nullable String category,
    @Nullable String areaId,
    @Nullable UUID improvementCaseId,
    @Nullable String summary,
    // Escalation info
    List<EscalationTrigger> escalationTriggers,
    double confidence,
    // Timing
    Instant queuedAt,
    @Nullable Instant resolvedAt,
    @Nullable Integer timeoutMinutes,
    // Resolution
    @Nullable ConductorDecision decision) {

  public enum Status { PENDING, APPROVED, REJECTED, REDIRECTED, TIMED_OUT }
}

public record ConductorDecision(
    ConductorInboxEntry.Status outcome,
    @Nullable String redirectTarget,
    @Nullable String reason,
    @Nullable String feedback) {}
```

### Research steering (D22) — checkpoint model

The research pipeline does NOT block threads. When a gate is reached, the pipeline **checkpoints** its intermediate state and **returns** immediately. The improvement case lifecycle persists the checkpoint and transitions to GATED state. When the conductor resolves the gate, a CDI event triggers pipeline resumption from the checkpoint.

```java
// api, io.casehub.api.model.stigmergy
public sealed interface ResearchPipelineResult
    permits ResearchPipelineResult.Completed,
            ResearchPipelineResult.AwaitingGate {

  record Completed(
      List<ImprovementHypothesis> hypotheses) implements ResearchPipelineResult {}

  record AwaitingGate(
      ImprovementStage stage,
      String inboxEntryId,
      Object checkpoint) implements ResearchPipelineResult {}
}
```

```java
// Enhanced ResearchPipelineOrchestrator
public ResearchPipelineResult execute(
    UUID caseId, String tenancyId,
    ResearchDepth depth, CapabilityAreaAssessment area,
    Map<String, String> driveContext, GatePolicy gatePolicy,
    EscalationPolicy escalationPolicy) {

  var scope = scoper.scope(depth, area, driveContext);

  // Checkpoint 1: Research scope gate
  if (shouldGate(gatePolicy.effectiveMode(ImprovementStage.RESEARCH_SCOPE),
      caseId, tenancyId, ImprovementStage.RESEARCH_SCOPE,
      scope, escalationPolicy)) {
    var entryId = enqueueForApproval(
        caseId, ImprovementStage.RESEARCH_SCOPE, scope);
    return new ResearchPipelineResult.AwaitingGate(
        ImprovementStage.RESEARCH_SCOPE, entryId, scope);
  }

  return executeFromScope(caseId, tenancyId, depth, area,
      driveContext, scope, gatePolicy, escalationPolicy);
}

public ResearchPipelineResult resume(
    UUID caseId, String tenancyId,
    ImprovementStage fromStage, ConductorDecision decision,
    ResearchDepth depth, CapabilityAreaAssessment area,
    Map<String, String> driveContext, GatePolicy gatePolicy,
    EscalationPolicy escalationPolicy, Object checkpoint) {

  return switch (fromStage) {
    case RESEARCH_SCOPE -> {
      var scope = applyDecision(decision, (ResearchScope) checkpoint);
      yield executeFromScope(caseId, tenancyId, depth, area,
          driveContext, scope, gatePolicy, escalationPolicy);
    }
    case HYPOTHESIS_APPROVAL -> {
      var hypotheses = applyDecision(
          decision, (List<ImprovementHypothesis>) checkpoint);
      yield new ResearchPipelineResult.Completed(hypotheses);
    }
    default -> throw new IllegalArgumentException(
        "Cannot resume from " + fromStage);
  };
}

private ResearchPipelineResult executeFromScope(
    UUID caseId, String tenancyId,
    ResearchDepth depth, CapabilityAreaAssessment area,
    Map<String, String> driveContext, ResearchScope scope,
    GatePolicy gatePolicy, EscalationPolicy escalationPolicy) {

  var candidates = searcher.search(scope, depth);
  var analysis = analyzer.analyze(candidates, scope, depth);
  corpus.store(candidates, analysis);
  var hypotheses = hypothesisFormer.form(analysis, area);

  // Checkpoint 2: Hypothesis approval gate
  if (shouldGate(gatePolicy.effectiveMode(
      ImprovementStage.HYPOTHESIS_APPROVAL),
      caseId, tenancyId, ImprovementStage.HYPOTHESIS_APPROVAL,
      hypotheses, escalationPolicy)) {
    var entryId = enqueueForApproval(
        caseId, ImprovementStage.HYPOTHESIS_APPROVAL, hypotheses);
    return new ResearchPipelineResult.AwaitingGate(
        ImprovementStage.HYPOTHESIS_APPROVAL, entryId, hypotheses);
  }

  return new ResearchPipelineResult.Completed(hypotheses);
}
```

**Gate resolution flow:**
1. Pipeline returns `AwaitingGate` → case lifecycle persists checkpoint to case context
2. Case transitions to GATED state (`StageProgress.StageStatus.GATED`)
3. `ConductorInboxManager` holds the pending entry with `GATE_PENDING` EventLog
4. Conductor resolves gate → `ConductorInboxManager.resolve()` writes `GATE_RESOLVED` EventLog → fires CDI event
5. Case lifecycle receives event → calls `pipeline.resume()` with checkpoint and decision
6. Pipeline continues from checkpoint with the conductor's modifications applied

**Timeout:** `ConductorInboxManager` checks pending entries on each tick. Entries past `gateTimeoutMinutes` are auto-rejected (`TIMED_OUT` status) with a `GATE_RESOLVED` EventLog entry. The case lifecycle handles the timeout like a REJECTED decision — the improvement case transitions to a terminal state.

**Restart recovery:** On restart, `ConductorInboxManager.onStartup()` reconstructs pending entries from `GATE_PENDING` events not matched by `GATE_RESOLVED`. Improvement cases in GATED state have their checkpoint persisted in case context. When a gate subsequently resolves, the case lifecycle resumes the pipeline normally.

**Concurrency:** Each improvement case has its own checkpoint and inbox entry — fully independent per-case.

### Manual coordination — semaphore (D26)

The `ConflictDetector` handles automatic file-level conflict avoidance. The conductor needs manual coordination for higher-level ordering: "don't start this improvement until that one completes" or "these two touch the same architectural concern, serialize them."

```java
// runtime-core, io.casehub.engine.internal.improvement
@ApplicationScoped
public class ImprovementCoordinator implements Resettable {

  private final ConcurrentHashMap<UUID, Map<UUID, UUID>> blocks =
      new ConcurrentHashMap<>();  // caseId → improvementId → blockedBy
  private final EventLogRepository eventLogRepository;

  // Startup reconstruction from EventLog
  void onStartup(@Observes StartupEvent event) {
    // Replay IMPROVEMENT_BLOCKED / IMPROVEMENT_UNBLOCKED events
    // → reconstruct active blocks
  }

  public void block(UUID caseId, UUID improvementId, UUID blockedBy) {
    blocks.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>())
        .put(improvementId, blockedBy);
    // Write EventLog entry (IMPROVEMENT_BLOCKED)
  }

  public void unblock(UUID caseId, UUID improvementId) {
    var caseBlocks = blocks.get(caseId);
    if (caseBlocks != null) caseBlocks.remove(improvementId);
    // Write EventLog entry (IMPROVEMENT_UNBLOCKED)
  }

  public boolean isBlocked(UUID caseId, UUID improvementId) {
    var caseBlocks = blocks.get(caseId);
    if (caseBlocks == null) return false;
    var blocker = caseBlocks.get(improvementId);
    return blocker != null;
  }

  public @Nullable UUID blockedBy(UUID caseId, UUID improvementId) {
    var caseBlocks = blocks.get(caseId);
    return caseBlocks != null ? caseBlocks.get(improvementId) : null;
  }
}
```

The `ImprovementCoordinator` complements `ConflictDetector`:

| Concern | Mechanism | Trigger |
|---------|-----------|---------|
| File-level conflicts | `ConflictDetector` (automatic) | `proposeImprovements()` checks `targetPaths` overlap |
| Architectural ordering | `ImprovementCoordinator` (manual) | Conductor blocks/unblocks via command centre mutation |

Manual blocks operate at two levels: (1) At **proposal time** — `EvolutionTicker.tick()` checks whether a proposed improvement's category/target is blocked, preventing new improvement cases from being spawned. (2) At **execution time** — the improvement case's worker dispatch checks whether the improvement case UUID is blocked, pausing execution of an already-spawned improvement until unblocked. The ticker handles "don't start this," the case lifecycle handles "pause this until the other one finishes."

## 5. Review — Artifact Trail (D24)

### ArtifactManifest

```java
// api, io.casehub.api.model.stigmergy
public record ArtifactManifest(
    UUID improvementCaseId,
    List<ArtifactEntry> entries) {}

public record ArtifactEntry(
    String path,
    ArtifactType type,
    ImprovementStage stage,
    Instant createdAt,
    @Nullable String summary) {

  public enum ArtifactType {
    ANALYSIS,
    LITERATURE_REVIEW,
    DESIGN,
    DECISION,
    ADVERSARIAL_DEBATE,
    DIFF,
    RESEARCH_FINDING,
    HYPOTHESIS
  }
}
```

Improvement workers record artifact entries as they produce outputs. The manifest is stored in the improvement case's working context (key `improvement:artifact-manifest`). The command centre query returns the manifest; content is read from disk via the existing diff viewer infrastructure.

### Artifact recording by improvement workers

Each improvement worker (introspect, research, implement, review) records its outputs:

```java
// In improvement workers — record artifact entry
var manifest = readOrCreateManifest(caseId, tenancyId);
manifest.entries().add(new ArtifactEntry(
    outputPath,
    ArtifactType.ANALYSIS,
    ImprovementStage.RESEARCH_SCOPE,
    Instant.now(),
    "Introspection result for dependency-update target"));
writeManifest(caseId, tenancyId, manifest);
```

## 6. API Surface — DefaultEngineEvolutionApi (D12)

```java
// rest, io.casehub.engine.rest.service
@ApplicationScoped
@McpDomain("engine/evolution")
public class DefaultEngineEvolutionApi {

  // --- OBSERVE ---

  @PlatformQuery("Get the current evolution state snapshot")
  public EvolutionStateSnapshot getEvolutionState(
      @PathParam UUID caseId, String tenancyId) { ... }

  @PlatformQuery("Get readiness report for a target compliance level")
  public ReadinessReport getReadinessReport(
      @PathParam UUID caseId, String tenancyId,
      String targetLevel) { ... }

  @PlatformQuery("Get recent tick trace history")
  public List<TickTrace> getTickHistory(
      @PathParam UUID caseId, String tenancyId,
      @Nullable Integer limit) { ... }

  @PlatformQuery("Get research corpus contents")
  public ResearchCorpusView getResearchCorpus(
      @PathParam UUID caseId, String tenancyId) { ... }

  @PlatformQuery("Get deny pattern configuration")
  public DenyPatternView getDenyPatterns(
      @PathParam UUID caseId, String tenancyId) { ... }

  // --- SUMMARIZE ---

  @PlatformQuery("Get layered evolution summary")
  public EvolutionSummary getSummary(
      @PathParam UUID caseId, String tenancyId,
      @Nullable String areaId, @Nullable String category,
      @Nullable Integer timeWindowMinutes,
      @Nullable UUID improvementCaseId) { ... }

  // --- CONTROL ---

  @PlatformMutation("Pause a category from receiving improvements")
  public void pauseCategory(
      @PathParam UUID caseId, String tenancyId,
      String category, int durationMinutes) { ... }

  @PlatformMutation("Unpause a previously paused category")
  public void unpauseCategory(
      @PathParam UUID caseId, String tenancyId,
      String category) { ... }

  @PlatformMutation("Manually reset the circuit breaker to CLOSED")
  public void resetCircuitBreaker(
      @PathParam UUID caseId, String tenancyId) { ... }

  @PlatformMutation("Add a runtime deny pattern")
  public void addDenyPattern(
      @PathParam UUID caseId, String tenancyId,
      String pattern) { ... }

  @PlatformMutation("Remove a runtime deny pattern")
  public void removeDenyPattern(
      @PathParam UUID caseId, String tenancyId,
      String pattern) { ... }

  @PlatformMutation("Trigger readiness validation")
  public ReadinessReport triggerReadinessValidation(
      @PathParam UUID caseId, String tenancyId,
      String targetLevel) { ... }

  // --- STEER ---

  @PlatformQuery("Get the conductor inbox — pending decisions and escalations")
  public List<ConductorInboxEntry> getInbox(
      @PathParam UUID caseId, String tenancyId) { ... }

  @PlatformMutation("Resolve a pending gate decision")
  public void resolveGate(
      @PathParam UUID caseId, String tenancyId,
      String entryId, String decision,
      @Nullable String redirectTarget,
      @Nullable String reason,
      @Nullable String feedback) { ... }

  @PlatformMutation("Set gate policy for lifecycle stages")
  public void setGatePolicy(
      @PathParam UUID caseId, String tenancyId,
      Map<String, String> stageModes) { ... }

  @PlatformMutation("Add a watch pattern for smart escalation")
  public void addWatchPattern(
      @PathParam UUID caseId, String tenancyId,
      @Nullable String category, @Nullable String areaId,
      @Nullable String targetPattern,
      @Nullable Integer minEstimatedSize) { ... }

  @PlatformMutation("Remove a watch pattern")
  public void removeWatchPattern(
      @PathParam UUID caseId, String tenancyId,
      String patternId) { ... }

  @PlatformMutation("Block an improvement until another completes")
  public void blockImprovement(
      @PathParam UUID caseId, String tenancyId,
      UUID improvementId, UUID blockedBy) { ... }

  @PlatformMutation("Unblock a previously blocked improvement")
  public void unblockImprovement(
      @PathParam UUID caseId, String tenancyId,
      UUID improvementId) { ... }

  // --- REVIEW ---

  @PlatformQuery("Get artifact trail for an improvement stream")
  public ArtifactManifest getArtifactTrail(
      @PathParam UUID caseId, String tenancyId,
      UUID improvementCaseId) { ... }

  @PlatformQuery("Get progress view for an active improvement stream")
  public ImprovementStreamView getStreamProgress(
      @PathParam UUID caseId, String tenancyId,
      UUID improvementCaseId) { ... }
}
```

## 7. Bootstrap from Zero (D14)

No separate infrastructure. The bootstrap is the L0→L1→L2→L3 readiness progression conducted through the command centre API.

### Flow

```
L0_INERT (default)
  │
  │ getReadinessReport(caseId, "L1_OBSERVE")
  │   → "stability-area-registered: satisfied"
  │   → "stability-has-terminal-events: NOT satisfied"
  │   → remediation: "Run cases to generate CASE_COMPLETED events"
  │
  │ [HIL runs cases, events accumulate]
  │
  │ triggerReadinessValidation(caseId, "L1_OBSERVE")
  │   → passed: true
  │
  ▼
L1_OBSERVE
  │ Health scores computed and tracked
  │ Dashboard shows per-area scores and trends
  │ Circuit breaker evaluated (but evolution disabled)
  │
  │ getReadinessReport(caseId, "L2_PROPOSE")
  │   → "evolution-enabled: NOT satisfied"
  │   → remediation: "Set evolutionEnabled=true in ImprovementConfig"
  │
  │ [HIL configures ImprovementConfig via case context]
  │
  │ triggerReadinessValidation(caseId, "L2_PROPOSE")
  │   → passed: true
  │
  ▼
L2_PROPOSE
  │ EvolutionTicker runs full gate pipeline
  │ Proposals generated, improvement cases spawned
  │ All gates initially GATED (or AUTO with conservative escalation)
  │ HIL conducts via inbox
  │
  │ getReadinessReport(caseId, "L3_AUTONOMOUS")
  │   → "rollback-policy-configured: NOT satisfied"
  │   → remediation: "Configure RollbackPolicy with autoRevertThreshold"
  │
  │ [HIL configures auto-integration and rollback]
  │
  ▼
L3_AUTONOMOUS
  │ Improvements can auto-integrate
  │ Rollback and circuit breaker provide safety net
  │ HIL conducts via inbox — escalations surface high-stakes decisions
```

## 8. Configuration Model

New fields on `ImprovementConfig`:

```java
// api/model/stigmergy — extended ImprovementConfig
public record ImprovementConfig(
    // ... existing fields ...
    // --- New fields for #1132 ---
    @Nullable GatePolicy gatePolicy,
    @Nullable EscalationPolicy escalationPolicy) {

  public GatePolicy effectiveGatePolicy() {
    return gatePolicy != null ? gatePolicy
        : new GatePolicy(Map.of(), null);
  }

  public EscalationPolicy effectiveEscalationPolicy() {
    return escalationPolicy != null ? escalationPolicy
        : new EscalationPolicy(null, null);
  }
}
```

### YAML record codegen

| Record | Codegen entry |
|--------|--------------|
| `GatePolicy` | `io.casehub.api.model.stigmergy.GatePolicy` |
| `EscalationPolicy` | `io.casehub.api.model.stigmergy.EscalationPolicy` |
| `CategoryEscalationRules` | `io.casehub.api.model.stigmergy.CategoryEscalationRules` |

`WatchPattern` is NOT in YAML codegen — it is EventLog-persisted runtime state, not case context configuration.

## 9. Module Placement

| Component | Module | Package |
|-----------|--------|---------|
| `TickTrace` | `api` | `io.casehub.api.model.stigmergy` |
| `GatePolicy` | `api` | `io.casehub.api.model.stigmergy` |
| `EscalationPolicy` | `api` | `io.casehub.api.model.stigmergy` |
| `CategoryEscalationRules` | `api` | `io.casehub.api.model.stigmergy` |
| `WatchPattern` | `api` | `io.casehub.api.model.stigmergy` |
| `ArtifactManifest` | `api` | `io.casehub.api.model.stigmergy` |
| `ArtifactEntry` | `api` | `io.casehub.api.model.stigmergy` |
| `EvolutionStateSnapshot` | `api` | `io.casehub.api.view` |
| `EvolutionSummary` | `api` | `io.casehub.api.view` |
| `DenyPatternView` | `api` | `io.casehub.api.view` |
| `EvolutionEvent` | `api` | `io.casehub.api.view` |
| `ConductorInboxEntry` | `api` | `io.casehub.api.model.stigmergy` |
| `ConductorDecision` | `api` | `io.casehub.api.model.stigmergy` |
| `ResearchPipelineResult` | `api` | `io.casehub.api.model.stigmergy` |
| `EscalationResult` | `api` | `io.casehub.api.model.stigmergy` |
| `EscalationContext` | `api` | `io.casehub.api.model.stigmergy` |
| `EscalationTrigger` | `api` | `io.casehub.api.model.stigmergy` |
| `SummaryScope` | `api` | `io.casehub.api.model.stigmergy` |
| `SummarizationProvider` SPI | `api` | `io.casehub.api.spi.improvement` |
| `EscalationProvider` SPI | `api` | `io.casehub.api.spi.improvement` |
| CDI events (4 types) | `engine-common` | `io.casehub.engine.common.spi.event` |
| `TickTraceBuffer` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ConductorInboxManager` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `DefaultSummarizationProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `DefaultEscalationProvider` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementCoordinator` | `runtime-core` | `io.casehub.engine.internal.improvement` |
| `ImprovementStreamView` | `api` | `io.casehub.api.view` |
| `ImprovementStage` | `api` | `io.casehub.api.model.stigmergy` |
| `StageProgress` | `api` | `io.casehub.api.view` |
| `DefaultEngineEvolutionApi` | `rest` | `io.casehub.engine.rest.service` |
| `EvolutionStreamBroadcaster` | `rest` | `io.casehub.engine.rest` |

## 10. Test Strategy

### Unit tests

| Test class | What it covers |
|------------|---------------|
| `TickTraceBufferTest` | Ring buffer: capacity, ordering, eviction, per-case isolation |
| `ConductorInboxManagerTest` | Enqueue, pending list, resolve, timeout, per-case isolation, EventLog restart reconstruction, watch pattern lifecycle |
| `DefaultSummarizationProviderTest` | Summary computation from EventLog data, scope filtering, trend calculation |
| `DefaultEscalationProviderTest` | All 3 layers: category rules, watch pattern matching, confidence scoring, composition (any-trigger union) |
| `GatePolicyTest` | Gate mode defaults, per-stage configuration, timeout handling |
| `EscalationPolicyTest` | Category rules, watch pattern matching, confidence threshold |
| `ArtifactManifestTest` | Entry recording, chronological ordering, type filtering |
| `DenyPatternDynamicTest` | Add/remove dynamic patterns, static immutability, effective deny set = static ∪ dynamic |
| `EvolutionTickerTickTraceTest` | tick() returns TickTrace with correct gate results, heartbeat emission |

### Integration tests

| Test class | What it covers |
|------------|---------------|
| `EvolutionApiIntegrationTest` | Full API surface: snapshot query, control mutations, gate resolution, stream subscription |
| `EvolutionStreamBroadcasterTest` | CDI events → broadcaster → stream filtering by caseId |
| `BootstrapProgressionIntegrationTest` | L0→L1→L2→L3 via command centre mutations, readiness validation at each level |
| `SmartEscalationIntegrationTest` | Composable escalation: category rule fires, watch pattern fires, confidence drops — all surface in inbox |
| `ResearchSteeringIntegrationTest` | Research scope gate → HIL modifies scope → modified scope used → hypothesis gate → HIL approves subset |
| `ArtifactTrailIntegrationTest` | Improvement lifecycle produces artifacts → manifest records entries → API returns chronological trail |
| `DenyPatternPersistenceTest` | Dynamic patterns survive restart via EventLog replay |
| `ConductorInboxPersistenceTest` | Pending gate decisions survive restart via EventLog replay |
| `ImprovementCoordinatorPersistenceTest` | Manual blocks survive restart via EventLog replay |
| `ResearchPipelineCheckpointTest` | Pipeline returns AwaitingGate at checkpoint → resume after gate resolution → pipeline continues from checkpoint |

### Critical test scenarios

1. **Dashboard snapshot composition:** All sources return data → single composed snapshot with correct fields from each source
2. **Tick trace gate visibility:** tick() blocked at circuit breaker → TickTrace shows `circuit_breaker_check: BLOCKED`, earlier gates `PASSED`
3. **Smart escalation union:** Category rule says "always escalate architecture", watch pattern doesn't match, confidence is high → item escalates (category rule alone is sufficient)
4. **Gate timeout:** GATED stage with 1440-minute timeout → entry expires → auto-reject with TIMED_OUT status
5. **Research scope modification:** HIL modifies research scope via `modifyResearchScope()` → modified scope used for search → research findings reflect new keywords
6. **Hypothesis selective approval:** HIL approves 2 of 5 hypotheses → only approved hypotheses become improvement signals → rejected hypotheses recorded in manifest
7. **Deny pattern safety:** Attempt to remove a static deny pattern via API → rejected (safety invariant preserved)
8. **Bootstrap L0→L1:** No events → readiness fails → generate events → readiness passes → compliance level changes → CDI event fired → stream receives notification
9. **Heartbeat liveness:** 24 uneventful ticks → TICK_HEARTBEAT EventLog entry persisted → restart → first tick has context ("loop was active")
10. **Artifact trail completeness:** Improvement runs introspect → research → implement → integrate → artifact manifest has 4+ entries in chronological order

## References

- `EvolutionTicker.java` (line 46 — current void tick() to return TickTrace; no production caller exists yet — only test callers)
- `ImprovementCircuitBreaker.java` (line 75 — manualReset())
- `ImprovementCategoryTracker.java` (line 93 — pauseCategory(), line 109 — unpauseCategory())
- `ImprovementBudgetEnforcer.java` — STRUCTURAL_DENIED_PATTERNS
- `HealthScoreTracker.java` (line 88 — latestSnapshot())
- `ReadinessValidator.java` — validate()
- `CaseStreamBroadcaster.java` — BroadcastProcessor pattern
- `ExecutionStateBroadcaster.java` — dedicated broadcaster precedent
- `DefaultEngineCaseControlApi.java` — @McpDomain pattern
- `DefaultEngineEventLogApi.java` — @PlatformQuery pattern
- `CaseHubEventType.java` — existing event types
- `ResearchPipelineOrchestrator.java` — research pipeline
- `HilQueueEntry.java` — existing research corpus queue entry (NOT modified; new ConductorInboxEntry is a separate type)
- `PlanItemStateChangedEvent.java` — CDI event precedent in engine-common
- `2026-09-20-continuous-evolution-loop-design.md` (#1115) — evolution loop infrastructure
- `2026-09-21-evolution-readiness-methodology-design.md` (#1131) — readiness methodology
- Decisions D10–D27 in `decisions.md`
- Memory: `command-centre-conductor` — developer as conductor via observable UI
- Memory: `evolution-from-zero` — bootstrap from zero with HIL briefing
