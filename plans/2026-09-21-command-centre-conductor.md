# Command Centre Conductor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1132 — feat: Command centre conductor
**Issue group:** #1131, #1132

**Goal:** Build the conductor's interface to the evolution loop — five layers (Observe, Summarize, Control, Steer, Review) exposed through a single `@McpDomain("engine/evolution")` API with real-time event streaming, configurable lifecycle gates, and smart escalation.

**Architecture:** Thin observability and control layer over existing evolution infrastructure. `EvolutionTicker.tick()` changes from void to returning `TickTrace`. Four new CDI events feed a dedicated `EvolutionStreamBroadcaster`. `ConductorInboxManager` manages lifecycle gates with composable 3-layer escalation. Research pipeline gains checkpoint/resume pattern for non-blocking gates. Single `DefaultEngineEvolutionApi` exposes all queries and mutations.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, Mutiny (BroadcastProcessor for SSE), Jackson (serialization)

## Global Constraints

- All model records in `api/model/stigmergy`, view records in `api/view`, SPIs in `api/spi/improvement`
- Runtime implementations in `runtime-core`, package `io.casehub.engine.internal.improvement`
- REST/API in `rest`, package `io.casehub.engine.rest.service` (API) and `io.casehub.engine.rest` (broadcaster)
- CDI events in `engine-common`, package `io.casehub.engine.common.spi.event`
- Test classes must be `*Test.java`, never `*IT.java`
- No `@DefaultBean` on multi-instance SPIs (D7 from #1131)
- `@DefaultBean` IS correct for 1:1 SPI replacement (SummarizationProvider, EscalationProvider)
- Build: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
- Test: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest=<TestClass>`
- `EventLogRepository.findByCaseAndTypes()` requires `tenancyId`
- Inject repos by SPI interface, not concrete class

---

## Batch 1: Observe — Tick Visibility and Event Streaming

After this batch: `EvolutionTicker.tick()` returns a `TickTrace` with per-gate results. Recent traces stored in a ring buffer. Notable outcomes produce EventLog entries. Four CDI events feed a dedicated `EvolutionStreamBroadcaster` for real-time SSE.

### Task 1: TickTrace model + ImprovementStage enum + CaseHubEventType additions

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/TickTrace.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementStage.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add 12 new enum values
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceTest.java`

**Interfaces:**
- Produces: `TickTrace` record (used by EvolutionTicker, TickTraceBuffer, EvolutionStreamBroadcaster), `ImprovementStage` enum (used by GatePolicy, ConductorInboxEntry, ArtifactEntry, StageProgress)

- [ ] **Step 1: Create ImprovementStage enum**

```java
// api/src/main/java/io/casehub/api/model/stigmergy/ImprovementStage.java
package io.casehub.api.model.stigmergy;

import java.util.Set;

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

- [ ] **Step 2: Create TickTrace record**

```java
// api/src/main/java/io/casehub/api/model/stigmergy/TickTrace.java
package io.casehub.api.model.stigmergy;

import jakarta.annotation.Nullable;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record TickTrace(
    UUID caseId,
    Instant timestamp,
    TickTrigger trigger,
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

- [ ] **Step 3: Add new CaseHubEventType values**

Add after `READINESS_EVALUATED` in `CaseHubEventType.java`:

```java
CATEGORY_PAUSED,
CATEGORY_UNPAUSED,
DENY_PATTERN_ADDED,
DENY_PATTERN_REMOVED,
TICK_EVALUATED,
TICK_HEARTBEAT,
GATE_PENDING,
GATE_RESOLVED,
WATCH_PATTERN_ADDED,
WATCH_PATTERN_REMOVED,
IMPROVEMENT_BLOCKED,
IMPROVEMENT_UNBLOCKED
```

- [ ] **Step 4: Write TickTrace unit test**

```java
// runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceTest.java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.ImprovementStage;
import io.casehub.api.model.stigmergy.TickTrace;
import io.casehub.api.model.stigmergy.TickTrace.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class TickTraceTest {

  @Test
  void traceRecordsGateResults() {
    var trace = new TickTrace(
        UUID.randomUUID(), Instant.now(), TickTrigger.EVENT_DRIVEN,
        List.of(
            new GateResult("evolution_enabled", GateResult.GateVerdict.PASSED, null),
            new GateResult("circuit_breaker_check", GateResult.GateVerdict.BLOCKED,
                "health below threshold")),
        null);

    assertThat(trace.gates()).hasSize(2);
    assertThat(trace.gates().get(0).verdict())
        .isEqualTo(GateResult.GateVerdict.PASSED);
    assertThat(trace.gates().get(1).verdict())
        .isEqualTo(GateResult.GateVerdict.BLOCKED);
    assertThat(trace.gates().get(1).reason()).isEqualTo("health below threshold");
  }

  @Test
  void proposalOutcomeIncludesFilteringSummary() {
    var summary = new SignalFilteringSummary(10, 8, 7, 5, 5, 4, 3, 3);
    var outcome = new TickOutcome.ProposalGenerated(3, summary);

    assertThat(outcome.goalCount()).isEqualTo(3);
    assertThat(outcome.filtering().consensusSignals()).isEqualTo(10);
    assertThat(outcome.filtering().proposed()).isEqualTo(3);
  }

  @Test
  void improvementStageGateCheckpoints() {
    assertThat(ImprovementStage.RESEARCH_SCOPE.isGateCheckpoint()).isTrue();
    assertThat(ImprovementStage.HYPOTHESIS_APPROVAL.isGateCheckpoint()).isTrue();
    assertThat(ImprovementStage.IMPLEMENTATION_PLAN.isGateCheckpoint()).isTrue();
    assertThat(ImprovementStage.PR_REVIEW.isGateCheckpoint()).isTrue();
    assertThat(ImprovementStage.INTROSPECT.isGateCheckpoint()).isFalse();
    assertThat(ImprovementStage.SEARCH.isGateCheckpoint()).isFalse();
    assertThat(ImprovementStage.IMPLEMENT.isGateCheckpoint()).isFalse();
  }
}
```

- [ ] **Step 5: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest=TickTraceTest`
Expected: 3 tests PASS

- [ ] **Step 6: Compile all modified modules**

Run: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/stigmergy/TickTrace.java \
       api/src/main/java/io/casehub/api/model/stigmergy/ImprovementStage.java \
       api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceTest.java
git commit -m "feat(#1132): add TickTrace, ImprovementStage, and new CaseHubEventType values"
```

### Task 2: TickTraceBuffer + EvolutionTicker instrumentation

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/TickTraceBuffer.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java` — tick() returns TickTrace
- Modify: All callers of `EvolutionTicker.tick()` (update to handle return value)
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceBufferTest.java`

**Interfaces:**
- Consumes: `TickTrace` (from Task 1)
- Produces: `TickTraceBuffer.recent(caseId, limit)` (used by EvolutionStateSnapshot composition), `EvolutionTicker.tick()` → `TickTrace` (used by EvolutionStreamBroadcaster, callers)

- [ ] **Step 1: Write TickTraceBuffer failing test**

```java
// runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceBufferTest.java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.TickTrace;
import io.casehub.api.model.stigmergy.TickTrace.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class TickTraceBufferTest {

  private TickTraceBuffer buffer;

  @BeforeEach
  void setUp() {
    buffer = new TickTraceBuffer();
  }

  @Test
  void emptyBufferReturnsEmptyList() {
    assertThat(buffer.recent(UUID.randomUUID(), 10)).isEmpty();
  }

  @Test
  void recordAndRetrieveTraces() {
    var caseId = UUID.randomUUID();
    var trace1 = makeTrace(caseId, Instant.now().minusSeconds(60));
    var trace2 = makeTrace(caseId, Instant.now());

    buffer.record(trace1);
    buffer.record(trace2);

    var recent = buffer.recent(caseId, 10);
    assertThat(recent).hasSize(2);
    assertThat(recent.get(0).timestamp()).isAfterOrEqualTo(recent.get(1).timestamp());
  }

  @Test
  void respectsLimitParameter() {
    var caseId = UUID.randomUUID();
    for (int i = 0; i < 10; i++) {
      buffer.record(makeTrace(caseId, Instant.now().plusSeconds(i)));
    }
    assertThat(buffer.recent(caseId, 3)).hasSize(3);
  }

  @Test
  void perCaseIsolation() {
    var case1 = UUID.randomUUID();
    var case2 = UUID.randomUUID();
    buffer.record(makeTrace(case1, Instant.now()));
    buffer.record(makeTrace(case2, Instant.now()));

    assertThat(buffer.recent(case1, 10)).hasSize(1);
    assertThat(buffer.recent(case2, 10)).hasSize(1);
  }

  @Test
  void evictsOldestWhenCapacityReached() {
    var caseId = UUID.randomUUID();
    for (int i = 0; i < 150; i++) {
      buffer.record(makeTrace(caseId, Instant.now().plusSeconds(i)));
    }
    assertThat(buffer.recent(caseId, 200)).hasSize(100);
  }

  private TickTrace makeTrace(UUID caseId, Instant timestamp) {
    return new TickTrace(caseId, timestamp, TickTrigger.EVENT_DRIVEN,
        List.of(new GateResult("evolution_enabled",
            GateResult.GateVerdict.PASSED, null)),
        null);
  }
}
```

- [ ] **Step 2: Run test, verify fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest=TickTraceBufferTest`
Expected: FAIL — `TickTraceBuffer` class not found

- [ ] **Step 3: Implement TickTraceBuffer**

```java
// runtime-core/src/main/java/io/casehub/engine/internal/improvement/TickTraceBuffer.java
package io.casehub.engine.internal.improvement;

import io.casehub.api.model.stigmergy.TickTrace;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.ArrayDeque;
import java.util.Comparator;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

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

  @Override
  public void reset() {
    buffers.clear();
  }
}
```

- [ ] **Step 4: Run test, verify passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest=TickTraceBufferTest`
Expected: 5 tests PASS

- [ ] **Step 5: Modify EvolutionTicker.tick() to return TickTrace**

Read `EvolutionTicker.java` via `ide_read_file`. Change the `tick()` method signature from `public void tick(...)` to `public TickTrace tick(...)`. Instrument each gate to build a `List<GateResult>`. Return the composed `TickTrace` at each exit point. Record the trace in `TickTraceBuffer`. Fire `TickEvaluatedEvent` CDI event on notable outcomes. Inject `TickTraceBuffer` and `Event<TickEvaluatedEvent>` as constructor dependencies.

Key gates to trace:
1. `evolution_enabled` — `config.effectiveEvolutionEnabled()` check
2. `health_refresh` — `healthTracker.refresh()` call
3. `regression_monitor` — `regressionDetector.checkActiveMonitors()` call
4. `circuit_breaker_evaluate` — `circuitBreaker.evaluate()` call
5. `circuit_breaker_check` — `circuitBreaker.state()` check

- [ ] **Step 6: Update EvolutionTicker callers**

Use `ide_find_references` on `EvolutionTicker.tick()` to find all callers. Update each caller to handle the `TickTrace` return value (typically ignore it — the trace is stored in the buffer automatically).

- [ ] **Step 7: Update EvolutionTickerTest**

Update existing `EvolutionTickerTest` assertions to verify the returned `TickTrace`. Assert gate results for each test scenario (evolution disabled → `evolution_enabled: BLOCKED`, circuit breaker open → `circuit_breaker_check: BLOCKED`, proposal generated → `ProposalGenerated` outcome with filtering summary).

- [ ] **Step 8: Run full test suite for runtime-core improvement package**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core -Dtest="io.casehub.engine.internal.improvement.*Test"`
Expected: All tests PASS (existing tests updated for new return type)

- [ ] **Step 9: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/improvement/TickTraceBuffer.java \
       runtime-core/src/main/java/io/casehub/engine/internal/improvement/EvolutionTicker.java \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/TickTraceBufferTest.java \
       runtime-core/src/test/java/io/casehub/engine/internal/improvement/EvolutionTickerTest.java
git commit -m "feat(#1132): add TickTraceBuffer and instrument EvolutionTicker with TickTrace return"
```

### Task 3: CDI events + EvolutionStreamBroadcaster

**Files:**
- Create: `engine-common/src/main/java/io/casehub/engine/common/spi/event/CircuitBreakerStateChangedEvent.java`
- Create: `engine-common/src/main/java/io/casehub/engine/common/spi/event/ComplianceLevelChangedEvent.java`
- Create: `engine-common/src/main/java/io/casehub/engine/common/spi/event/RegressionDetectedEvent.java`
- Create: `engine-common/src/main/java/io/casehub/engine/common/spi/event/TickEvaluatedEvent.java`
- Create: `api/src/main/java/io/casehub/api/view/EvolutionEvent.java`
- Create: `rest/src/main/java/io/casehub/engine/rest/EvolutionStreamBroadcaster.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCircuitBreaker.java` — fire CDI event on state transitions
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ReadinessValidator.java` — fire CDI event on compliance level change
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/RegressionDetector.java` — fire CDI event on regression detection
- Test: `rest/src/test/java/io/casehub/engine/rest/EvolutionStreamBroadcasterTest.java`

**Interfaces:**
- Consumes: `TickTrace` (from Task 1), `CircuitBreakerState` (existing), `ComplianceLevel` (existing)
- Produces: `EvolutionStreamBroadcaster.stream(caseId)` → `Multi<EvolutionEvent>` (used by API), CDI events consumed by broadcaster

- [ ] **Step 1: Create 4 CDI event records in engine-common**

Each is a simple record in `io.casehub.engine.common.spi.event`:

```java
public record CircuitBreakerStateChangedEvent(
    UUID caseId, CircuitBreakerState oldState, CircuitBreakerState newState) {}

public record ComplianceLevelChangedEvent(
    UUID caseId, ComplianceLevel oldLevel, ComplianceLevel newLevel) {}

public record RegressionDetectedEvent(
    UUID caseId, UUID improvementCaseId, double confidence, String category) {}

public record TickEvaluatedEvent(UUID caseId, TickTrace trace) {}
```

- [ ] **Step 2: Create EvolutionEvent view record**

```java
// api/src/main/java/io/casehub/api/view/EvolutionEvent.java
package io.casehub.api.view;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

public record EvolutionEvent(
    UUID caseId, String type, Map<String, String> data, Instant timestamp) {}
```

- [ ] **Step 3: Write EvolutionStreamBroadcaster test**

Follow the pattern from `ExecutionStateBroadcasterTest`. Test that CDI events produce correctly typed `EvolutionEvent` instances on the stream, filtered by caseId.

- [ ] **Step 4: Implement EvolutionStreamBroadcaster**

Follow the spec §1 code. `@ApplicationScoped` with `BroadcastProcessor<EvolutionEvent>`. Four `@ObservesAsync` handlers. Log `BackPressureFailure` instead of silently swallowing.

- [ ] **Step 5: Wire CDI event firing into existing components**

Inject `Event<CircuitBreakerStateChangedEvent>` into `ImprovementCircuitBreaker`. Fire on state transitions in `evaluate()` and `manualReset()`.

Inject `Event<ComplianceLevelChangedEvent>` into `ReadinessValidator`. Fire when computed level differs from cached level.

Inject `Event<RegressionDetectedEvent>` into `RegressionDetector`. Fire in `onMetricsDegraded()`.

`TickEvaluatedEvent` is already fired from `EvolutionTicker` (Task 2).

- [ ] **Step 6: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core,rest`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add engine-common/src/main/java/io/casehub/engine/common/spi/event/CircuitBreakerStateChangedEvent.java \
       engine-common/src/main/java/io/casehub/engine/common/spi/event/ComplianceLevelChangedEvent.java \
       engine-common/src/main/java/io/casehub/engine/common/spi/event/RegressionDetectedEvent.java \
       engine-common/src/main/java/io/casehub/engine/common/spi/event/TickEvaluatedEvent.java \
       api/src/main/java/io/casehub/api/view/EvolutionEvent.java \
       rest/src/main/java/io/casehub/engine/rest/EvolutionStreamBroadcaster.java \
       rest/src/test/java/io/casehub/engine/rest/EvolutionStreamBroadcasterTest.java
git commit -m "feat(#1132): add CDI evolution events and EvolutionStreamBroadcaster"
```

## Batch 2: Control — Manual Overrides and Dynamic Deny List

After this batch: The conductor can pause/unpause categories, reset the circuit breaker, and manage a two-layer deny list (static safety base + dynamic operator additions) — all via direct method calls ready for API wiring. Dynamic deny patterns survive restart via EventLog.

### Task 4: Dynamic deny list + control mutations foundation

**Files:**
- Create: `api/src/main/java/io/casehub/api/view/DenyPatternView.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementBudgetEnforcer.java` — add dynamic deny patterns
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/DenyPatternDynamicTest.java`

**Interfaces:**
- Consumes: `CaseHubEventType.DENY_PATTERN_ADDED/REMOVED` (from Task 1)
- Produces: `ImprovementBudgetEnforcer.addDenyPattern()`, `removeDenyPattern()`, `isDenied()`, `dynamicDenyPatterns()`, `staticDenyPatterns()` (used by API)

- [ ] **Step 1: Write DenyPatternDynamicTest**

Test cases: add dynamic pattern, remove dynamic pattern, static patterns immutable, effective deny set = static ∪ dynamic, `isDenied()` checks both layers, EventLog reconstruction on startup.

- [ ] **Step 2: Create DenyPatternView**

```java
// api/src/main/java/io/casehub/api/view/DenyPatternView.java
package io.casehub.api.view;

import java.time.Instant;
import java.util.List;

public record DenyPatternView(
    List<String> staticPatterns,
    List<DynamicDenyEntry> dynamicPatterns) {

  public record DynamicDenyEntry(
      String pattern, String addedBy, Instant addedAt) {}
}
```

- [ ] **Step 3: Implement dynamic deny list on ImprovementBudgetEnforcer**

Add `ConcurrentHashMap<UUID, Set<String>> dynamicDenyPatterns`. Add methods: `addDenyPattern(caseId, pattern)`, `removeDenyPattern(caseId, pattern)`, `isDenied(caseId, request)` checking both static and dynamic, `dynamicDenyPatterns(caseId)` for query, `staticDenyPatterns()` exposing the static set.

- [ ] **Step 4: Run test, verify passes**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#1132): add dynamic deny patterns to ImprovementBudgetEnforcer"
```

### Task 5: ConductorInboxManager + ConductorInboxEntry

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ConductorInboxEntry.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ConductorDecision.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/GateResolutionPayload.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ConductorInboxManager.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ConductorInboxManagerTest.java`

**Interfaces:**
- Consumes: `ImprovementStage` (from Task 1), `CaseHubEventType.GATE_PENDING/GATE_RESOLVED` (from Task 1), `WatchPattern` (created in Task 6)
- Produces: `ConductorInboxManager.enqueue()`, `pending()`, `pendingCount()`, `resolve()`, `addWatchPattern()`, `removeWatchPattern()`, `activeWatchPatterns()` (used by API, EscalationProvider, EvolutionStateSnapshot)

- [ ] **Step 1: Create ConductorInboxEntry, ConductorDecision, GateResolutionPayload**

Follow spec §4 code exactly. `ConductorInboxEntry` with Status enum (PENDING, APPROVED, REJECTED, REDIRECTED, TIMED_OUT, AUTO_APPROVED). `GateResolutionPayload` as sealed interface with `ScopeModification` and `HypothesisSelection` variants.

- [ ] **Step 2: Write ConductorInboxManagerTest**

Test cases: enqueue and retrieve pending, resolve gate (APPROVED/REJECTED/REDIRECTED), pendingCount, per-case isolation, timeout handling, EventLog reconstruction (pending entries survive restart), watch pattern add/remove/list.

- [ ] **Step 3: Implement ConductorInboxManager**

Follow spec §4 code. `@ApplicationScoped` with ConcurrentHashMap for queues and watch patterns. `@Observes StartupEvent` for EventLog reconstruction. Inject `EventLogRepository` and `Event<GatePendingEvent>`.

- [ ] **Step 4: Run test, verify passes**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(#1132): add ConductorInboxManager with EventLog persistence"
```

## Batch 3: Steer — Escalation, Gates, and Research Checkpoints

After this batch: The conductor has configurable lifecycle gates with 3-layer smart escalation (category rules, watch patterns, confidence scoring). The research pipeline supports non-blocking checkpoints for scope steering and hypothesis approval. Manual coordination via ImprovementCoordinator.

### Task 6: GatePolicy + EscalationPolicy + EscalationProvider

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/GatePolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/EscalationPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/CategoryEscalationRules.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/WatchPattern.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/EscalationResult.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/EscalationContext.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/EscalationTrigger.java`
- Create: `api/src/main/java/io/casehub/api/spi/improvement/EscalationProvider.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultEscalationProvider.java`
- Modify: `api/src/main/java/io/casehub/api/model/stigmergy/ImprovementConfig.java` — add gatePolicy, escalationPolicy fields
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/DefaultEscalationProviderTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/GatePolicyTest.java`

**Interfaces:**
- Consumes: `ImprovementStage` (from Task 1), `WatchPattern` (managed by ConductorInboxManager from Task 5)
- Produces: `EscalationProvider.evaluate()` (used by ResearchPipelineOrchestrator), `GatePolicy.effectiveMode()` (used by ResearchPipelineOrchestrator, API)

- [ ] **Step 1: Create all escalation model types**

GatePolicy (Map-based with `ImprovementStage` keys, compact record constructor validates gate checkpoints), EscalationPolicy, CategoryEscalationRules, WatchPattern, EscalationResult, EscalationContext, EscalationTrigger. Follow spec §4 code exactly.

- [ ] **Step 2: Create EscalationProvider SPI**

```java
// api/src/main/java/io/casehub/api/spi/improvement/EscalationProvider.java
public interface EscalationProvider {
  EscalationResult evaluate(UUID caseId, String tenancyId,
      ImprovementStage stage, EscalationContext context,
      EscalationPolicy policy, List<WatchPattern> activeWatchPatterns);
}
```

- [ ] **Step 3: Write GatePolicyTest**

Test: default mode (AUTO for all except PR_REVIEW which is GATED), custom mode per stage, reject non-checkpoint stages, timeout defaults.

- [ ] **Step 4: Write DefaultEscalationProviderTest**

Test all 3 layers independently and composed: category rule triggers alone, watch pattern triggers alone, confidence below threshold triggers alone, neverEscalate suppresses category layer only (not watch patterns), union of any triggers = escalate, no triggers = no escalation.

- [ ] **Step 5: Implement DefaultEscalationProvider**

Follow spec §4 code. `@DefaultBean @ApplicationScoped`. Three-layer evaluation. Heuristic confidence scoring (1.0 baseline, deductions for novelty, failure rate, scope size).

- [ ] **Step 6: Extend ImprovementConfig**

Add `@Nullable GatePolicy gatePolicy` and `@Nullable EscalationPolicy escalationPolicy` fields. Add `effectiveGatePolicy()` and `effectiveEscalationPolicy()` methods. Update the backwards-compatible constructor.

- [ ] **Step 7: Add YAML codegen entries for GatePolicy, EscalationPolicy, CategoryEscalationRules**

Add to `schema/src/main/resources/schema/yaml-record-mappings.yaml`.

- [ ] **Step 8: Run tests, verify pass**

- [ ] **Step 9: Commit**

```bash
git commit -m "feat(#1132): add GatePolicy, EscalationProvider SPI, and DefaultEscalationProvider"
```

### Task 7: Research pipeline checkpoint/resume + ImprovementCoordinator

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/GateCheckpoint.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ResearchPipelineResult.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ResearchPipelineOrchestrator.java` — add gate checkpoints, resume(), evaluateGate()
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/ImprovementCoordinator.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ResearchPipelineCheckpointTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/ImprovementCoordinatorTest.java`

**Interfaces:**
- Consumes: `GatePolicy` (from Task 6), `EscalationProvider` (from Task 6), `ConductorInboxManager` (from Task 5)
- Produces: `ResearchPipelineResult` (Completed or AwaitingGate), `ImprovementCoordinator.block()/unblock()/isBlocked()` (used by API, EvolutionTicker)

- [ ] **Step 1: Create GateCheckpoint and ResearchPipelineResult**

Sealed interfaces per spec §4. `GateCheckpoint.ScopeCheckpoint(ResearchScope)` and `GateCheckpoint.HypothesisCheckpoint(List<ImprovementHypothesis>)`. `ResearchPipelineResult.Completed(hypotheses)` and `ResearchPipelineResult.AwaitingGate(stage, entryId, checkpoint)`.

- [ ] **Step 2: Write ResearchPipelineCheckpointTest**

Test: pipeline returns AwaitingGate at scope checkpoint when GATED, pipeline returns Completed when AUTO and no escalation, resume() with ScopeModification applies changes, resume() with HypothesisSelection filters hypotheses, NOTIFY mode proceeds + creates inbox entry.

- [ ] **Step 3: Modify ResearchPipelineOrchestrator**

Add `evaluateGate()` private method per spec. Change `execute()` to return `ResearchPipelineResult`. Add `resume()` method. Extract `executeFromScope()` for resumability. Add `GatePolicy`, `EscalationPolicy`, `ConductorInboxManager`, `EscalationProvider` as constructor dependencies.

- [ ] **Step 4: Write ImprovementCoordinatorTest**

Test: block/unblock, isBlocked, blockedBy, per-case isolation, EventLog reconstruction.

- [ ] **Step 5: Implement ImprovementCoordinator**

Follow spec §4 code. `@ApplicationScoped` with ConcurrentHashMap. EventLog persistence for blocks (`IMPROVEMENT_BLOCKED`/`IMPROVEMENT_UNBLOCKED`). Startup reconstruction.

- [ ] **Step 6: Run tests, verify pass**

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#1132): add research pipeline checkpoints and ImprovementCoordinator"
```

## Batch 4: Summarize, Review, and API Surface

After this batch: Full command centre API operational. `DefaultEngineEvolutionApi` exposes all 5 layers through `@McpDomain("engine/evolution")`. Layered summaries via `SummarizationProvider`. Artifact trail per improvement stream. `EvolutionStateSnapshot` composed from all sources.

### Task 8: SummarizationProvider + ArtifactManifest

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/SummaryScope.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ArtifactManifest.java`
- Create: `api/src/main/java/io/casehub/api/model/stigmergy/ArtifactEntry.java`
- Create: `api/src/main/java/io/casehub/api/view/EvolutionSummary.java` (includes CategorySummary, ResearchDirectionSummary, NotableEvent)
- Create: `api/src/main/java/io/casehub/api/spi/improvement/SummarizationProvider.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/improvement/DefaultSummarizationProvider.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/improvement/DefaultSummarizationProviderTest.java`

**Interfaces:**
- Consumes: `EventLogRepository` (existing), `ImprovementStage` (from Task 1), `CaseHubEventType` improvement events (existing)
- Produces: `SummarizationProvider.summarize()` (used by API), `ArtifactManifest` (used by API)

- [ ] **Step 1: Create all summary and artifact model types**

SummaryScope, EvolutionSummary (with CategorySummary, ResearchDirectionSummary, NotableEvent), ArtifactManifest, ArtifactEntry (with ArtifactType enum). Follow spec §2 and §5 code exactly.

- [ ] **Step 2: Create SummarizationProvider SPI**

- [ ] **Step 3: Write DefaultSummarizationProviderTest**

Test: project-wide summary from EventLog events, area-scoped summary, category-scoped summary, time-window filtering, empty EventLog returns zero counts.

- [ ] **Step 4: Implement DefaultSummarizationProvider**

`@DefaultBean @ApplicationScoped`. Query EventLog for improvement lifecycle events. Filter by scope parameters. Compute counts, ratios, trends.

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#1132): add SummarizationProvider SPI, DefaultSummarizationProvider, and ArtifactManifest"
```

### Task 9: EvolutionStateSnapshot + DefaultEngineEvolutionApi

**Files:**
- Create: `api/src/main/java/io/casehub/api/view/EvolutionStateSnapshot.java` (includes CategoryStateView, ImprovementStreamView, StageProgress)
- Create: `rest/src/main/java/io/casehub/engine/rest/service/DefaultEngineEvolutionApi.java`
- Test: `rest/src/test/java/io/casehub/engine/rest/service/EvolutionApiTest.java`

**Interfaces:**
- Consumes: All beans from Tasks 1-8 (HealthScoreTracker, ImprovementCircuitBreaker, ImprovementCategoryTracker, TickTraceBuffer, ImprovementBudgetEnforcer, ReadinessValidator, ConductorInboxManager, SummarizationProvider, ImprovementCoordinator, EvolutionStreamBroadcaster, ResearchCorpus)
- Produces: The full API surface — all queries and mutations per spec §6

- [ ] **Step 1: Create EvolutionStateSnapshot and supporting view records**

EvolutionStateSnapshot, CategoryStateView, ImprovementStreamView, StageProgress (with StageStatus enum). Follow spec §1 code exactly.

- [ ] **Step 2: Write EvolutionApiTest**

Test key endpoints: getEvolutionState returns composed snapshot, pauseCategory delegates to tracker, resetCircuitBreaker delegates, addDenyPattern/removeDenyPattern, getInbox returns pending entries, resolveGate updates entry status, getDenyPatterns shows both layers.

- [ ] **Step 3: Implement DefaultEngineEvolutionApi**

`@ApplicationScoped @McpDomain("engine/evolution")`. Inject all required beans. Implement each query and mutation per spec §6. Snapshot composition reads from in-memory caches — no expensive queries. `resolveGate()` deserializes `GateResolutionPayload` based on entry stage. `triggerReadinessValidation()` calls `ReadinessValidator.validate()`.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Run full project build**

Run: `/opt/homebrew/bin/mvn install -pl api,schema,codegen,common-core,engine-support-core,runtime-core,rest -am -Dcheckstyle.skip=true -Dspotless.check.skip=true -DskipTests`
Expected: BUILD SUCCESS

- [ ] **Step 6: Run all tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn clean test -pl runtime-core,rest`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#1132): add EvolutionStateSnapshot and DefaultEngineEvolutionApi Closes #1132"
```

## References

- [2026-09-21-command-centre-conductor-design.md] — design spec this plan implements
- [EvolutionTicker.java:46] — current void tick() to be changed
- [ImprovementCircuitBreaker.java:75] — manualReset()
- [ImprovementCategoryTracker.java:93] — pauseCategory()
- [ImprovementBudgetEnforcer.java] — STRUCTURAL_DENIED_PATTERNS
- [HealthScoreTracker.java:88] — latestSnapshot()
- [CaseStreamBroadcaster.java] — BroadcastProcessor pattern
- [ExecutionStateBroadcaster.java] — dedicated broadcaster precedent
- [DefaultEngineCaseControlApi.java] — @McpDomain API pattern
- [CaseHubEventType.java] — existing event types
- [ResearchPipelineOrchestrator.java] — research pipeline to modify
- [PlanItemStateChangedEvent.java] — CDI event precedent
- [GitHub #1132] — focal issue
- [GitHub #1131] — prerequisite issue
- Decisions D10–D27 in decisions.md
