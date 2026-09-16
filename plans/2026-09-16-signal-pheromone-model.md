# Signal/Pheromone Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1106 — feat: Signal/pheromone model — temporal signals with decay & reinforcement in CaseContext
**Issue group:** #1104 (Hive Mind epic), #1105 (complete)

**Goal:** Add a temporal signal model where agents deposit named signals that decay exponentially and are reinforced by repeat deposits, enabling stigmergic coordination.

**Architecture:** Signals live in a dedicated `SignalRegistry` (engine-common, `@ApplicationScoped`, `Resettable`) — not in CaseContext. Workers deposit/perceive via `WorkerRuntime` methods. Observers see signals via `ObservationContext.signals()`. Decay is computed lazily at read time using exponential half-life.

**Tech Stack:** Java 21, Quarkus 3.32.2, AssertJ, JUnit 5

## Global Constraints

- All new types follow Apache 2.0 license header (copy from existing files in the same package)
- Records use compact constructors with validation where needed
- `@ApplicationScoped` beans use constructor injection (no `@Inject` on fields)
- Default methods on interfaces for backward compatibility
- Engine-api types must not reference engine-common or runtime types

---

## Batch 1: Foundation types + SignalRegistry

### Task 1: Foundation types (Signal, PerceivedSignal, SignalConfig, SignalDecay)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/signal/Signal.java`
- Create: `api/src/main/java/io/casehub/api/model/signal/PerceivedSignal.java`
- Create: `api/src/main/java/io/casehub/api/model/signal/SignalConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/signal/SignalDecay.java`
- Test: `api/src/test/java/io/casehub/api/model/signal/SignalTest.java`
- Test: `api/src/test/java/io/casehub/api/model/signal/SignalDecayTest.java`
- Test: `api/src/test/java/io/casehub/api/model/signal/SignalConfigTest.java`

**Interfaces:**
- Consumes: nothing (foundation types)
- Produces:
  - `Signal(String name, double strength, Instant firstDeposited, Instant lastReinforced, Duration halfLife, String lastSource, int reinforcementCount, boolean expired)` — stored value type
  - `PerceivedSignal(String name, double effectiveStrength, int reinforcementCount, String lastSource, Duration age)` — read model
  - `SignalConfig(Duration defaultHalfLife, double effectiveZeroThreshold, int maxSignalsPerCase)` — per-case config
  - `SignalDecay.effectiveStrength(double strength, Instant lastReinforced, Duration halfLife, Instant now) → double` — static utility

- [ ] **Step 1: Write Signal record tests**

```java
// api/src/test/java/io/casehub/api/model/signal/SignalTest.java
package io.casehub.api.model.signal;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import java.time.Instant;
import org.junit.jupiter.api.Test;

class SignalTest {

  @Test
  void validSignal_createsSuccessfully() {
    Instant now = Instant.now();
    Signal signal = new Signal("trail", 0.8, now, now, Duration.ofMinutes(5), "agent-a", 1, false);
    assertThat(signal.name()).isEqualTo("trail");
    assertThat(signal.strength()).isEqualTo(0.8);
    assertThat(signal.firstDeposited()).isEqualTo(now);
    assertThat(signal.lastReinforced()).isEqualTo(now);
    assertThat(signal.halfLife()).isEqualTo(Duration.ofMinutes(5));
    assertThat(signal.lastSource()).isEqualTo("agent-a");
    assertThat(signal.reinforcementCount()).isEqualTo(1);
    assertThat(signal.expired()).isFalse();
  }

  @Test
  void strength_belowZero_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new Signal("x", -0.1, Instant.now(), Instant.now(),
            Duration.ofMinutes(1), "a", 1, false));
  }

  @Test
  void strength_aboveOne_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new Signal("x", 1.1, Instant.now(), Instant.now(),
            Duration.ofMinutes(1), "a", 1, false));
  }

  @Test
  void halfLife_zero_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new Signal("x", 0.5, Instant.now(), Instant.now(),
            Duration.ZERO, "a", 1, false));
  }

  @Test
  void halfLife_negative_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new Signal("x", 0.5, Instant.now(), Instant.now(),
            Duration.ofMinutes(-1), "a", 1, false));
  }
}
```

- [ ] **Step 2: Write SignalDecay tests**

```java
// api/src/test/java/io/casehub/api/model/signal/SignalDecayTest.java
package io.casehub.api.model.signal;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import java.time.Instant;
import org.junit.jupiter.api.Test;

class SignalDecayTest {

  @Test
  void noElapsedTime_returnsOriginalStrength() {
    Instant now = Instant.now();
    assertThat(SignalDecay.effectiveStrength(0.8, now, Duration.ofMinutes(5), now))
        .isEqualTo(0.8);
  }

  @Test
  void oneHalfLife_returnsHalfStrength() {
    Instant start = Instant.parse("2026-01-01T00:00:00Z");
    Instant after = start.plus(Duration.ofMinutes(5));
    double result = SignalDecay.effectiveStrength(1.0, start, Duration.ofMinutes(5), after);
    assertThat(result).isCloseTo(0.5, within(0.001));
  }

  @Test
  void twoHalfLives_returnsQuarterStrength() {
    Instant start = Instant.parse("2026-01-01T00:00:00Z");
    Instant after = start.plus(Duration.ofMinutes(10));
    double result = SignalDecay.effectiveStrength(1.0, start, Duration.ofMinutes(5), after);
    assertThat(result).isCloseTo(0.25, within(0.001));
  }

  @Test
  void futureTimestamp_returnsOriginalStrength() {
    Instant start = Instant.parse("2026-01-01T00:05:00Z");
    Instant before = Instant.parse("2026-01-01T00:00:00Z");
    assertThat(SignalDecay.effectiveStrength(0.8, start, Duration.ofMinutes(5), before))
        .isEqualTo(0.8);
  }

  @Test
  void veryLongElapsed_decaysNearZero() {
    Instant start = Instant.parse("2026-01-01T00:00:00Z");
    Instant after = start.plus(Duration.ofHours(1));
    double result = SignalDecay.effectiveStrength(1.0, start, Duration.ofMinutes(5), after);
    assertThat(result).isLessThan(0.001);
  }

  @Test
  void partialStrength_decaysProportionally() {
    Instant start = Instant.parse("2026-01-01T00:00:00Z");
    Instant after = start.plus(Duration.ofMinutes(5));
    double result = SignalDecay.effectiveStrength(0.6, start, Duration.ofMinutes(5), after);
    assertThat(result).isCloseTo(0.3, within(0.001));
  }
}
```

- [ ] **Step 3: Write SignalConfig tests**

```java
// api/src/test/java/io/casehub/api/model/signal/SignalConfigTest.java
package io.casehub.api.model.signal;

import static org.assertj.core.api.Assertions.*;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class SignalConfigTest {

  @Test
  void defaults_returnsExpectedValues() {
    SignalConfig config = SignalConfig.defaults();
    assertThat(config.defaultHalfLife()).isEqualTo(Duration.ofMinutes(5));
    assertThat(config.effectiveZeroThreshold()).isEqualTo(0.01);
    assertThat(config.maxSignalsPerCase()).isEqualTo(100);
  }

  @Test
  void threshold_negative_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new SignalConfig(Duration.ofMinutes(5), -0.01, 100));
  }

  @Test
  void threshold_aboveOne_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new SignalConfig(Duration.ofMinutes(5), 1.1, 100));
  }

  @Test
  void maxSignals_zero_throws() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new SignalConfig(Duration.ofMinutes(5), 0.01, 0));
  }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest="io.casehub.api.model.signal.*" -DfailIfNoTests=false -q`
Expected: Compilation failure — classes don't exist yet

- [ ] **Step 5: Implement Signal record**

Create `api/src/main/java/io/casehub/api/model/signal/Signal.java`:

```java
package io.casehub.api.model.signal;

import java.time.Duration;
import java.time.Instant;

public record Signal(
    String name,
    double strength,
    Instant firstDeposited,
    Instant lastReinforced,
    Duration halfLife,
    String lastSource,
    int reinforcementCount,
    boolean expired) {

  public Signal {
    if (strength < 0.0 || strength > 1.0)
      throw new IllegalArgumentException("strength must be in [0.0, 1.0], got: " + strength);
    if (halfLife.isNegative() || halfLife.isZero())
      throw new IllegalArgumentException("halfLife must be positive, got: " + halfLife);
  }
}
```

- [ ] **Step 6: Implement PerceivedSignal record**

Create `api/src/main/java/io/casehub/api/model/signal/PerceivedSignal.java`:

```java
package io.casehub.api.model.signal;

import java.time.Duration;

public record PerceivedSignal(
    String name,
    double effectiveStrength,
    int reinforcementCount,
    String lastSource,
    Duration age) {}
```

- [ ] **Step 7: Implement SignalConfig record**

Create `api/src/main/java/io/casehub/api/model/signal/SignalConfig.java`:

```java
package io.casehub.api.model.signal;

import java.time.Duration;

public record SignalConfig(
    Duration defaultHalfLife,
    double effectiveZeroThreshold,
    int maxSignalsPerCase) {

  public static final Duration DEFAULT_HALF_LIFE = Duration.ofMinutes(5);
  public static final double DEFAULT_EFFECTIVE_ZERO_THRESHOLD = 0.01;
  public static final int DEFAULT_MAX_SIGNALS_PER_CASE = 100;

  public SignalConfig {
    if (effectiveZeroThreshold < 0.0 || effectiveZeroThreshold > 1.0)
      throw new IllegalArgumentException(
          "effectiveZeroThreshold must be in [0.0, 1.0], got: " + effectiveZeroThreshold);
    if (maxSignalsPerCase < 1)
      throw new IllegalArgumentException(
          "maxSignalsPerCase must be >= 1, got: " + maxSignalsPerCase);
    if (defaultHalfLife.isNegative() || defaultHalfLife.isZero())
      throw new IllegalArgumentException(
          "defaultHalfLife must be positive, got: " + defaultHalfLife);
  }

  public static SignalConfig defaults() {
    return new SignalConfig(DEFAULT_HALF_LIFE,
        DEFAULT_EFFECTIVE_ZERO_THRESHOLD, DEFAULT_MAX_SIGNALS_PER_CASE);
  }
}
```

- [ ] **Step 8: Implement SignalDecay utility**

Create `api/src/main/java/io/casehub/api/model/signal/SignalDecay.java`:

```java
package io.casehub.api.model.signal;

import java.time.Duration;
import java.time.Instant;

public final class SignalDecay {

  private SignalDecay() {}

  public static double effectiveStrength(
      double strength, Instant lastReinforced, Duration halfLife, Instant now) {
    long elapsedMs = Duration.between(lastReinforced, now).toMillis();
    if (elapsedMs <= 0) return strength;
    double lambda = Math.log(2) / halfLife.toMillis();
    return strength * Math.exp(-lambda * elapsedMs);
  }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="io.casehub.api.model.signal.*" -q`
Expected: All 13 tests PASS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/signal/ api/src/test/java/io/casehub/api/model/signal/
git commit -m "feat: add Signal foundation types (Signal, PerceivedSignal, SignalConfig, SignalDecay) Refs #1106"
```

### Task 2: SignalRegistry

**Files:**
- Create: `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java`
- Test: `common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryTest.java`

**Interfaces:**
- Consumes: `Signal`, `PerceivedSignal`, `SignalConfig`, `SignalDecay` (from Task 1)
- Produces:
  - `deposit(UUID caseId, String name, double strength, Duration halfLife, String source, int maxPerCase) → boolean`
  - `perceive(UUID caseId, double effectiveZeroThreshold) → Map<String, PerceivedSignal>`
  - `findNewlyExpired(UUID caseId, double effectiveZeroThreshold) → List<Signal>`
  - `markExpired(UUID caseId, String name) → void`
  - `evictByCase(UUID caseId) → void`
  - `signalCount(UUID caseId) → int`
  - `reset() → void`

- [ ] **Step 1: Write SignalRegistry tests**

```java
// common-core/src/test/java/io/casehub/engine/common/internal/signal/SignalRegistryTest.java
package io.casehub.engine.common.internal.signal;

import static org.assertj.core.api.Assertions.*;

import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.model.signal.Signal;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class SignalRegistryTest {

  private SignalRegistry registry;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    registry = new SignalRegistry();
    caseId = UUID.randomUUID();
  }

  @Test
  void deposit_newSignal_returnsTrue() {
    boolean result = registry.deposit(caseId, "trail", 0.8,
        Duration.ofMinutes(5), "agent-a", 100);
    assertThat(result).isTrue();
    assertThat(registry.signalCount(caseId)).isEqualTo(1);
  }

  @Test
  void deposit_reinforcement_updatesStrengthAndCount() {
    registry.deposit(caseId, "trail", 0.5, Duration.ofMinutes(5), "agent-a", 100);
    registry.deposit(caseId, "trail", 0.8, Duration.ofMinutes(5), "agent-b", 100);

    Map<String, PerceivedSignal> perceived = registry.perceive(caseId, 0.01);
    assertThat(perceived).containsKey("trail");
    PerceivedSignal signal = perceived.get("trail");
    assertThat(signal.reinforcementCount()).isEqualTo(2);
    assertThat(signal.lastSource()).isEqualTo("agent-b");
    assertThat(signal.effectiveStrength()).isGreaterThanOrEqualTo(0.8);
  }

  @Test
  void deposit_exceedsMax_returnsFalse() {
    registry.deposit(caseId, "s1", 0.5, Duration.ofMinutes(5), "a", 2);
    registry.deposit(caseId, "s2", 0.5, Duration.ofMinutes(5), "a", 2);
    boolean result = registry.deposit(caseId, "s3", 0.5, Duration.ofMinutes(5), "a", 2);
    assertThat(result).isFalse();
    assertThat(registry.signalCount(caseId)).isEqualTo(2);
  }

  @Test
  void deposit_reinforcementDoesNotCountAgainstMax() {
    registry.deposit(caseId, "s1", 0.5, Duration.ofMinutes(5), "a", 2);
    registry.deposit(caseId, "s2", 0.5, Duration.ofMinutes(5), "a", 2);
    boolean result = registry.deposit(caseId, "s1", 0.9, Duration.ofMinutes(5), "b", 2);
    assertThat(result).isTrue();
  }

  @Test
  void perceive_filtersExpiredSignals() {
    registry.deposit(caseId, "trail", 0.5, Duration.ofMinutes(5), "a", 100);
    registry.markExpired(caseId, "trail");
    Map<String, PerceivedSignal> perceived = registry.perceive(caseId, 0.01);
    assertThat(perceived).isEmpty();
  }

  @Test
  void perceive_emptyCase_returnsEmpty() {
    Map<String, PerceivedSignal> perceived = registry.perceive(UUID.randomUUID(), 0.01);
    assertThat(perceived).isEmpty();
  }

  @Test
  void findNewlyExpired_detectsCrossedThreshold() throws InterruptedException {
    registry.deposit(caseId, "fast-decay", 0.02,
        Duration.ofMillis(1), "a", 100);
    Thread.sleep(20);
    List<Signal> expired = registry.findNewlyExpired(caseId, 0.01);
    assertThat(expired).hasSize(1);
    assertThat(expired.get(0).name()).isEqualTo("fast-decay");
  }

  @Test
  void findNewlyExpired_alreadyMarked_notReturned() throws InterruptedException {
    registry.deposit(caseId, "fast-decay", 0.02,
        Duration.ofMillis(1), "a", 100);
    Thread.sleep(20);
    registry.findNewlyExpired(caseId, 0.01);
    registry.markExpired(caseId, "fast-decay");
    List<Signal> expired = registry.findNewlyExpired(caseId, 0.01);
    assertThat(expired).isEmpty();
  }

  @Test
  void evictByCase_removesAllSignals() {
    registry.deposit(caseId, "s1", 0.5, Duration.ofMinutes(5), "a", 100);
    registry.deposit(caseId, "s2", 0.5, Duration.ofMinutes(5), "a", 100);
    registry.evictByCase(caseId);
    assertThat(registry.signalCount(caseId)).isZero();
    assertThat(registry.perceive(caseId, 0.01)).isEmpty();
  }

  @Test
  void reset_clearsAllCases() {
    UUID case1 = UUID.randomUUID();
    UUID case2 = UUID.randomUUID();
    registry.deposit(case1, "s1", 0.5, Duration.ofMinutes(5), "a", 100);
    registry.deposit(case2, "s2", 0.5, Duration.ofMinutes(5), "a", 100);
    registry.reset();
    assertThat(registry.signalCount(case1)).isZero();
    assertThat(registry.signalCount(case2)).isZero();
  }

  @Test
  void deposit_expiredSignal_treatedAsNew() {
    registry.deposit(caseId, "trail", 0.5, Duration.ofMinutes(5), "agent-a", 100);
    registry.markExpired(caseId, "trail");
    boolean result = registry.deposit(caseId, "trail", 0.9, Duration.ofMinutes(5), "agent-b", 100);
    assertThat(result).isTrue();
    Map<String, PerceivedSignal> perceived = registry.perceive(caseId, 0.01);
    assertThat(perceived.get("trail").reinforcementCount()).isEqualTo(1);
    assertThat(perceived.get("trail").lastSource()).isEqualTo("agent-b");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl common-core -Dtest="io.casehub.engine.common.internal.signal.*" -DfailIfNoTests=false -q`
Expected: Compilation failure — `SignalRegistry` doesn't exist yet

- [ ] **Step 3: Implement SignalRegistry**

Create `common-core/src/main/java/io/casehub/engine/common/internal/signal/SignalRegistry.java`:

```java
package io.casehub.engine.common.internal.signal;

import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.model.signal.Signal;
import io.casehub.api.model.signal.SignalDecay;
import io.casehub.engine.common.spi.Resettable;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SignalRegistry implements Resettable {

  private static final Logger LOG = Logger.getLogger(SignalRegistry.class);

  private final ConcurrentHashMap<UUID, ConcurrentHashMap<String, Signal>> signals =
      new ConcurrentHashMap<>();

  public boolean deposit(UUID caseId, String name, double strength,
      Duration halfLife, String source, int maxPerCase) {
    ConcurrentHashMap<String, Signal> caseSignals =
        signals.computeIfAbsent(caseId, k -> new ConcurrentHashMap<>());

    Signal existing = caseSignals.get(name);
    Instant now = Instant.now();

    if (existing != null && !existing.expired()) {
      double currentEffective = SignalDecay.effectiveStrength(
          existing.strength(), existing.lastReinforced(), existing.halfLife(), now);
      double newStrength = Math.max(currentEffective, strength);
      caseSignals.put(name, new Signal(name, newStrength, existing.firstDeposited(),
          now, halfLife, source, existing.reinforcementCount() + 1, false));
      return true;
    }

    if (existing != null && existing.expired()) {
      caseSignals.put(name, new Signal(name, strength, now, now, halfLife, source, 1, false));
      return true;
    }

    if (caseSignals.size() >= maxPerCase) {
      LOG.warnf("Signal cap reached for case=%s (max=%d), dropping signal=%s",
          caseId, maxPerCase, name);
      return false;
    }

    caseSignals.put(name, new Signal(name, strength, now, now, halfLife, source, 1, false));
    return true;
  }

  public Map<String, PerceivedSignal> perceive(UUID caseId, double effectiveZeroThreshold) {
    ConcurrentHashMap<String, Signal> caseSignals = signals.get(caseId);
    if (caseSignals == null) return Map.of();

    Instant now = Instant.now();
    Map<String, PerceivedSignal> result = new LinkedHashMap<>();
    for (Signal signal : caseSignals.values()) {
      if (signal.expired()) continue;
      double effective = SignalDecay.effectiveStrength(
          signal.strength(), signal.lastReinforced(), signal.halfLife(), now);
      if (effective < effectiveZeroThreshold) continue;
      result.put(signal.name(), new PerceivedSignal(signal.name(), effective,
          signal.reinforcementCount(), signal.lastSource(),
          Duration.between(signal.lastReinforced(), now)));
    }
    return Collections.unmodifiableMap(result);
  }

  public List<Signal> findNewlyExpired(UUID caseId, double effectiveZeroThreshold) {
    ConcurrentHashMap<String, Signal> caseSignals = signals.get(caseId);
    if (caseSignals == null) return List.of();

    Instant now = Instant.now();
    List<Signal> expired = new ArrayList<>();
    for (Signal signal : caseSignals.values()) {
      if (signal.expired()) continue;
      double effective = SignalDecay.effectiveStrength(
          signal.strength(), signal.lastReinforced(), signal.halfLife(), now);
      if (effective < effectiveZeroThreshold) {
        expired.add(signal);
      }
    }
    return expired;
  }

  public void markExpired(UUID caseId, String name) {
    ConcurrentHashMap<String, Signal> caseSignals = signals.get(caseId);
    if (caseSignals == null) return;
    Signal existing = caseSignals.get(name);
    if (existing == null) return;
    caseSignals.put(name, new Signal(existing.name(), existing.strength(),
        existing.firstDeposited(), existing.lastReinforced(),
        existing.halfLife(), existing.lastSource(),
        existing.reinforcementCount(), true));
  }

  public void evictByCase(UUID caseId) {
    signals.remove(caseId);
  }

  public int signalCount(UUID caseId) {
    ConcurrentHashMap<String, Signal> caseSignals = signals.get(caseId);
    return caseSignals == null ? 0 : caseSignals.size();
  }

  @Override
  public void reset() {
    signals.clear();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl common-core -Dtest="io.casehub.engine.common.internal.signal.*" -q`
Expected: All 12 tests PASS

- [ ] **Step 5: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/signal/ common-core/src/test/java/io/casehub/engine/common/internal/signal/
git commit -m "feat: add SignalRegistry with deposit, perceive, expiry, and eviction Refs #1106"
```

## Batch 2: CaseDefinition + Worker API + Pipeline integration

### Task 3: CaseDefinition signalConfig + YAML parsing + CaseHubEventType

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `signalConfig` field, getter, setter, builder method
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse `signalConfig:` YAML block
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add `SIGNAL_DEPOSITED`, `SIGNAL_EXPIRED`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperSignalConfigTest.java`
- Create: `api/src/test/resources/definitions/signal-config-test.yaml`

**Interfaces:**
- Consumes: `SignalConfig` (from Task 1)
- Produces:
  - `CaseDefinition.getSignalConfig() → SignalConfig` (returns defaults when null)
  - `CaseDefinition.Builder.signalConfig(SignalConfig) → Builder`
  - `CaseHubEventType.SIGNAL_DEPOSITED`
  - `CaseHubEventType.SIGNAL_EXPIRED`

- [ ] **Step 1: Write YAML test fixture**

Create `api/src/test/resources/definitions/signal-config-test.yaml`:

```yaml
spec:
  name: signal-test
  namespace: test
  version: "1.0"
  signalConfig:
    defaultHalfLife: PT10M
    effectiveZeroThreshold: 0.05
    maxSignalsPerCase: 50
  capabilities:
    - name: analyse
  workers:
    - name: analyser
      capabilities: [analyse]
  bindings:
    - name: run-analyser
      capability: analyse
      on:
        contextChanged: ".input != null"
```

- [ ] **Step 2: Write YAML mapper test**

```java
// api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperSignalConfigTest.java
package io.casehub.api.model.converter;

import static org.assertj.core.api.Assertions.*;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.signal.SignalConfig;
import java.io.InputStream;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperSignalConfigTest {

  @Test
  void signalConfig_parsedFromYaml() {
    CaseDefinition def = loadDefinition("signal-config-test.yaml");
    SignalConfig config = def.getSignalConfig();
    assertThat(config.defaultHalfLife()).isEqualTo(Duration.ofMinutes(10));
    assertThat(config.effectiveZeroThreshold()).isEqualTo(0.05);
    assertThat(config.maxSignalsPerCase()).isEqualTo(50);
  }

  @Test
  void signalConfig_absent_returnsDefaults() {
    CaseDefinition def = loadDefinition("basic-test.yaml");
    SignalConfig config = def.getSignalConfig();
    assertThat(config.defaultHalfLife()).isEqualTo(Duration.ofMinutes(5));
    assertThat(config.effectiveZeroThreshold()).isEqualTo(0.01);
    assertThat(config.maxSignalsPerCase()).isEqualTo(100);
  }

  @Test
  void signalConfig_viaBuilder() {
    SignalConfig config = new SignalConfig(Duration.ofMinutes(3), 0.02, 200);
    CaseDefinition def = CaseDefinition.builder()
        .name("test").namespace("test").version("1.0")
        .signalConfig(config)
        .build();
    assertThat(def.getSignalConfig()).isEqualTo(config);
  }

  private static CaseDefinition loadDefinition(String filename) {
    InputStream is = CaseDefinitionYamlMapperSignalConfigTest.class
        .getResourceAsStream("/definitions/" + filename);
    return CaseDefinitionYamlMapper.fromYaml(is);
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest="CaseDefinitionYamlMapperSignalConfigTest" -DfailIfNoTests=false -q`
Expected: Compilation failure — `getSignalConfig()`, `signalConfig()` builder method don't exist

- [ ] **Step 4: Add signalConfig to CaseDefinition**

Use `ide_insert_member` to add field, getter, setter, and builder method to `CaseDefinition.java`, following the exact pattern of `observationConfig`:

1. Add field (after `observationConfig` around line 193): `private io.casehub.api.model.signal.SignalConfig signalConfig;`
2. Add getter (after `getObservationConfig()` around line 648):
```java
public io.casehub.api.model.signal.SignalConfig getSignalConfig() {
  return signalConfig != null
      ? signalConfig
      : io.casehub.api.model.signal.SignalConfig.defaults();
}

public void setSignalConfig(io.casehub.api.model.signal.SignalConfig signalConfig) {
  this.signalConfig = signalConfig;
}
```
3. Add builder field (after builder's `observationConfig` field): `private io.casehub.api.model.signal.SignalConfig signalConfig;`
4. Add builder method (after builder's `observationConfig()` method):
```java
public Builder signalConfig(io.casehub.api.model.signal.SignalConfig signalConfig) {
  this.signalConfig = signalConfig;
  return this;
}
```
5. In `build()`: `caseHubDefinition.setSignalConfig(signalConfig);`

- [ ] **Step 5: Add YAML parsing for signalConfig**

In `CaseDefinitionYamlMapper`, add parsing logic after the `observationConfig` parsing block. Look for `signalConfig` node under `spec`:

```java
JsonNode signalConfigNode = specNode.get("signalConfig");
if (signalConfigNode != null) {
  Duration halfLife = signalConfigNode.has("defaultHalfLife")
      ? Duration.parse(signalConfigNode.get("defaultHalfLife").asText())
      : SignalConfig.DEFAULT_HALF_LIFE;
  double threshold = signalConfigNode.has("effectiveZeroThreshold")
      ? signalConfigNode.get("effectiveZeroThreshold").asDouble()
      : SignalConfig.DEFAULT_EFFECTIVE_ZERO_THRESHOLD;
  int maxSignals = signalConfigNode.has("maxSignalsPerCase")
      ? signalConfigNode.get("maxSignalsPerCase").asInt()
      : SignalConfig.DEFAULT_MAX_SIGNALS_PER_CASE;
  builder.signalConfig(new SignalConfig(halfLife, threshold, maxSignals));
}
```

- [ ] **Step 6: Add CaseHubEventType values**

Use `ide_edit_member` to add two enum constants to `CaseHubEventType`:

```java
SIGNAL_DEPOSITED,
SIGNAL_EXPIRED
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest="CaseDefinitionYamlMapperSignalConfigTest" -q`
Expected: All 3 tests PASS

- [ ] **Step 8: Run full api module tests to check for regressions**

Run: `mvn test -pl api -q`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add api/
git commit -m "feat: add signalConfig to CaseDefinition + YAML parsing + SIGNAL_DEPOSITED/EXPIRED event types Refs #1106"
```

### Task 4: WorkerRuntime signal methods + DefaultWorkerRuntime

**Files:**
- Modify: `api/src/main/java/io/casehub/api/engine/WorkerRuntime.java` — add `depositSignal()` and `perceiveSignals()` default methods
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java` — implement signal methods
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java` — inject `SignalRegistry`
- Test: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeSignalTest.java`

**Interfaces:**
- Consumes: `SignalRegistry` (Task 2), `SignalConfig` (Task 1), `CaseDefinition.getSignalConfig()` (Task 3), `CaseHubEventType.SIGNAL_DEPOSITED` (Task 3)
- Produces:
  - `WorkerRuntime.depositSignal(String name, double strength)` — default no-op
  - `WorkerRuntime.depositSignal(String name, double strength, Duration halfLife)` — default no-op
  - `WorkerRuntime.perceiveSignals() → Map<String, PerceivedSignal>` — default returns `Map.of()`

- [ ] **Step 1: Write DefaultWorkerRuntime signal tests**

```java
// runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeSignalTest.java
package io.casehub.engine.internal.executor;

import static org.assertj.core.api.Assertions.*;

import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.model.signal.SignalConfig;
import io.casehub.engine.common.internal.signal.SignalRegistry;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultWorkerRuntimeSignalTest {

  private SignalRegistry signalRegistry;
  private DefaultWorkerRuntime runtime;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    signalRegistry = new SignalRegistry();
    caseId = UUID.randomUUID();
    SignalConfig config = SignalConfig.defaults();
    runtime = createRuntime(caseId, "test-agent", config);
  }

  @Test
  void depositSignal_createsSignalInRegistry() {
    runtime.depositSignal("trail", 0.8);
    assertThat(signalRegistry.signalCount(caseId)).isEqualTo(1);
  }

  @Test
  void depositSignal_withCustomHalfLife() {
    runtime.depositSignal("trail", 0.8, Duration.ofMinutes(10));
    Map<String, PerceivedSignal> perceived = signalRegistry.perceive(caseId, 0.01);
    assertThat(perceived).containsKey("trail");
    assertThat(perceived.get("trail").effectiveStrength()).isCloseTo(0.8, within(0.01));
  }

  @Test
  void perceiveSignals_returnsActiveSignals() {
    signalRegistry.deposit(caseId, "trail", 0.8, Duration.ofMinutes(5), "other-agent", 100);
    Map<String, PerceivedSignal> perceived = runtime.perceiveSignals();
    assertThat(perceived).containsKey("trail");
    assertThat(perceived.get("trail").effectiveStrength()).isGreaterThan(0.0);
  }

  @Test
  void perceiveSignals_emptyWhenNoSignals() {
    Map<String, PerceivedSignal> perceived = runtime.perceiveSignals();
    assertThat(perceived).isEmpty();
  }

  // Helper — constructs a DefaultWorkerRuntime with signal support.
  // Uses null for non-signal dependencies (tested elsewhere).
  private DefaultWorkerRuntime createRuntime(UUID caseId, String workerName,
      SignalConfig config) {
    // This test will need a constructor or factory that accepts SignalRegistry + SignalConfig.
    // The exact wiring depends on the implementation — see Step 3.
    return null; // placeholder — replaced by actual constructor in Step 3
  }
}
```

Note: The `createRuntime` helper will be completed in Step 3 when the constructor signature is finalized.

- [ ] **Step 2: Add default methods to WorkerRuntime**

Use `ide_insert_member` to add three default methods to `WorkerRuntime.java` (after `registerObserver`):

```java
default void depositSignal(String name, double strength) {}

default void depositSignal(String name, double strength, java.time.Duration halfLife) {}

default java.util.Map<String, io.casehub.api.model.signal.PerceivedSignal> perceiveSignals() {
  return java.util.Map.of();
}
```

- [ ] **Step 3: Implement signal methods in DefaultWorkerRuntime**

1. Add `SignalRegistry` and `SignalConfig` fields to `DefaultWorkerRuntime`
2. Add them to the 13-arg constructor (the one with `observationRegistry`, `workerName`, `bindingName`)
3. Implement `depositSignal(String, double)` — delegates to registry with `config.defaultHalfLife()`
4. Implement `depositSignal(String, double, Duration)` — delegates to registry with explicit halfLife
5. Implement `perceiveSignals()` — delegates to registry with `config.effectiveZeroThreshold()`

```java
// Fields
private final SignalRegistry signalRegistry;
private final SignalConfig signalConfig;

// In depositSignal(name, strength):
@Override
public void depositSignal(String name, double strength) {
  depositSignal(name, strength, signalConfig.defaultHalfLife());
}

// In depositSignal(name, strength, halfLife):
@Override
public void depositSignal(String name, double strength, Duration halfLife) {
  if (signalRegistry == null) return;
  signalRegistry.deposit(caseId, name, strength, halfLife, workerName,
      signalConfig.maxSignalsPerCase());
}

// In perceiveSignals():
@Override
public Map<String, PerceivedSignal> perceiveSignals() {
  if (signalRegistry == null) return Map.of();
  return signalRegistry.perceive(caseId, signalConfig.effectiveZeroThreshold());
}
```

- [ ] **Step 4: Update WorkerRuntimeFactory**

1. Add `SignalRegistry signalRegistry` field and constructor parameter
2. In the 6-arg `create()` (with workerName, bindingName): pass `signalRegistry` to `DefaultWorkerRuntime`
3. Look up `SignalConfig` from `CaseDefinition` via `definitionRegistry` in the factory, or pass a default

- [ ] **Step 5: Complete test helper and run tests**

Update the test `createRuntime` method to use the actual constructor with signal parameters. Run:

Run: `mvn test -pl runtime -Dtest="DefaultWorkerRuntimeSignalTest" -q`
Expected: All 4 tests PASS

- [ ] **Step 6: Run full runtime module tests to check for regressions**

Run: `mvn test -pl runtime -q`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/engine/WorkerRuntime.java runtime-core/src/main/java/io/casehub/engine/internal/executor/ runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeSignalTest.java
git commit -m "feat: add depositSignal/perceiveSignals to WorkerRuntime + DefaultWorkerRuntime Refs #1106"
```

### Task 5: ObservationContext + handler integration + eviction + SignalStrengthObserver

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/observation/ObservationContext.java` — add `signals` field
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — inject `SignalRegistry`, integrate into `observations()`
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — add `signalRegistry.evictByCase()` call
- Create: `runtime-core/src/main/java/io/casehub/engine/internal/observation/SignalStrengthObserver.java`
- Test: `runtime-core/src/test/java/io/casehub/engine/internal/observation/SignalStrengthObserverTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/EngineResetServiceTest.java` — verify `SignalRegistry` is reset

**Interfaces:**
- Consumes: `SignalRegistry` (Task 2), `SignalConfig` (Task 1), `CaseDefinition.getSignalConfig()` (Task 3), `CaseHubEventType.SIGNAL_EXPIRED` (Task 3), `PerceivedSignal` (Task 1)
- Produces:
  - `ObservationContext.signals() → Map<String, PerceivedSignal>`
  - `SignalStrengthObserver.of(String signalName, ThresholdObserver.Operator operator, double threshold)`

- [ ] **Step 1: Write SignalStrengthObserver tests**

```java
// runtime-core/src/test/java/io/casehub/engine/internal/observation/SignalStrengthObserverTest.java
package io.casehub.engine.internal.observation;

import static org.assertj.core.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.JsonNodeFactory;
import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.spi.observation.Observation;
import io.casehub.api.spi.observation.ObservationContext;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class SignalStrengthObserverTest {

  @Test
  void observerType_returnsSignalStrength() {
    SignalStrengthObserver observer = SignalStrengthObserver.of(
        "trail", ThresholdObserver.Operator.GT, 0.5);
    assertThat(observer.observerType()).isEqualTo("signal-strength");
  }

  @Test
  void watchedKeys_returnsEmpty() {
    SignalStrengthObserver observer = SignalStrengthObserver.of(
        "trail", ThresholdObserver.Operator.GT, 0.5);
    assertThat(observer.watchedKeys()).isEmpty();
  }

  @Test
  void observe_signalAboveThreshold_returnsObservation() {
    SignalStrengthObserver observer = SignalStrengthObserver.of(
        "trail", ThresholdObserver.Operator.GT, 0.5);
    PerceivedSignal signal = new PerceivedSignal("trail", 0.8, 3, "agent-a",
        Duration.ofSeconds(30));
    ObservationContext ctx = contextWithSignals(Map.of("trail", signal));

    List<Observation> results = observer.observe(ctx);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).patternId()).isEqualTo("signal-strength-crossing");
    assertThat(results.get(0).confidence()).isEqualTo(1.0);
  }

  @Test
  void observe_signalBelowThreshold_returnsEmpty() {
    SignalStrengthObserver observer = SignalStrengthObserver.of(
        "trail", ThresholdObserver.Operator.GT, 0.5);
    PerceivedSignal signal = new PerceivedSignal("trail", 0.3, 1, "agent-a",
        Duration.ofSeconds(30));
    ObservationContext ctx = contextWithSignals(Map.of("trail", signal));

    List<Observation> results = observer.observe(ctx);
    assertThat(results).isEmpty();
  }

  @Test
  void observe_signalAbsent_returnsEmpty() {
    SignalStrengthObserver observer = SignalStrengthObserver.of(
        "trail", ThresholdObserver.Operator.GT, 0.5);
    ObservationContext ctx = contextWithSignals(Map.of());

    List<Observation> results = observer.observe(ctx);
    assertThat(results).isEmpty();
  }

  private ObservationContext contextWithSignals(Map<String, PerceivedSignal> signals) {
    JsonNode emptyNode = JsonNodeFactory.instance.objectNode();
    return new ObservationContext(emptyNode, Set.of(), List.of(),
        "test-agent", "tenant-1", UUID.randomUUID(), signals);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime-core -Dtest="SignalStrengthObserverTest" -DfailIfNoTests=false -q`
Expected: Compilation failure

- [ ] **Step 3: Add signals field to ObservationContext**

Modify `ObservationContext` record to add a 7th field:

```java
public record ObservationContext(
    JsonNode snapshot,
    Set<String> changedKeys,
    List<ContextSnapshot> history,
    String agentId,
    String tenancyId,
    UUID caseId,
    Map<String, PerceivedSignal> signals) {

  public ObservationContext(JsonNode snapshot, Set<String> changedKeys,
      List<ContextSnapshot> history, String agentId, String tenancyId, UUID caseId) {
    this(snapshot, changedKeys, history, agentId, tenancyId, caseId, Map.of());
  }
}
```

- [ ] **Step 4: Implement SignalStrengthObserver**

Create `runtime-core/src/main/java/io/casehub/engine/internal/observation/SignalStrengthObserver.java`:

```java
package io.casehub.engine.internal.observation;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.DoubleNode;
import com.fasterxml.jackson.databind.node.IntNode;
import com.fasterxml.jackson.databind.node.TextNode;
import io.casehub.api.model.signal.PerceivedSignal;
import io.casehub.api.spi.observation.EnvironmentObserver;
import io.casehub.api.spi.observation.Observation;
import io.casehub.api.spi.observation.ObservationContext;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class SignalStrengthObserver implements EnvironmentObserver {

  private final String signalName;
  private final ThresholdObserver.Operator operator;
  private final double threshold;

  private SignalStrengthObserver(String signalName, ThresholdObserver.Operator operator,
      double threshold) {
    this.signalName = signalName;
    this.operator = operator;
    this.threshold = threshold;
  }

  public static SignalStrengthObserver of(String signalName,
      ThresholdObserver.Operator operator, double threshold) {
    return new SignalStrengthObserver(signalName, operator, threshold);
  }

  @Override
  public String observerType() {
    return "signal-strength";
  }

  @Override
  public Set<String> watchedKeys() {
    return Set.of();
  }

  @Override
  public List<Observation> observe(ObservationContext ctx) {
    PerceivedSignal signal = ctx.signals().get(signalName);
    if (signal == null) return List.of();

    double value = signal.effectiveStrength();
    boolean crossed = switch (operator) {
      case GT -> value > threshold;
      case LT -> value < threshold;
      case GTE -> value >= threshold;
      case LTE -> value <= threshold;
      case EQ -> Double.compare(value, threshold) == 0;
    };
    if (!crossed) return List.of();

    Map<String, JsonNode> details = new LinkedHashMap<>();
    details.put("signalName", TextNode.valueOf(signalName));
    details.put("effectiveStrength", DoubleNode.valueOf(value));
    details.put("threshold", DoubleNode.valueOf(threshold));
    details.put("operator", TextNode.valueOf(operator.name()));
    details.put("reinforcementCount", IntNode.valueOf(signal.reinforcementCount()));
    return List.of(new Observation("signal-strength-crossing", 1.0, details, Instant.now()));
  }
}
```

- [ ] **Step 5: Run SignalStrengthObserver tests**

Run: `mvn test -pl runtime-core -Dtest="SignalStrengthObserverTest" -q`
Expected: All 5 tests PASS

- [ ] **Step 6: Integrate signals into observations() pipeline**

In `CaseContextChangedEventHandler`:

1. `SignalRegistry` is already injected (field exists from constructor)
2. In `observations()` method, before building `ObservationContext`:
   - Read `SignalConfig` from definition: `SignalConfig signalConfig = definition.getSignalConfig();`
   - Perceive signals: `Map<String, PerceivedSignal> signals = signalRegistry.perceive(caseInstance.getUuid(), signalConfig.effectiveZeroThreshold());`
   - Detect expiry: iterate `signalRegistry.findNewlyExpired(caseId, threshold)`, for each publish `SIGNAL_EXPIRED` EventLog and call `markExpired()`
3. Pass `signals` as the 7th argument to `ObservationContext` constructor

- [ ] **Step 7: Add eviction to CaseStatusChangedHandler**

In `CaseStatusChangedHandler`, in the terminal-state cleanup block (where `observationRegistry.unregisterByCase()` and `contextHistoryBuffer.evict()` are called):

1. Inject `SignalRegistry` via constructor
2. Add: `signalRegistry.evictByCase(caseInstance.getUuid());`

- [ ] **Step 8: Run full runtime tests to check for regressions**

Run: `mvn test -pl runtime -q`
Expected: All tests PASS

- [ ] **Step 9: Verify EngineResetService discovers SignalRegistry**

The `EngineResetService` uses `Instance<Resettable>` — `SignalRegistry` is `@ApplicationScoped` and implements `Resettable`, so it should be auto-discovered. Check `EngineResetServiceTest` to confirm (may need a new assertion).

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/observation/ObservationContext.java runtime-core/src/main/java/ runtime-core/src/test/java/ runtime/src/test/java/
git commit -m "feat: integrate signal perception into observation pipeline + SignalStrengthObserver + eviction Refs #1106"
```

- [ ] **Step 11: Update CLAUDE.md with signal model documentation**

Add a `## Signal/Pheromone Model` section to CLAUDE.md documenting the signal types, registry, configuration, and integration points.

```bash
git add CLAUDE.md
git commit -m "docs: add signal/pheromone model to CLAUDE.md Refs #1106"
```

## References

- [2026-09-16-signal-pheromone-model-design.md] — design spec this plan implements
- [ObservationRegistry.java:27] — per-case registry pattern
- [ContextHistoryBuffer.java:28] — Resettable + eviction pattern
- [WorkerRuntime.java:24] — existing coordination API surface
- [DefaultWorkerRuntime.java:43] — runtime delegation implementation
- [WorkerRuntimeFactory.java:25] — factory with dependency injection
- [CaseContextChangedEventHandler.java:1168] — observations() pipeline integration point
- [CaseStatusChangedHandler.java:53] — terminal state eviction point
- [CaseDefinition.java:193] — observationConfig field pattern
- [CaseDefinitionYamlMapper.java:42] — YAML parsing pattern
- [ThresholdObserver.java:30] — classical observer pattern
- [CaseHubEventType.java:18] — event type enum
- [D10-D18] — design decisions
- [engine#1106] — focal issue
- [engine#1104] — parent epic
