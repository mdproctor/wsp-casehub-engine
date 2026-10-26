# Evolution Loop Gap Closure — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1115 — Continuous evolution loop — self-directed growth with quality gates
**Issue group:** #1115

**Goal:** Wire the evolution loop closed in production — connect the existing,
tested-in-isolation components so the trigger→proposal→execution→outcome→feedback
cycle actually runs end-to-end with all safety gates active.

**Architecture:** The components exist and pass unit tests. The work is CDI wiring
(RuntimeBeans.java producers), adding the timer backstop (runtime module), connecting
the outcome capture (CaseOutcomeObserver SPI), and filling remaining gaps (rollback
spawning, restart recovery, circuit breaker hysteresis, YAML codegen, DevTown workers).

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (ArC), Maven multi-module

## Global Constraints

- **Module boundary:** `runtime-core` is framework-agnostic (no CDI annotations).
  All `@ApplicationScoped`, `@Observes`, `@Scheduled` live in `runtime` module.
  runtime-core classes use constructor injection; `RuntimeBeans.java` provides producers.
- **Event pattern:** Improvement subsystem uses `Consumer<Event>` callbacks, not CDI
  `@ObservesAsync`. New components follow this pattern.
- **Test naming:** `*Test.java` only. Never `*IT.java`.
- **TESTCONTAINERS_RYUK_DISABLED=true** on all test runs.
- **`mvn install -DskipTests -q`** before module-specific tests.
- All commits reference `Refs #1115`.

---

## Batch 1: Close the Trigger Loop

After this batch: EvolutionTicker is called from both the convergence handler and a
periodic timer. Per-case locking prevents concurrent ticks. Safety gates (circuit
breaker, health check, regression monitoring) are active in the event-driven path.

### Task 1: Wire EvolutionTicker into convergence handler

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/EvolutionTicker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/EvolutionTickerTest.java`

**Interfaces:**
- Consumes: `EvolutionTicker.tick(UUID, String, ImprovementConfig, TickTrigger)` (exists)
- Produces: Per-case tick locking via `ConcurrentHashMap<UUID, ReentrantLock>`

- [ ] **Step 1: Write failing test for per-case tick serialization**

In `EvolutionTickerTest`, add test that verifies concurrent ticks on the same case
are serialized (second tick returns a skipped trace):

```java
@Test
void concurrentTickOnSameCaseIsSkipped() throws Exception {
    var latch = new CountDownLatch(1);
    var slowGoalFormation = mock(ImprovementGoalFormationStrategy.class);
    when(slowGoalFormation.proposeImprovements(any(), any(), any()))
        .thenAnswer(inv -> { latch.await(); return emptyProposal(); });

    var ticker = new EvolutionTicker(slowGoalFormation, circuitBreaker,
        healthTracker, regressionDetector, goalFormationService,
        traceBuffer, event -> {});

    var config = enabledConfig();
    var future = CompletableFuture.supplyAsync(() ->
        ticker.tick(CASE_ID, TENANCY_ID, config, TickTrigger.TIMER));

    Thread.sleep(50); // Let first tick enter doTick
    var secondTrace = ticker.tick(CASE_ID, TENANCY_ID, config, TickTrigger.EVENT_DRIVEN);

    assertThat(secondTrace.outcome()).isInstanceOf(TickTrace.TickOutcome.Skipped.class);
    latch.countDown();
    var firstTrace = future.get();
    assertThat(firstTrace.outcome()).isNotInstanceOf(TickTrace.TickOutcome.Skipped.class);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=EvolutionTickerTest#concurrentTickOnSameCaseIsSkipped -q`
Expected: FAIL — `Skipped` variant doesn't exist yet, or `tryLock` not implemented.

- [ ] **Step 3: Add Skipped outcome variant to TickTrace**

In `api/.../model/stigmergy/TickTrace.java`, add `Skipped` to the sealed `TickOutcome`:

```java
record Skipped(String reason) implements TickOutcome {}
```

- [ ] **Step 4: Add per-case lock to EvolutionTicker**

Use `ide_edit_member` on `EvolutionTicker`:
- Add field: `private final ConcurrentHashMap<UUID, ReentrantLock> tickLocks = new ConcurrentHashMap<>();`
- Modify `tick(UUID, String, ImprovementConfig, TickTrigger)` to acquire per-case lock:

```java
public TickTrace tick(UUID caseId, String tenancyId,
    ImprovementConfig config, TickTrigger trigger) {
  var lock = tickLocks.computeIfAbsent(caseId, k -> new ReentrantLock());
  if (!lock.tryLock()) {
    var trace = new TickTrace(caseId, Instant.now(), trigger,
        List.of(), new TickTrace.TickOutcome.Skipped("concurrent tick in progress"));
    traceBuffer.record(trace);
    return trace;
  }
  try {
    return doTick(caseId, tenancyId, config, trigger);
  } finally {
    lock.unlock();
  }
}
```

Rename the existing `tick` body to `private TickTrace doTick(...)`.

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=EvolutionTickerTest -q`
Expected: All tests PASS.

- [ ] **Step 6: Add EvolutionTicker to CaseContextChangedEventHandler**

Add `Instance<EvolutionTicker>` as a constructor parameter to `CaseContextChangedEventHandler`.
In `convergenceDetection()`, replace the direct `proposeImprovements()` call with:

```java
if (evolutionTickerInstance.isResolvable()) {
    evolutionTickerInstance.get().tick(
        caseInstance.getUuid(), caseInstance.tenancyId(), improvementConfig,
        TickTrace.TickTrigger.EVENT_DRIVEN);
} else {
    // Fallback: existing direct path for non-evolution cases
    var proposal = improvementStrategy.proposeImprovements(
        caseInstance.getUuid(), caseInstance.tenancyId(), improvementConfig);
    if (proposal != null && !proposal.goals().isEmpty()
        && goalFormationServiceInstance.isResolvable()) {
        goalFormationServiceInstance.get().propose(
            agentId, caseInstance.tenancyId(), proposal);
    }
}
```

- [ ] **Step 7: Wire EvolutionTicker into handler producer in RuntimeBeans.java**

Find the `CaseContextChangedEventHandler` producer method in `RuntimeBeans.java`.
Add `Instance<EvolutionTicker> evolutionTickerInstance` as a parameter and pass it
to the handler constructor.

- [ ] **Step 8: Run full improvement test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest="*Improvement*,*Evolution*,*Ticker*" -q`
Expected: All tests PASS.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#1115): wire EvolutionTicker into convergence handler with per-case lock

Route event-driven improvement proposals through the complete gate
pipeline (evolution-enabled, health refresh, regression monitoring,
circuit breaker). Add per-case ReentrantLock to prevent concurrent
ticks on the same case.

Refs #1115"
```

---

### Task 2: Timer backstop

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/quarkus/improvement/EvolutionTimerBackstop.java`
- Test: `runtime/src/test/java/io/casehub/engine/quarkus/improvement/EvolutionTimerBackstopTest.java`

**Interfaces:**
- Consumes: `EvolutionTicker.tick(UUID, String, ImprovementConfig, TickTrigger.TIMER)`
- Consumes: `CaseInstanceRepository.findActive()`, `CaseDefinitionRegistry.get(String)`
- Produces: Periodic tick invocations for evolution-enabled cases

- [ ] **Step 1: Write failing test**

```java
@Test
void ticksOnlyEvolutionEnabledCases() {
    var enabledCase = mockCaseInstance("case-1", "tenant-1", "def-enabled");
    var disabledCase = mockCaseInstance("case-2", "tenant-2", "def-disabled");
    when(caseRepository.findActive()).thenReturn(List.of(enabledCase, disabledCase));
    when(definitionRegistry.get("def-enabled")).thenReturn(definitionWithEvolution(true));
    when(definitionRegistry.get("def-disabled")).thenReturn(definitionWithEvolution(false));

    backstop.tick();

    verify(ticker).tick(eq(enabledCase.getUuid()), eq("tenant-1"), any(), eq(TickTrigger.TIMER));
    verify(ticker, never()).tick(eq(disabledCase.getUuid()), any(), any(), any());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `EvolutionTimerBackstop` doesn't exist.

- [ ] **Step 3: Create EvolutionTimerBackstop**

```java
package io.casehub.engine.quarkus.improvement;

@ApplicationScoped
public class EvolutionTimerBackstop {

    private final EvolutionTicker ticker;
    private final CaseInstanceRepository caseRepository;
    private final CaseDefinitionRegistry definitionRegistry;

    @Inject
    public EvolutionTimerBackstop(EvolutionTicker ticker,
            CaseInstanceRepository caseRepository,
            CaseDefinitionRegistry definitionRegistry) {
        this.ticker = ticker;
        this.caseRepository = caseRepository;
        this.definitionRegistry = definitionRegistry;
    }

    @Scheduled(every = "${casehub.evolution.timer-interval:60m}")
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

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=EvolutionTimerBackstopTest -q`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1115): add evolution timer backstop

Periodic @Scheduled trigger (default 60m) ticks all evolution-enabled
cases through EvolutionTicker. Complements the event-driven path.
Lives in runtime module (Quarkus-specific).

Refs #1115"
```

---

## Batch 2: Close the Feedback Loop

After this batch: Improvement outcomes are captured when improvement cases complete,
feeding back into category tracking, regression detection, and CBR.

### Task 3: Wire ImprovementOutcomeEventCapture as CaseOutcomeObserver

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementOutcomeEventCapture.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ImprovementOutcomeEventCaptureTest.java`

**Interfaces:**
- Consumes: `CaseOutcomeObserver.onOutcome(CaseOutcomeEvent)` SPI
- Produces: Calls to all 6 layers (outcomeRecorder, signalProjector, cbrProjector, budgetEnforcer, categoryTracker, regressionDetector)

- [ ] **Step 1: Write failing test**

```java
@Test
void capturesOutcomeForImprovementCase() {
    var event = new CaseOutcomeEvent(CASE_ID, TENANCY_ID,
        CaseOutcome.COMPLETED, improvementCaseContext());

    capture.onOutcome(event);

    verify(outcomeRecorder).record(eq(CASE_ID), eq(TENANCY_ID), any());
    verify(signalProjector).project(eq(CASE_ID), any());
    verify(budgetEnforcer).recordCompletion(any());
    verify(categoryTracker).recordOutcome(eq(CASE_ID), eq("dependency-update"), any());
    verify(regressionDetector).onOutcome(eq(CASE_ID), any());
}

@Test
void ignoresNonImprovementCase() {
    var event = new CaseOutcomeEvent(CASE_ID, TENANCY_ID,
        CaseOutcome.COMPLETED, regularCaseContext());

    capture.onOutcome(event);

    verifyNoInteractions(outcomeRecorder);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `ImprovementOutcomeEventCapture` doesn't implement `CaseOutcomeObserver`.

- [ ] **Step 3: Update ImprovementOutcomeEventCapture to implement CaseOutcomeObserver**

Use `ide_replace_member` to update the class declaration and `onImprovementComplete`:

```java
public class ImprovementOutcomeEventCapture implements CaseOutcomeObserver {

    // ... existing fields ...

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
        regressionDetector.onOutcome(event.caseId(), outcome);
    }

    private boolean isImprovementCase(CaseOutcomeEvent event) {
        var templateId = event.context().get("caseTemplateId");
        return "self-improvement".equals(templateId)
            || "self-improvement-rollback".equals(templateId);
    }

    private ImprovementOutcome extractOutcome(CaseOutcomeEvent event) {
        // Extract from case context working layer
        var ctx = event.context();
        // ... build ImprovementOutcome from context fields ...
    }
}
```

Keep the existing `onImprovementComplete(ImprovementCaseCompleted)` method for
backwards compatibility with tests, delegating to the same internal logic.

- [ ] **Step 4: Add producer in RuntimeBeans.java**

Add `ImprovementOutcomeEventCapture` as a `@Produces @ApplicationScoped` bean
and register it as a `CaseOutcomeObserver`:

```java
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

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ImprovementOutcomeEventCaptureTest -q`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#1115): wire ImprovementOutcomeEventCapture as CaseOutcomeObserver

Close the feedback loop: improvement case completions now feed through
all 6 outcome capture layers (EventLog, signals, CBR, budget, category
tracker, regression detector). Uses existing CaseOutcomeObserver SPI.

Refs #1115"
```

---

### Task 4: CBR projector and goal formation CBR read

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementCbrProjector.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementGoalFormationStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ImprovementCbrProjectorTest.java`

**Interfaces:**
- Consumes: `Instance<CbrRetrieverService>` (optional, from neocortex)
- Produces: CBR write on outcome, CBR read on proposal

- [ ] **Step 1: Write failing test for CBR projector**

```java
@Test
void projectsToServiceWhenAvailable() {
    var cbrService = mock(CbrRetrieverService.class);
    var projector = new ImprovementCbrProjector(resolvableInstance(cbrService));
    var outcome = testOutcome("dependency-update", "pom.xml");

    projector.project("tenant-1", outcome);

    verify(cbrService).store(eq("tenant-1"), eq("improvement-outcomes"), any());
}

@Test
void noOpWhenServiceUnavailable() {
    var projector = new ImprovementCbrProjector(unresolvableInstance());
    var outcome = testOutcome("dependency-update", "pom.xml");

    projector.project("tenant-1", outcome);
    // No exception, no interaction
}
```

- [ ] **Step 2: Update ImprovementCbrProjector**

Replace the logging stub with optional CBR service injection:

```java
public class ImprovementCbrProjector {

    private final Instance<CbrRetrieverService> cbrService;

    public ImprovementCbrProjector(Instance<CbrRetrieverService> cbrService) {
        this.cbrService = cbrService;
    }

    public void project(String tenancyId, ImprovementOutcome outcome) {
        if (!cbrService.isResolvable()) return;
        cbrService.get().store(tenancyId, "improvement-outcomes",
            buildCbrCase(outcome));
    }

    private Map<String, Object> buildCbrCase(ImprovementOutcome outcome) {
        return Map.of(
            "category", outcome.category(),
            "target", outcome.target(),
            "status", outcome.status().name(),
            "timestamp", Instant.now().toString());
    }
}
```

- [ ] **Step 3: Add optional CBR read to ImprovementGoalFormationStrategy**

Add `Instance<CbrRetrieverService> cbrService` constructor parameter. In
`proposeImprovements()`, after anti-oscillation filter and before deny filter:

```java
if (cbrService.isResolvable()) {
    var similar = cbrService.get().findSimilar(tenancyId,
        "improvement-outcomes", request.category(), request.target(), 5);
    if (computeFailureRate(similar) > 0.6) {
        continue;
    }
}
```

- [ ] **Step 4: Update RuntimeBeans producers**

Add `Instance<CbrRetrieverService>` to both the `ImprovementCbrProjector` and
`ImprovementGoalFormationStrategy` producers.

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest="*CbrProjector*,*GoalFormation*" -q`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#1115): wire CBR projector and goal formation CBR read

CBR projector writes improvement outcomes when CbrRetrieverService is
available (neocortex). Goal formation optionally reads CBR for similar
past failures and skips proposals with >60% failure rate. Graceful
no-op when CBR unavailable.

Refs #1115"
```

---

## Batch 3: Rollback Wiring

After this batch: When regression is detected with high confidence, a rollback case
is automatically spawned from the self-improvement-rollback template. Anti-cascade
check prevents rollback when dependent improvements are active.

### Task 5: RollbackRequested event and MonitoredImprovement targetPaths

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/spi/event/RollbackRequested.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/RegressionDetector.java` (MonitoredImprovement record)
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/RegressionDetectorTest.java`

**Interfaces:**
- Produces: `RollbackRequested` event record, `MonitoredImprovement` with targetPaths

- [ ] **Step 1: Create RollbackRequested event**

```java
package io.casehub.engine.common.spi.event;

public record RollbackRequested(
    UUID caseId,
    String tenancyId,
    UUID improvementCaseId,
    String category,
    String target,
    List<String> targetPaths,
    double confidence) {}
```

- [ ] **Step 2: Add targetPaths to MonitoredImprovement**

Use `ide_edit_member` on `RegressionDetector.MonitoredImprovement` to add
`List<String> targetPaths` field. Update `onOutcome()` to extract target paths
from the improvement outcome and pass them when creating the monitor.

- [ ] **Step 3: Write test verifying targetPaths stored in monitor**

```java
@Test
void onOutcomeCapturesTargetPaths() {
    var outcome = mergedOutcome(List.of("src/main/java/Foo.java", "pom.xml"));
    detector.onOutcome(CASE_ID, outcome);
    assertThat(detector.activeMonitorCount(CASE_ID)).isEqualTo(1);
}
```

- [ ] **Step 4: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=RegressionDetectorTest -q`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1115): add RollbackRequested event and MonitoredImprovement targetPaths

RollbackRequested in common-core spi.event for rollback case spawning.
MonitoredImprovement now stores targetPaths for anti-cascade checks.

Refs #1115"
```

---

### Task 6: RegressionDetector rollback callback and RollbackCaseSpawner

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/RegressionDetector.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/RollbackCaseSpawner.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/RollbackCaseSpawnerTest.java`

**Interfaces:**
- Consumes: `Consumer<RollbackRequested>` callback on RegressionDetector
- Consumes: `GoalFormationService.propose()`, `ImprovementBudgetEnforcer.activeImprovementRequests()`
- Produces: Rollback case creation via GoalFormationService

- [ ] **Step 1: Write failing test for RollbackCaseSpawner**

```java
@Test
void spawnsRollbackCaseWhenNoConflicts() {
    when(budgetEnforcer.activeImprovementRequests()).thenReturn(Map.of());

    spawner.onRollbackRequested(new RollbackRequested(
        CASE_ID, TENANCY_ID, IMPROVEMENT_ID,
        "dependency-update", "pom.xml",
        List.of("pom.xml"), 0.95));

    verify(goalFormationService).propose(eq("regression-detector"), eq(TENANCY_ID), any());
}

@Test
void skipsRollbackWhenActiveImprovementOverlaps() {
    var activeRequest = new ImprovementRequest("lint-fix", "pom.xml", 5,
        Map.of(), List.of("pom.xml"), "code-evolution");
    when(budgetEnforcer.activeImprovementRequests())
        .thenReturn(Map.of(UUID.randomUUID(), activeRequest));

    spawner.onRollbackRequested(new RollbackRequested(
        CASE_ID, TENANCY_ID, IMPROVEMENT_ID,
        "dependency-update", "pom.xml",
        List.of("pom.xml"), 0.95));

    verify(goalFormationService, never()).propose(any(), any(), any());
}
```

- [ ] **Step 2: Create RollbackCaseSpawner**

```java
public class RollbackCaseSpawner {

    private final GoalFormationService goalFormationService;
    private final ImprovementBudgetEnforcer budgetEnforcer;

    public RollbackCaseSpawner(GoalFormationService goalFormationService,
            ImprovementBudgetEnforcer budgetEnforcer) {
        this.goalFormationService = goalFormationService;
        this.budgetEnforcer = budgetEnforcer;
    }

    public void onRollbackRequested(RollbackRequested event) {
        if (hasOverlappingPaths(event.targetPaths(),
                budgetEnforcer.activeImprovementRequests())) {
            return;
        }
        var proposal = GoalFormationProposal.of(List.of(ProposedGoal.builder()
            .name("rollback-" + event.improvementCaseId())
            .description("Revert regressed improvement")
            .caseTemplateId("self-improvement-rollback")
            .context(Map.of(
                "originalImprovementCaseId", event.improvementCaseId().toString(),
                "category", event.category(),
                "target", event.target(),
                "confidence", String.valueOf(event.confidence())))
            .build()));
        goalFormationService.propose("regression-detector", event.tenancyId(), proposal);
    }

    private boolean hasOverlappingPaths(List<String> targetPaths,
            Map<UUID, ImprovementRequest> active) {
        for (var req : active.values()) {
            for (String tp : targetPaths) {
                for (String ap : req.targetPaths()) {
                    if (tp.equals(ap) || sameDirectory(tp, ap)) return true;
                }
            }
        }
        return false;
    }

    private boolean sameDirectory(String a, String b) {
        int lastA = a.lastIndexOf('/');
        int lastB = b.lastIndexOf('/');
        return lastA > 0 && lastB > 0
            && a.substring(0, lastA).equals(b.substring(0, lastB));
    }
}
```

- [ ] **Step 3: Add rollback callback to RegressionDetector**

Add `Consumer<RollbackRequested> rollbackRequestedCallback` constructor parameter.
In `onMetricsDegraded()`, fire callback when confidence >= autoRevertThreshold
(alongside existing `regressionDetectedEvent` and rollbackHistory/categoryTracker calls).

- [ ] **Step 4: Wire in RuntimeBeans.java**

Create `RollbackCaseSpawner` producer. Pass `spawner::onRollbackRequested` as the
`Consumer<RollbackRequested>` to the `RegressionDetector` producer.

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest="*RollbackCaseSpawner*,*RegressionDetector*" -q`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#1115): add RollbackCaseSpawner with anti-cascade guard

On high-confidence regression, spawn rollback case from
self-improvement-rollback template via GoalFormationService. Anti-cascade
check prevents rollback when active improvements have overlapping paths.

Refs #1115"
```

---

## Batch 4: Robustness and Restart Recovery

After this batch: Circuit breaker has hysteresis. All safety-critical in-memory state
(circuit breaker, category tracker, regression monitors) is recovered from EventLog
on restart.

### Task 7: Circuit breaker hysteresis

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/improvement/HealthPolicy.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementCircuitBreaker.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ImprovementCircuitBreakerTest.java`

**Interfaces:**
- Consumes: `HealthPolicy.effectiveHysteresisMargin()` (new field)
- Produces: Circuit breaker with hysteresis gap between trip and recovery thresholds

- [ ] **Step 1: Write failing test**

```java
@Test
void doesNotRecoverWithinHysteresisMargin() {
    // Trip the breaker
    circuitBreaker.evaluate(CASE_ID, TENANCY_ID, trackerAt(0.5), healthPolicy());
    assertThat(circuitBreaker.state(CASE_ID)).isEqualTo(CircuitBreakerState.OPEN);

    // Health recovers to threshold but not past hysteresis margin
    circuitBreaker.evaluate(CASE_ID, TENANCY_ID, trackerAt(0.62), healthPolicy());
    assertThat(circuitBreaker.state(CASE_ID)).isEqualTo(CircuitBreakerState.OPEN);

    // Health recovers past threshold + margin (0.6 + 0.05 = 0.65)
    circuitBreaker.evaluate(CASE_ID, TENANCY_ID, trackerAt(0.66), healthPolicy());
    assertThat(circuitBreaker.state(CASE_ID)).isEqualTo(CircuitBreakerState.HALF_OPEN);
}
```

- [ ] **Step 2: Add hysteresisMargin to HealthPolicy**

```java
@Nullable Double hysteresisMargin
// ...
public double effectiveHysteresisMargin() {
    return hysteresisMargin != null ? hysteresisMargin : 0.05;
}
```

- [ ] **Step 3: Update circuit breaker OPEN → HALF_OPEN transition**

In `evaluate()`, change the recovery check to require `score >= threshold + hysteresisMargin`.

- [ ] **Step 4: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ImprovementCircuitBreakerTest -q`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1115): add circuit breaker hysteresis

Recovery from OPEN requires health to exceed threshold + hysteresis
margin (default 0.05). Prevents state thrashing when health oscillates
near the trip threshold.

Refs #1115"
```

---

### Task 8: Startup recovery

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/quarkus/improvement/EvolutionStartupRecovery.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementCategoryTracker.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/RegressionDetector.java`
- Test: `runtime/src/test/java/io/casehub/engine/quarkus/improvement/EvolutionStartupRecoveryTest.java`

**Interfaces:**
- Consumes: `ImprovementCircuitBreaker.restoreFromEventLog()` (exists)
- Produces: `ImprovementCategoryTracker.restoreFromEventLog()` (new), `RegressionDetector.restoreFromEventLog()` (new)

- [ ] **Step 1: Add restoreFromEventLog to ImprovementCategoryTracker**

```java
public void restoreFromEventLog(UUID caseId, String tenancyId,
        EventLogRepository eventLogRepository) {
    // Query REGRESSION_DETECTED events with active pause durations
    // Reconstruct paused categories with remaining durations
}
```

- [ ] **Step 2: Add restoreFromEventLog to RegressionDetector**

```java
public void restoreFromEventLog(UUID caseId, String tenancyId,
        EventLogRepository eventLogRepository) {
    // Query recent IMPROVEMENT_OUTCOME events with MERGED status
    // within regression window. Reconstruct active monitors.
}
```

- [ ] **Step 3: Create EvolutionStartupRecovery**

```java
@ApplicationScoped
public class EvolutionStartupRecovery {

    @Inject ImprovementCircuitBreaker circuitBreaker;
    @Inject RegressionDetector regressionDetector;
    @Inject ImprovementCategoryTracker categoryTracker;
    @Inject CaseInstanceRepository caseRepository;
    @Inject CaseDefinitionRegistry definitionRegistry;
    @Inject EventLogRepository eventLogRepository;

    void onStartup(@Observes @Priority(200) StartupEvent event) {
        for (var ci : caseRepository.findActive()) {
            var def = definitionRegistry.get(ci.caseDefinitionId());
            if (def == null) continue;
            var stig = def.stigmergyConfig();
            if (stig == null || stig.improvement() == null) continue;
            if (!stig.improvement().effectiveEvolutionEnabled()) continue;

            circuitBreaker.restoreFromEventLog(ci.getUuid(), ci.tenancyId());
            categoryTracker.restoreFromEventLog(ci.getUuid(), ci.tenancyId(),
                eventLogRepository);
            regressionDetector.restoreFromEventLog(ci.getUuid(), ci.tenancyId(),
                eventLogRepository);
        }
    }
}
```

- [ ] **Step 4: Write test**

Test that recovery calls all three restore methods for evolution-enabled cases.

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=EvolutionStartupRecoveryTest -q`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat(#1115): add EvolutionStartupRecovery

Recover safety-critical in-memory state from EventLog on startup:
circuit breaker state, category pause state, and active regression
monitors. Runs at StartupEvent priority 200.

Refs #1115"
```

---

## Batch 5: Observability and Configuration

After this batch: Tick traces include full filtering statistics. YAML case definitions
can deserialize RollbackPolicy, HealthPolicy, and ResearchMethodology.

### Task 9: ProposalFilteringSummary and YAML codegen

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementGoalFormationStrategy.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/EvolutionTicker.java`
- Modify: `schema/src/main/resources/schema/yaml-record-mappings.yaml`
- Test: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ImprovementGoalFormationStrategyTest.java`

**Interfaces:**
- Produces: `ProposalResult` record wrapping `GoalFormationProposal` + `ProposalFilteringSummary`

- [ ] **Step 1: Write failing test**

```java
@Test
void proposeImprovementsReturnsFilteringSummary() {
    // Set up consensus with 3 proposals, 1 suppressed, 1 budget-denied
    var result = strategy.proposeImprovements(CASE_ID, TENANCY_ID, config);
    assertThat(result.filtering()).isNotNull();
    assertThat(result.filtering().proposalsBySource()).isEqualTo(3);
    assertThat(result.filtering().proposed()).isEqualTo(1);
}
```

- [ ] **Step 2: Change proposeImprovements return type**

Create `ProposalResult` record. Update `proposeImprovements()` to track counts at
each filter stage and return `ProposalResult`. Update `EvolutionTicker.doTick()` to
use the summary when constructing `TickTrace.TickOutcome.ProposalGenerated`.

- [ ] **Step 3: Add YAML codegen entries**

In `yaml-record-mappings.yaml`, add:

```yaml
io.casehub.api.model.improvement.RollbackPolicy:
  yamlName: YamlRollbackPolicy

io.casehub.api.model.improvement.HealthPolicy:
  yamlName: YamlHealthPolicy

io.casehub.api.model.stigmergy.ResearchMethodology:
  yamlName: YamlResearchMethodology
```

- [ ] **Step 4: Rebuild codegen and test**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest="*GoalFormation*,*Ticker*" -q`

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1115): add ProposalFilteringSummary and YAML codegen entries

proposeImprovements() now returns ProposalResult with filtering stats.
EvolutionTicker uses summary in TickTrace. Added YAML codegen for
RollbackPolicy, HealthPolicy, ResearchMethodology.

Refs #1115"
```

---

## Batch 6: Integration Tests

After this batch: End-to-end scenarios verified — regression→rollback workflow,
circuit breaker lifecycle with restart recovery, conflict serialization.

### Task 10: Integration tests and research pipeline test enhancement

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/runtime/improvement/RegressionRollbackIntegrationTest.java`
- Create: `runtime/src/test/java/io/casehub/engine/runtime/improvement/CircuitBreakerIntegrationTest.java`
- Create: `runtime/src/test/java/io/casehub/engine/runtime/improvement/ConflictSerializationIntegrationTest.java`
- Modify: `runtime-core/src/test/java/io/casehub/engine/runtime/improvement/ResearchPipelineOrchestratorTest.java`

**Interfaces:**
- Consumes: All components wired in previous batches

- [ ] **Step 1: Write RegressionRollbackIntegrationTest**

Full workflow: signal consensus → improvement case → outcome MERGED → health drops →
RegressionDetector fires → RollbackCaseSpawner creates rollback case.

- [ ] **Step 2: Write CircuitBreakerIntegrationTest**

Full lifecycle including restart recovery: trip OPEN → verify blocked → recover
past hysteresis → HALF_OPEN → CLOSED. Restart scenario with `restoreFromEventLog`.

- [ ] **Step 3: Write ConflictSerializationIntegrationTest**

Two overlapping improvements → first proceeds, second blocked → first completes →
next tick → second proceeds.

- [ ] **Step 4: Enhance ResearchPipelineOrchestratorTest**

Expand to 5+ tests covering each depth tier, corpus storage, HIL queue, empty
results, and drive context propagation.

- [ ] **Step 5: Run all tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="*Integration*" -q`
Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=ResearchPipelineOrchestratorTest -q`

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "test(#1115): add integration tests for evolution loop

RegressionRollbackIntegrationTest: full rollback workflow.
CircuitBreakerIntegrationTest: lifecycle with restart recovery.
ConflictSerializationIntegrationTest: overlapping improvement ordering.
ResearchPipelineOrchestratorTest: expanded to 5 tests.

Refs #1115"
```

---

## Batch 7: DevTown Workers

After this batch: DevTown's evolution loop can execute operational improvements
end-to-end — observe, propose, review, apply, observe.

### Task 11: DevTown improvement workers and services

**Files (in casehubio/devtown):**
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/DevtownReviewerCalibrationWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/DevtownRoutingAdjustmentWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/DevtownSlaWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/DevtownGateTighteningWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/worker/DevtownQueueOptimizationWorker.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/service/ReviewerTrustService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/service/RoutingConfigService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/service/SlaConfigService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/service/GatePolicyService.java`
- Create: `app/src/main/java/io/casehub/devtown/app/evolution/service/QueueConfigService.java`
- Test: `app/src/test/java/io/casehub/devtown/app/evolution/worker/DevtownEvolutionWorkerTest.java`

**Interfaces:**
- Consumes: `ImprovementRequest` metadata for change parameters
- Produces: `WorkerResult.completed()` with change summary

- [ ] **Step 1: Create service interfaces** (5 thin CDI beans)

Each service modifies its domain's operational configuration. Implementation details
(case context, database, config) are devtown-specific.

- [ ] **Step 2: Create workers** (5 workers, one per category)

Each worker reads `ImprovementRequest` metadata, calls the service, records the
change in case context, returns `WorkerResult.completed()`.

- [ ] **Step 3: Write tests**

One test class with 5 test methods (one per worker), mocking the service and verifying
the correct call with correct parameters extracted from the improvement request.

- [ ] **Step 4: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl app -Dtest=DevtownEvolutionWorkerTest -q`
(from devtown project root)

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(#1115): add DevTown evolution workers and services

5 workers (reviewer-calibration, routing-adjustment, sla-tuning,
gate-tightening, queue-optimization) calling internal services to apply
operational improvements. Closes the apply stage in DevTown's 5-stage
evolution lifecycle.

Refs #1115"
```

---

## References

- `specs/issue-1115-continuous-evolution-loop/2026-10-26-evolution-loop-gap-closure-design.md` — design spec
- `docs/specs/issue-1104-hive-mind/2026-09-20-continuous-evolution-loop-design.md` — parent spec
- `specs/issue-1115-continuous-evolution-loop/gap-analysis.md` — gap analysis
- `specs/issue-1115-continuous-evolution-loop/decisions.md` — D1–D5
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:1498` — convergence detection call site
- `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java:1360` — EvolutionTicker producer
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/EvolutionTicker.java` — gate pipeline
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementGoalFormationStrategy.java` — filter pipeline
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementOutcomeEventCapture.java` — 6-layer capture
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/RegressionDetector.java` — regression detection
- `runtime-core/src/main/java/io/casehub/engine/runtime/improvement/ImprovementCircuitBreaker.java` — circuit breaker
- `api/src/main/java/io/casehub/api/spi/CaseOutcomeObserver.java` — case completion SPI
- `common-core/src/main/java/io/casehub/engine/common/spi/event/RegressionDetectedEvent.java` — existing event pattern
- GitHub #1115, #1104 (Hive Mind epic)
