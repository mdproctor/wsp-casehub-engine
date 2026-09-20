# Convergence Detection & Termination Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1110 — feat: Convergence detection & termination — emergent completion, runaway & collusion prevention
**Issue group:** #1104 (Hive Mind epic), #1105-#1115

**Goal:** Add activity tracking, budget enforcement, convergence detection, and output convergence monitoring to the engine evaluation pipeline.

**Architecture:** Five new components: `ActivityTracker` (common-core, per-case metrics + sliding window rates), `ConvergenceDetector` + `BudgetEnforcer` + `OutputConvergenceMonitor` (runtime-core, pipeline integration), and three config records (engine-api). Convergence fires synthetic `GoalReachedEvent("_converged")`; budget exhaustion faults the case. Output convergence fires informational events.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ

## Global Constraints

- All new types follow the Apache 2.0 license header (copy from any existing file in the module)
- Records use compact constructor validation where applicable (see `SignalConfig.java` pattern)
- `@ApplicationScoped` beans that hold per-case mutable state implement `Resettable`
- Per-case state uses `ConcurrentHashMap<UUID, ...>` (same pattern as `SignalRegistry`, `ObservationRegistry`, `RuleRegistry`)
- New `CaseHubEventType` values are added to the enum in alphabetical order within their logical group
- YAML schema changes go in `schema/src/main/resources/schema/CaseDefinition.yaml`
- YAML record mappings go in `schema/src/main/resources/schema/yaml-record-mappings.yaml`
- Test classes use `*Test.java` naming (never `*IT.java`)
- `CaseDefinition` builder methods follow the existing pattern: private field, getter, builder setter returning `this`

---

## Batch 1: Foundation types + ActivityTracker

### Task 1: Config records + CaseHubEventType additions

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/convergence/BudgetConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/convergence/ConvergenceThresholdConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/convergence/OutputConvergenceConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`
- Test: `api/src/test/java/io/casehub/api/model/convergence/BudgetConfigTest.java`
- Test: `api/src/test/java/io/casehub/api/model/convergence/ConvergenceThresholdConfigTest.java`
- Test: `api/src/test/java/io/casehub/api/model/convergence/OutputConvergenceConfigTest.java`

**Interfaces:**
- Produces: `BudgetConfig(Integer maxDispatches, Integer maxSignalDeposits, Integer maxContextMutations, Integer maxEvaluationCycles)`, `ConvergenceThresholdConfig(Double dispatchRateThreshold, Double signalDepositRateThreshold, Double contextMutationRateThreshold, Double evaluationRateThreshold, Duration stabilityWindow, Duration rateWindow, Integer maxWindowEntries)`, `OutputConvergenceConfig(Double convergenceThreshold, Integer convergenceMinSamples, Integer outputWindowSize)`, `CaseDefinition.getBudgetConfig()`, `CaseDefinition.getConvergenceThresholdConfig()`, `CaseDefinition.getOutputConvergenceConfig()`

- [ ] **Step 1: Write tests for BudgetConfig**

```java
package io.casehub.api.model.convergence;

import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.Test;

class BudgetConfigTest {

  @Test
  void defaults_returns_all_null_caps() {
    var config = BudgetConfig.defaults();
    assertThat(config.maxDispatches()).isNull();
    assertThat(config.maxSignalDeposits()).isNull();
    assertThat(config.maxContextMutations()).isNull();
    assertThat(config.maxEvaluationCycles()).isNull();
  }

  @Test
  void rejects_negative_values() {
    assertThatThrownBy(() -> new BudgetConfig(-1, null, null, null))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void accepts_zero_as_valid_cap() {
    var config = new BudgetConfig(0, 0, 0, 0);
    assertThat(config.maxDispatches()).isEqualTo(0);
  }

  @Test
  void isExceeded_returns_true_when_count_exceeds_cap() {
    var config = new BudgetConfig(100, null, null, null);
    assertThat(config.isExceeded("dispatches", 101, config.maxDispatches())).isTrue();
    assertThat(config.isExceeded("dispatches", 100, config.maxDispatches())).isFalse();
    assertThat(config.isExceeded("signals", 9999, config.maxSignalDeposits())).isFalse();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=BudgetConfigTest -q 2>&1 | tail -5`
Expected: FAIL — class not found

- [ ] **Step 3: Implement BudgetConfig**

```java
package io.casehub.api.model.convergence;

public record BudgetConfig(
    Integer maxDispatches,
    Integer maxSignalDeposits,
    Integer maxContextMutations,
    Integer maxEvaluationCycles) {

  public BudgetConfig {
    if (maxDispatches != null && maxDispatches < 0)
      throw new IllegalArgumentException("maxDispatches must be >= 0, got: " + maxDispatches);
    if (maxSignalDeposits != null && maxSignalDeposits < 0)
      throw new IllegalArgumentException("maxSignalDeposits must be >= 0, got: " + maxSignalDeposits);
    if (maxContextMutations != null && maxContextMutations < 0)
      throw new IllegalArgumentException("maxContextMutations must be >= 0, got: " + maxContextMutations);
    if (maxEvaluationCycles != null && maxEvaluationCycles < 0)
      throw new IllegalArgumentException("maxEvaluationCycles must be >= 0, got: " + maxEvaluationCycles);
  }

  public static BudgetConfig defaults() {
    return new BudgetConfig(null, null, null, null);
  }

  public static boolean isExceeded(String metric, long count, Integer cap) {
    return cap != null && count > cap;
  }
}
```

- [ ] **Step 4: Write tests for ConvergenceThresholdConfig**

```java
package io.casehub.api.model.convergence;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class ConvergenceThresholdConfigTest {

  @Test
  void defaults_returns_sensible_values() {
    var config = ConvergenceThresholdConfig.defaults();
    assertThat(config.dispatchRateThreshold()).isEqualTo(0.1);
    assertThat(config.stabilityWindow()).isEqualTo(Duration.ofSeconds(30));
    assertThat(config.rateWindow()).isEqualTo(Duration.ofSeconds(60));
    assertThat(config.maxWindowEntries()).isNull();
  }

  @Test
  void rejects_negative_thresholds() {
    assertThatThrownBy(() -> new ConvergenceThresholdConfig(
        -0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(30), Duration.ofSeconds(60), null))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void rejects_zero_stability_window() {
    assertThatThrownBy(() -> new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ZERO, Duration.ofSeconds(60), null))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void effectiveMaxWindowEntries_derived_from_rateWindow_when_null() {
    var config = ConvergenceThresholdConfig.defaults();
    assertThat(config.effectiveMaxWindowEntries()).isEqualTo(600);
  }

  @Test
  void effectiveMaxWindowEntries_uses_override_when_set() {
    var config = new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(30), Duration.ofSeconds(60), 200);
    assertThat(config.effectiveMaxWindowEntries()).isEqualTo(200);
  }
}
```

- [ ] **Step 5: Implement ConvergenceThresholdConfig**

```java
package io.casehub.api.model.convergence;

import java.time.Duration;

public record ConvergenceThresholdConfig(
    Double dispatchRateThreshold,
    Double signalDepositRateThreshold,
    Double contextMutationRateThreshold,
    Double evaluationRateThreshold,
    Duration stabilityWindow,
    Duration rateWindow,
    Integer maxWindowEntries) {

  public static final double DEFAULT_DISPATCH_RATE = 0.1;
  public static final double DEFAULT_SIGNAL_DEPOSIT_RATE = 0.1;
  public static final double DEFAULT_CONTEXT_MUTATION_RATE = 0.1;
  public static final double DEFAULT_EVALUATION_RATE = 0.5;
  public static final Duration DEFAULT_STABILITY_WINDOW = Duration.ofSeconds(30);
  public static final Duration DEFAULT_RATE_WINDOW = Duration.ofSeconds(60);

  public ConvergenceThresholdConfig {
    if (dispatchRateThreshold != null && dispatchRateThreshold < 0)
      throw new IllegalArgumentException("dispatchRateThreshold must be >= 0");
    if (signalDepositRateThreshold != null && signalDepositRateThreshold < 0)
      throw new IllegalArgumentException("signalDepositRateThreshold must be >= 0");
    if (contextMutationRateThreshold != null && contextMutationRateThreshold < 0)
      throw new IllegalArgumentException("contextMutationRateThreshold must be >= 0");
    if (evaluationRateThreshold != null && evaluationRateThreshold < 0)
      throw new IllegalArgumentException("evaluationRateThreshold must be >= 0");
    if (stabilityWindow != null && (stabilityWindow.isNegative() || stabilityWindow.isZero()))
      throw new IllegalArgumentException("stabilityWindow must be positive");
    if (rateWindow != null && (rateWindow.isNegative() || rateWindow.isZero()))
      throw new IllegalArgumentException("rateWindow must be positive");
    if (maxWindowEntries != null && maxWindowEntries < 1)
      throw new IllegalArgumentException("maxWindowEntries must be >= 1");
  }

  public static ConvergenceThresholdConfig defaults() {
    return new ConvergenceThresholdConfig(
        DEFAULT_DISPATCH_RATE, DEFAULT_SIGNAL_DEPOSIT_RATE,
        DEFAULT_CONTEXT_MUTATION_RATE, DEFAULT_EVALUATION_RATE,
        DEFAULT_STABILITY_WINDOW, DEFAULT_RATE_WINDOW, null);
  }

  public int effectiveMaxWindowEntries() {
    if (maxWindowEntries != null) return maxWindowEntries;
    Duration window = rateWindow != null ? rateWindow : DEFAULT_RATE_WINDOW;
    return (int) (window.toSeconds() * 10);
  }
}
```

- [ ] **Step 6: Write tests for OutputConvergenceConfig**

```java
package io.casehub.api.model.convergence;

import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.Test;

class OutputConvergenceConfigTest {

  @Test
  void defaults_returns_sensible_values() {
    var config = OutputConvergenceConfig.defaults();
    assertThat(config.convergenceThreshold()).isEqualTo(0.9);
    assertThat(config.convergenceMinSamples()).isEqualTo(3);
    assertThat(config.outputWindowSize()).isEqualTo(10);
  }

  @Test
  void rejects_threshold_above_one() {
    assertThatThrownBy(() -> new OutputConvergenceConfig(1.1, 3, 10))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void rejects_min_samples_below_two() {
    assertThatThrownBy(() -> new OutputConvergenceConfig(0.9, 1, 10))
        .isInstanceOf(IllegalArgumentException.class);
  }
}
```

- [ ] **Step 7: Implement OutputConvergenceConfig**

```java
package io.casehub.api.model.convergence;

public record OutputConvergenceConfig(
    double convergenceThreshold, int convergenceMinSamples, int outputWindowSize) {

  public static final double DEFAULT_CONVERGENCE_THRESHOLD = 0.9;
  public static final int DEFAULT_CONVERGENCE_MIN_SAMPLES = 3;
  public static final int DEFAULT_OUTPUT_WINDOW_SIZE = 10;

  public OutputConvergenceConfig {
    if (convergenceThreshold < 0.0 || convergenceThreshold > 1.0)
      throw new IllegalArgumentException("convergenceThreshold must be in [0.0, 1.0]");
    if (convergenceMinSamples < 2)
      throw new IllegalArgumentException("convergenceMinSamples must be >= 2");
    if (outputWindowSize < 2)
      throw new IllegalArgumentException("outputWindowSize must be >= 2");
  }

  public static OutputConvergenceConfig defaults() {
    return new OutputConvergenceConfig(
        DEFAULT_CONVERGENCE_THRESHOLD, DEFAULT_CONVERGENCE_MIN_SAMPLES,
        DEFAULT_OUTPUT_WINDOW_SIZE);
  }
}
```

- [ ] **Step 8: Add CaseHubEventType values**

Add three new values to `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` after `RULE_FIRED`:

```java
  BUDGET_EXHAUSTED, // cumulative activity budget exceeded — case faulted
  CONVERGENCE_DETECTED, // all activity rates below threshold for stability window
  OUTPUT_CONVERGENCE_DETECTED // per-binding output structural similarity exceeds threshold
```

- [ ] **Step 9: Add fields to CaseDefinition**

Add to `api/src/main/java/io/casehub/api/model/CaseDefinition.java`:
- Private field `private BudgetConfig budgetConfig;`
- Private field `private ConvergenceThresholdConfig convergenceThresholdConfig;`
- Private field `private OutputConvergenceConfig outputConvergenceConfig;`
- Getter and builder setter for each (follow the existing `signalConfig` pattern)

- [ ] **Step 10: Run all tests and commit**

Run: `mvn test -pl api -q 2>&1 | tail -5`
Expected: All PASS

```bash
git add api/src/main/java/io/casehub/api/model/convergence/ api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java api/src/main/java/io/casehub/api/model/CaseDefinition.java api/src/test/java/io/casehub/api/model/convergence/
git commit -m "feat: add convergence config records + CaseHubEventType values Refs #1110"
```

### Task 2: SlidingWindowCounter + ActivityTracker

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/convergence/SlidingWindowCounter.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/convergence/CaseActivityState.java`
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/convergence/ActivityTracker.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/convergence/SlidingWindowCounterTest.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/convergence/ActivityTrackerTest.java`

**Interfaces:**
- Consumes: `BudgetConfig.isExceeded()`, `ConvergenceThresholdConfig.effectiveMaxWindowEntries()`, `Resettable` interface from `common-core/src/main/java/io/casehub/engine/common/spi/Resettable.java`
- Produces: `ActivityTracker.recordDispatch(UUID caseId)`, `ActivityTracker.recordSignalDeposit(UUID caseId)`, `ActivityTracker.recordContextMutation(UUID caseId, int keyCount)`, `ActivityTracker.recordEvaluationCycle(UUID caseId)`, `ActivityTracker.getState(UUID caseId) → CaseActivityState`, `ActivityTracker.evictByCase(UUID caseId)`, `CaseActivityState.totalDispatches()`, `CaseActivityState.dispatchRate(Duration window, Instant now)`, and corresponding methods for all four metrics

- [ ] **Step 1: Write tests for SlidingWindowCounter**

```java
package io.casehub.engine.common.internal.convergence;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import java.time.Instant;
import org.junit.jupiter.api.Test;

class SlidingWindowCounterTest {

  @Test
  void empty_counter_returns_zero_rate() {
    var counter = new SlidingWindowCounter(100);
    assertThat(counter.rate(Duration.ofSeconds(60), Instant.now())).isEqualTo(0.0);
  }

  @Test
  void single_event_returns_correct_rate() {
    var counter = new SlidingWindowCounter(100);
    var now = Instant.now();
    counter.record(now);
    assertThat(counter.rate(Duration.ofSeconds(60), now)).isCloseTo(1.0 / 60, within(0.001));
  }

  @Test
  void events_outside_window_excluded() {
    var counter = new SlidingWindowCounter(100);
    var base = Instant.now();
    counter.record(base.minusSeconds(120));
    counter.record(base.minusSeconds(30));
    counter.record(base);
    assertThat(counter.rate(Duration.ofSeconds(60), base)).isCloseTo(2.0 / 60, within(0.001));
  }

  @Test
  void oldest_evicted_when_capacity_exceeded() {
    var counter = new SlidingWindowCounter(3);
    var base = Instant.now();
    counter.record(base.minusSeconds(3));
    counter.record(base.minusSeconds(2));
    counter.record(base.minusSeconds(1));
    counter.record(base);
    assertThat(counter.count()).isEqualTo(3);
  }

  @Test
  void total_returns_cumulative_count() {
    var counter = new SlidingWindowCounter(100);
    var now = Instant.now();
    counter.record(now);
    counter.record(now);
    counter.record(now);
    assertThat(counter.total()).isEqualTo(3);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl common-core -Dtest=SlidingWindowCounterTest -q 2>&1 | tail -5`
Expected: FAIL — class not found

- [ ] **Step 3: Implement SlidingWindowCounter**

```java
package io.casehub.engine.common.internal.convergence;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicLong;

public final class SlidingWindowCounter {

  private final Instant[] buffer;
  private int head;
  private int size;
  private final AtomicLong totalCount = new AtomicLong();

  public SlidingWindowCounter(int capacity) {
    if (capacity < 1) throw new IllegalArgumentException("capacity must be >= 1");
    this.buffer = new Instant[capacity];
  }

  public synchronized void record(Instant timestamp) {
    buffer[head] = timestamp;
    head = (head + 1) % buffer.length;
    if (size < buffer.length) size++;
    totalCount.incrementAndGet();
  }

  public synchronized double rate(Duration window, Instant now) {
    if (size == 0) return 0.0;
    Instant cutoff = now.minus(window);
    int count = 0;
    for (int i = 0; i < size; i++) {
      int idx = (head - 1 - i + buffer.length) % buffer.length;
      if (!buffer[idx].isBefore(cutoff)) count++;
    }
    return (double) count / window.toSeconds();
  }

  public synchronized int count() {
    return size;
  }

  public long total() {
    return totalCount.get();
  }

  public synchronized void reset() {
    head = 0;
    size = 0;
    totalCount.set(0);
  }
}
```

- [ ] **Step 4: Write tests for ActivityTracker**

```java
package io.casehub.engine.common.internal.convergence;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class ActivityTrackerTest {

  private final ActivityTracker tracker = new ActivityTracker();

  @Test
  void getState_returns_empty_state_for_unknown_case() {
    var state = tracker.getState(UUID.randomUUID());
    assertThat(state).isNotNull();
    assertThat(state.totalDispatches()).isEqualTo(0);
    assertThat(state.totalSignalDeposits()).isEqualTo(0);
    assertThat(state.totalContextMutations()).isEqualTo(0);
    assertThat(state.totalEvaluationCycles()).isEqualTo(0);
  }

  @Test
  void recordDispatch_increments_counter() {
    var caseId = UUID.randomUUID();
    tracker.recordDispatch(caseId);
    tracker.recordDispatch(caseId);
    assertThat(tracker.getState(caseId).totalDispatches()).isEqualTo(2);
  }

  @Test
  void recordSignalDeposit_increments_counter() {
    var caseId = UUID.randomUUID();
    tracker.recordSignalDeposit(caseId);
    assertThat(tracker.getState(caseId).totalSignalDeposits()).isEqualTo(1);
  }

  @Test
  void recordContextMutation_increments_by_key_count() {
    var caseId = UUID.randomUUID();
    tracker.recordContextMutation(caseId, 5);
    tracker.recordContextMutation(caseId, 3);
    assertThat(tracker.getState(caseId).totalContextMutations()).isEqualTo(8);
  }

  @Test
  void recordEvaluationCycle_increments_counter() {
    var caseId = UUID.randomUUID();
    tracker.recordEvaluationCycle(caseId);
    assertThat(tracker.getState(caseId).totalEvaluationCycles()).isEqualTo(1);
  }

  @Test
  void evictByCase_removes_state() {
    var caseId = UUID.randomUUID();
    tracker.recordDispatch(caseId);
    tracker.evictByCase(caseId);
    assertThat(tracker.getState(caseId).totalDispatches()).isEqualTo(0);
  }

  @Test
  void rates_return_zero_for_new_case() {
    var caseId = UUID.randomUUID();
    var state = tracker.getState(caseId);
    var now = Instant.now();
    assertThat(state.dispatchRate(Duration.ofSeconds(60), now)).isEqualTo(0.0);
  }

  @Test
  void reset_clears_all_state() {
    var caseId = UUID.randomUUID();
    tracker.recordDispatch(caseId);
    tracker.reset();
    assertThat(tracker.getState(caseId).totalDispatches()).isEqualTo(0);
  }
}
```

- [ ] **Step 5: Implement CaseActivityState and ActivityTracker**

`CaseActivityState`:
```java
package io.casehub.engine.common.internal.convergence;

import java.time.Duration;
import java.time.Instant;

public final class CaseActivityState {
  private final SlidingWindowCounter dispatches;
  private final SlidingWindowCounter signalDeposits;
  private final SlidingWindowCounter contextMutations;
  private final SlidingWindowCounter evaluationCycles;

  public CaseActivityState(int maxWindowEntries) {
    this.dispatches = new SlidingWindowCounter(maxWindowEntries);
    this.signalDeposits = new SlidingWindowCounter(maxWindowEntries);
    this.contextMutations = new SlidingWindowCounter(maxWindowEntries);
    this.evaluationCycles = new SlidingWindowCounter(maxWindowEntries);
  }

  public void recordDispatch(Instant now) { dispatches.record(now); }
  public void recordSignalDeposit(Instant now) { signalDeposits.record(now); }
  public void recordContextMutation(Instant now, int keyCount) {
    for (int i = 0; i < keyCount; i++) contextMutations.record(now);
  }
  public void recordEvaluationCycle(Instant now) { evaluationCycles.record(now); }

  public long totalDispatches() { return dispatches.total(); }
  public long totalSignalDeposits() { return signalDeposits.total(); }
  public long totalContextMutations() { return contextMutations.total(); }
  public long totalEvaluationCycles() { return evaluationCycles.total(); }

  public double dispatchRate(Duration window, Instant now) { return dispatches.rate(window, now); }
  public double signalDepositRate(Duration window, Instant now) { return signalDeposits.rate(window, now); }
  public double contextMutationRate(Duration window, Instant now) { return contextMutations.rate(window, now); }
  public double evaluationRate(Duration window, Instant now) { return evaluationCycles.rate(window, now); }
}
```

`ActivityTracker`:
```java
package io.casehub.engine.common.internal.convergence;

import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ActivityTracker implements Resettable {

  static final int DEFAULT_MAX_WINDOW_ENTRIES = 600;
  private final ConcurrentHashMap<UUID, CaseActivityState> states = new ConcurrentHashMap<>();
  private volatile int maxWindowEntries = DEFAULT_MAX_WINDOW_ENTRIES;

  public void setMaxWindowEntries(int maxWindowEntries) {
    this.maxWindowEntries = maxWindowEntries;
  }

  public CaseActivityState getState(UUID caseId) {
    return states.computeIfAbsent(caseId, k -> new CaseActivityState(maxWindowEntries));
  }

  public void recordDispatch(UUID caseId) {
    getState(caseId).recordDispatch(Instant.now());
  }

  public void recordSignalDeposit(UUID caseId) {
    getState(caseId).recordSignalDeposit(Instant.now());
  }

  public void recordContextMutation(UUID caseId, int keyCount) {
    getState(caseId).recordContextMutation(Instant.now(), keyCount);
  }

  public void recordEvaluationCycle(UUID caseId) {
    getState(caseId).recordEvaluationCycle(Instant.now());
  }

  public void evictByCase(UUID caseId) {
    states.remove(caseId);
  }

  @Override
  public void reset() {
    states.clear();
  }
}
```

- [ ] **Step 6: Run all tests and commit**

Run: `mvn test -pl common-core -Dtest="SlidingWindowCounterTest,ActivityTrackerTest" -q 2>&1 | tail -5`
Expected: All PASS

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/convergence/ common-core/src/test/java/io/casehub/engine/common/internal/convergence/
git commit -m "feat: add SlidingWindowCounter + ActivityTracker Refs #1110"
```

## Batch 2: Pipeline integration — ConvergenceDetector + BudgetEnforcer

### Task 3: ConvergenceDetector + BudgetEnforcer + pipeline wiring

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/convergence/ConvergenceDetector.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/convergence/BudgetEnforcer.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/convergence/ConvergenceDetectorTest.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/convergence/BudgetEnforcerTest.java`

**Interfaces:**
- Consumes: `ActivityTracker.getState()`, `ActivityTracker.recordDispatch()`, `ActivityTracker.recordEvaluationCycle()`, `ActivityTracker.recordContextMutation()`, `ActivityTracker.evictByCase()`, `BudgetConfig`, `ConvergenceThresholdConfig`, `CaseDefinition.getBudgetConfig()`, `CaseDefinition.getConvergenceThresholdConfig()`, `GoalReachedEvent(CaseInstance, List<Goal>)`, `CaseHubEventType.CONVERGENCE_DETECTED`, `CaseHubEventType.BUDGET_EXHAUSTED`
- Produces: `ConvergenceDetector.evaluate(UUID caseId, CaseInstance instance, CaseDefinition definition)`, `BudgetEnforcer.checkBudget(UUID caseId, String metric, long count, Integer cap, CaseInstance instance)`

- [ ] **Step 1: Write tests for BudgetEnforcer**

```java
package io.casehub.engine.internal.convergence;

import static org.assertj.core.api.Assertions.*;
import io.casehub.api.model.convergence.BudgetConfig;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class BudgetEnforcerTest {

  @Test
  void no_budget_config_returns_false() {
    var enforcer = new BudgetEnforcer();
    assertThat(enforcer.isExhausted(null, 0, 0, 0, 0)).isFalse();
  }

  @Test
  void all_null_caps_returns_false() {
    var config = BudgetConfig.defaults();
    assertThat(new BudgetEnforcer().isExhausted(config, 999, 999, 999, 999)).isFalse();
  }

  @Test
  void dispatch_cap_exceeded_returns_true() {
    var config = new BudgetConfig(10, null, null, null);
    assertThat(new BudgetEnforcer().isExhausted(config, 11, 0, 0, 0)).isTrue();
  }

  @Test
  void dispatch_cap_not_exceeded_returns_false() {
    var config = new BudgetConfig(10, null, null, null);
    assertThat(new BudgetEnforcer().isExhausted(config, 10, 0, 0, 0)).isFalse();
  }

  @Test
  void signal_cap_exceeded_returns_true() {
    var config = new BudgetConfig(null, 5, null, null);
    assertThat(new BudgetEnforcer().isExhausted(config, 0, 6, 0, 0)).isTrue();
  }

  @Test
  void exhaustedMetric_identifies_correct_metric() {
    var config = new BudgetConfig(null, null, 100, null);
    var enforcer = new BudgetEnforcer();
    assertThat(enforcer.exhaustedMetric(config, 0, 0, 101, 0)).isEqualTo("contextMutations");
  }
}
```

- [ ] **Step 2: Implement BudgetEnforcer**

```java
package io.casehub.engine.internal.convergence;

import io.casehub.api.model.convergence.BudgetConfig;
import jakarta.annotation.Nullable;

public class BudgetEnforcer {

  public boolean isExhausted(
      @Nullable BudgetConfig config,
      long dispatches, long signalDeposits, long contextMutations, long evaluationCycles) {
    if (config == null) return false;
    return BudgetConfig.isExceeded("dispatches", dispatches, config.maxDispatches())
        || BudgetConfig.isExceeded("signalDeposits", signalDeposits, config.maxSignalDeposits())
        || BudgetConfig.isExceeded("contextMutations", contextMutations, config.maxContextMutations())
        || BudgetConfig.isExceeded("evaluationCycles", evaluationCycles, config.maxEvaluationCycles());
  }

  public @Nullable String exhaustedMetric(
      @Nullable BudgetConfig config,
      long dispatches, long signalDeposits, long contextMutations, long evaluationCycles) {
    if (config == null) return null;
    if (BudgetConfig.isExceeded("dispatches", dispatches, config.maxDispatches())) return "dispatches";
    if (BudgetConfig.isExceeded("signalDeposits", signalDeposits, config.maxSignalDeposits())) return "signalDeposits";
    if (BudgetConfig.isExceeded("contextMutations", contextMutations, config.maxContextMutations())) return "contextMutations";
    if (BudgetConfig.isExceeded("evaluationCycles", evaluationCycles, config.maxEvaluationCycles())) return "evaluationCycles";
    return null;
  }
}
```

- [ ] **Step 3: Write tests for ConvergenceDetector**

```java
package io.casehub.engine.internal.convergence;

import static org.assertj.core.api.Assertions.*;
import io.casehub.api.model.convergence.ConvergenceThresholdConfig;
import io.casehub.engine.common.internal.convergence.ActivityTracker;
import io.casehub.engine.common.internal.convergence.CaseActivityState;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class ConvergenceDetectorTest {

  private final ConvergenceDetector detector = new ConvergenceDetector();

  @Test
  void no_config_returns_not_converged() {
    var state = new CaseActivityState(100);
    assertThat(detector.evaluate(UUID.randomUUID(), state, null, Instant.now())).isFalse();
  }

  @Test
  void all_rates_below_threshold_but_stability_window_not_met() {
    var caseId = UUID.randomUUID();
    var state = new CaseActivityState(100);
    var config = ConvergenceThresholdConfig.defaults();
    var now = Instant.now();
    assertThat(detector.evaluate(caseId, state, config, now)).isFalse();
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(10))).isFalse();
  }

  @Test
  void convergence_detected_after_stability_window() {
    var caseId = UUID.randomUUID();
    var state = new CaseActivityState(100);
    var config = new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(5), Duration.ofSeconds(60), null);
    var now = Instant.now();
    detector.evaluate(caseId, state, config, now);
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(6))).isTrue();
  }

  @Test
  void convergence_fires_only_once() {
    var caseId = UUID.randomUUID();
    var state = new CaseActivityState(100);
    var config = new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(1), Duration.ofSeconds(60), null);
    var now = Instant.now();
    detector.evaluate(caseId, state, config, now);
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(2))).isTrue();
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(3))).isFalse();
  }

  @Test
  void activity_resets_quiet_period() {
    var caseId = UUID.randomUUID();
    var state = new CaseActivityState(100);
    var config = new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(5), Duration.ofSeconds(60), null);
    var now = Instant.now();
    detector.evaluate(caseId, state, config, now);
    state.recordDispatch(now.plusSeconds(3));
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(4))).isFalse();
  }

  @Test
  void evict_clears_convergence_state() {
    var caseId = UUID.randomUUID();
    var state = new CaseActivityState(100);
    var config = new ConvergenceThresholdConfig(
        0.1, 0.1, 0.1, 0.5, Duration.ofSeconds(1), Duration.ofSeconds(60), null);
    var now = Instant.now();
    detector.evaluate(caseId, state, config, now);
    detector.evaluate(caseId, state, config, now.plusSeconds(2));
    detector.evictByCase(caseId);
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(10))).isFalse();
    assertThat(detector.evaluate(caseId, state, config, now.plusSeconds(12))).isTrue();
  }
}
```

- [ ] **Step 4: Implement ConvergenceDetector**

```java
package io.casehub.engine.internal.convergence;

import io.casehub.api.model.convergence.ConvergenceThresholdConfig;
import io.casehub.engine.common.internal.convergence.CaseActivityState;
import io.casehub.engine.common.spi.Resettable;
import jakarta.annotation.Nullable;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ConvergenceDetector implements Resettable {

  private final ConcurrentHashMap<UUID, ConvergenceState> states = new ConcurrentHashMap<>();

  public boolean evaluate(
      UUID caseId,
      CaseActivityState activity,
      @Nullable ConvergenceThresholdConfig config,
      Instant now) {
    if (config == null) return false;

    var state = states.computeIfAbsent(caseId, k -> new ConvergenceState());
    if (state.converged) return false;

    Duration rateWindow = config.rateWindow() != null
        ? config.rateWindow() : ConvergenceThresholdConfig.DEFAULT_RATE_WINDOW;

    boolean allQuiet =
        activity.dispatchRate(rateWindow, now) < config.dispatchRateThreshold()
            && activity.signalDepositRate(rateWindow, now) < config.signalDepositRateThreshold()
            && activity.contextMutationRate(rateWindow, now) < config.contextMutationRateThreshold()
            && activity.evaluationRate(rateWindow, now) < config.evaluationRateThreshold();

    if (!allQuiet) {
      state.firstQuietCycle = null;
      return false;
    }

    if (state.firstQuietCycle == null) {
      state.firstQuietCycle = now;
      return false;
    }

    Duration quietDuration = Duration.between(state.firstQuietCycle, now);
    Duration stabilityWindow = config.stabilityWindow() != null
        ? config.stabilityWindow() : ConvergenceThresholdConfig.DEFAULT_STABILITY_WINDOW;

    if (quietDuration.compareTo(stabilityWindow) >= 0) {
      state.converged = true;
      return true;
    }
    return false;
  }

  public void evictByCase(UUID caseId) {
    states.remove(caseId);
  }

  @Override
  public void reset() {
    states.clear();
  }

  private static class ConvergenceState {
    Instant firstQuietCycle;
    boolean converged;
  }
}
```

- [ ] **Step 5: Wire instrumentation into CaseContextChangedEventHandler**

Modify `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`:

1. Inject `ActivityTracker activityTracker`
2. In `evaluateAndDispatch()` at the top: call `activityTracker.recordEvaluationCycle(caseInstance.getUuid())` and `activityTracker.recordContextMutation(caseInstance.getUuid(), event.changedKeys() != null ? event.changedKeys().size() : 0)`
3. In `publishWorkerSchedule()` after successful dispatch: call `activityTracker.recordDispatch(caseInstance.getUuid())`
4. Add `convergenceDetection()` as 5th phase after `localRules()` call in `evaluateAndDispatch()`

The `convergenceDetection()` method:
```java
private void convergenceDetection(CaseInstance caseInstance, CaseDefinition definition) {
  var budgetConfig = definition.getBudgetConfig();
  var convergenceConfig = definition.getConvergenceThresholdConfig();
  if (budgetConfig == null && convergenceConfig == null) return;

  var state = activityTracker.getState(caseInstance.getUuid());
  var now = Instant.now();

  // Budget enforcement
  if (budgetEnforcer.isExhausted(budgetConfig,
      state.totalDispatches(), state.totalSignalDeposits(),
      state.totalContextMutations(), state.totalEvaluationCycles())) {
    String metric = budgetEnforcer.exhaustedMetric(budgetConfig,
        state.totalDispatches(), state.totalSignalDeposits(),
        state.totalContextMutations(), state.totalEvaluationCycles());
    // Write BUDGET_EXHAUSTED EventLog and fault the case
    writeBudgetExhaustedEvent(caseInstance, metric, state, budgetConfig);
    eventDispatcher.dispatch(new CaseStatusChanged(caseInstance,
        caseInstance.getState().name(), "FAULTED", null, "Budget exhausted: " + metric));
    return;
  }

  // Convergence detection
  if (convergenceDetector.evaluate(caseInstance.getUuid(), state, convergenceConfig, now)) {
    writeConvergenceDetectedEvent(caseInstance, state, convergenceConfig, now);
    var convergedGoal = new Goal("_converged", null);
    eventDispatcher.dispatch(new GoalReachedEvent(caseInstance, List.of(convergedGoal)));
  }
}
```

- [ ] **Step 6: Wire signal deposit instrumentation into SignalRegistry**

Modify `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java`:

1. Add `private ActivityTracker activityTracker;` field with `@Inject` (use `Instance<ActivityTracker>` with `isResolvable()` guard for backward compat)
2. In `deposit()` method, after the signal is deposited or reinforced: call `activityTracker.get().recordSignalDeposit(caseId)` if resolvable

- [ ] **Step 7: Wire eviction into CaseStatusChangedHandler**

Modify `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`:

Add `activityTracker.evictByCase(caseId)` and `convergenceDetector.evictByCase(caseId)` to the terminal state cleanup section (alongside `signalRegistry.evictByCase()`, `ruleRegistry.evictByCase()`, etc.).

- [ ] **Step 8: Run all tests and commit**

Run: `mvn test -pl common-core,runtime-core -q 2>&1 | tail -10`
Expected: All PASS

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java common-core/src/main/java/io/casehub/engine/common/internal/convergence/ common-core/src/test/java/io/casehub/engine/common/internal/convergence/ runtime-core/src/main/java/io/casehub/engine/internal/convergence/ runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java runtime-core/src/test/java/io/casehub/engine/internal/convergence/
git commit -m "feat: wire ConvergenceDetector + BudgetEnforcer into evaluation pipeline Refs #1110"
```

## Batch 3: OutputConvergenceMonitor + YAML + CLAUDE.md

### Task 4: OutputConvergenceMonitor

**Files:**
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/convergence/OutputConvergenceMonitor.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/convergence/OutputConvergenceMonitorTest.java`

**Interfaces:**
- Consumes: `OutputConvergenceConfig`, `CaseHubEventType.OUTPUT_CONVERGENCE_DETECTED`, `WorkflowExecutionCompleted` (record — `outcome()`, `worker()`, `bindingName()`)
- Produces: `OutputConvergenceMonitor.recordOutput(UUID caseId, String bindingName, String executorName, Map<String, Object> output)`, `OutputConvergenceMonitor.evictByCase(UUID caseId)`

- [ ] **Step 1: Write tests for OutputConvergenceMonitor**

```java
package io.casehub.engine.internal.convergence;

import static org.assertj.core.api.Assertions.*;
import io.casehub.api.model.convergence.OutputConvergenceConfig;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class OutputConvergenceMonitorTest {

  private final OutputConvergenceMonitor monitor = new OutputConvergenceMonitor();

  @Test
  void no_detection_below_min_samples() {
    var caseId = UUID.randomUUID();
    var config = OutputConvergenceConfig.defaults();
    monitor.recordOutput(caseId, "binding1", "agent1", Map.of("key", "value1"));
    monitor.recordOutput(caseId, "binding1", "agent2", Map.of("key", "value2"));
    assertThat(monitor.detectConvergence(caseId, "binding1", config)).isEmpty();
  }

  @Test
  void detects_identical_outputs() {
    var caseId = UUID.randomUUID();
    var config = new OutputConvergenceConfig(0.9, 3, 10);
    var output = Map.<String, Object>of("result", "same", "score", 42);
    monitor.recordOutput(caseId, "binding1", "agent1", output);
    monitor.recordOutput(caseId, "binding1", "agent2", output);
    monitor.recordOutput(caseId, "binding1", "agent3", output);
    var result = monitor.detectConvergence(caseId, "binding1", config);
    assertThat(result).isPresent();
    assertThat(result.get().averageJaccard()).isEqualTo(1.0);
  }

  @Test
  void no_detection_for_diverse_outputs() {
    var caseId = UUID.randomUUID();
    var config = new OutputConvergenceConfig(0.9, 3, 10);
    monitor.recordOutput(caseId, "binding1", "agent1", Map.of("a", 1));
    monitor.recordOutput(caseId, "binding1", "agent2", Map.of("b", 2));
    monitor.recordOutput(caseId, "binding1", "agent3", Map.of("c", 3));
    assertThat(monitor.detectConvergence(caseId, "binding1", config)).isEmpty();
  }

  @Test
  void evict_clears_state() {
    var caseId = UUID.randomUUID();
    monitor.recordOutput(caseId, "binding1", "agent1", Map.of("key", "val"));
    monitor.evictByCase(caseId);
    var config = new OutputConvergenceConfig(0.9, 2, 10);
    monitor.recordOutput(caseId, "binding1", "agent2", Map.of("key", "val"));
    assertThat(monitor.detectConvergence(caseId, "binding1", config)).isEmpty();
  }

  @Test
  void window_evicts_old_entries() {
    var caseId = UUID.randomUUID();
    var config = new OutputConvergenceConfig(0.9, 3, 3);
    monitor.recordOutput(caseId, "b", "a1", Map.of("old", 1));
    monitor.recordOutput(caseId, "b", "a2", Map.of("same", 1));
    monitor.recordOutput(caseId, "b", "a3", Map.of("same", 1));
    monitor.recordOutput(caseId, "b", "a4", Map.of("same", 1));
    var result = monitor.detectConvergence(caseId, "b", config);
    assertThat(result).isPresent();
  }

  @Test
  void cross_binding_isolation() {
    var caseId = UUID.randomUUID();
    var config = new OutputConvergenceConfig(0.9, 3, 10);
    var output = Map.<String, Object>of("key", "same");
    monitor.recordOutput(caseId, "binding1", "a1", output);
    monitor.recordOutput(caseId, "binding1", "a2", output);
    monitor.recordOutput(caseId, "binding2", "a3", output);
    assertThat(monitor.detectConvergence(caseId, "binding1", config)).isEmpty();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime-core -Dtest=OutputConvergenceMonitorTest -q 2>&1 | tail -5`
Expected: FAIL — class not found

- [ ] **Step 3: Implement OutputConvergenceMonitor**

```java
package io.casehub.engine.internal.convergence;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.convergence.OutputConvergenceConfig;
import io.casehub.engine.common.spi.Resettable;
import jakarta.annotation.Nullable;
import jakarta.enterprise.context.ApplicationScoped;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class OutputConvergenceMonitor implements Resettable {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, List<OutputFingerprint>>> state =
      new ConcurrentHashMap<>();

  public void recordOutput(UUID caseId, String bindingName, String executorName,
      Map<String, Object> output) {
    var perCase = state.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>());
    var window = perCase.computeIfAbsent(bindingName, k -> new ArrayList<>());
    synchronized (window) {
      window.add(new OutputFingerprint(executorName, output.keySet(), hashValues(output)));
    }
  }

  public Optional<ConvergenceResult> detectConvergence(
      UUID caseId, String bindingName, @Nullable OutputConvergenceConfig config) {
    if (config == null) return Optional.empty();
    var perCase = state.get(caseId);
    if (perCase == null) return Optional.empty();
    var window = perCase.get(bindingName);
    if (window == null) return Optional.empty();

    List<OutputFingerprint> snapshot;
    synchronized (window) {
      while (window.size() > config.outputWindowSize()) window.remove(0);
      snapshot = List.copyOf(window);
    }

    if (snapshot.size() < config.convergenceMinSamples()) return Optional.empty();

    double totalJaccard = 0;
    int pairs = 0;
    int matchingValueCount = 0;
    Set<String> agents = new HashSet<>();

    for (int i = 0; i < snapshot.size(); i++) {
      agents.add(snapshot.get(i).executorName);
      for (int j = i + 1; j < snapshot.size(); j++) {
        var a = snapshot.get(i);
        var b = snapshot.get(j);
        double jaccard = jaccard(a.keySet, b.keySet);
        totalJaccard += jaccard;
        pairs++;
        if (jaccard >= config.convergenceThreshold() && a.valueHashes.equals(b.valueHashes)) {
          matchingValueCount++;
        }
      }
    }

    double avgJaccard = pairs > 0 ? totalJaccard / pairs : 0;
    if (avgJaccard >= config.convergenceThreshold() && matchingValueCount > 0) {
      return Optional.of(new ConvergenceResult(
          avgJaccard, matchingValueCount, snapshot.size(), List.copyOf(agents)));
    }
    return Optional.empty();
  }

  public void evictByCase(UUID caseId) { state.remove(caseId); }

  @Override
  public void reset() { state.clear(); }

  private static double jaccard(Set<String> a, Set<String> b) {
    if (a.isEmpty() && b.isEmpty()) return 1.0;
    Set<String> union = new HashSet<>(a);
    union.addAll(b);
    Set<String> intersection = new HashSet<>(a);
    intersection.retainAll(b);
    return (double) intersection.size() / union.size();
  }

  private static Map<String, String> hashValues(Map<String, Object> output) {
    Map<String, String> hashes = new TreeMap<>();
    for (var entry : output.entrySet()) {
      try {
        String json = MAPPER.writeValueAsString(entry.getValue());
        var digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(json.getBytes(StandardCharsets.UTF_8));
        hashes.put(entry.getKey(), Base64.getEncoder().encodeToString(hash));
      } catch (Exception e) {
        hashes.put(entry.getKey(), String.valueOf(entry.getValue().hashCode()));
      }
    }
    return hashes;
  }

  record OutputFingerprint(String executorName, Set<String> keySet, Map<String, String> valueHashes) {
    OutputFingerprint {
      keySet = Set.copyOf(keySet);
      valueHashes = Map.copyOf(valueHashes);
    }
  }

  public record ConvergenceResult(
      double averageJaccard, int matchingOutputCount, int totalSamples, List<String> affectedAgents) {}
}
```

- [ ] **Step 4: Wire into WorkflowExecutionCompletedHandler**

Modify `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`:

1. Inject `Instance<OutputConvergenceMonitor> outputConvergenceMonitor`
2. In the success path (`recordSuccessOutcome()`), after output is applied to context:
```java
if (outputConvergenceMonitor.isResolvable()) {
  var monitor = outputConvergenceMonitor.get();
  monitor.recordOutput(caseInstance.getUuid(), bindingName, worker.name(), outputData);
  var config = definition.getOutputConvergenceConfig();
  monitor.detectConvergence(caseInstance.getUuid(), bindingName, config)
      .ifPresent(result -> writeOutputConvergenceEvent(caseInstance, bindingName, result));
}
```

- [ ] **Step 5: Wire eviction into CaseStatusChangedHandler**

Add `outputConvergenceMonitor.evictByCase(caseId)` to the terminal state cleanup section (inject via `Instance<>` with `isResolvable()` guard).

- [ ] **Step 6: Run all tests and commit**

Run: `mvn test -pl runtime-core -Dtest=OutputConvergenceMonitorTest -q 2>&1 | tail -5`
Expected: All PASS

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/convergence/OutputConvergenceMonitor.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java runtime-core/src/test/java/io/casehub/engine/internal/convergence/
git commit -m "feat: add OutputConvergenceMonitor with pipeline wiring Refs #1110"
```

### Task 5: YAML schema + converter + CLAUDE.md update

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java`
- Modify: `CLAUDE.md` (convergence section)
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperConvergenceTest.java`

**Interfaces:**
- Consumes: `BudgetConfig`, `ConvergenceThresholdConfig`, `OutputConvergenceConfig`, `CaseDefinition.Builder` setters
- Produces: YAML parsing for `budgetConfig:`, `convergenceThresholdConfig:`, `outputConvergenceConfig:` blocks

- [ ] **Step 1: Write YAML parsing test**

```java
package io.casehub.api.model.converter;

import static org.assertj.core.api.Assertions.*;
import io.casehub.api.model.CaseDefinition;
import java.io.IOException;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperConvergenceTest {

  @Test
  void budgetConfig_parsed_from_yaml() throws IOException {
    String yaml = """
        spec:
          name: test-case
          budgetConfig:
            maxDispatches: 1000
            maxSignalDeposits: 5000
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getBudgetConfig()).isNotNull();
    assertThat(def.getBudgetConfig().maxDispatches()).isEqualTo(1000);
    assertThat(def.getBudgetConfig().maxSignalDeposits()).isEqualTo(5000);
    assertThat(def.getBudgetConfig().maxContextMutations()).isNull();
  }

  @Test
  void convergenceThresholdConfig_parsed_from_yaml() throws IOException {
    String yaml = """
        spec:
          name: test-case
          convergenceThresholdConfig:
            dispatchRateThreshold: 0.2
            stabilityWindow: PT30S
            rateWindow: PT60S
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getConvergenceThresholdConfig()).isNotNull();
    assertThat(def.getConvergenceThresholdConfig().dispatchRateThreshold()).isEqualTo(0.2);
    assertThat(def.getConvergenceThresholdConfig().stabilityWindow()).isEqualTo(Duration.ofSeconds(30));
  }

  @Test
  void outputConvergenceConfig_parsed_from_yaml() throws IOException {
    String yaml = """
        spec:
          name: test-case
          outputConvergenceConfig:
            convergenceThreshold: 0.8
            convergenceMinSamples: 5
            outputWindowSize: 20
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getOutputConvergenceConfig()).isNotNull();
    assertThat(def.getOutputConvergenceConfig().convergenceThreshold()).isEqualTo(0.8);
  }

  @Test
  void all_configs_null_when_absent() throws IOException {
    String yaml = """
        spec:
          name: test-case
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getBudgetConfig()).isNull();
    assertThat(def.getConvergenceThresholdConfig()).isNull();
    assertThat(def.getOutputConvergenceConfig()).isNull();
  }
}
```

- [ ] **Step 2: Add properties to CaseDefinition.yaml schema**

Add `budgetConfig`, `convergenceThresholdConfig`, `outputConvergenceConfig` properties to the `CaseDefinitionSpec` in the YAML schema. Follow the existing `signalConfig` pattern for structure.

- [ ] **Step 3: Add conversion logic to YamlCaseDefinitionConverter**

Add `convertBudgetConfig()`, `convertConvergenceThresholdConfig()`, `convertOutputConvergenceConfig()` methods following the existing `convertSignalConfig()` pattern. Wire them into the main conversion method.

- [ ] **Step 4: Run YAML converter tests**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperConvergenceTest -q 2>&1 | tail -5`
Expected: All PASS

- [ ] **Step 5: Update CLAUDE.md**

Add a new section to CLAUDE.md documenting the convergence infrastructure:
- `## Convergence Detection & Termination` section covering ActivityTracker, BudgetEnforcer, ConvergenceDetector, OutputConvergenceMonitor
- Config records and their YAML blocks
- Event types and their metadata
- The `_converged` goal convention
- Pipeline integration (5th phase after localRules)

- [ ] **Step 6: Run full build and commit**

Run: `mvn clean test -q 2>&1 | tail -10`
Expected: All PASS

```bash
git add schema/src/main/resources/schema/CaseDefinition.yaml api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperConvergenceTest.java CLAUDE.md
git commit -m "feat: add YAML schema + converter + docs for convergence config Closes #1110"
```

## References

- [2026-09-17-convergence-detection-termination-design.md] — design spec this plan implements
- [SignalConfig.java:20] — config record pattern
- [SignalRegistry.java:30] — per-case ConcurrentHashMap + Resettable pattern
- [CaseContextChangedEventHandler.java:219-266] — evaluation pipeline structure
- [CaseStatusChangedHandler.java:53] — terminal state cleanup
- [WorkflowExecutionCompletedHandler.java:81] — worker completion handler
- [GoalReachedEvent.java:22] — goal reached event record
- [CaseHubEventType.java:18] — event type enum
- [CaseDefinition.java:36] — case definition builder
- [YamlCaseDefinitionConverter.java:136] — YAML converter
- [Resettable.java:24] — reset interface
- [D45-D56] — design decisions
- [GitHub #1110] — focal issue
- [GitHub #1104] — Hive Mind epic
