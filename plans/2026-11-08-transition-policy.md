# TransitionPolicy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1290 — TransitionPolicy — graduated topology transition model
**Issue group:** #1290

**Goal:** Model graduated topology transitions as vocabulary types + SPI extension, and remove GRADUATED from TopologyType.

**Architecture:** Three new vocabulary types in `model.coordination` (TransitionTrigger sealed interface, TransitionStrategy enum, TransitionPolicy record) plus two default methods on the existing TopologyProvider SPI. GRADUATED removed from TopologyType as the design debt cleanup that motivated this work.

**Tech Stack:** Java 21 (sealed interfaces, records, pattern matching), JUnit 5, AssertJ

## Global Constraints

- All new types in `io.casehub.api.model.coordination` (api module)
- SPI extension on `io.casehub.api.spi.coordination.TopologyProvider` (api module)
- Follow existing test conventions: package-private class, `snake_case` methods, AssertJ
- All commits reference `Refs #1290` or `Closes #1290`
- Run spotless before committing: `mvn spotless:apply -pl api -q`

---

## Batch 1: Vocabulary Types + GRADUATED Cleanup

### Task 1: TransitionTrigger sealed interface + TransitionStrategy enum

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/TransitionTrigger.java`
- Create: `api/src/main/java/io/casehub/api/model/coordination/TransitionStrategy.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/TransitionTriggerTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `TransitionTrigger` sealed interface with `Threshold(String metric, double value)`, `Time(Duration duration)`, `Convergence(double threshold)` record variants; `TransitionStrategy` enum with `IMMEDIATE`, `PHASED`

- [ ] **Step 1: Write the failing tests for TransitionTrigger variants**

Create `api/src/test/java/io/casehub/api/model/coordination/TransitionTriggerTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.time.Duration;
import org.junit.jupiter.api.Test;

class TransitionTriggerTest {

  @Test
  void threshold_valid_construction() {
    var trigger = new TransitionTrigger.Threshold("swarm.size", 5.0);
    assertThat(trigger.metric()).isEqualTo("swarm.size");
    assertThat(trigger.value()).isEqualTo(5.0);
  }

  @Test
  void threshold_null_metric_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new TransitionTrigger.Threshold(null, 5.0))
        .withMessageContaining("metric");
  }

  @Test
  void threshold_blank_metric_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TransitionTrigger.Threshold("  ", 5.0))
        .withMessageContaining("metric");
  }

  @Test
  void time_valid_construction() {
    var trigger = new TransitionTrigger.Time(Duration.ofMinutes(10));
    assertThat(trigger.duration()).isEqualTo(Duration.ofMinutes(10));
  }

  @Test
  void time_null_duration_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new TransitionTrigger.Time(null))
        .withMessageContaining("duration");
  }

  @Test
  void time_zero_duration_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TransitionTrigger.Time(Duration.ZERO))
        .withMessageContaining("duration");
  }

  @Test
  void time_negative_duration_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TransitionTrigger.Time(Duration.ofMinutes(-1)))
        .withMessageContaining("duration");
  }

  @Test
  void convergence_valid_construction() {
    var trigger = new TransitionTrigger.Convergence(0.85);
    assertThat(trigger.threshold()).isEqualTo(0.85);
  }

  @Test
  void convergence_below_zero_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TransitionTrigger.Convergence(-0.1))
        .withMessageContaining("threshold");
  }

  @Test
  void convergence_above_one_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new TransitionTrigger.Convergence(1.1))
        .withMessageContaining("threshold");
  }

  @Test
  void convergence_boundary_zero_accepted() {
    var trigger = new TransitionTrigger.Convergence(0.0);
    assertThat(trigger.threshold()).isEqualTo(0.0);
  }

  @Test
  void convergence_boundary_one_accepted() {
    var trigger = new TransitionTrigger.Convergence(1.0);
    assertThat(trigger.threshold()).isEqualTo(1.0);
  }

  @Test
  void sealed_interface_permits_three_variants() {
    assertThat(TransitionTrigger.class.getPermittedSubclasses())
        .hasSize(3)
        .extracting(Class::getSimpleName)
        .containsExactlyInAnyOrder("Threshold", "Time", "Convergence");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TransitionTriggerTest -q`
Expected: compilation error — `TransitionTrigger` does not exist

- [ ] **Step 3: Implement TransitionTrigger and TransitionStrategy**

Create `api/src/main/java/io/casehub/api/model/coordination/TransitionTrigger.java`:

```java
package io.casehub.api.model.coordination;

import java.time.Duration;
import java.util.Objects;

public sealed interface TransitionTrigger
    permits TransitionTrigger.Threshold, TransitionTrigger.Time, TransitionTrigger.Convergence {

  record Threshold(String metric, double value) implements TransitionTrigger {
    public Threshold {
      Objects.requireNonNull(metric, "metric must not be null");
      if (metric.isBlank()) throw new IllegalArgumentException("metric must not be blank");
    }
  }

  record Time(Duration duration) implements TransitionTrigger {
    public Time {
      Objects.requireNonNull(duration, "duration must not be null");
      if (duration.isNegative() || duration.isZero())
        throw new IllegalArgumentException("duration must be positive");
    }
  }

  record Convergence(double threshold) implements TransitionTrigger {
    public Convergence {
      if (threshold < 0.0 || threshold > 1.0)
        throw new IllegalArgumentException("threshold must be in [0.0, 1.0]");
    }
  }
}
```

Create `api/src/main/java/io/casehub/api/model/coordination/TransitionStrategy.java`:

```java
package io.casehub.api.model.coordination;

public enum TransitionStrategy {
  IMMEDIATE,
  PHASED
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TransitionTriggerTest -q`
Expected: all 12 tests PASS

- [ ] **Step 5: Run spotless and commit**

```bash
mvn spotless:apply -pl api -q
git add api/src/main/java/io/casehub/api/model/coordination/TransitionTrigger.java \
        api/src/main/java/io/casehub/api/model/coordination/TransitionStrategy.java \
        api/src/test/java/io/casehub/api/model/coordination/TransitionTriggerTest.java
git commit -m "feat: add TransitionTrigger sealed interface and TransitionStrategy enum

Refs #1290"
```

### Task 2: TransitionPolicy record

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/TransitionPolicy.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/TransitionPolicyTest.java`

**Interfaces:**
- Consumes: `TransitionTrigger` (sealed interface from Task 1), `TransitionStrategy` (enum from Task 1), `TopologyType` (existing enum)
- Produces: `TransitionPolicy(TopologyType from, TopologyType to, TransitionTrigger trigger, TransitionStrategy strategy, boolean reversible, @Nullable TransitionTrigger rollbackTrigger)` record

- [ ] **Step 1: Write the failing tests for TransitionPolicy**

Create `api/src/test/java/io/casehub/api/model/coordination/TransitionPolicyTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.time.Duration;
import org.junit.jupiter.api.Test;

class TransitionPolicyTest {

  private static final TransitionTrigger.Threshold SWARM_SIZE_TRIGGER =
      new TransitionTrigger.Threshold("swarm.size", 5.0);

  private static final TransitionTrigger.Convergence CONVERGENCE_TRIGGER =
      new TransitionTrigger.Convergence(0.9);

  @Test
  void valid_immediate_non_reversible() {
    var policy =
        new TransitionPolicy(
            TopologyType.CENTRALISED,
            TopologyType.DISTRIBUTED,
            SWARM_SIZE_TRIGGER,
            TransitionStrategy.IMMEDIATE,
            false,
            null);
    assertThat(policy.from()).isEqualTo(TopologyType.CENTRALISED);
    assertThat(policy.to()).isEqualTo(TopologyType.DISTRIBUTED);
    assertThat(policy.trigger()).isEqualTo(SWARM_SIZE_TRIGGER);
    assertThat(policy.strategy()).isEqualTo(TransitionStrategy.IMMEDIATE);
    assertThat(policy.reversible()).isFalse();
    assertThat(policy.rollbackTrigger()).isNull();
  }

  @Test
  void valid_phased_reversible_with_rollback() {
    var rollback = new TransitionTrigger.Convergence(0.3);
    var policy =
        new TransitionPolicy(
            TopologyType.CENTRALISED,
            TopologyType.HYBRID,
            SWARM_SIZE_TRIGGER,
            TransitionStrategy.PHASED,
            true,
            rollback);
    assertThat(policy.reversible()).isTrue();
    assertThat(policy.rollbackTrigger()).isEqualTo(rollback);
  }

  @Test
  void valid_reversible_without_rollback_trigger() {
    var policy =
        new TransitionPolicy(
            TopologyType.CENTRALISED,
            TopologyType.DISTRIBUTED,
            SWARM_SIZE_TRIGGER,
            TransitionStrategy.IMMEDIATE,
            true,
            null);
    assertThat(policy.reversible()).isTrue();
    assertThat(policy.rollbackTrigger()).isNull();
  }

  @Test
  void same_from_and_to_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    TopologyType.CENTRALISED,
                    TopologyType.CENTRALISED,
                    SWARM_SIZE_TRIGGER,
                    TransitionStrategy.IMMEDIATE,
                    false,
                    null))
        .withMessageContaining("from and to must differ");
  }

  @Test
  void rollback_trigger_without_reversible_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    TopologyType.CENTRALISED,
                    TopologyType.DISTRIBUTED,
                    SWARM_SIZE_TRIGGER,
                    TransitionStrategy.IMMEDIATE,
                    false,
                    CONVERGENCE_TRIGGER))
        .withMessageContaining("rollbackTrigger requires reversible");
  }

  @Test
  void null_from_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    null,
                    TopologyType.DISTRIBUTED,
                    SWARM_SIZE_TRIGGER,
                    TransitionStrategy.IMMEDIATE,
                    false,
                    null));
  }

  @Test
  void null_to_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    TopologyType.CENTRALISED,
                    null,
                    SWARM_SIZE_TRIGGER,
                    TransitionStrategy.IMMEDIATE,
                    false,
                    null));
  }

  @Test
  void null_trigger_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    TopologyType.CENTRALISED,
                    TopologyType.DISTRIBUTED,
                    null,
                    TransitionStrategy.IMMEDIATE,
                    false,
                    null));
  }

  @Test
  void null_strategy_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () ->
                new TransitionPolicy(
                    TopologyType.CENTRALISED,
                    TopologyType.DISTRIBUTED,
                    SWARM_SIZE_TRIGGER,
                    null,
                    false,
                    null));
  }

  @Test
  void time_trigger_with_phased_strategy() {
    var policy =
        new TransitionPolicy(
            TopologyType.HYBRID,
            TopologyType.DISTRIBUTED,
            new TransitionTrigger.Time(Duration.ofHours(1)),
            TransitionStrategy.PHASED,
            false,
            null);
    assertThat(policy.trigger()).isInstanceOf(TransitionTrigger.Time.class);
    assertThat(policy.strategy()).isEqualTo(TransitionStrategy.PHASED);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TransitionPolicyTest -q`
Expected: compilation error — `TransitionPolicy` does not exist

- [ ] **Step 3: Implement TransitionPolicy**

Create `api/src/main/java/io/casehub/api/model/coordination/TransitionPolicy.java`:

```java
package io.casehub.api.model.coordination;

import jakarta.annotation.Nullable;
import java.util.Objects;

public record TransitionPolicy(
    TopologyType from,
    TopologyType to,
    TransitionTrigger trigger,
    TransitionStrategy strategy,
    boolean reversible,
    @Nullable TransitionTrigger rollbackTrigger) {

  public TransitionPolicy {
    Objects.requireNonNull(from, "from must not be null");
    Objects.requireNonNull(to, "to must not be null");
    Objects.requireNonNull(trigger, "trigger must not be null");
    Objects.requireNonNull(strategy, "strategy must not be null");
    if (from == to)
      throw new IllegalArgumentException(
          "from and to must differ — a transition to the same topology is a no-op");
    if (!reversible && rollbackTrigger != null)
      throw new IllegalArgumentException("rollbackTrigger requires reversible=true");
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TransitionPolicyTest -q`
Expected: all 10 tests PASS

- [ ] **Step 5: Run spotless and commit**

```bash
mvn spotless:apply -pl api -q
git add api/src/main/java/io/casehub/api/model/coordination/TransitionPolicy.java \
        api/src/test/java/io/casehub/api/model/coordination/TransitionPolicyTest.java
git commit -m "feat: add TransitionPolicy record composing trigger, strategy, and rollback

Refs #1290"
```

### Task 3: TopologyProvider SPI extension + GRADUATED cleanup

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/coordination/TopologyProvider.java:21-27` — add two default methods
- Modify: `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java:71-80` — remove GRADUATED constant
- Modify: `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java:102-117` — remove 3 GRADUATED tests
- Test: `api/src/test/java/io/casehub/api/spi/coordination/TopologyProviderTest.java`

**Interfaces:**
- Consumes: `TransitionPolicy` (record from Task 2), `TopologyType` (existing enum)
- Produces: `TopologyProvider.transitionPolicy(String agentId, String tenancyId)` returning `Optional<TransitionPolicy>`, `TopologyProvider.evaluateTransition(String agentId, String tenancyId)` returning `Optional<TopologyType>`

- [ ] **Step 1: Write the failing tests for TopologyProvider default methods**

Create `api/src/test/java/io/casehub/api/spi/coordination/TopologyProviderTest.java`:

```java
package io.casehub.api.spi.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import java.util.Map;
import org.junit.jupiter.api.Test;

class TopologyProviderTest {

  private final TopologyProvider defaultProvider =
      (agentId, tenancyId) -> new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());

  @Test
  void default_transition_policy_returns_empty() {
    assertThat(defaultProvider.transitionPolicy("agent-1", "tenant-1")).isEmpty();
  }

  @Test
  void default_evaluate_transition_returns_empty() {
    assertThat(defaultProvider.evaluateTransition("agent-1", "tenant-1")).isEmpty();
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TopologyProviderTest -q`
Expected: compilation error — `transitionPolicy` method does not exist

- [ ] **Step 3: Add default methods to TopologyProvider**

In `api/src/main/java/io/casehub/api/spi/coordination/TopologyProvider.java`, add two default methods after the existing `effectiveSelfDetermination` method. The file should become:

```java
package io.casehub.api.spi.coordination;

import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import io.casehub.api.model.coordination.TransitionPolicy;
import java.util.Optional;
import java.util.OptionalDouble;

public interface TopologyProvider {
  TopologyDescriptor topologyFor(String agentId, String tenancyId);

  default OptionalDouble effectiveSelfDetermination(String agentId, String tenancyId) {
    return OptionalDouble.empty();
  }

  default Optional<TransitionPolicy> transitionPolicy(String agentId, String tenancyId) {
    return Optional.empty();
  }

  default Optional<TopologyType> evaluateTransition(String agentId, String tenancyId) {
    return Optional.empty();
  }
}
```

- [ ] **Step 4: Run TopologyProvider tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=TopologyProviderTest -q`
Expected: all 2 tests PASS

- [ ] **Step 5: Remove GRADUATED from TopologyType**

In `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java`, delete the GRADUATED constant (lines 71-80):

```java
  // DELETE this entire block:
  GRADUATED {
    @Override
    public DecisionAuthority deriveAuthority(StructuralRole role) {
      return switch (role) {
        case INITIATOR -> DecisionAuthority.CENTRAL;
        case PEER -> DecisionAuthority.COORDINATED;
        case CONTRACTOR -> DecisionAuthority.LOCAL;
      };
    }
  },
```

Adjust the trailing comma on the preceding constant — `HIERARCHICAL`'s closing brace should have a comma (connecting to `MULTI_SWARM`).

- [ ] **Step 6: Remove GRADUATED tests from TopologyTypeTest**

In `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java`, delete the three test methods (lines 101-117):

```java
  // DELETE these three methods:
  @Test
  void graduated_initiator_derives_central() { ... }

  @Test
  void graduated_peer_derives_coordinated() { ... }

  @Test
  void graduated_contractor_derives_local() { ... }
```

- [ ] **Step 7: Run full api test suite to verify nothing breaks**

First install dependencies:
Run: `mvn install -DskipTests -q`

Then run all api tests:
Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q`
Expected: all tests PASS, including the new TransitionTriggerTest, TransitionPolicyTest, TopologyProviderTest, and all existing tests

- [ ] **Step 8: Run spotless and commit**

```bash
mvn spotless:apply -pl api -q
git add api/src/main/java/io/casehub/api/spi/coordination/TopologyProvider.java \
        api/src/main/java/io/casehub/api/model/coordination/TopologyType.java \
        api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java \
        api/src/test/java/io/casehub/api/spi/coordination/TopologyProviderTest.java
git commit -m "feat: extend TopologyProvider with transition evaluation, remove GRADUATED

TopologyProvider gains transitionPolicy() and evaluateTransition() default
methods. GRADUATED removed from TopologyType — TransitionPolicy is the
canonical replacement for graduated topology transitions.

Closes #1290"
```

---

## References

- [2026-11-08-transition-policy-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/model/coordination/TopologyType.java:71-80` — GRADUATED to remove
- `api/src/test/java/io/casehub/api/model/coordination/TopologyTypeTest.java:102-117` — GRADUATED tests to remove
- `api/src/main/java/io/casehub/api/spi/coordination/TopologyProvider.java:21-27` — SPI to extend
- `api/src/main/java/io/casehub/api/model/convergence/BudgetConfig.java` — convergence threshold reference
- `api/src/main/java/io/casehub/api/spi/stigmergy/AgentWorkLifecycleListener.java:42` — onTopologyChanged hook
- `specs/issue-1282-coordination-vocabulary-types/SPEC.md` — vocabulary foundation
- GitHub #1290, #1282, #1284
