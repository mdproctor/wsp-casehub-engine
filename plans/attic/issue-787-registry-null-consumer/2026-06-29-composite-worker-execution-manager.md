# Composite WorkerExecutionManager Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable co-deployment of multiple `WorkerExecutionManager` backends (Quartz + Camel + HTTP + ...) by introducing a composite that routes via a pluggable strategy.

**Architecture:** A `@WorkerBackend` CDI qualifier separates individual backends from the composite. The composite injects `@WorkerBackend Instance<WorkerExecutionManager>` and delegates `submit()` routing to a `WorkerExecutionRoutingStrategy` SPI. Default strategy: first supporter by `@Priority` order.

**Tech Stack:** Java 21, Quarkus 3.32.2 (CDI/ARC), Mutiny, JUnit 5, Mockito

**Spec:** `docs/superpowers/specs/2026-06-29-composite-worker-execution-manager-design.md`

## Global Constraints

- `supports()` is abstract on the interface — breaking change; every implementation must implement it
- `schedulePersistedEvent()` becomes a `default` no-op — external backends delete their override
- `@WorkerBackend` qualifier removes implicit `@Default` — concrete-type injection sites must add `@WorkerBackend`
- External backends: `@Priority(10)`. Quartz: `@Priority(0)`.
- All new SPIs in `casehub-engine-common/spi/scheduler/`
- Composite and default strategy in engine `runtime/`
- TDD: write failing test → verify failure → implement → verify pass → commit
- Every commit references `Refs #461`

## Repos Involved

| Repo | Path | Changes |
|------|------|---------|
| casehub-engine | `/Users/mdproctor/claude/casehub/engine` | Interface, composite, strategy, Quartz migration |
| casehub-workers | `/Users/mdproctor/claude/casehub/workers` | 5 backend migrations (Camel, HTTP, Script, MCP, GH Actions) |
| casehub-claudony | `/Users/mdproctor/claude/casehub/claudony` | 1 backend + 4 injection site fixes |

---

### Task 1: SPI Interface Changes (engine-common)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionManager.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerBackend.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionRoutingStrategy.java`

**Produces:**
- `WorkerBackend` qualifier annotation
- `WorkerExecutionManager.supports(String capabilityName, String tenancyId): boolean` (abstract)
- `WorkerExecutionManager.schedulePersistedEvent(EventLog): default Uni<Void>` (no-op)
- `WorkerExecutionRoutingStrategy.select(List<WorkerExecutionManager>, Worker, Capability, String): Optional<WorkerExecutionManager>`

- [ ] **Step 1: Create `@WorkerBackend` qualifier**

Create `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerBackend.java`:

```java
package io.casehub.engine.common.spi.scheduler;

import static java.lang.annotation.ElementType.*;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import jakarta.inject.Qualifier;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

@Qualifier
@Retention(RUNTIME)
@Target({TYPE, METHOD, FIELD, PARAMETER})
public @interface WorkerBackend {}
```

- [ ] **Step 2: Add `supports()` and make `schedulePersistedEvent()` default on `WorkerExecutionManager`**

In `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionManager.java`, add the abstract method before `submit()` and change `schedulePersistedEvent()` to a default:

```java
boolean supports(String capabilityName, String tenancyId);
```

Change `schedulePersistedEvent` from:
```java
Uni<Void> schedulePersistedEvent(EventLog scheduledEventLog);
```
to:
```java
default Uni<Void> schedulePersistedEvent(EventLog scheduledEventLog) {
    return Uni.createFrom().voidItem();
}
```

- [ ] **Step 3: Create `WorkerExecutionRoutingStrategy`**

Create `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionRoutingStrategy.java`:

```java
package io.casehub.engine.common.spi.scheduler;

import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import java.util.List;
import java.util.Optional;

public interface WorkerExecutionRoutingStrategy {

  Optional<WorkerExecutionManager> select(
      List<WorkerExecutionManager> candidates,
      Worker worker,
      Capability capability,
      String tenancyId);
}
```

- [ ] **Step 4: Verify engine-common compiles**

Run: `mvn compile -pl casehub-engine-common -q`

Expected: BUILD SUCCESS (interface changes are additive/default — no implementations broken yet because `supports()` is abstract and implementations haven't been updated, but the common module itself compiles).

- [ ] **Step 5: Commit**

```
feat(#461): add WorkerBackend qualifier, supports() SPI, and WorkerExecutionRoutingStrategy

Adds the three SPI building blocks for composite WorkerExecutionManager:
- @WorkerBackend CDI qualifier for backend discovery
- supports() abstract method on WorkerExecutionManager
- WorkerExecutionRoutingStrategy SPI for pluggable routing
- schedulePersistedEvent() becomes default no-op

Refs #461
```

---

### Task 2: Default Routing Strategy (engine runtime)

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/FirstSupportedRoutingStrategy.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/FirstSupportedRoutingStrategyTest.java`

**Consumes:** `WorkerExecutionRoutingStrategy` from Task 1
**Produces:** `FirstSupportedRoutingStrategy` — `@DefaultBean @ApplicationScoped`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/engine/internal/routing/FirstSupportedRoutingStrategyTest.java`:

```java
package io.casehub.engine.internal.routing;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.engine.common.spi.scheduler.WorkerExecutionManager;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class FirstSupportedRoutingStrategyTest {

  private FirstSupportedRoutingStrategy strategy;
  private Worker worker;
  private Capability capability;

  @BeforeEach
  void setUp() {
    strategy = new FirstSupportedRoutingStrategy();
    capability = new Capability("test-cap", null, null);
    worker =
        Worker.builder()
            .name("test-worker")
            .capability(capability)
            .function(input -> WorkerResult.of(Map.of()))
            .build();
  }

  @Test
  void selectsFirstSupportingBackend() {
    WorkerExecutionManager backend1 = mockBackend(false);
    WorkerExecutionManager backend2 = mockBackend(true);
    WorkerExecutionManager backend3 = mockBackend(true);

    Optional<WorkerExecutionManager> result =
        strategy.select(List.of(backend1, backend2, backend3), worker, capability, "tenant-1");

    assertThat(result).isPresent().containsSame(backend2);
  }

  @Test
  void returnsEmptyWhenNoBackendSupports() {
    WorkerExecutionManager backend1 = mockBackend(false);
    WorkerExecutionManager backend2 = mockBackend(false);

    Optional<WorkerExecutionManager> result =
        strategy.select(List.of(backend1, backend2), worker, capability, "tenant-1");

    assertThat(result).isEmpty();
  }

  @Test
  void returnsEmptyForEmptyCandidateList() {
    Optional<WorkerExecutionManager> result =
        strategy.select(List.of(), worker, capability, "tenant-1");

    assertThat(result).isEmpty();
  }

  @Test
  void respectsCandidateOrdering() {
    WorkerExecutionManager highPriority = mockBackend(true);
    WorkerExecutionManager lowPriority = mockBackend(true);

    Optional<WorkerExecutionManager> result =
        strategy.select(List.of(highPriority, lowPriority), worker, capability, "tenant-1");

    assertThat(result).isPresent().containsSame(highPriority);
  }

  private WorkerExecutionManager mockBackend(boolean supports) {
    WorkerExecutionManager mock = mock(WorkerExecutionManager.class);
    when(mock.supports("test-cap", "tenant-1")).thenReturn(supports);
    return mock;
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=FirstSupportedRoutingStrategyTest -q`

Expected: FAIL — `FirstSupportedRoutingStrategy` class does not exist.

- [ ] **Step 3: Implement `FirstSupportedRoutingStrategy`**

Create `runtime/src/main/java/io/casehub/engine/internal/routing/FirstSupportedRoutingStrategy.java`:

```java
package io.casehub.engine.internal.routing;

import io.casehub.engine.common.spi.scheduler.WorkerExecutionManager;
import io.casehub.engine.common.spi.scheduler.WorkerExecutionRoutingStrategy;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class FirstSupportedRoutingStrategy implements WorkerExecutionRoutingStrategy {

  @Override
  public Optional<WorkerExecutionManager> select(
      List<WorkerExecutionManager> candidates,
      Worker worker,
      Capability capability,
      String tenancyId) {
    for (WorkerExecutionManager candidate : candidates) {
      if (candidate.supports(capability.name(), tenancyId)) {
        return Optional.of(candidate);
      }
    }
    return Optional.empty();
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=FirstSupportedRoutingStrategyTest -q`

Expected: BUILD SUCCESS — all 4 tests pass.

- [ ] **Step 5: Commit**

```
feat(#461): add FirstSupportedRoutingStrategy default routing

@DefaultBean @ApplicationScoped — iterates candidates in priority order,
returns first where supports() is true.

Refs #461
```

---

### Task 3: Composite WorkerExecutionManager (engine runtime)

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/CompositeWorkerExecutionManager.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/worker/CompositeWorkerExecutionManagerTest.java`
- Delete: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerExecutionManager.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java` — remove 4 NoOp WEM tests

**Consumes:** `WorkerBackend`, `WorkerExecutionRoutingStrategy`, `WorkerExecutionManager` from Tasks 1-2
**Produces:** `CompositeWorkerExecutionManager` — `@ApplicationScoped`, replaces `NoOpWorkerExecutionManager`

- [ ] **Step 1: Write failing tests**

Create `runtime/src/test/java/io/casehub/engine/internal/worker/CompositeWorkerExecutionManagerTest.java`:

```java
package io.casehub.engine.internal.worker;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.scheduler.WorkerBackend;
import io.casehub.engine.common.spi.scheduler.WorkerExecutionManager;
import io.casehub.engine.common.spi.scheduler.WorkerExecutionRoutingStrategy;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.helpers.test.UniAssertSubscriber;
import jakarta.annotation.Priority;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CompositeWorkerExecutionManagerTest {

  private CompositeWorkerExecutionManager composite;
  private WorkerExecutionRoutingStrategy strategy;
  private WorkerExecutionManager backend1;
  private WorkerExecutionManager backend2;
  private Worker worker;
  private Capability capability;
  private CaseInstance instance;

  @BeforeEach
  void setUp() {
    strategy = mock(WorkerExecutionRoutingStrategy.class);
    backend1 = mock(WorkerExecutionManager.class);
    backend2 = mock(WorkerExecutionManager.class);

    capability = new Capability("test-cap", null, null);
    worker =
        Worker.builder()
            .name("test-worker")
            .capability(capability)
            .function(input -> WorkerResult.of(Map.of()))
            .build();

    instance = new CaseInstance();
    instance.tenancyId = "tenant-1";
    instance.setUuid(UUID.randomUUID());

    composite = new CompositeWorkerExecutionManager(strategy, List.of(backend1, backend2));
  }

  @Test
  void submit_routesToSelectedBackend() {
    when(strategy.select(any(), any(), any(), anyString())).thenReturn(Optional.of(backend2));
    when(backend2.submit(any(), any(), any(), any(), any()))
        .thenReturn(Uni.createFrom().voidItem());

    composite
        .submit(1L, instance, worker, capability, Map.of())
        .subscribe()
        .withSubscriber(UniAssertSubscriber.create())
        .assertCompleted();

    verify(backend2).submit(1L, instance, worker, capability, Map.of());
  }

  @Test
  void submit_throwsProvisioningExceptionWhenNoBackendSelected() {
    when(strategy.select(any(), any(), any(), anyString())).thenReturn(Optional.empty());

    assertThatThrownBy(
            () ->
                composite
                    .submit(1L, instance, worker, capability, Map.of())
                    .await()
                    .indefinitely())
        .hasCauseInstanceOf(
            io.casehub.api.spi.ProvisioningException.class);
  }

  @Test
  void submit_throwsProvisioningExceptionWhenNoBackendsDiscovered() {
    CompositeWorkerExecutionManager empty =
        new CompositeWorkerExecutionManager(strategy, List.of());

    assertThatThrownBy(
            () ->
                empty
                    .submit(1L, instance, worker, capability, Map.of())
                    .await()
                    .indefinitely())
        .hasCauseInstanceOf(
            io.casehub.api.spi.ProvisioningException.class);
  }

  @Test
  void getActiveWorkCount_sumsAcrossBackends() {
    when(backend1.getActiveWorkCount("w1")).thenReturn(3);
    when(backend2.getActiveWorkCount("w1")).thenReturn(5);

    assertThat(composite.getActiveWorkCount("w1")).isEqualTo(8);
  }

  @Test
  void getActiveCaseIds_unionsAcrossBackends() {
    UUID id1 = UUID.randomUUID();
    UUID id2 = UUID.randomUUID();
    when(backend1.getActiveCaseIds("w1")).thenReturn(List.of(id1));
    when(backend2.getActiveCaseIds("w1")).thenReturn(List.of(id2));

    assertThat(composite.getActiveCaseIds("w1")).containsExactlyInAnyOrder(id1, id2);
  }

  @Test
  void supports_returnsTrueIfAnyBackendSupports() {
    when(backend1.supports("cap-a", "t1")).thenReturn(false);
    when(backend2.supports("cap-a", "t1")).thenReturn(true);

    assertThat(composite.supports("cap-a", "t1")).isTrue();
  }

  @Test
  void supports_returnsFalseWhenNoneSupport() {
    when(backend1.supports("cap-a", "t1")).thenReturn(false);
    when(backend2.supports("cap-a", "t1")).thenReturn(false);

    assertThat(composite.supports("cap-a", "t1")).isFalse();
  }

  @Test
  void schedulePersistedEvent_delegatesToAllBackends() {
    EventLog eventLog = new EventLog();
    when(backend1.schedulePersistedEvent(eventLog)).thenReturn(Uni.createFrom().voidItem());
    when(backend2.schedulePersistedEvent(eventLog)).thenReturn(Uni.createFrom().voidItem());

    composite
        .schedulePersistedEvent(eventLog)
        .subscribe()
        .withSubscriber(UniAssertSubscriber.create())
        .assertCompleted();

    verify(backend1).schedulePersistedEvent(eventLog);
    verify(backend2).schedulePersistedEvent(eventLog);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=CompositeWorkerExecutionManagerTest -q`

Expected: FAIL — `CompositeWorkerExecutionManager` class does not exist.

- [ ] **Step 3: Implement `CompositeWorkerExecutionManager`**

Create `runtime/src/main/java/io/casehub/engine/internal/worker/CompositeWorkerExecutionManager.java`:

```java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.ProvisioningException;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.scheduler.WorkerBackend;
import io.casehub.engine.common.spi.scheduler.WorkerExecutionManager;
import io.casehub.engine.common.spi.scheduler.WorkerExecutionRoutingStrategy;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.Worker;
import io.smallrye.mutiny.Uni;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class CompositeWorkerExecutionManager implements WorkerExecutionManager {

  private static final Logger LOG = Logger.getLogger(CompositeWorkerExecutionManager.class);

  private final WorkerExecutionRoutingStrategy routingStrategy;
  private final List<WorkerExecutionManager> backends;

  @Inject
  public CompositeWorkerExecutionManager(
      WorkerExecutionRoutingStrategy routingStrategy,
      @WorkerBackend Instance<WorkerExecutionManager> discoveredBackends) {
    this.routingStrategy = routingStrategy;
    this.backends = sortByPriority(discoveredBackends);
    LOG.infof("CompositeWorkerExecutionManager initialized with %d backend(s)", backends.size());
  }

  CompositeWorkerExecutionManager(
      WorkerExecutionRoutingStrategy routingStrategy,
      List<WorkerExecutionManager> backends) {
    this.routingStrategy = routingStrategy;
    this.backends = List.copyOf(backends);
  }

  @Override
  public boolean supports(String capabilityName, String tenancyId) {
    for (WorkerExecutionManager backend : backends) {
      if (backend.supports(capabilityName, tenancyId)) {
        return true;
      }
    }
    return false;
  }

  @Override
  public Uni<Void> submit(
      Long eventLogId,
      CaseInstance instance,
      Worker worker,
      Capability capability,
      Map<String, Object> inputData) {
    if (backends.isEmpty()) {
      return Uni.createFrom()
          .failure(
              new ProvisioningException(
                  "No WorkerExecutionManager backend configured"));
    }
    Optional<WorkerExecutionManager> selected =
        routingStrategy.select(backends, worker, capability, instance.tenancyId);
    if (selected.isEmpty()) {
      return Uni.createFrom()
          .failure(
              new ProvisioningException(
                  "No backend supports capability '"
                      + capability.name()
                      + "' for tenant '"
                      + instance.tenancyId
                      + "'"));
    }
    return selected.get().submit(eventLogId, instance, worker, capability, inputData);
  }

  @Override
  public Uni<Void> schedulePersistedEvent(EventLog scheduledEventLog) {
    List<Uni<Void>> results = new ArrayList<>();
    for (WorkerExecutionManager backend : backends) {
      results.add(backend.schedulePersistedEvent(scheduledEventLog));
    }
    if (results.isEmpty()) {
      return Uni.createFrom().voidItem();
    }
    return Uni.join().all(results).andFailFast().replaceWithVoid();
  }

  @Override
  public int getActiveWorkCount(String workerId) {
    int total = 0;
    for (WorkerExecutionManager backend : backends) {
      total += backend.getActiveWorkCount(workerId);
    }
    return total;
  }

  @Override
  public List<UUID> getActiveCaseIds(String workerId) {
    List<UUID> all = new ArrayList<>();
    for (WorkerExecutionManager backend : backends) {
      all.addAll(backend.getActiveCaseIds(workerId));
    }
    return Collections.unmodifiableList(all);
  }

  private static List<WorkerExecutionManager> sortByPriority(
      Instance<WorkerExecutionManager> instances) {
    List<WorkerExecutionManager> sorted = new ArrayList<>();
    for (WorkerExecutionManager wem : instances) {
      sorted.add(wem);
    }
    sorted.sort(
        Comparator.comparingInt(
                (WorkerExecutionManager wem) -> {
                  Priority p = wem.getClass().getAnnotation(Priority.class);
                  return p != null ? p.value() : 0;
                })
            .reversed());
    return Collections.unmodifiableList(sorted);
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=CompositeWorkerExecutionManagerTest -q`

Expected: BUILD SUCCESS — all 8 tests pass.

- [ ] **Step 5: Delete `NoOpWorkerExecutionManager` and update its tests**

Delete `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerExecutionManager.java`.

In `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java`, remove these 4 test methods:
- `noOpWorkerExecutionManager_submit_uniFails_withProvisioningException`
- `noOpWorkerExecutionManager_schedulePersistedEvent_completesWithoutException`
- `noOpWorkerExecutionManager_getActiveWorkCount_returnsZero`
- `noOpWorkerExecutionManager_getActiveCaseIds_returnsEmptyList`

Also remove any `NoOpWorkerExecutionManager` import and instantiation.

- [ ] **Step 6: Verify remaining tests still pass**

Run: `mvn test -pl runtime -Dtest=DefaultWorkerSpiImplementationsTest -q`

Expected: BUILD SUCCESS — remaining tests unaffected.

- [ ] **Step 7: Commit**

```
feat(#461): add CompositeWorkerExecutionManager, remove NoOpWorkerExecutionManager

Composite routes submit() via WorkerExecutionRoutingStrategy, aggregates
getActiveWorkCount/getActiveCaseIds across all backends, delegates
schedulePersistedEvent to all.

Replaces NoOpWorkerExecutionManager — empty-backends case handled
internally with ProvisioningException.

Refs #461
```

---

### Task 4: Quartz Backend Migration (scheduler-quartz)

**Files:**
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java`
- Modify: `scheduler-quartz/src/test/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManagerTest.java`

- [ ] **Step 1: Write failing `supports()` test**

Add to `QuartzWorkerExecutionManagerTest.java`:

```java
@Test
void supports_alwaysReturnsTrue() {
    assertThat(manager.supports("any-capability", "any-tenant")).isTrue();
    assertThat(manager.supports("another-cap", null)).isTrue();
    assertThat(manager.supports("", "")).isTrue();
}
```

Where `manager` is the `QuartzWorkerExecutionManager` under test (already instantiated in the test class).

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl scheduler-quartz -Dtest=QuartzWorkerExecutionManagerTest#supports_alwaysReturnsTrue -q`

Expected: FAIL — `supports()` not implemented.

- [ ] **Step 3: Add annotations and implement `supports()`**

In `QuartzWorkerExecutionManager.java`:

Add annotations to the class declaration:
```java
@WorkerBackend
@Priority(0)
@ApplicationScoped
public class QuartzWorkerExecutionManager implements WorkerExecutionManager {
```

Add imports:
```java
import io.casehub.engine.common.spi.scheduler.WorkerBackend;
import jakarta.annotation.Priority;
```

Add the `supports()` method:
```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return true;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl scheduler-quartz -Dtest=QuartzWorkerExecutionManagerTest -q`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
feat(#461): migrate QuartzWorkerExecutionManager to @WorkerBackend

@WorkerBackend @Priority(0) — catch-all fallback for in-process execution.
supports() returns true for all capabilities.

Refs #461
```

---

### Task 5: Publish Engine and Verify Build

**Produces:** Engine artifacts in local Maven repo for workers/claudony compilation

- [ ] **Step 1: Install engine to local repo**

Run: `mvn install -DskipTests -q` (from engine root)

Expected: BUILD SUCCESS — all engine modules published to `~/.m2/repository`.

- [ ] **Step 2: Verify no compilation errors across engine**

Run: `mvn compile -q` (from engine root)

Expected: BUILD SUCCESS.

- [ ] **Step 3: Run full engine test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -q` (from engine root)

Expected: BUILD SUCCESS. If any tests fail due to `supports()` being abstract, fix them (likely test stubs or mocks that need updating).

- [ ] **Step 4: Commit any test fixes**

If any tests needed updating for the abstract `supports()` method, commit:

```
fix(#461): update test stubs for abstract supports() method

Refs #461
```

---

### Task 6: Workers-Common canResolve() Default (workers repo)

**Files:**
- Modify: `workers-common/src/main/java/io/casehub/workers/common/WorkerCapabilityResolver.java`

**Produces:** `canResolve(String, String): boolean` default method on the resolver interface

- [ ] **Step 1: Add `canResolve()` default method**

In `WorkerCapabilityResolver.java`, add:

```java
default boolean canResolve(String capabilityTag, String tenancyId) {
    return capabilities().contains(capabilityTag);
}
```

This provides a baseline for all resolvers. Backends with fallback logic (EndpointRegistry) override with richer checks.

- [ ] **Step 2: Commit**

```
feat(#461): add canResolve() default to WorkerCapabilityResolver

Boolean, no-throw capability check used by WorkerExecutionManager.supports().
Default: delegates to capabilities().contains(). Backends with fallback
logic (EndpointRegistry) override.

Refs #461
```

---

### Task 7: Camel Backend Migration (workers repo)

**Files:**
- Modify: `workers-camel/src/main/java/io/casehub/workers/camel/CamelWorkerExecutionManager.java`
- Modify: `workers-camel/src/test/java/io/casehub/workers/camel/CamelWorkerExecutionManagerTest.java`

- [ ] **Step 1: Write failing `supports()` test**

Add to `CamelWorkerExecutionManagerTest.java`:

```java
@Test
void supports_returnsTrueForRegisteredCapability() {
    // camelCapabilityResolver is already set up in the test with known capabilities
    assertThat(manager.supports("registered-cap", "tenant-1")).isTrue();
}

@Test
void supports_returnsFalseForUnregisteredCapability() {
    assertThat(manager.supports("unknown-cap", "tenant-1")).isFalse();
}
```

Adjust the capability name to match whatever the existing test setup registers. Read the existing test `@BeforeEach` to find the registered capability name.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl workers-camel -Dtest=CamelWorkerExecutionManagerTest#supports_returnsTrueForRegisteredCapability -q`

Expected: FAIL — `supports()` not implemented.

- [ ] **Step 3: Add annotations and implement `supports()`**

In `CamelWorkerExecutionManager.java`:

Change annotations:
```java
@WorkerBackend
@Priority(10)
@ApplicationScoped
public class CamelWorkerExecutionManager implements WorkerExecutionManager {
```

Add imports:
```java
import io.casehub.engine.common.spi.scheduler.WorkerBackend;
import jakarta.annotation.Priority;
```

Add `supports()`:
```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return camelCapabilityResolver.canResolve(capabilityName, tenancyId);
}
```

Remove the `schedulePersistedEvent()` override (now inherits the default no-op).

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl workers-camel -Dtest=CamelWorkerExecutionManagerTest -q`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
feat(#461): migrate CamelWorkerExecutionManager to @WorkerBackend

@WorkerBackend @Priority(10). supports() delegates to
CamelCapabilityResolver.canResolve(). Removed schedulePersistedEvent()
override (inherits default no-op).

Refs #461
```

---

### Task 8: HTTP Backend Migration (workers repo)

**Files:**
- Modify: `workers-http/src/main/java/io/casehub/workers/http/HttpWorkerExecutionManager.java`
- Modify: `workers-http/src/main/java/io/casehub/workers/http/HttpEndpointResolver.java` (override `canResolve()` for EndpointRegistry fallback)
- Modify: `workers-http/src/test/java/io/casehub/workers/http/HttpWorkerExecutionManagerTest.java`

- [ ] **Step 1: Write failing `supports()` test**

Add to `HttpWorkerExecutionManagerTest.java`:

```java
@Test
void supports_returnsTrueForRegisteredCapability() {
    assertThat(manager.supports("registered-cap", "tenant-1")).isTrue();
}

@Test
void supports_returnsFalseForUnregisteredCapability() {
    assertThat(manager.supports("unknown-cap", "tenant-1")).isFalse();
}
```

Adjust capability name to match the test setup.

- [ ] **Step 2: Override `canResolve()` in `HttpEndpointResolver`**

`HttpEndpointResolver` uses an `EndpointRegistry` fallback beyond its `resolvedEndpoints` map. Override `canResolve()` to check both:

```java
@Override
public boolean canResolve(String capabilityTag, String tenancyId) {
    if (resolvedEndpoints.containsKey(capabilityTag)) {
        return true;
    }
    if (registry != null) {
        return registry.lookup(capabilityTag, tenancyId).isPresent();
    }
    return false;
}
```

Verify this matches the logic in `resolve()` — the `canResolve()` must return `true` for exactly the same inputs that `resolve()` would succeed on.

- [ ] **Step 3: Add annotations and implement `supports()` in `HttpWorkerExecutionManager`**

Same pattern as Camel:

```java
@WorkerBackend
@Priority(10)
@ApplicationScoped
public class HttpWorkerExecutionManager implements WorkerExecutionManager {
```

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return httpEndpointResolver.canResolve(capabilityName, tenancyId);
}
```

Remove `schedulePersistedEvent()` override.

- [ ] **Step 4: Run tests**

Run: `mvn test -pl workers-http -Dtest=HttpWorkerExecutionManagerTest -q`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
feat(#461): migrate HttpWorkerExecutionManager to @WorkerBackend

@WorkerBackend @Priority(10). supports() delegates to
HttpEndpointResolver.canResolve() (checks map + EndpointRegistry fallback).

Refs #461
```

---

### Task 9: Script, MCP, GitHub Actions Backend Migrations (workers repo)

Same pattern for each. All three follow the Camel pattern (no EndpointRegistry override needed for Script; MCP needs the registry override like HTTP; GitHub Actions uses constant matching).

**Files:**
- Modify: `workers-script/src/main/java/io/casehub/workers/script/ScriptWorkerExecutionManager.java`
- Modify: `workers-mcp/src/main/java/io/casehub/workers/mcp/McpWorkerExecutionManager.java`
- Modify: `workers-mcp/src/main/java/io/casehub/workers/mcp/McpServerResolver.java` (override `canResolve()` for EndpointRegistry)
- Modify: `workers-github-actions/src/main/java/io/casehub/workers/githubactions/GitHubActionsWorkerExecutionManager.java`
- Modify: corresponding test files

- [ ] **Step 1: Migrate ScriptWorkerExecutionManager**

Add `@WorkerBackend @Priority(10)`, implement `supports()`:

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return scriptDefinitionResolver.canResolve(capabilityName, tenancyId);
}
```

Remove `schedulePersistedEvent()` override. Add `supports()` tests.

- [ ] **Step 2: Migrate McpWorkerExecutionManager**

Add `@WorkerBackend @Priority(10)`, implement `supports()`:

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return mcpServerResolver.canResolve(capabilityName, tenancyId);
}
```

Override `canResolve()` in `McpServerResolver` to check `capabilityToServerName` map + EndpointRegistry:

```java
@Override
public boolean canResolve(String capabilityTag, String tenancyId) {
    if (capabilityToServerName.containsKey(capabilityTag)) {
        return true;
    }
    String serverName = parseServerName(capabilityTag);
    if (registry != null) {
        return registry.lookup(serverName, tenancyId).isPresent();
    }
    return false;
}
```

Remove `schedulePersistedEvent()` override. Add `supports()` tests.

- [ ] **Step 3: Migrate GitHubActionsWorkerExecutionManager**

Add `@WorkerBackend @Priority(10)`, implement `supports()`:

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return GitHubActionsWorkerConstants.CAPABILITY_WORKFLOW_DISPATCH.equals(capabilityName)
        || GitHubActionsWorkerConstants.CAPABILITY_REPOSITORY_DISPATCH.equals(capabilityName);
}
```

Remove `schedulePersistedEvent()` override. Add `supports()` tests.

- [ ] **Step 4: Run all worker tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl workers-script,workers-mcp,workers-github-actions -q`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
feat(#461): migrate Script, MCP, GitHub Actions backends to @WorkerBackend

All three: @WorkerBackend @Priority(10), supports() via resolver.
McpServerResolver.canResolve() checks map + EndpointRegistry fallback.
GitHubActionsWorkerExecutionManager matches known capability constants.

Refs #461
```

---

### Task 10: Claudony Migration (claudony repo)

**Files:**
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyWorkerExecutionManager.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyReactiveWorkerProvisioner.java`
- Modify: `casehub/src/main/java/io/casehub/claudony/casehub/ClaudonyLedgerEventCapture.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/CasehubStartupService.java`
- Modify: `app/src/main/java/io/casehub/claudony/server/ServerStartup.java`
- Modify: `app/src/test/java/io/casehub/claudony/NoOpWorkerExecutionManager.java` (test stub)

- [ ] **Step 1: Install engine + workers to local repo**

Run from engine root: `mvn install -DskipTests -q`
Run from workers root: `mvn install -DskipTests -q`

- [ ] **Step 2: Add `@WorkerBackend @Priority(10)` and `supports()` to `ClaudonyWorkerExecutionManager`**

```java
@WorkerBackend
@Priority(10)
@ApplicationScoped
public class ClaudonyWorkerExecutionManager implements WorkerExecutionManager {
```

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    // Claudony handles all capabilities when deployed — it IS the worker execution backend
    return true;
}
```

Add imports for `WorkerBackend` and `Priority`.

Remove `schedulePersistedEvent()` override if it's a no-op.

- [ ] **Step 3: Fix concrete-type injection sites — add `@WorkerBackend`**

In `ClaudonyReactiveWorkerProvisioner.java` (constructor param):
```java
@WorkerBackend ClaudonyWorkerExecutionManager execManager,
```

In `ClaudonyLedgerEventCapture.java` (field injection):
```java
@Inject @WorkerBackend ClaudonyWorkerExecutionManager execManager;
```

In `CasehubStartupService.java` (constructor param):
```java
@WorkerBackend ClaudonyWorkerExecutionManager execManager) {
```

In `ServerStartup.java` (Instance injection):
```java
@Inject @WorkerBackend Instance<ClaudonyWorkerExecutionManager> workerExecManager;
```

- [ ] **Step 4: Update test stub `NoOpWorkerExecutionManager`**

In `app/src/test/java/io/casehub/claudony/NoOpWorkerExecutionManager.java`, add:

```java
@Override
public boolean supports(String capabilityName, String tenancyId) {
    return false;
}
```

- [ ] **Step 5: Verify claudony compiles and tests pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -q` (from claudony root)

Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```
feat(#461): migrate ClaudonyWorkerExecutionManager to @WorkerBackend

@WorkerBackend @Priority(10). Fixed 4 concrete-type injection sites
that need @WorkerBackend qualifier (CDI §2.3.1 — adding a qualifier
removes implicit @Default).

Refs #461
```

---

### Task 11: Protocol Update and CLAUDE.md Sync

**Files:**
- Modify: `CLAUDE.md` in engine repo — update NoOpWorkerExecutionManager references

- [ ] **Step 1: Update CLAUDE.md**

In the `## Worker Provisioner SPIs` section, update the `NoOpWorkerExecutionManager` entry to reference `CompositeWorkerExecutionManager`:

Replace the `NoOpWorkerExecutionManager` line in the no-op defaults list with:
```
- `CompositeWorkerExecutionManager` — `@ApplicationScoped` (not `@DefaultBean`); replaces `NoOpWorkerExecutionManager`; routes `submit()` via `WorkerExecutionRoutingStrategy`, aggregates monitoring methods across `@WorkerBackend` backends
```

Add `FirstSupportedRoutingStrategy` to the defaults list:
```
- `FirstSupportedRoutingStrategy` — `@DefaultBean @ApplicationScoped` in `runtime/internal/routing/`; iterates candidates by `@Priority` (descending), returns first where `supports()` is true
```

- [ ] **Step 2: Commit**

```
docs(#461): update CLAUDE.md for composite WorkerExecutionManager

Replace NoOpWorkerExecutionManager references with
CompositeWorkerExecutionManager. Add FirstSupportedRoutingStrategy
to defaults list.

Refs #461
```

---

## Execution Order Summary

| Task | Repo | Depends On | Can Parallelize With |
|------|------|-----------|---------------------|
| 1 | engine | — | — |
| 2 | engine | 1 | — |
| 3 | engine | 1, 2 | — |
| 4 | engine | 1 | 2, 3 |
| 5 | engine | 1-4 | — |
| 6 | workers | 5 | — |
| 7 | workers | 5, 6 | — |
| 8 | workers | 5, 6 | 7 |
| 9 | workers | 5, 6 | 7, 8 |
| 10 | claudony | 5 | 7, 8, 9 |
| 11 | engine | 3 | 7-10 |
