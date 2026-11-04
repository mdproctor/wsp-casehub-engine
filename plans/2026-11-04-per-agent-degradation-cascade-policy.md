# Per-Agent Degradation + Cascade Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #383 — Per-agent degradation + topology-owned cascade policy
**Issue group:** #377 (epic), #383

**Goal:** When a decision loop's budget or latency threshold is hit, switch
from LLM reasoning to heuristic fallback. When enough loops degrade, the
topology responds with a one-way structural cascade.

**Architecture:** Composable `DegradationTrigger` SPI with adapter pattern
for budget and latency signals. `DegradationAwareSummariser` decorator in
the existing summariser chain delegates to heuristic or LLM per call.
Push-model cascade tracking via `SwarmDegradedEvent` to a new
`CoordinationTopology.onDegradation()` method. Each topology implements
its own one-way cascade response.

**Tech Stack:** Java 21, JUnit 5, AssertJ, agency-core module

## Global Constraints

- All new classes in `io.casehub.agency.orchestration.degradation`
- Source: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/`
- Tests: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/`
- `ContentSummariser<T, R>` signature: `CompletionStage<R> summarise(List<T> items, @Nullable R previous)`
- Test pattern: JUnit 5 + AssertJ, `StubSummariser` inner class, `CompletableFuture.get()` for sync assertions
- Use `ide_insert_member` for new methods, `ide_replace_member` for modifying existing ones
- Use `ide_refactor_rename` for renames, never bash
- Build: `mvn -pl agency-core test -q` from `/Users/mdproctor/claude/casehub/slots/220/agency`

---

## Batch 1: Foundation SPI

### Task 1: DegradationTrigger + BudgetDegradationTrigger + CompositeDegradationTrigger + CascadeDegradationTrigger

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/DegradationTrigger.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/BudgetDegradationTrigger.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/CompositeDegradationTrigger.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/CascadeDegradationTrigger.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/DegradationTriggerTest.java`

**Interfaces:**
- Produces: `DegradationTrigger` — `@FunctionalInterface boolean shouldDegrade()`
- Produces: `BudgetDegradationTrigger(AgentBudgetView view)` — `shouldDegrade()` delegates to `view.isBudgetExhausted()`
- Produces: `CompositeDegradationTrigger(List<DegradationTrigger> triggers)` — OR semantics
- Produces: `CascadeDegradationTrigger` — `void activate()`, `shouldDegrade()` reads `volatile boolean`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.AgentBudgetView;
import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.budget.AutonomyBudgetPool;
import io.casehub.agency.orchestration.budget.BudgetPriority;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class DegradationTriggerTest {

    private AutonomyBudgetPool pool(int maxCalls) {
        return new AutonomyBudgetPool(
            new AutonomyBudgetPolicy(maxCalls, Long.MAX_VALUE, 0.0, maxCalls, Long.MAX_VALUE), "test");
    }

    @Test
    void budgetTrigger_doesNotFire_whenBudgetAvailable() {
        var p = pool(10);
        var view = p.createView("agent-1", BudgetPriority.NORMAL);
        var trigger = new BudgetDegradationTrigger(view);
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void budgetTrigger_fires_whenBudgetExhausted() {
        var p = pool(1);
        var view = p.createView("agent-1", BudgetPriority.NORMAL);
        view.tryAcquire();
        var trigger = new BudgetDegradationTrigger(view);
        assertThat(trigger.shouldDegrade()).isTrue();
    }

    @Test
    void composite_doesNotFire_whenNoTriggersFire() {
        var trigger = new CompositeDegradationTrigger(List.of(() -> false, () -> false));
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void composite_fires_whenAnyTriggerFires() {
        var trigger = new CompositeDegradationTrigger(List.of(() -> false, () -> true));
        assertThat(trigger.shouldDegrade()).isTrue();
    }

    @Test
    void composite_emptyList_doesNotFire() {
        var trigger = new CompositeDegradationTrigger(List.of());
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void cascadeTrigger_doesNotFire_beforeActivation() {
        var trigger = new CascadeDegradationTrigger();
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void cascadeTrigger_fires_afterActivation() {
        var trigger = new CascadeDegradationTrigger();
        trigger.activate();
        assertThat(trigger.shouldDegrade()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=DegradationTriggerTest -q` from `/Users/mdproctor/claude/casehub/slots/220/agency`
Expected: compilation failure — classes don't exist yet

- [ ] **Step 3: Implement DegradationTrigger**

```java
package io.casehub.agency.orchestration.degradation;

@FunctionalInterface
public interface DegradationTrigger {
    boolean shouldDegrade();
}
```

- [ ] **Step 4: Implement BudgetDegradationTrigger**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.AgentBudgetView;
import java.util.Objects;

public class BudgetDegradationTrigger implements DegradationTrigger {

    private final AgentBudgetView view;

    public BudgetDegradationTrigger(AgentBudgetView view) {
        this.view = Objects.requireNonNull(view, "view must not be null");
    }

    @Override
    public boolean shouldDegrade() {
        return view.isBudgetExhausted();
    }
}
```

- [ ] **Step 5: Implement CompositeDegradationTrigger**

```java
package io.casehub.agency.orchestration.degradation;

import java.util.List;
import java.util.Objects;

public class CompositeDegradationTrigger implements DegradationTrigger {

    private final List<DegradationTrigger> triggers;

    public CompositeDegradationTrigger(List<DegradationTrigger> triggers) {
        this.triggers = List.copyOf(Objects.requireNonNull(triggers));
    }

    @Override
    public boolean shouldDegrade() {
        return triggers.stream().anyMatch(DegradationTrigger::shouldDegrade);
    }
}
```

- [ ] **Step 6: Implement CascadeDegradationTrigger**

```java
package io.casehub.agency.orchestration.degradation;

public class CascadeDegradationTrigger implements DegradationTrigger {

    private volatile boolean activated;

    public void activate() {
        activated = true;
    }

    @Override
    public boolean shouldDegrade() {
        return activated;
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest=DegradationTriggerTest -q` from `/Users/mdproctor/claude/casehub/slots/220/agency`
Expected: all 7 tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add DegradationTrigger SPI with budget, composite, and cascade triggers Refs #383"
```

### Task 2: DegradationPolicy + CascadeThresholdType

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/DegradationPolicy.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/CascadeThresholdType.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/DegradationPolicyTest.java`

**Interfaces:**
- Consumes: `BudgetPriority` from `io.casehub.agency.orchestration.budget`
- Produces: `DegradationPolicy` record — `thresholdType`, `cascadeThreshold`, `priorityWeights`, `latencyBreachWindow`, `latencyBreachCount`, `latencyBreachWindowDuration`
- Produces: `CascadeThresholdType` enum — `SIMPLE_FRACTION`, `PRIORITY_WEIGHTED`
- Produces: `DegradationPolicy.defaults()` — factory method returning sensible defaults

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class DegradationPolicyTest {

    @Test
    void defaults_returnsValidPolicy() {
        var policy = DegradationPolicy.defaults();
        assertThat(policy.thresholdType()).isEqualTo(CascadeThresholdType.SIMPLE_FRACTION);
        assertThat(policy.cascadeThreshold()).isEqualTo(0.5);
        assertThat(policy.latencyBreachWindow()).isEqualTo(5);
        assertThat(policy.latencyBreachCount()).isEqualTo(3);
        assertThat(policy.latencyBreachWindowDuration()).isEqualTo(Duration.ofMinutes(2));
        assertThat(policy.priorityWeights()).containsEntry(BudgetPriority.RESERVED, 2.0);
        assertThat(policy.priorityWeights()).containsEntry(BudgetPriority.NORMAL, 1.0);
    }

    @Test
    void rejects_zeroCascadeThreshold() {
        assertThatThrownBy(() -> new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.0,
            Map.of(), 5, 3, Duration.ofMinutes(2)))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejects_negativeBreachWindowDuration() {
        assertThatThrownBy(() -> new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.5,
            Map.of(), 5, 3, Duration.ofMinutes(-1)))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejects_breachCountExceedingWindow() {
        assertThatThrownBy(() -> new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.5,
            Map.of(), 3, 5, Duration.ofMinutes(2)))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=DegradationPolicyTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement CascadeThresholdType**

```java
package io.casehub.agency.orchestration.degradation;

public enum CascadeThresholdType {
    SIMPLE_FRACTION,
    PRIORITY_WEIGHTED
}
```

- [ ] **Step 4: Implement DegradationPolicy**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;

import java.time.Duration;
import java.util.Map;
import java.util.Objects;

public record DegradationPolicy(
    CascadeThresholdType thresholdType,
    double cascadeThreshold,
    Map<BudgetPriority, Double> priorityWeights,
    int latencyBreachWindow,
    int latencyBreachCount,
    Duration latencyBreachWindowDuration
) {
    public DegradationPolicy {
        Objects.requireNonNull(thresholdType, "thresholdType must not be null");
        if (cascadeThreshold <= 0.0 || cascadeThreshold > 1.0)
            throw new IllegalArgumentException("cascadeThreshold must be in (0.0, 1.0]");
        Objects.requireNonNull(priorityWeights, "priorityWeights must not be null");
        priorityWeights = Map.copyOf(priorityWeights);
        if (latencyBreachWindow < 1)
            throw new IllegalArgumentException("latencyBreachWindow must be >= 1");
        if (latencyBreachCount < 1 || latencyBreachCount > latencyBreachWindow)
            throw new IllegalArgumentException(
                "latencyBreachCount must be in [1, latencyBreachWindow]");
        Objects.requireNonNull(latencyBreachWindowDuration,
            "latencyBreachWindowDuration must not be null");
        if (latencyBreachWindowDuration.isNegative() || latencyBreachWindowDuration.isZero())
            throw new IllegalArgumentException(
                "latencyBreachWindowDuration must be positive");
    }

    public static DegradationPolicy defaults() {
        return new DegradationPolicy(
            CascadeThresholdType.SIMPLE_FRACTION, 0.5,
            Map.of(BudgetPriority.RESERVED, 2.0, BudgetPriority.NORMAL, 1.0),
            5, 3, Duration.ofMinutes(2));
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest=DegradationPolicyTest -q`
Expected: all 4 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add DegradationPolicy config record + CascadeThresholdType enum Refs #383"
```

---

## Batch 2: Cascade Thresholds

### Task 3: CascadeThreshold + SimpleFractionThreshold + PriorityWeightedThreshold

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/CascadeThreshold.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/SimpleFractionThreshold.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/PriorityWeightedThreshold.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/CascadeThresholdTest.java`

**Interfaces:**
- Consumes: `BudgetPriority` from `io.casehub.agency.orchestration.budget`
- Produces: `CascadeThreshold` — `boolean isBreached(Set<String> degradedLoopIds, int totalLoops, Map<String, BudgetPriority> loopPriorities)`
- Produces: `SimpleFractionThreshold(double fraction)` — `degradedLoopIds.size() / totalLoops > fraction`
- Produces: `PriorityWeightedThreshold(double fraction, Map<BudgetPriority, Double> weights)` — weighted sum comparison

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class CascadeThresholdTest {

    @Test
    void simpleFraction_notBreached_belowThreshold() {
        var threshold = new SimpleFractionThreshold(0.5);
        var result = threshold.isBreached(Set.of("loop-1"), 3,
            Map.of("loop-1", BudgetPriority.NORMAL, "loop-2", BudgetPriority.NORMAL, "loop-3", BudgetPriority.NORMAL));
        assertThat(result).isFalse();
    }

    @Test
    void simpleFraction_breached_atThreshold() {
        var threshold = new SimpleFractionThreshold(0.5);
        var result = threshold.isBreached(Set.of("loop-1", "loop-2"), 3,
            Map.of("loop-1", BudgetPriority.NORMAL, "loop-2", BudgetPriority.NORMAL, "loop-3", BudgetPriority.NORMAL));
        assertThat(result).isTrue();
    }

    @Test
    void simpleFraction_notBreached_zeroTotalLoops() {
        var threshold = new SimpleFractionThreshold(0.5);
        assertThat(threshold.isBreached(Set.of(), 0, Map.of())).isFalse();
    }

    @Test
    void priorityWeighted_reservedCountsMore() {
        var weights = Map.of(BudgetPriority.RESERVED, 2.0, BudgetPriority.NORMAL, 1.0);
        var threshold = new PriorityWeightedThreshold(0.5, weights);
        var priorities = Map.of("coord", BudgetPriority.RESERVED, "w1", BudgetPriority.NORMAL, "w2", BudgetPriority.NORMAL);
        // total weight = 2+1+1 = 4, degraded weight = 2 (reserved), ratio = 0.5 >= 0.5
        assertThat(threshold.isBreached(Set.of("coord"), 3, priorities)).isTrue();
    }

    @Test
    void priorityWeighted_normalOnlyNotEnough() {
        var weights = Map.of(BudgetPriority.RESERVED, 2.0, BudgetPriority.NORMAL, 1.0);
        var threshold = new PriorityWeightedThreshold(0.5, weights);
        var priorities = Map.of("coord", BudgetPriority.RESERVED, "w1", BudgetPriority.NORMAL, "w2", BudgetPriority.NORMAL);
        // total weight = 4, degraded weight = 1 (one normal), ratio = 0.25 < 0.5
        assertThat(threshold.isBreached(Set.of("w1"), 3, priorities)).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=CascadeThresholdTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement CascadeThreshold**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;

import java.util.Map;
import java.util.Set;

public interface CascadeThreshold {
    boolean isBreached(Set<String> degradedLoopIds, int totalLoops,
                       Map<String, BudgetPriority> loopPriorities);
}
```

- [ ] **Step 4: Implement SimpleFractionThreshold**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;

import java.util.Map;
import java.util.Set;

public class SimpleFractionThreshold implements CascadeThreshold {

    private final double fraction;

    public SimpleFractionThreshold(double fraction) {
        if (fraction <= 0.0 || fraction > 1.0)
            throw new IllegalArgumentException("fraction must be in (0.0, 1.0]");
        this.fraction = fraction;
    }

    @Override
    public boolean isBreached(Set<String> degradedLoopIds, int totalLoops,
                              Map<String, BudgetPriority> loopPriorities) {
        if (totalLoops == 0) return false;
        return (double) degradedLoopIds.size() / totalLoops >= fraction;
    }
}
```

- [ ] **Step 5: Implement PriorityWeightedThreshold**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetPriority;

import java.util.Map;
import java.util.Objects;
import java.util.Set;

public class PriorityWeightedThreshold implements CascadeThreshold {

    private final double fraction;
    private final Map<BudgetPriority, Double> weights;

    public PriorityWeightedThreshold(double fraction, Map<BudgetPriority, Double> weights) {
        if (fraction <= 0.0 || fraction > 1.0)
            throw new IllegalArgumentException("fraction must be in (0.0, 1.0]");
        this.fraction = fraction;
        this.weights = Map.copyOf(Objects.requireNonNull(weights));
    }

    @Override
    public boolean isBreached(Set<String> degradedLoopIds, int totalLoops,
                              Map<String, BudgetPriority> loopPriorities) {
        if (totalLoops == 0) return false;
        double totalWeight = loopPriorities.values().stream()
            .mapToDouble(p -> weights.getOrDefault(p, 1.0)).sum();
        if (totalWeight == 0.0) return false;
        double degradedWeight = degradedLoopIds.stream()
            .filter(loopPriorities::containsKey)
            .mapToDouble(id -> weights.getOrDefault(loopPriorities.get(id), 1.0)).sum();
        return degradedWeight / totalWeight >= fraction;
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest=CascadeThresholdTest -q`
Expected: all 5 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add CascadeThreshold SPI with fraction and priority-weighted impls Refs #383"
```

---

## Batch 3: Heuristic Summarisers

### Task 4: NoOpHeuristicSummariser + HeuristicRoutingSummariser

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/NoOpHeuristicSummariser.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/HeuristicRoutingSummariser.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/HeuristicSummariserTest.java`

**Interfaces:**
- Consumes: `ContentSummariser<T, R>` from `io.casehub.agency.summarisation`
- Consumes: `SwarmFallbackStrategy.fallbackRouting(List<RoutingCandidate>)` → `RoutingDecision`
- Consumes: `DecisionOutcome.Route(RoutingDecision)` from `io.casehub.agency.orchestration.loop`
- Produces: `NoOpHeuristicSummariser<T, R>` — returns `previous` (pass-through)
- Produces: `HeuristicRoutingSummariser` — `ContentSummariser<ObservationEvent, DecisionOutcome>` wrapping `fallbackRouting()`

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.core.AgentRef;
import io.casehub.agency.core.RoutingCandidate;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.routing.RoutingDecision;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class HeuristicSummariserTest {

    @Test
    void noOp_returnsPrevious() throws Exception {
        var summariser = new NoOpHeuristicSummariser<String, String>();
        var result = summariser.summarise(List.of("input"), "previous-value").toCompletableFuture().get();
        assertThat(result).isEqualTo("previous-value");
    }

    @Test
    void noOp_returnsNull_whenNoPrevious() throws Exception {
        var summariser = new NoOpHeuristicSummariser<String, String>();
        var result = summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(result).isNull();
    }

    @Test
    void routing_producesRouteOutcome() throws Exception {
        var fallback = new SwarmFallbackStrategy();
        var candidate = RoutingCandidate.of(AgentRef.of("agent-1", "Agent One"), List.of("cap"));
        var summariser = new HeuristicRoutingSummariser(fallback, () -> List.of(candidate));

        var result = summariser.summarise(List.of(), null).toCompletableFuture().get();
        assertThat(result).isInstanceOf(DecisionOutcome.Route.class);
        var route = (DecisionOutcome.Route) result;
        assertThat(route.decision()).isInstanceOf(RoutingDecision.SwarmAssignment.class);
        var assignment = (RoutingDecision.SwarmAssignment) route.decision();
        assertThat(assignment.strategy()).isEqualTo("heuristic-all");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=HeuristicSummariserTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement NoOpHeuristicSummariser**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class NoOpHeuristicSummariser<T, R> implements ContentSummariser<T, R> {

    @Override
    public CompletionStage<R> summarise(List<T> items, @Nullable R previous) {
        return CompletableFuture.completedFuture(previous);
    }
}
```

- [ ] **Step 4: Implement HeuristicRoutingSummariser**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.core.RoutingCandidate;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.function.Supplier;

public class HeuristicRoutingSummariser implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final SwarmFallbackStrategy fallback;
    private final Supplier<List<RoutingCandidate>> candidateSupplier;

    public HeuristicRoutingSummariser(SwarmFallbackStrategy fallback,
                                      Supplier<List<RoutingCandidate>> candidateSupplier) {
        this.fallback = Objects.requireNonNull(fallback);
        this.candidateSupplier = Objects.requireNonNull(candidateSupplier);
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(List<ObservationEvent> items,
                                                       @Nullable DecisionOutcome previous) {
        var decision = fallback.fallbackRouting(candidateSupplier.get());
        return CompletableFuture.completedFuture(new DecisionOutcome.Route(decision));
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest=HeuristicSummariserTest -q`
Expected: all 3 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add NoOpHeuristicSummariser + HeuristicRoutingSummariser Refs #383"
```

### Task 5: HeuristicAggregationSummariser + HeuristicProvisioningSummariser

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/HeuristicAggregationSummariser.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/HeuristicProvisioningSummariser.java`
- Modify: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/HeuristicSummariserTest.java` (add tests)

**Interfaces:**
- Consumes: `SwarmFallbackStrategy.fallbackAggregation(List<AgentResult>)` → `AggregationResult`
- Consumes: `SwarmFallbackStrategy.fallbackProvisioningAdvice(int, int)` → `boolean`
- Consumes: `DecisionOutcome.Aggregate(AggregationResult)`, `DecisionOutcome.Provision(ProvisioningAdvice)`
- Produces: `HeuristicAggregationSummariser` — wraps `fallbackAggregation()`
- Produces: `HeuristicProvisioningSummariser` — wraps `fallbackProvisioningAdvice()`

- [ ] **Step 1: Write failing tests**

Add to `HeuristicSummariserTest.java`:

```java
@Test
void aggregation_producesAggregateOutcome() throws Exception {
    var fallback = new SwarmFallbackStrategy();
    var summariser = new HeuristicAggregationSummariser(fallback);

    var agentResult = new AgentResult(AgentRef.of("a1", "Agent 1"),
        "output text", AgentResult.Status.COMPLETED, List.of());
    var event = ObservationEvent.of("member-completed", agentResult);

    var result = summariser.summarise(List.of(event), null).toCompletableFuture().get();
    assertThat(result).isInstanceOf(DecisionOutcome.Aggregate.class);
}

@Test
void provisioning_producesProvisionOutcome() throws Exception {
    var fallback = new SwarmFallbackStrategy();
    var summariser = new HeuristicProvisioningSummariser(fallback, 3, 10);

    var result = summariser.summarise(List.of(), null).toCompletableFuture().get();
    assertThat(result).isInstanceOf(DecisionOutcome.Provision.class);
    var provision = (DecisionOutcome.Provision) result;
    assertThat(provision.advice().shouldProvision()).isTrue();
}

@Test
void provisioning_doesNotProvision_whenAtMax() throws Exception {
    var fallback = new SwarmFallbackStrategy();
    var summariser = new HeuristicProvisioningSummariser(fallback, 10, 10);

    var result = summariser.summarise(List.of(), null).toCompletableFuture().get();
    var provision = (DecisionOutcome.Provision) result;
    assertThat(provision.advice().shouldProvision()).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=HeuristicSummariserTest -q`
Expected: compilation failure

- [ ] **Step 3: Implement HeuristicAggregationSummariser**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.core.AgentResult;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class HeuristicAggregationSummariser implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final SwarmFallbackStrategy fallback;

    public HeuristicAggregationSummariser(SwarmFallbackStrategy fallback) {
        this.fallback = Objects.requireNonNull(fallback);
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(List<ObservationEvent> items,
                                                       @Nullable DecisionOutcome previous) {
        var results = items.stream()
            .filter(e -> "member-completed".equals(e.key()))
            .map(e -> (AgentResult) e.payload())
            .toList();
        var aggregation = fallback.fallbackAggregation(results);
        return CompletableFuture.completedFuture(new DecisionOutcome.Aggregate(aggregation));
    }
}
```

- [ ] **Step 4: Implement HeuristicProvisioningSummariser**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmFallbackStrategy;
import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.api.spi.stigmergy.SwarmProvisioningAdvisor.ProvisioningAdvice;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

public class HeuristicProvisioningSummariser implements ContentSummariser<ObservationEvent, DecisionOutcome> {

    private final SwarmFallbackStrategy fallback;
    private final int currentSize;
    private final int maxSize;

    public HeuristicProvisioningSummariser(SwarmFallbackStrategy fallback,
                                           int currentSize, int maxSize) {
        this.fallback = Objects.requireNonNull(fallback);
        this.currentSize = currentSize;
        this.maxSize = maxSize;
    }

    @Override
    public CompletionStage<DecisionOutcome> summarise(List<ObservationEvent> items,
                                                       @Nullable DecisionOutcome previous) {
        boolean shouldProvision = fallback.fallbackProvisioningAdvice(currentSize, maxSize);
        var advice = new ProvisioningAdvice(shouldProvision, null, null, "heuristic-fallback");
        return CompletableFuture.completedFuture(new DecisionOutcome.Provision(advice));
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest=HeuristicSummariserTest -q`
Expected: all 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add HeuristicAggregation + HeuristicProvisioning summarisers Refs #383"
```

---

## Batch 4: Core Decorator + Factory Wiring

### Task 6: DegradationAwareSummariser + LatencyDegradationTrigger

**Files:**
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/DegradationAwareSummariser.java`
- Create: `agency-core/src/main/java/io/casehub/agency/orchestration/degradation/LatencyDegradationTrigger.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/DegradationAwareSummariserTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/LatencyDegradationTriggerTest.java`

**Interfaces:**
- Consumes: `DegradationTrigger` (from Task 1), `ContentSummariser<T, R>`, `SwarmDegradedEvent`
- Consumes: `BudgetTelemetryEvent.Outcome` (from existing budget package), `LlmInvocationPolicy`
- Produces: `DegradationAwareSummariser<T, R>(delegate, heuristic, trigger, eventSink, loopName)` — decorator
- Produces: `LatencyDegradationTrigger(LlmInvocationPolicy, int windowSize, int breachCount, Duration windowDuration)` — implements `DegradationTrigger` and `Consumer<BudgetTelemetryEvent>`

- [ ] **Step 1: Write DegradationAwareSummariser failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.swarm.SwarmDegradedEvent;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import static org.assertj.core.api.Assertions.assertThat;

class DegradationAwareSummariserTest {

    static class StubSummariser implements ContentSummariser<String, String> {
        int callCount = 0;
        @Override
        public CompletionStage<String> summarise(List<String> items, @Nullable String previous) {
            callCount++;
            return CompletableFuture.completedFuture("llm-" + callCount);
        }
    }

    static class HeuristicStub implements ContentSummariser<String, String> {
        int callCount = 0;
        @Override
        public CompletionStage<String> summarise(List<String> items, @Nullable String previous) {
            callCount++;
            return CompletableFuture.completedFuture("heuristic-" + callCount);
        }
    }

    @Test
    void delegatesToLlm_whenNotDegraded() throws Exception {
        var llm = new StubSummariser();
        var heuristic = new HeuristicStub();
        var events = new ArrayList<SwarmDegradedEvent>();
        var summariser = new DegradationAwareSummariser<>(llm, heuristic, () -> false, events::add, "test-loop");

        var result = summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(result).isEqualTo("llm-1");
        assertThat(heuristic.callCount).isZero();
        assertThat(events).isEmpty();
    }

    @Test
    void delegatesToHeuristic_whenDegraded() throws Exception {
        var llm = new StubSummariser();
        var heuristic = new HeuristicStub();
        var events = new ArrayList<SwarmDegradedEvent>();
        var summariser = new DegradationAwareSummariser<>(llm, heuristic, () -> true, events::add, "test-loop");

        var result = summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(result).isEqualTo("heuristic-1");
        assertThat(llm.callCount).isZero();
    }

    @Test
    void emitsDegradationEvent_onTransition() throws Exception {
        var llm = new StubSummariser();
        var heuristic = new HeuristicStub();
        var events = new ArrayList<SwarmDegradedEvent>();
        boolean[] degraded = {false};
        var summariser = new DegradationAwareSummariser<>(llm, heuristic, () -> degraded[0], events::add, "test-loop");

        summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(events).isEmpty();

        degraded[0] = true;
        summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(events).hasSize(1);
        assertThat(events.get(0).fallbackActive()).isTrue();
        assertThat(events.get(0).capability()).isEqualTo("test-loop");
    }

    @Test
    void emitsRecoveryEvent_onRecovery() throws Exception {
        var llm = new StubSummariser();
        var heuristic = new HeuristicStub();
        var events = new ArrayList<SwarmDegradedEvent>();
        boolean[] degraded = {true};
        var summariser = new DegradationAwareSummariser<>(llm, heuristic, () -> degraded[0], events::add, "test-loop");

        summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(events).hasSize(1);
        assertThat(events.get(0).fallbackActive()).isTrue();

        degraded[0] = false;
        summariser.summarise(List.of("input"), null).toCompletableFuture().get();
        assertThat(events).hasSize(2);
        assertThat(events.get(1).fallbackActive()).isFalse();
    }

    @Test
    void doesNotEmitDuplicate_whenAlreadyDegraded() throws Exception {
        var llm = new StubSummariser();
        var heuristic = new HeuristicStub();
        var events = new ArrayList<SwarmDegradedEvent>();
        var summariser = new DegradationAwareSummariser<>(llm, heuristic, () -> true, events::add, "test-loop");

        summariser.summarise(List.of("a"), null).toCompletableFuture().get();
        summariser.summarise(List.of("b"), null).toCompletableFuture().get();
        assertThat(events).hasSize(1);
    }
}
```

- [ ] **Step 2: Write LatencyDegradationTrigger failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetTelemetryEvent;
import io.casehub.agency.stigmergy.llm.LlmInvocationPolicy;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;

import static org.assertj.core.api.Assertions.assertThat;

class LatencyDegradationTriggerTest {

    private LlmInvocationPolicy policy() {
        return LlmInvocationPolicy.of(
            Duration.ofSeconds(5), Duration.ofSeconds(8), Duration.ofSeconds(10));
    }

    private BudgetTelemetryEvent.Outcome outcome(long latencyMs) {
        return new BudgetTelemetryEvent.Outcome("corr-1", "model", 100, latencyMs, Instant.now());
    }

    @Test
    void doesNotFire_withNoEvents() {
        var trigger = new LatencyDegradationTrigger(policy(), 5, 3, Duration.ofMinutes(2));
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void doesNotFire_belowBreachCount() {
        var trigger = new LatencyDegradationTrigger(policy(), 5, 3, Duration.ofMinutes(2));
        trigger.accept(outcome(11_000));
        trigger.accept(outcome(11_000));
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void fires_atBreachCount() {
        var trigger = new LatencyDegradationTrigger(policy(), 5, 3, Duration.ofMinutes(2));
        trigger.accept(outcome(11_000));
        trigger.accept(outcome(11_000));
        trigger.accept(outcome(11_000));
        assertThat(trigger.shouldDegrade()).isTrue();
    }

    @Test
    void doesNotFire_whenLatencyBelowErrorThreshold() {
        var trigger = new LatencyDegradationTrigger(policy(), 5, 3, Duration.ofMinutes(2));
        trigger.accept(outcome(5_000));
        trigger.accept(outcome(5_000));
        trigger.accept(outcome(5_000));
        assertThat(trigger.shouldDegrade()).isFalse();
    }

    @Test
    void recovers_whenEntriesExpire() throws Exception {
        var trigger = new LatencyDegradationTrigger(policy(), 5, 3, Duration.ofMillis(100));
        trigger.accept(outcome(11_000));
        trigger.accept(outcome(11_000));
        trigger.accept(outcome(11_000));
        assertThat(trigger.shouldDegrade()).isTrue();

        Thread.sleep(150);
        assertThat(trigger.shouldDegrade()).isFalse();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest="DegradationAwareSummariserTest,LatencyDegradationTriggerTest" -q`
Expected: compilation failure

- [ ] **Step 4: Implement DegradationAwareSummariser**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.swarm.SwarmDegradedEvent;
import io.casehub.agency.summarisation.ContentSummariser;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Objects;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.function.Consumer;

public class DegradationAwareSummariser<T, R> implements ContentSummariser<T, R> {

    private final ContentSummariser<T, R> delegate;
    private final ContentSummariser<T, R> heuristic;
    private final DegradationTrigger trigger;
    private final Consumer<SwarmDegradedEvent> eventSink;
    private final String loopName;
    private final AtomicBoolean previouslyDegraded = new AtomicBoolean(false);

    public DegradationAwareSummariser(ContentSummariser<T, R> delegate,
                                      ContentSummariser<T, R> heuristic,
                                      DegradationTrigger trigger,
                                      Consumer<SwarmDegradedEvent> eventSink,
                                      String loopName) {
        this.delegate = Objects.requireNonNull(delegate);
        this.heuristic = Objects.requireNonNull(heuristic);
        this.trigger = Objects.requireNonNull(trigger);
        this.eventSink = Objects.requireNonNull(eventSink);
        this.loopName = Objects.requireNonNull(loopName);
    }

    @Override
    public CompletionStage<R> summarise(List<T> items, @Nullable R previous) {
        if (trigger.shouldDegrade()) {
            if (previouslyDegraded.compareAndSet(false, true)) {
                eventSink.accept(new SwarmDegradedEvent(loopName, "degradation trigger fired", true));
            }
            return heuristic.summarise(items, previous);
        }
        if (previouslyDegraded.compareAndSet(true, false)) {
            eventSink.accept(new SwarmDegradedEvent(loopName, "degradation cleared", false));
        }
        return delegate.summarise(items, previous);
    }
}
```

- [ ] **Step 5: Implement LatencyDegradationTrigger**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.BudgetTelemetryEvent;
import io.casehub.agency.stigmergy.llm.LlmInvocationPolicy;

import java.time.Duration;
import java.time.Instant;
import java.util.Objects;
import java.util.concurrent.ConcurrentLinkedDeque;
import java.util.function.Consumer;

public class LatencyDegradationTrigger implements DegradationTrigger, Consumer<BudgetTelemetryEvent> {

    private final long errorAfterMs;
    private final int breachCount;
    private final long windowDurationMs;
    private final ConcurrentLinkedDeque<Instant> breaches = new ConcurrentLinkedDeque<>();

    public LatencyDegradationTrigger(LlmInvocationPolicy policy,
                                     int windowSize, int breachCount,
                                     Duration windowDuration) {
        Objects.requireNonNull(policy, "policy must not be null");
        this.errorAfterMs = policy.errorAfter().toMillis();
        this.breachCount = breachCount;
        this.windowDurationMs = windowDuration.toMillis();
    }

    @Override
    public void accept(BudgetTelemetryEvent event) {
        if (event instanceof BudgetTelemetryEvent.Outcome outcome) {
            if (outcome.latencyMs() > errorAfterMs) {
                breaches.addLast(outcome.timestamp());
            }
        }
    }

    @Override
    public boolean shouldDegrade() {
        pruneExpired();
        return breaches.size() >= breachCount;
    }

    private void pruneExpired() {
        Instant cutoff = Instant.now().minusMillis(windowDurationMs);
        while (!breaches.isEmpty() && breaches.peekFirst().isBefore(cutoff)) {
            breaches.pollFirst();
        }
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn -pl agency-core test -Dtest="DegradationAwareSummariserTest,LatencyDegradationTriggerTest" -q`
Expected: all 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add agency-core/src/main/java/io/casehub/agency/orchestration/degradation/ agency-core/src/test/java/io/casehub/agency/orchestration/degradation/
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add DegradationAwareSummariser decorator + LatencyDegradationTrigger Refs #383"
```

### Task 7: DecisionLoopFactory + AgentDecisionLoop + CoordinationTopology + TopologyContext changes

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/DecisionLoopFactory.java` — add heuristic, trigger, sinks to `create()`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/loop/AgentDecisionLoop.java` — remove budget guards from `tick()` and `flush()`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CoordinationTopology.java` — add `onDegradation(SwarmDegradedEvent)`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/TopologyContext.java` — add `DegradationPolicy degradationPolicy`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/DecisionLoopFactoryDegradationTest.java`

**Interfaces:**
- Consumes: `DegradationAwareSummariser` (from Task 6), `DegradationTrigger`, `SwarmDegradedEvent`
- Produces: `DecisionLoopFactory.create(descriptor, context, reasoner, heuristic, pool, trigger, degradationSink, budgetEventSink)` — extended signature
- Produces: `CoordinationTopology.onDegradation(SwarmDegradedEvent)` — new interface method
- Produces: `TopologyContext(..., DegradationPolicy degradationPolicy)` — extended record

- [ ] **Step 1: Write failing test for DecisionLoopFactory degradation wiring**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.budget.AutonomyBudgetPool;
import io.casehub.agency.orchestration.budget.BudgetPriority;
import io.casehub.agency.orchestration.budget.BudgetTelemetryEvent;
import io.casehub.agency.orchestration.loop.DecisionContext;
import io.casehub.agency.orchestration.loop.DecisionLoopDescriptor;
import io.casehub.agency.orchestration.loop.DecisionLoopFactory;
import io.casehub.agency.orchestration.loop.DecisionOutcome;
import io.casehub.agency.orchestration.loop.ObservationEvent;
import io.casehub.agency.orchestration.swarm.SwarmDegradedEvent;
import io.casehub.agency.summarisation.ContentSummariser;
import io.casehub.agency.summarisation.EventStreamBus;
import io.casehub.api.model.coordination.CommunicationMode;
import io.casehub.api.model.coordination.DecisionScope;
import io.casehub.api.model.coordination.ObservationStrategy;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import org.jspecify.annotations.Nullable;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;

import static org.assertj.core.api.Assertions.assertThat;

class DecisionLoopFactoryDegradationTest {

    static class StubReasoner implements ContentSummariser<ObservationEvent, DecisionOutcome> {
        int callCount = 0;
        @Override
        public CompletionStage<DecisionOutcome> summarise(List<ObservationEvent> items,
                                                           @Nullable DecisionOutcome previous) {
            callCount++;
            return CompletableFuture.completedFuture(
                new DecisionOutcome.WriteBack(List.of(), List.of(), List.of()));
        }
    }

    @Test
    void factoryCreate_wiresChainCorrectly() {
        var descriptor = new DecisionLoopDescriptor("test-loop",
            ObservationStrategy.EVENT_DRIVEN, DecisionScope.FULL_AUTONOMY,
            CommunicationMode.BROADCAST, BudgetPriority.NORMAL, List.of());
        var context = new DecisionContext(UUID.randomUUID(), "test-loop",
            DecisionScope.FULL_AUTONOMY, CommunicationMode.BROADCAST,
            new TopologyDescriptor(TopologyType.DISTRIBUTED, Map.of()));
        var pool = new AutonomyBudgetPool(
            new AutonomyBudgetPolicy(10, Long.MAX_VALUE, 0.0, 10, Long.MAX_VALUE), "test");
        var reasoner = new StubReasoner();
        var heuristic = new NoOpHeuristicSummariser<ObservationEvent, DecisionOutcome>();
        var degradationEvents = new ArrayList<SwarmDegradedEvent>();
        var budgetEvents = new ArrayList<BudgetTelemetryEvent>();

        var registry = new io.casehub.agency.orchestration.loop.MapPromptSectionRegistry(Map.of());
        var bus = EventStreamBus.<DecisionOutcome>create();
        var factory = new DecisionLoopFactory(registry, bus);

        var loop = factory.create(descriptor, context, reasoner, heuristic, pool,
            () -> false, degradationEvents::add, budgetEvents::add);
        assertThat(loop).isNotNull();
        assertThat(loop.descriptor()).isEqualTo(descriptor);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl agency-core test -Dtest=DecisionLoopFactoryDegradationTest -q`
Expected: compilation failure — new `create()` signature doesn't exist yet

- [ ] **Step 3: Add onDegradation() to CoordinationTopology**

Use `ide_insert_member` to add to `CoordinationTopology.java`:

```java
default void onDegradation(SwarmDegradedEvent event) {}
```

Add import for `io.casehub.agency.orchestration.swarm.SwarmDegradedEvent`.

- [ ] **Step 4: Add DegradationPolicy to TopologyContext**

Use `ide_replace_member` to replace the `TopologyContext` record body. The new record adds the `degradationPolicy` field:

```java
public record TopologyContext(
        UUID caseId,
        TopologyDescriptor descriptor,
        AgentProvider agentProvider,
        DecisionLoopFactory loopFactory,
        AutonomyBudgetPolicy budgetPolicy,
        DegradationPolicy degradationPolicy
) {
    public TopologyContext {
        Objects.requireNonNull(caseId, "caseId must not be null");
        Objects.requireNonNull(descriptor, "descriptor must not be null");
        Objects.requireNonNull(agentProvider, "agentProvider must not be null");
        Objects.requireNonNull(loopFactory, "loopFactory must not be null");
        Objects.requireNonNull(budgetPolicy, "budgetPolicy must not be null");
        Objects.requireNonNull(degradationPolicy, "degradationPolicy must not be null");
    }
}
```

Add import for `io.casehub.agency.orchestration.degradation.DegradationPolicy`.

- [ ] **Step 5: Update DecisionLoopFactory.create() signature and wiring**

Use `ide_replace_member` to replace the `create` method:

```java
public AgentDecisionLoop create(
        DecisionLoopDescriptor descriptor,
        DecisionContext context,
        ContentSummariser<ObservationEvent, DecisionOutcome> reasoner,
        ContentSummariser<ObservationEvent, DecisionOutcome> heuristic,
        AutonomyBudgetPool pool,
        DegradationTrigger trigger,
        Consumer<SwarmDegradedEvent> degradationSink,
        Consumer<BudgetTelemetryEvent> budgetEventSink) {

    var sections = sectionRegistry.resolveAll(descriptor.promptSectionNames());
    var view     = pool.createView(context.agentId(), descriptor.budgetPriority());

    ContentSummariser<ObservationEvent, DecisionOutcome> withSections =
            new SectionAwareContentSummariser(reasoner, sections, context);

    ContentSummariser<ObservationEvent, DecisionOutcome> budgetAware =
            new BudgetAwareSummariser<>(withSections, view, budgetEventSink, r -> 0L, "unknown");

    var degradationAware = new DegradationAwareSummariser<>(
            budgetAware, heuristic, trigger, degradationSink, descriptor.name());

    var gate = new InFlightGate<>(degradationAware);

    EmissionPolicy<ObservationEvent, ?> emissionPolicy = buildEmissionPolicy(
            descriptor.observationStrategy());

    var runner = SummarisationRunner.<ObservationEvent, DecisionOutcome>builder(
                                            gate.summariser().asSummariser(),
                                            outputBus,
                                            DecisionLoopLevels.DECISION)
                                    .emissionPolicy(gate.gating(emissionPolicy))
                                    .build();

    return new AgentDecisionLoop(descriptor, context, runner, view);
}
```

Add imports for `DegradationTrigger`, `DegradationAwareSummariser`, `SwarmDegradedEvent`, `BudgetTelemetryEvent`, `Consumer`.

- [ ] **Step 6: Remove budget guards from AgentDecisionLoop**

Use `ide_replace_member` to replace `tick()`:

```java
public CompletionStage<Void> tick(long now) {
    return runner.tick(now);
}
```

Use `ide_replace_member` to replace `flush()`:

```java
public CompletionStage<Void> flush() {
    return runner.flush();
}
```

- [ ] **Step 7: Fix compilation — update all callers of the old TopologyContext and DecisionLoopFactory.create() signatures**

All 6 topology `configure()` methods and all test files constructing `TopologyContext` must add the `DegradationPolicy` parameter. All calls to `factory.create(descriptor, context, reasoner, pool)` must add the 4 new parameters.

For now, wire no-op defaults in each topology (these will be replaced in Batch 5):

In each topology `configure()`:
- `DegradationPolicy` parameter comes from `context.degradationPolicy()`
- `DegradationTrigger trigger = () -> false` (no degradation)
- `ContentSummariser heuristic = new NoOpHeuristicSummariser<>()` (no-op)
- `Consumer<SwarmDegradedEvent> degradationSink = e -> {}` (discard)
- `Consumer<BudgetTelemetryEvent> budgetEventSink = e -> {}` (discard)

For `TopologyContext` callers, add `DegradationPolicy.defaults()` as the new 6th argument.

- [ ] **Step 8: Run full test suite**

Run: `mvn -pl agency-core test -q`
Expected: all existing + new tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add .
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: wire DegradationAwareSummariser into DecisionLoopFactory + remove budget guards Refs #383"
```

---

## Batch 5: Topology Wiring

### Task 8: CentralisedTopology + DistributedTopology cascade wiring

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/CentralisedTopology.java` — wire real triggers + heuristics in `configure()`, implement `onDegradation()`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/DistributedTopology.java` — wire real triggers in `configure()`, `onDegradation()` no-op
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/CentralisedTopologyCascadeTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/DistributedTopologyCascadeTest.java`

**Interfaces:**
- Consumes: All degradation types from Tasks 1-6
- Consumes: `SwarmFallbackStrategy`, `CentralisedTopology.fallbackStrategy()`
- Produces: CentralisedTopology with cascade: all-heuristic response
- Produces: DistributedTopology with cascade: no-op (independent)

- [ ] **Step 1: Write CentralisedTopology cascade test**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.budget.AutonomyBudgetPolicy;
import io.casehub.agency.orchestration.budget.AutonomyBudgetPool;
import io.casehub.agency.orchestration.budget.BudgetPriority;
import io.casehub.agency.orchestration.swarm.SwarmDegradedEvent;
import io.casehub.agency.orchestration.topology.CentralisedTopology;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CentralisedTopologyCascadeTest {

    @Test
    void onDegradation_cascadesAllHeuristic_whenThresholdBreached() {
        // Build a CentralisedTopology, configure it, then send enough
        // degradation events to cross the threshold.
        // After cascade, verify all loops' cascade triggers are activated.
        // Full wiring test — verifies the topology correctly constructs
        // triggers and responds to cascade threshold breach.
        // Implementation details depend on CentralisedTopology's cascade
        // wiring — write the full test body after the implementation is in place.
    }
}
```

Note: the test body requires constructing a full CentralisedTopology with
all its dependencies (SwarmContext, SwarmRoutingStrategy, etc.). Use the
existing `CentralisedTopologyTest` as a template for the test setup.

- [ ] **Step 2: Implement CentralisedTopology cascade wiring**

In `CentralisedTopology`:

Add fields:
```java
private final Set<String> degradedLoops = ConcurrentHashMap.newKeySet();
private final AtomicBoolean cascadeTriggered = new AtomicBoolean(false);
private CascadeThreshold cascadeThreshold;
private Map<String, BudgetPriority> loopPriorities;
private List<CascadeDegradationTrigger> cascadeTriggers;
```

In `configure()`, replace no-op defaults with real triggers per loop:

For each of the 3 loops (team-formation, collective-reasoning, provisioning):
1. Create `BudgetDegradationTrigger` from the loop's `AgentBudgetView`
2. Create `LatencyDegradationTrigger` from `context.degradationPolicy()`
3. Create `CascadeDegradationTrigger`
4. Compose into `CompositeDegradationTrigger`
5. Create the appropriate heuristic summariser (`HeuristicRoutingSummariser`, `HeuristicAggregationSummariser`, `HeuristicProvisioningSummariser`)
6. Wire the composite consumer: `event -> { this.onDegradation(event); executionEventListener.onSwarmDegraded(event); }`

Build `CascadeThreshold` from `DegradationPolicy`:
```java
this.cascadeThreshold = switch (context.degradationPolicy().thresholdType()) {
    case SIMPLE_FRACTION -> new SimpleFractionThreshold(context.degradationPolicy().cascadeThreshold());
    case PRIORITY_WEIGHTED -> new PriorityWeightedThreshold(
        context.degradationPolicy().cascadeThreshold(), context.degradationPolicy().priorityWeights());
};
```

Implement `onDegradation()`:
```java
@Override
public void onDegradation(SwarmDegradedEvent event) {
    if (event.fallbackActive()) {
        degradedLoops.add(event.capability());
    } else {
        degradedLoops.remove(event.capability());
    }
    if (event.fallbackActive() && !cascadeTriggered.get()
        && cascadeThreshold.isBreached(degradedLoops, loopPriorities.size(), loopPriorities)
        && cascadeTriggered.compareAndSet(false, true)) {
        cascadeTriggers.forEach(CascadeDegradationTrigger::activate);
    }
}
```

- [ ] **Step 3: Implement DistributedTopology cascade wiring**

In `DistributedTopology.configure()`, wire real triggers per loop:
1. `BudgetDegradationTrigger` + `LatencyDegradationTrigger` composed in `CompositeDegradationTrigger`
2. `NoOpHeuristicSummariser` as the heuristic (general reasoning loops)
3. No `CascadeDegradationTrigger` — distributed topology has no cascade

`onDegradation()` remains the default no-op.

- [ ] **Step 4: Write and run full CentralisedTopologyCascadeTest**

Complete the test from Step 1 using the real topology setup.

- [ ] **Step 5: Run tests**

Run: `mvn -pl agency-core test -Dtest="CentralisedTopologyCascadeTest,DistributedTopologyCascadeTest" -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add .
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: wire CentralisedTopology all-heuristic + DistributedTopology independent cascade Refs #383"
```

### Task 9: GraduatedTopology + GraduationState.forceScope

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduatedTopology.java` — wire triggers, implement `onDegradation()` (centralise authority)
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/GraduationState.java` — add `forceScope()` with `scopeFloor`, clamp `upgrade()` against floor
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/GraduatedTopologyCascadeTest.java`

**Interfaces:**
- Consumes: `GraduationState.scopeFor()`, `upgrade()`, `downgrade()`
- Produces: `GraduationState.forceScope(String agentId, DecisionScope scope)` — sets scope floor
- Produces: GraduatedTopology cascade: all agents forced to STIMULUS_DRIVEN

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.topology.GraduationState;
import io.casehub.api.model.coordination.DecisionScope;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GraduatedTopologyCascadeTest {

    @Test
    void forceScope_setsToStimulusDriven() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 5);
        state.forceScope("agent-1", DecisionScope.STIMULUS_DRIVEN);
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void upgrade_clampedByScopeFloor() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.FULL_AUTONOMY), 5);
        state.forceScope("agent-1", DecisionScope.STIMULUS_DRIVEN);
        state.upgrade("agent-1");
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }

    @Test
    void downgrade_worksNormally_withFloor() {
        var state = new GraduationState(
            Map.of("agent-1", DecisionScope.TASK_ADAPTATION), 5);
        state.forceScope("agent-1", DecisionScope.STIMULUS_DRIVEN);
        assertThat(state.scopeFor("agent-1")).isEqualTo(DecisionScope.STIMULUS_DRIVEN);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl agency-core test -Dtest=GraduatedTopologyCascadeTest -q`
Expected: compilation failure — `forceScope()` doesn't exist

- [ ] **Step 3: Implement GraduationState.forceScope()**

Add to `GraduationState`:

```java
private volatile DecisionScope scopeFloor;

public void forceScope(String agentId, DecisionScope scope) {
    this.scopeFloor = scope;
    agentScopes.put(agentId, scope);
}
```

Modify `upgrade()` to clamp against the floor:

```java
public void upgrade(String agentId) {
    agentScopes.compute(agentId, (id, scope) -> {
        var upgraded = switch (scope != null ? scope : DecisionScope.STIMULUS_DRIVEN) {
            case STIMULUS_DRIVEN -> DecisionScope.TASK_ADAPTATION;
            case TASK_ADAPTATION -> DecisionScope.FULL_AUTONOMY;
            case FULL_AUTONOMY -> DecisionScope.FULL_AUTONOMY;
        };
        if (scopeFloor != null && upgraded.ordinal() > scopeFloor.ordinal()) {
            return scopeFloor;
        }
        return upgraded;
    });
}
```

Note: `DecisionScope` ordinals: `STIMULUS_DRIVEN(0)`, `TASK_ADAPTATION(1)`, `FULL_AUTONOMY(2)`. The floor clamps upgrades — if the floor is STIMULUS_DRIVEN, no agent can upgrade past it. Check the actual ordinal ordering in the enum before implementing.

- [ ] **Step 4: Wire GraduatedTopology cascade**

In `GraduatedTopology`, add cascade fields and implement `onDegradation()`:

```java
@Override
public void onDegradation(SwarmDegradedEvent event) {
    if (event.fallbackActive()) {
        degradedLoops.add(event.capability());
    } else {
        degradedLoops.remove(event.capability());
    }
    if (event.fallbackActive() && !cascadeTriggered.get()
        && cascadeThreshold.isBreached(degradedLoops, loopPriorities.size(), loopPriorities)
        && cascadeTriggered.compareAndSet(false, true)) {
        for (String agentId : agentRoles.keySet()) {
            graduationState.forceScope(agentId, DecisionScope.STIMULUS_DRIVEN);
        }
    }
}
```

Wire triggers in `configure()` — same pattern as Distributed but with `CascadeDegradationTrigger` added.

- [ ] **Step 5: Run tests**

Run: `mvn -pl agency-core test -Dtest=GraduatedTopologyCascadeTest -q`
Expected: all 3 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add .
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: add GraduationState.forceScope + GraduatedTopology cascade response Refs #383"
```

### Task 10: HybridTopology + HierarchicalTopology + HiveTopology cascade wiring

**Files:**
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HybridTopology.java`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HierarchicalTopology.java`
- Modify: `agency-core/src/main/java/io/casehub/agency/orchestration/topology/HiveTopology.java` — `patternType` becomes `volatile`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/HybridTopologyCascadeTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/HierarchicalTopologyCascadeTest.java`
- Test: `agency-core/src/test/java/io/casehub/agency/orchestration/degradation/HiveTopologyCascadeTest.java`

**Interfaces:**
- Consumes: All degradation types from Tasks 1-6
- Produces: HybridTopology cascade: force-degrade NORMAL loops
- Produces: HierarchicalTopology cascade: force-degrade leaf loops
- Produces: HiveTopology cascade: simplify pattern + force-degrade workers

- [ ] **Step 1: Write failing tests**

```java
// HybridTopologyCascadeTest.java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.swarm.SwarmDegradedEvent;
import io.casehub.agency.orchestration.topology.HybridTopology;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class HybridTopologyCascadeTest {

    @Test
    void onDegradation_forceDegradesNormalLoops() {
        // After cascade threshold breach, verify NORMAL-priority
        // local loop CascadeDegradationTriggers are activated
        // while RESERVED coordinator trigger is NOT activated
    }
}
```

```java
// HierarchicalTopologyCascadeTest.java
package io.casehub.agency.orchestration.degradation;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class HierarchicalTopologyCascadeTest {

    @Test
    void onDegradation_forceDegradesLeaves() {
        // After cascade threshold breach, verify leaf loop
        // CascadeDegradationTriggers are activated while
        // root loop trigger is NOT activated
    }
}
```

```java
// HiveTopologyCascadeTest.java
package io.casehub.agency.orchestration.degradation;

import io.casehub.agency.orchestration.topology.OrchestrationPatternType;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class HiveTopologyCascadeTest {

    @Test
    void onDegradation_simplifiesPattern_debateToVoting() {
        // After cascade, verify pattern downgraded from DEBATE to VOTING
        // and worker CascadeDegradationTriggers are activated
    }

    @Test
    void onDegradation_simplifiesPattern_votingToSupervisor() {
        // After cascade when already at VOTING, verify pattern
        // downgraded to SUPERVISOR
    }
}
```

- [ ] **Step 2: Implement HybridTopology cascade**

Add cascade fields. In `onDegradation()`:
```java
@Override
public void onDegradation(SwarmDegradedEvent event) {
    if (event.fallbackActive()) {
        degradedLoops.add(event.capability());
    } else {
        degradedLoops.remove(event.capability());
    }
    if (event.fallbackActive() && !cascadeTriggered.get()
        && cascadeThreshold.isBreached(degradedLoops, loopPriorities.size(), loopPriorities)
        && cascadeTriggered.compareAndSet(false, true)) {
        normalCascadeTriggers.forEach(CascadeDegradationTrigger::activate);
    }
}
```

Wire triggers in `configure()`:
- Coordinator loop: `CompositeDegradationTrigger` with budget + latency only (no `CascadeDegradationTrigger`)
- Local loops: `CompositeDegradationTrigger` with budget + latency + `CascadeDegradationTrigger`

- [ ] **Step 3: Implement HierarchicalTopology cascade**

Same pattern as Hybrid but partitioned by tree position:
- Root loops (depth 0): no `CascadeDegradationTrigger`
- Leaf loops: `CascadeDegradationTrigger` included

`onDegradation()` activates leaf cascade triggers on threshold breach.

- [ ] **Step 4: Implement HiveTopology cascade**

Change `patternType` from `final` to `volatile`:
```java
private volatile OrchestrationPatternType patternType;
```

In `onDegradation()`:
```java
@Override
public void onDegradation(SwarmDegradedEvent event) {
    if (event.fallbackActive()) {
        degradedLoops.add(event.capability());
    } else {
        degradedLoops.remove(event.capability());
    }
    if (event.fallbackActive() && !cascadeTriggered.get()
        && cascadeThreshold.isBreached(degradedLoops, loopPriorities.size(), loopPriorities)
        && cascadeTriggered.compareAndSet(false, true)) {
        patternType = switch (patternType) {
            case DEBATE -> OrchestrationPatternType.VOTING;
            case VOTING -> OrchestrationPatternType.SUPERVISOR;
            case SUPERVISOR -> OrchestrationPatternType.SUPERVISOR;
        };
        workerCascadeTriggers.forEach(CascadeDegradationTrigger::activate);
    }
}
```

Update `onFeedback()` to read from the volatile field:
```java
@Override
public void onFeedback(CoordinationFeedback feedback) {
    patternAdapter.onFeedback(patternType, feedback);
}
```

Wire triggers in `configure()`:
- Coordinator: budget + latency only
- Workers: budget + latency + `CascadeDegradationTrigger`

- [ ] **Step 5: Complete and run all topology cascade tests**

Run: `mvn -pl agency-core test -Dtest="HybridTopologyCascadeTest,HierarchicalTopologyCascadeTest,HiveTopologyCascadeTest" -q`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `mvn -pl agency-core test -q`
Expected: all tests PASS (existing + new)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/220/agency add .
git -C /Users/mdproctor/claude/casehub/slots/220/agency commit -m "feat: wire Hybrid, Hierarchical, Hive topology cascade responses Refs #383"
```

---

## References

- [2026-11-04-per-agent-degradation-cascade-policy-design.md] — design spec this plan implements
- [decisions.md] — 11 design decisions with rationale
- `CoordinationTopology.java` — SPI interface being extended
- `BudgetAwareSummariser.java` — existing budget decorator pattern (model for `DegradationAwareSummariser`)
- `SwarmFallbackStrategy.java` — heuristic methods being adapted
- `SwarmDegradedEvent.java` — existing scaffolded event type
- `DecisionLoopFactory.java:25-53` — factory wiring point (current `create()` signature)
- `AgentDecisionLoop.java:36-51` — budget guards being removed (`tick()` and `flush()`)
- `GraduationState.java` — scope adjustment mechanism for Graduated cascade
- `HiveTopology.java:25` — `patternType` field changing from `final` to `volatile`
- `CentralisedTopology.java:70-130` — `configure()` method showing loop construction pattern
- `BudgetAwareSummariserTest.java` — test pattern template (StubSummariser, pool construction)
- GitHub casehubio/agency#383, casehubio/agency#377
