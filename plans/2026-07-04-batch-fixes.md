# Batch Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement 6 engine issues (#629, #636, #637, #641, #642, #643) covering health check semantics, WorkerRuntime case orchestration, PlanItem concurrency safety, gate evaluation placement, strategy resolver determinism, and platform documentation.

**Architecture:** Each issue is an independent concern — task ordering follows dependency: #629 (no deps) → #637 (PlanItem CAS, no deps) → #642 (resolver, no deps) → #641 (gate eval, no deps) → #636 (spawnCase/awaitCase, depends on CaseStatusChangedHandler changes) → #643 (docs, depends on all code being final). TDD throughout — tests written before implementation for all Java code.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny, Vert.x event bus, AtomicReference, CompletableFuture, Quarkus ARC InjectableBean API.

## Global Constraints

- All tests are `*Test.java` (never `*IT.java` — picked up by failsafe instead of surefire)
- Maven module tests: `mvn install -DskipTests -q` before module-specific `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl <module>`
- SPI additions use `default` methods with safe no-op returns (PP-spi-evolution-default-methods)
- Engine SPI no-op defaults use `@DefaultBean` (PP-engine-spi-noops-defaultbean)
- All commits reference an issue: `Refs #N` or `Closes #N`
- Breaking changes are preferred over backward-compatibility shims (PP-no-workarounds-fix-the-design)

---

### Task 1: WorkerRecoveryHealthCheck @Readiness (#629)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/WorkerRecoveryHealthCheck.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/recovery/WorkerRecoveryHealthCheckTest.java`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: nothing
- Produces: nothing (annotation-only change, no API surface affected)

- [ ] **Step 1: Verify current test passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=WorkerRecoveryHealthCheckTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 3 tests pass

- [ ] **Step 2: Change @Liveness to @Readiness**

In `runtime/src/main/java/io/casehub/engine/internal/engine/recovery/WorkerRecoveryHealthCheck.java`:

Replace import:
```java
// Before
import org.eclipse.microprofile.health.Liveness;
// After
import org.eclipse.microprofile.health.Readiness;
```

Replace annotation:
```java
// Before
@Liveness
@ApplicationScoped
// After
@Readiness
@ApplicationScoped
```

- [ ] **Step 3: Run test to verify it still passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=WorkerRecoveryHealthCheckTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 3 tests pass (the test checks `HealthCheckResponse` behavior, not the annotation — no test changes needed)

- [ ] **Step 4: Update CLAUDE.md**

Find the line mentioning `WorkerRecoveryHealthCheck` and `@Liveness`. Change `@Liveness` to `@Readiness`. The line is in the "Worker Execution Architecture" section:

```
`WorkerRecoveryHealthCheck` (`@Liveness`) reports the status at `/q/health/live`.
```
→
```
`WorkerRecoveryHealthCheck` (`@Readiness`) reports the status at `/q/health/ready`.
```

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/engine/recovery/WorkerRecoveryHealthCheck.java CLAUDE.md
git commit -m "refactor(#629): WorkerRecoveryHealthCheck @Liveness → @Readiness

Library modules should not define liveness checks — liveness is a
deployment-layer concern. A failed recovery doesn't mean the JVM is
hung; the engine is still serving requests. Readiness is the correct
semantic: divert traffic until recovery completes.

Closes #629"
```

---

### Task 2: PlanItem CAS Guard (#637)

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java` (create if absent)
- Modify: `blackboard/src/test/java/io/casehub/blackboard/control/PlanningStrategyLoopControlTest.java` (create if absent)

**Interfaces:**
- Consumes: nothing
- Produces: `PlanItem.tryMarkRunning(): boolean` — CAS PENDING→RUNNING, returns true only if caller won the race. `PlanItem.getStatus(): PlanItemStatus` — now reads from `AtomicReference`. All existing `mark*()` methods unchanged in signature but use `AtomicReference` internally.

- [ ] **Step 1: Write PlanItem CAS tests**

Create `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java`:

```java
package io.casehub.blackboard.plan;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.engine.common.internal.model.PlanItemStatus;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.Test;

class PlanItemTest {

  @Test
  void tryMarkRunning_fromPending_returnsTrue() {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    assertTrue(item.tryMarkRunning());
    assertEquals(PlanItemStatus.RUNNING, item.getStatus());
  }

  @Test
  void tryMarkRunning_fromRunning_returnsFalse() {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    item.markRunning();
    assertFalse(item.tryMarkRunning());
  }

  @Test
  void tryMarkRunning_fromCompleted_returnsFalse() {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    item.markRunning();
    item.markCompleted();
    assertFalse(item.tryMarkRunning());
  }

  @Test
  void tryMarkRunning_concurrentCallers_exactlyOneWins() throws Exception {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    int threadCount = 10;
    CountDownLatch ready = new CountDownLatch(threadCount);
    CountDownLatch go = new CountDownLatch(1);
    AtomicInteger wins = new AtomicInteger(0);

    Thread[] threads = new Thread[threadCount];
    for (int i = 0; i < threadCount; i++) {
      threads[i] = Thread.ofVirtual().start(() -> {
        ready.countDown();
        try { go.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        if (item.tryMarkRunning()) {
          wins.incrementAndGet();
        }
      });
    }
    ready.await();
    go.countDown();
    for (Thread t : threads) t.join();

    assertEquals(1, wins.get());
    assertEquals(PlanItemStatus.RUNNING, item.getStatus());
  }

  @Test
  void markRunning_fromPending_succeeds() {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    item.markRunning();
    assertEquals(PlanItemStatus.RUNNING, item.getStatus());
  }

  @Test
  void markRunning_fromRunning_throws() {
    PlanItem item = PlanItem.create("b1", "w1", 0, null);
    item.markRunning();
    assertThrows(IllegalStateException.class, item::markRunning);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard -Dtest=PlanItemTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: `tryMarkRunning` tests FAIL (method does not exist)

- [ ] **Step 3: Convert PlanItem to AtomicReference and add tryMarkRunning()**

In `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`:

Add import:
```java
import java.util.concurrent.atomic.AtomicReference;
```

Replace field:
```java
// Before
private volatile PlanItemStatus status;

// After
private final AtomicReference<PlanItemStatus> status;
```

Update both constructors to initialize the AtomicReference:
```java
// In first constructor (line 44):
this.status = new AtomicReference<>(PlanItemStatus.PENDING);

// In second constructor (line 56, restore path):
this.status = new AtomicReference<>(status);
```

Update `getStatus()`:
```java
public PlanItemStatus getStatus() {
    return status.get();
}
```

Add `tryMarkRunning()`:
```java
public boolean tryMarkRunning() {
    return status.compareAndSet(PlanItemStatus.PENDING, PlanItemStatus.RUNNING);
}
```

Update `markRunning()` to use CAS:
```java
public void markRunning() {
    if (!status.compareAndSet(PlanItemStatus.PENDING, PlanItemStatus.RUNNING)) {
        throw new IllegalStateException(
            "Cannot transition to RUNNING from " + status.get() + " (planItemId=" + planItemId + ")");
    }
}
```

Update `markDelegated()`:
```java
public void markDelegated() {
    if (!status.compareAndSet(PlanItemStatus.PENDING, PlanItemStatus.DELEGATED)) {
        throw new IllegalStateException(
            "Cannot transition to DELEGATED from " + status.get() + " (planItemId=" + planItemId + ")");
    }
}
```

Update `markCompleted()`:
```java
public void markCompleted() {
    PlanItemStatus current = status.get();
    if (current != PlanItemStatus.RUNNING && current != PlanItemStatus.DELEGATED) {
        throw new IllegalStateException(
            "Cannot transition to COMPLETED from " + current + " (planItemId=" + planItemId + ")");
    }
    status.set(PlanItemStatus.COMPLETED);
}
```

Update `markFaulted()`:
```java
public void markFaulted() {
    PlanItemStatus current = status.get();
    if (current.isTerminal()) {
        throw new IllegalStateException(
            "Cannot fault a terminal PlanItem (status=" + current + ", planItemId=" + planItemId + ")");
    }
    status.set(PlanItemStatus.FAULTED);
}
```

Update `markRejected()`:
```java
public void markRejected() {
    if (!status.compareAndSet(PlanItemStatus.DELEGATED, PlanItemStatus.REJECTED)) {
        throw new IllegalStateException(
            "Cannot transition to REJECTED from " + status.get() + " (planItemId=" + planItemId + ")");
    }
}
```

Update `markObsolete()`:
```java
public void markObsolete() {
    PlanItemStatus current = status.get();
    if (current.isTerminal()) {
        throw new IllegalStateException(
            "Cannot obsolete a terminal PlanItem (status=" + current + ", planItemId=" + planItemId + ")");
    }
    status.set(PlanItemStatus.OBSOLETE);
}
```

Update `markSuspended()`:
```java
public void markSuspended() {
    if (!status.compareAndSet(PlanItemStatus.DELEGATED, PlanItemStatus.SUSPENDED)) {
        throw new IllegalStateException(
            "Cannot suspend from " + status.get() + " (planItemId=" + planItemId + ")");
    }
}
```

Update `markResumed()`:
```java
public void markResumed() {
    if (!status.compareAndSet(PlanItemStatus.SUSPENDED, PlanItemStatus.DELEGATED)) {
        throw new IllegalStateException(
            "Cannot resume from " + status.get() + " (planItemId=" + planItemId + ")");
    }
}
```

Update `markCancelled()`:
```java
public void markCancelled() {
    PlanItemStatus current = status.get();
    if (current.isTerminal()) {
        throw new IllegalStateException(
            "Cannot cancel a terminal PlanItem (status=" + current + ", planItemId=" + planItemId + ")");
    }
    status.set(PlanItemStatus.CANCELLED);
}
```

- [ ] **Step 4: Run PlanItem tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard -Dtest=PlanItemTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 6 tests pass

- [ ] **Step 5: Merge filterToDispatchable + indexSelectedForCompletion in PlanningStrategyLoopControl**

In `blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java`:

Replace the terminal `.map()` + `.invoke()` chain (around line 165-166):
```java
// Before
        .map(selected -> filterToDispatchable(plan, selected))
        .invoke(dispatchable -> indexSelectedForCompletion(caseId, dispatchable, plan));

// After
        .map(selected -> filterAndIndexForDispatch(caseId, plan, selected));
```

Replace both `filterToDispatchable()` and `indexSelectedForCompletion()` methods with a single method:

```java
private List<Binding> filterAndIndexForDispatch(
    UUID caseId, CasePlanModel plan, List<Binding> selected) {
  List<Binding> dispatched = new ArrayList<>();
  for (Binding binding : selected) {
    Optional<PlanItem> piOpt = plan.getPlanItemByBindingName(binding.getName());
    if (piOpt.isEmpty()) {
      dispatched.add(binding);
      continue;
    }
    PlanItem pi = piOpt.get();
    if (binding.target() instanceof CapabilityTarget) {
      if (pi.tryMarkRunning()) {
        registry.indexForCompletion(caseId, pi.getWorkerName(), pi.getPlanItemId());
        dispatched.add(binding);
      }
    } else {
      if (pi.getStatus() == PlanItemStatus.PENDING) {
        dispatched.add(binding);
      }
    }
  }
  return dispatched;
}
```

Add import for `Optional` if not present:
```java
import java.util.Optional;
```

- [ ] **Step 6: Run all blackboard tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard`
Expected: All tests pass

- [ ] **Step 7: Run the integration test that exercises sequential strategy**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=HybridOrchestrationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Both tests pass (tier2_sequentialStrategy_firesBindingsOneAtATime and signalAndAwait_resolvesAfterWorkerCompletes)

- [ ] **Step 8: File follow-on issue for Option B (per-case serialization)**

```bash
gh issue create --repo casehubio/engine \
  --title "refactor: per-case CONTEXT_CHANGED serialization" \
  --label "scale: M,complexity: Med" \
  --body "$(cat <<'BODY'
## Context

engine#637 Option A (PlanItem CAS guard) prevents double-dispatch for CapabilityTarget bindings via atomic `tryMarkRunning()`. For non-CapabilityTarget bindings (HumanTask, SubCase, Extension), the handler owns the transition and the PENDING status check in `filterAndIndexForDispatch` remains a TOCTOU gap.

## What's needed

Serialize the `CONTEXT_CHANGED` evaluation pipeline per caseId to eliminate the race class for ALL target types. Options:

- `ConcurrentHashMap<UUID, ReentrantLock>` in `CaseContextChangedEventHandler`
- Vert.x ordered consumer per caseId
- Striped lock (hash-based, bounded lock count)

## Analysis required

- Re-entrant paths: handlers that re-publish `CONTEXT_CHANGED` (e.g., `WorkflowExecutionCompletedHandler` publishes `CONTEXT_CHANGED` after applying output). Lock must be re-entrant OR the re-publish must be deferred.
- Deadlock surface: nested case operations (`WorkerRuntime.spawnCase`) could create cross-case lock dependencies.
- Contention impact: high-throughput cases may queue excessive events.

## Scale
M

## Complexity
Med — concurrency design with re-entrancy and deadlock analysis
BODY
)"
```

- [ ] **Step 9: Commit**

```
git add blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java blackboard/src/main/java/io/casehub/blackboard/control/PlanningStrategyLoopControl.java blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java
git commit -m "fix(#637): PlanItem CAS guard — AtomicReference for concurrent dispatch safety

Replace volatile PlanItemStatus with AtomicReference. Add tryMarkRunning()
for atomic PENDING→RUNNING CAS in dispatch paths. Merge filterToDispatchable
and indexSelectedForCompletion into single atomic filterAndIndexForDispatch
step — only the thread that wins the CAS dispatches.

Existing markRunning() uses CAS too — consistent violation detection.
Non-CapabilityTarget bindings retain status check (handler owns transition).

Refs #637"
```

---

### Task 3: EngineStrategyResolver Default Determinism (#642)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/EngineStrategyResolverTest.java`

**Interfaces:**
- Consumes: `io.quarkus.arc.InjectableBean.isDefaultBean()` from Quarkus ARC
- Produces: Deterministic `defaultStrategy(type)` — `@DefaultBean` strategies always win regardless of CDI iteration order. Throws on duplicate `@DefaultBean` for same type.

- [ ] **Step 1: Write EngineStrategyResolver unit test**

Create `runtime/src/test/java/io/casehub/engine/internal/routing/EngineStrategyResolverTest.java`:

```java
package io.casehub.engine.internal.routing;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.platform.api.routing.NamedStrategy;
import io.casehub.platform.api.routing.StrategyResolver;
import jakarta.enterprise.inject.Instance;
import java.util.List;
import org.junit.jupiter.api.Test;

class EngineStrategyResolverTest {

  interface TestStrategy extends NamedStrategy {}

  static class StrategyA implements TestStrategy {
    @Override public String id() { return "a"; }
  }

  static class StrategyB implements TestStrategy {
    @Override public String id() { return "b"; }
  }

  @Test
  void defaultBean_winsRegardlessOfIterationOrder() {
    // StrategyA is NOT @DefaultBean, StrategyB IS @DefaultBean.
    // Even though A iterates first, B should be the default.
    var resolver = buildResolver(
        List.of(handle(new StrategyA(), false), handle(new StrategyB(), true)));

    TestStrategy defaultStrategy = resolver.defaultStrategy(TestStrategy.class);
    assertEquals("b", defaultStrategy.id());
  }

  @Test
  void noDefaultBean_fallsBackToFirst() {
    var resolver = buildResolver(
        List.of(handle(new StrategyA(), false), handle(new StrategyB(), false)));

    TestStrategy defaultStrategy = resolver.defaultStrategy(TestStrategy.class);
    assertEquals("a", defaultStrategy.id());
  }

  @Test
  void duplicateDefaultBean_throws() {
    assertThrows(IllegalStateException.class, () ->
        buildResolver(
            List.of(handle(new StrategyA(), true), handle(new StrategyB(), true))));
  }

  // Helper: build a resolver using the new registerStrategies method.
  // This requires making registerStrategies testable or using a test-visible constructor.
  // See Step 3 for the actual implementation approach.
  private EngineStrategyResolver buildResolver(
      List<TestHandle<? extends NamedStrategy>> handles) {
    return EngineStrategyResolver.forTest(handles);
  }

  // Minimal test handle
  record TestHandle<T extends NamedStrategy>(T strategy, boolean isDefaultBean) {}

  private <T extends NamedStrategy> TestHandle<T> handle(T strategy, boolean isDefaultBean) {
    return new TestHandle<>(strategy, isDefaultBean);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=EngineStrategyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `forTest` method does not exist

- [ ] **Step 3: Implement @DefaultBean detection in EngineStrategyResolver**

In `runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java`:

Add import:
```java
import io.quarkus.arc.InjectableBean;
```

Add a package-private record and static factory for testing:
```java
record StrategyEntry(NamedStrategy strategy, boolean isDefaultBean) {}

static EngineStrategyResolver forTest(
    List<? extends Object> handles) {
  var resolver = new EngineStrategyResolver();
  for (Object h : handles) {
    if (h instanceof EngineStrategyResolverTest.TestHandle<?> th) {
      resolver.registerEntry(th.strategy(), th.isDefaultBean());
    }
  }
  return resolver;
}
```

Actually, a cleaner approach — extract the registration logic into a method that takes strategy + isDefaultBean, and add a test-visible constructor:

Replace the entire constructor and add a helper method:

```java
private EngineStrategyResolver() {
    this.index = new HashMap<>();
    this.defaults = new HashMap<>();
}

@Inject
public EngineStrategyResolver(
    @Any Instance<io.casehub.api.spi.routing.AgentRoutingStrategy> agentStrategies,
    @Any Instance<io.casehub.api.spi.routing.ImplementationRoutingStrategy> implStrategies,
    @Any Instance<io.casehub.api.spi.routing.CandidateMatchingStrategy> matchStrategies,
    @Any Instance<io.casehub.api.spi.routing.CandidateSetStrategy> candidateSetStrategies,
    @Any Instance<io.casehub.engine.common.spi.scheduler.WorkerExecutionRoutingStrategy> execStrategies,
    @Any Instance<io.casehub.api.spi.routing.TrustRoutingPolicyProvider> trustStrategies) {
  this();
  registerStrategies(agentStrategies);
  registerStrategies(implStrategies);
  registerStrategies(matchStrategies);
  registerStrategies(candidateSetStrategies);
  registerStrategies(execStrategies);
  registerStrategies(trustStrategies);

  org.jboss.logging.Logger.getLogger(EngineStrategyResolver.class)
      .infof("EngineStrategyResolver discovered %d strategies, defaults: %s",
          index.values().stream().mapToInt(Map::size).sum(),
          defaults.entrySet().stream()
              .map(e -> e.getKey().getSimpleName() + "=" + e.getValue().id())
              .toList());
}

private <T extends NamedStrategy> void registerStrategies(Instance<T> instance) {
  for (Instance.Handle<T> handle : instance.handles()) {
    T strategy = handle.get();
    boolean isDefault = (handle.getBean() instanceof InjectableBean<?> ib) && ib.isDefaultBean();
    registerEntry(strategy, isDefault);
  }
}

void registerEntry(NamedStrategy strategy, boolean isDefault) {
  for (Class<?> iface : resolveStrategyTypes(strategy.getClass())) {
    Map<String, NamedStrategy> byId = index.computeIfAbsent(iface, k -> new LinkedHashMap<>());
    NamedStrategy existing = byId.put(strategy.id(), strategy);
    if (existing != null) {
      throw new IllegalStateException(
          "Duplicate strategy id '" + strategy.id() + "' for type " + iface.getSimpleName()
          + ": " + existing.getClass().getName() + " and " + strategy.getClass().getName());
    }
    if (isDefault) {
      NamedStrategy existingDefault = defaults.put(iface, strategy);
      if (existingDefault != null && existingDefault != strategy) {
        throw new IllegalStateException(
            "Multiple @DefaultBean strategies for type " + iface.getSimpleName()
            + ": " + existingDefault.getClass().getName()
            + " and " + strategy.getClass().getName());
      }
    } else {
      defaults.putIfAbsent(iface, strategy);
    }
  }
}

static EngineStrategyResolver forTest(List<?> handles) {
  var resolver = new EngineStrategyResolver();
  for (Object h : handles) {
    @SuppressWarnings("unchecked")
    var th = (EngineStrategyResolverTest.TestHandle<? extends NamedStrategy>) h;
    resolver.registerEntry(th.strategy(), th.isDefaultBean());
  }
  return resolver;
}
```

- [ ] **Step 4: Run EngineStrategyResolver tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=EngineStrategyResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 3 tests pass

- [ ] **Step 5: Run full runtime tests to verify no regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: All tests pass

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java runtime/src/test/java/io/casehub/engine/internal/routing/EngineStrategyResolverTest.java
git commit -m "refactor(#642): EngineStrategyResolver default determinism via @DefaultBean detection

Use Quarkus ARC InjectableBean.isDefaultBean() to identify @DefaultBean
strategies as defaults, regardless of CDI iteration order. Throw on
duplicate @DefaultBean for the same strategy type. Non-@DefaultBean
strategies fall back to first-wins only when no @DefaultBean exists.

Closes #642"
```

---

### Task 4: GateRequired CandidateSetStrategy Evaluation Upstream (#641)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateScheduleEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/ActionGateWorkItemHandler.java`
- Modify: relevant test files for the handler

**Interfaces:**
- Consumes: `CandidateSetStrategy.evaluate(CandidateSetContext)`, `CaseContext.panel(ContextPanel.WORKING).asJsonNode()`
- Produces: `ActionGateScheduleEvent.resolvedCandidateGroups(): Set<String>` — pre-evaluated groups, no routing dependency in work-adapter

- [ ] **Step 1: Add resolvedCandidateGroups to ActionGateScheduleEvent**

In `common/src/main/java/io/casehub/engine/common/internal/event/ActionGateScheduleEvent.java`:

Add import:
```java
import java.util.Set;
```

Update record:
```java
public record ActionGateScheduleEvent(
    UUID caseId,
    String tenancyId,
    long gateId,
    PlannedAction plannedAction,
    RiskDecision.GateRequired gateRequired,
    Set<String> resolvedCandidateGroups) {}
```

- [ ] **Step 2: Update WorkflowExecutionCompletedHandler.handleGate() to evaluate upstream**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`:

Add imports:
```java
import io.casehub.api.context.ContextPanel;
import io.casehub.api.spi.routing.CandidateSetContext;
import java.util.Set;
```

Replace the `handleGate()` method. The key change is to evaluate `gate.candidateGroups()` before publishing the event. Wrap the existing method body in a `groupsUni.chain()`:

```java
private Uni<Void> handleGate(
    final WorkflowExecutionCompleted event,
    final PlannedAction plannedAction,
    final RiskDecision.GateRequired gate,
    final String traceId) {
  final CaseInstance caseInstance = event.caseInstance();
  final Worker worker = event.worker();
  final Map<String, Object> rawOutput = event.output() == null ? Map.of() : event.output();
  final Instant now = Instant.now();

  Uni<Set<String>> groupsUni;
  if (gate.candidateGroups() != null) {
    JsonNode contextNode = caseInstance.getCaseContext()
        .panel(ContextPanel.WORKING).asJsonNode();
    groupsUni = gate.candidateGroups()
        .evaluate(new CandidateSetContext(contextNode))
        .onFailure()
        .recoverWithUni(t -> {
          LOG.warnf(t,
              "CandidateSetStrategy evaluation failed for caseId=%s — "
              + "proceeding with empty candidate groups",
              caseInstance.getUuid());
          return Uni.createFrom().item(Set.of());
        });
  } else {
    groupsUni = Uni.createFrom().item(Set.of());
  }

  return groupsUni.chain(resolvedGroups -> {
    final EventLog gateEventLog =
        buildGateEventLog(caseInstance, worker, rawOutput, plannedAction, gate,
            event.idempotency(), now);

    return eventLogRepository
        .append(gateEventLog, caseInstance.tenancyId)
        .chain(() -> {
          caseInstance.setPendingActionGate(
              new PendingActionGate(
                  gateEventLog.id, worker.name(), event.idempotency(),
                  rawOutput, plannedAction));
          return caseInstanceRepository.update(caseInstance, caseInstance.tenancyId);
        })
        .invoke(() ->
            eventBus.publish(
                EventBusAddresses.ACTION_GATE_SCHEDULE,
                new ActionGateScheduleEvent(
                    caseInstance.getUuid(), caseInstance.tenancyId,
                    gateEventLog.id, plannedAction, gate, resolvedGroups)))
        .invoke(() ->
            lifecycleEvents
                .fireAsync(new CaseLifecycleEvent(
                    caseInstance.getUuid(), caseInstance.tenancyId,
                    "ActionGate", "ActionGatePending",
                    caseInstance.getState().name(), worker.name(), "WORKER", traceId))
                .whenComplete((v, t) -> {
                  if (t != null)
                    LOG.warnf(t,
                        "CaseLifecycleEvent observer failed for caseId=%s event=ActionGatePending",
                        caseInstance.getUuid());
                }))
        .replaceWithVoid()
        .onFailure()
        .invoke(t -> LOG.error(
            "Failed to handle action gate for caseId: " + caseInstance.getUuid(), t));
  });
}
```

- [ ] **Step 3: Simplify ActionGateWorkItemHandler to use pre-resolved groups**

In `work-adapter/src/main/java/io/casehub/workadapter/ActionGateWorkItemHandler.java`:

Remove the `candidateGroupsCsv()` method entirely (lines 84-94). Update `onActionGateSchedule()`:

```java
@ConsumeEvent(value = EventBusAddresses.ACTION_GATE_SCHEDULE, blocking = true)
@Transactional
public void onActionGateSchedule(final ActionGateScheduleEvent event) {
  final String callerRef = GateCallerRef.encode(event.caseId(), event.gateId());
  final Instant expiresAt =
      event.gateRequired().expiresIn() != null
          ? Instant.now().plus(event.gateRequired().expiresIn())
          : null;

  final Set<String> groups = event.resolvedCandidateGroups();
  final String candidateGroupsCsv =
      (groups == null || groups.isEmpty()) ? null : String.join(",", groups);

  final WorkItemCreateRequest request =
      WorkItemCreateRequest.builder()
          .title(event.gateRequired().reason())
          .candidateGroups(candidateGroupsCsv)
          .createdBy("casehub-engine")
          .payload(buildPayload(event))
          .expiresAt(expiresAt)
          .callerRef(callerRef)
          .scope(event.gateRequired().scope())
          .build();

  workItemCreator.create(request);
  LOG.infof("Gate WorkItem created: caseId=%s gateId=%d callerRef=%s expiresAt=%s",
      event.caseId(), event.gateId(), callerRef, expiresAt);
}
```

Add import:
```java
import java.util.Set;
```

Remove unused import for `io.casehub.api.spi.routing.CandidateSetContext` if present.

- [ ] **Step 4: Run work-adapter tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl work-adapter`
Expected: All tests pass

- [ ] **Step 5: Run runtime tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: All tests pass

- [ ] **Step 6: Commit**

```
git add common/src/main/java/io/casehub/engine/common/internal/event/ActionGateScheduleEvent.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java work-adapter/src/main/java/io/casehub/workadapter/ActionGateWorkItemHandler.java
git commit -m "refactor(#641): evaluate GateRequired CandidateSetStrategy upstream

Move CandidateSetStrategy evaluation from ActionGateWorkItemHandler
(work-adapter, no case context) to WorkflowExecutionCompletedHandler
(runtime, CaseInstance in scope). Pre-resolved groups carried in
ActionGateScheduleEvent. Handler reads data, not behavior.

Evaluation failure recovers with empty groups + warning — gate still
fires with manual review, just without pre-populated candidate groups.

Closes #641"
```

---

### Task 5: WorkerRuntime.spawnCase() / awaitCase() (#636)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/model/CaseTerminatedException.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseCompletionTracker.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/CaseCompletionTrackerTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.startCase(CaseDefinition, Object, UUID, PropagationContext)`, `CaseInstanceCache.get(UUID)`, `CaseContext.snapshot()`
- Produces: `CaseDefinitionRegistry.findByName(String): Optional<CaseDefinition>`, `CaseCompletionTracker.register(UUID): CompletableFuture<CaseContext>`, `CaseCompletionTracker.complete(UUID, CaseContext)`, `CaseCompletionTracker.completeExceptionally(UUID, Throwable)`, `CaseCompletionTracker.remove(UUID)`, `CaseTerminatedException(UUID, CaseStatus)`

- [ ] **Step 1: Create CaseTerminatedException**

Create `common/src/main/java/io/casehub/engine/common/internal/model/CaseTerminatedException.java`:

```java
package io.casehub.engine.common.internal.model;

import io.casehub.api.model.CaseStatus;
import java.util.UUID;

public class CaseTerminatedException extends RuntimeException {

  private final UUID caseId;
  private final CaseStatus terminalStatus;

  public CaseTerminatedException(UUID caseId, CaseStatus terminalStatus) {
    super("Case " + caseId + " terminated with status " + terminalStatus);
    this.caseId = caseId;
    this.terminalStatus = terminalStatus;
  }

  public UUID caseId() {
    return caseId;
  }

  public CaseStatus terminalStatus() {
    return terminalStatus;
  }
}
```

- [ ] **Step 2: Add findByName() to CaseDefinitionRegistry**

In `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java`, add at the end of the interface:

```java
default Optional<CaseDefinition> findByName(String name) {
    return Optional.empty();
}
```

- [ ] **Step 3: Write findByName() test**

In `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java`, add tests. Read the existing file first to understand the test setup, then add:

```java
@Test
void findByName_existingDefinition_returnsDefinition() {
    // Register a definition, then look it up by name
    // (Exact setup depends on existing test infrastructure)
}

@Test
void findByName_nonExistent_returnsEmpty() {
    // Lookup a name that doesn't exist
}

@Test
void findByName_ambiguous_throws() {
    // Register two definitions with the same name but different namespaces
}
```

Note: The exact test setup depends on the existing test infrastructure in `DefaultCaseDefinitionRegistryTest`. The implementer should read the existing test file and follow the established patterns for registering definitions.

- [ ] **Step 4: Implement findByName() in DefaultCaseDefinitionRegistry**

In `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`, add:

```java
@Override
public Optional<CaseDefinition> findByName(String name) {
    List<RegistryEntry> matches = registry.values().stream()
        .filter(e -> name.equals(e.definition().getName()))
        .toList();
    if (matches.isEmpty()) return Optional.empty();
    if (matches.size() > 1) {
        throw new IllegalArgumentException(
            "Ambiguous caseType '" + name + "' — matches " + matches.size()
            + " definitions across namespaces. Use qualified lookup to disambiguate.");
    }
    return Optional.of(matches.get(0).definition());
}
```

- [ ] **Step 5: Run registry tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests pass

- [ ] **Step 6: Write CaseCompletionTracker test**

Create `runtime/src/test/java/io/casehub/engine/internal/engine/CaseCompletionTrackerTest.java`:

```java
package io.casehub.engine.internal.engine;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.context.CaseContext;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CaseCompletionTrackerTest {

  private CaseCompletionTracker tracker;

  @BeforeEach
  void setUp() {
    tracker = new CaseCompletionTracker();
  }

  @Test
  void register_thenComplete_resolvesFuture() throws Exception {
    UUID caseId = UUID.randomUUID();
    CompletableFuture<CaseContext> future = tracker.register(caseId);

    CaseContext mockContext = new io.casehub.engine.internal.context.CaseContextImpl();
    tracker.complete(caseId, mockContext);

    CaseContext result = future.get(1, TimeUnit.SECONDS);
    assertSame(mockContext, result);
  }

  @Test
  void complete_withoutRegister_isNoOp() {
    UUID caseId = UUID.randomUUID();
    CaseContext mockContext = new io.casehub.engine.internal.context.CaseContextImpl();
    assertDoesNotThrow(() -> tracker.complete(caseId, mockContext));
  }

  @Test
  void register_thenCompleteExceptionally_failsFuture() {
    UUID caseId = UUID.randomUUID();
    CompletableFuture<CaseContext> future = tracker.register(caseId);

    tracker.completeExceptionally(caseId, new RuntimeException("case faulted"));

    ExecutionException ex = assertThrows(ExecutionException.class,
        () -> future.get(1, TimeUnit.SECONDS));
    assertEquals("case faulted", ex.getCause().getMessage());
  }

  @Test
  void register_thenTimeout_throwsTimeoutException() {
    UUID caseId = UUID.randomUUID();
    CompletableFuture<CaseContext> future = tracker.register(caseId);

    assertThrows(TimeoutException.class,
        () -> future.get(50, TimeUnit.MILLISECONDS));
  }

  @Test
  void remove_cleansUpEntry() throws Exception {
    UUID caseId = UUID.randomUUID();
    tracker.register(caseId);
    tracker.remove(caseId);

    // A second register creates a new future (not the removed one)
    CompletableFuture<CaseContext> future2 = tracker.register(caseId);
    assertFalse(future2.isDone());
  }

  @Test
  void register_idempotent_returnsSameFuture() {
    UUID caseId = UUID.randomUUID();
    CompletableFuture<CaseContext> f1 = tracker.register(caseId);
    CompletableFuture<CaseContext> f2 = tracker.register(caseId);
    assertSame(f1, f2);
  }
}
```

- [ ] **Step 7: Create CaseCompletionTracker**

Create `runtime/src/main/java/io/casehub/engine/internal/engine/CaseCompletionTracker.java`:

```java
package io.casehub.engine.internal.engine;

import io.casehub.api.context.CaseContext;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class CaseCompletionTracker {

  private final ConcurrentHashMap<UUID, CompletableFuture<CaseContext>> pending =
      new ConcurrentHashMap<>();

  public CompletableFuture<CaseContext> register(UUID caseId) {
    return pending.computeIfAbsent(caseId, k -> new CompletableFuture<>());
  }

  public void complete(UUID caseId, CaseContext context) {
    CompletableFuture<CaseContext> future = pending.get(caseId);
    if (future != null) {
      future.complete(context);
    }
  }

  public void completeExceptionally(UUID caseId, Throwable t) {
    CompletableFuture<CaseContext> future = pending.get(caseId);
    if (future != null) {
      future.completeExceptionally(t);
    }
  }

  public void remove(UUID caseId) {
    pending.remove(caseId);
  }
}
```

- [ ] **Step 8: Run CaseCompletionTracker tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=CaseCompletionTrackerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 6 tests pass

- [ ] **Step 9: Wire CaseCompletionTracker into CaseStatusChangedHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`:

Add injection:
```java
@Inject CaseCompletionTracker caseCompletionTracker;
```

Add imports:
```java
import io.casehub.engine.internal.engine.CaseCompletionTracker;
import io.casehub.engine.common.internal.model.CaseTerminatedException;
import io.casehub.api.context.CaseContext;
```

In `onCaseStatusChangedHandler()`, inside the `isTerminalState(newState)` block (around line 106), add tracker notification BEFORE the existing channel close/gate cancel/trigger cancel logic:

```java
if (isTerminalState(newState)) {
    CaseContext contextSnapshot = caseInstance.getCaseContext().snapshot();
    if (newState == CaseStatus.COMPLETED) {
        caseCompletionTracker.complete(caseInstance.getUuid(), contextSnapshot);
    } else {
        caseCompletionTracker.completeExceptionally(
            caseInstance.getUuid(),
            new CaseTerminatedException(caseInstance.getUuid(), newState));
    }
    // existing: channel close, gate cancel, trigger cancel
    caseChannelProvider.listChannels(caseInstance.getUuid())
        .forEach(caseChannelProvider::closeChannel);
    // ... rest of existing code
```

- [ ] **Step 10: Update DefaultWorkerRuntime with spawnCase/awaitCase**

In `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java`:

Add imports:
```java
import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseTerminatedException;
import io.casehub.engine.internal.engine.CaseCompletionTracker;
import io.casehub.api.engine.SettlementTimeoutException;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
```

Update constructor to accept tracker:
```java
private final CaseCompletionTracker tracker;

DefaultWorkerRuntime(
    UUID caseId,
    CaseHubRuntime caseHubRuntime,
    CaseDefinitionRegistry definitionRegistry,
    CaseInstanceCache caseInstanceCache,
    CaseCompletionTracker tracker) {
  this.caseId = caseId;
  this.caseHubRuntime = caseHubRuntime;
  this.definitionRegistry = definitionRegistry;
  this.caseInstanceCache = caseInstanceCache;
  this.tracker = tracker;
}
```

Replace `spawnCase`:
```java
@Override
public UUID spawnCase(String caseType, Map<String, Object> input) {
  CaseDefinition definition = definitionRegistry.findByName(caseType)
      .orElseThrow(() -> new IllegalArgumentException(
          "No case definition found for caseType: " + caseType));

  CaseInstance parentInstance = caseInstanceCache.get(caseId);
  PropagationContext propagation = parentInstance != null
      ? parentInstance.getPropagationContext() : null;

  try {
    return caseHubRuntime.startCase(definition, input, caseId, propagation)
        .toCompletableFuture().join();
  } catch (Exception e) {
    throw new RuntimeException("Failed to spawn case '" + caseType + "'", e);
  }
}
```

Replace `awaitCase`:
```java
@Override
public CaseContext awaitCase(UUID childCaseId, Duration timeout) {
  CompletableFuture<CaseContext> future = tracker.register(childCaseId);

  CaseInstance child = caseInstanceCache.get(childCaseId);
  if (child != null && isTerminal(child.getState())) {
    CaseContext snapshot = child.getCaseContext().snapshot();
    if (child.getState() == CaseStatus.COMPLETED) {
      future.complete(snapshot);
    } else {
      future.completeExceptionally(
          new CaseTerminatedException(childCaseId, child.getState()));
    }
  }

  try {
    return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
  } catch (TimeoutException e) {
    throw new SettlementTimeoutException(
        "Child case " + childCaseId + " did not complete within " + timeout);
  } catch (ExecutionException e) {
    if (e.getCause() instanceof CaseTerminatedException cte) {
      throw cte;
    }
    throw new RuntimeException("Child case " + childCaseId + " failed", e.getCause());
  } catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException("Interrupted while awaiting case " + childCaseId, e);
  } finally {
    tracker.remove(childCaseId);
  }
}

private static boolean isTerminal(CaseStatus status) {
  return status == CaseStatus.COMPLETED
      || status == CaseStatus.FAULTED
      || status == CaseStatus.CANCELLED;
}
```

- [ ] **Step 11: Update WorkerRuntimeFactory**

In `runtime/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java`:

Add import:
```java
import io.casehub.engine.internal.engine.CaseCompletionTracker;
```

Add field and update constructor:
```java
private final CaseCompletionTracker caseCompletionTracker;

@Inject
public WorkerRuntimeFactory(
    CaseHubRuntime caseHubRuntime,
    CaseDefinitionRegistry definitionRegistry,
    CaseInstanceCache caseInstanceCache,
    CaseCompletionTracker caseCompletionTracker) {
  this.caseHubRuntime = caseHubRuntime;
  this.definitionRegistry = definitionRegistry;
  this.caseInstanceCache = caseInstanceCache;
  this.caseCompletionTracker = caseCompletionTracker;
}
```

Update `create()`:
```java
public WorkerRuntime create(UUID caseId) {
  return new DefaultWorkerRuntime(
      caseId, caseHubRuntime, definitionRegistry, caseInstanceCache, caseCompletionTracker);
}
```

- [ ] **Step 12: Update DefaultWorkerRuntimeTest**

In `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeTest.java`:

Update setUp to pass tracker (null for execute-only tests):
```java
@BeforeEach
void setUp() {
  runtime = new DefaultWorkerRuntime(CASE_ID, null, null, null, null);
}
```

Add spawnCase and awaitCase tests — these require mocking `CaseDefinitionRegistry`, `CaseHubRuntime`, `CaseInstanceCache`, and `CaseCompletionTracker`. Use test doubles, not Mockito:

```java
@Test
void spawnCase_existingDefinition_returnsCaseId() {
  UUID childId = UUID.randomUUID();
  var definition = io.casehub.api.model.CaseDefinition.builder()
      .namespace("test").name("child").version("1.0.0").build();
  var registry = new io.casehub.engine.common.spi.CaseDefinitionRegistry() {
    @Override public io.smallrye.mutiny.Uni<io.casehub.engine.common.internal.model.CaseMetaModel> registerCaseDefinition(io.casehub.api.model.CaseDefinition m) { return null; }
    @Override public io.casehub.api.model.CaseDefinition getCaseDefinition(io.casehub.engine.common.internal.model.CaseMetaModel m) { return null; }
    @Override public io.casehub.engine.common.internal.model.CaseMetaModel getCaseMetaModel(io.casehub.api.model.CaseDefinition d) { return null; }
    @Override public java.util.Optional<io.casehub.api.model.CaseDefinition> findByName(String name) {
      return "child".equals(name) ? java.util.Optional.of(definition) : java.util.Optional.empty();
    }
  };
  var caseHubRuntime = new io.casehub.api.engine.CaseHubRuntime() {
    @Override public java.util.concurrent.CompletionStage<UUID> startCase(io.casehub.api.model.CaseDefinition d) { return null; }
    @Override public java.util.concurrent.CompletionStage<UUID> startCase(io.casehub.api.model.CaseDefinition d, Object i) { return null; }
    @Override public java.util.concurrent.CompletionStage<UUID> startCase(io.casehub.api.model.CaseDefinition d, Object i, UUID p, io.casehub.api.context.PropagationContext pc) {
      return java.util.concurrent.CompletableFuture.completedFuture(childId);
    }
    @Override public java.util.concurrent.CompletionStage<UUID> startCase(io.casehub.api.model.CaseDefinition d, Object i, Map<String, Object> s) { return null; }
    @Override public java.util.concurrent.CompletionStage<UUID> startCase(io.casehub.api.model.CaseDefinition d, Object i, Map<String, Object> s, UUID p, io.casehub.api.context.PropagationContext pc) { return null; }
    @Override public java.util.concurrent.CompletionStage<Void> signal(UUID id, String p, Object v) { return null; }
    @Override public void cancelCase(UUID id) {}
    @Override public void suspendCase(UUID id) {}
    @Override public void resumeCase(UUID id) {}
    @Override public java.util.concurrent.CompletionStage<Object> query(UUID id, String p) { return null; }
    @Override public <T> java.util.concurrent.CompletionStage<T> query(UUID id, String p, Class<T> c) { return null; }
    @Override public java.util.concurrent.CompletionStage<java.util.List<io.casehub.api.model.event.CaseEventLogRecord>> eventLog(UUID id) { return null; }
    @Override public java.util.concurrent.CompletionStage<java.util.List<io.casehub.api.model.event.CaseEventLogRecord>> eventLog(UUID id, java.util.Set<io.casehub.api.model.event.CaseHubEventType> t) { return null; }
    @Override public java.util.concurrent.CompletionStage<java.util.List<io.casehub.api.model.event.CaseEventLogRecord>> eventLog(UUID id, java.util.Set<io.casehub.api.model.event.CaseHubEventType> t, java.util.Set<io.casehub.api.model.event.EventStreamType> s) { return null; }
  };

  var rt = new DefaultWorkerRuntime(CASE_ID, caseHubRuntime, registry, null, null);
  UUID result = rt.spawnCase("child", Map.of("key", "value"));
  assertEquals(childId, result);
}

@Test
void spawnCase_unknownDefinition_throws() {
  var registry = new io.casehub.engine.common.spi.CaseDefinitionRegistry() {
    @Override public io.smallrye.mutiny.Uni<io.casehub.engine.common.internal.model.CaseMetaModel> registerCaseDefinition(io.casehub.api.model.CaseDefinition m) { return null; }
    @Override public io.casehub.api.model.CaseDefinition getCaseDefinition(io.casehub.engine.common.internal.model.CaseMetaModel m) { return null; }
    @Override public io.casehub.engine.common.internal.model.CaseMetaModel getCaseMetaModel(io.casehub.api.model.CaseDefinition d) { return null; }
  };

  var rt = new DefaultWorkerRuntime(CASE_ID, null, registry, null, null);
  assertThrows(IllegalArgumentException.class, () -> rt.spawnCase("unknown", Map.of()));
}
```

- [ ] **Step 13: Run all runtime tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: All tests pass

- [ ] **Step 14: Commit**

```
git add common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java common/src/main/java/io/casehub/engine/common/internal/model/CaseTerminatedException.java runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java runtime/src/main/java/io/casehub/engine/internal/engine/CaseCompletionTracker.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerRuntime.java runtime/src/main/java/io/casehub/engine/internal/executor/WorkerRuntimeFactory.java runtime/src/test/java/io/casehub/engine/internal/engine/CaseCompletionTrackerTest.java runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerRuntimeTest.java
git commit -m "feat(#636): implement WorkerRuntime.spawnCase() and awaitCase()

- CaseDefinitionRegistry.findByName() — name-only lookup, ambiguity throws
- CaseCompletionTracker — CompletableFuture per awaited case, no orphans
- CaseStatusChangedHandler notifies tracker with snapshot on terminal state
  (COMPLETED → complete, FAULTED/CANCELLED → completeExceptionally)
- DefaultWorkerRuntime.spawnCase() — links parent-child, inherits propagation
- DefaultWorkerRuntime.awaitCase() — race guard for out-of-order completion,
  CaseTerminatedException for non-success, finally-block cleanup
- WorkerRuntimeFactory injects CaseCompletionTracker

Refs #636"
```

- [ ] **Step 15: Add integration test for spawnCase/awaitCase**

Add a new test in `runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java`:

```java
@Inject SpawnAwaitBean spawnAwaitBean;

@Test
void spawnAndAwait_childCaseCompletesAndReturnsContext() throws Exception {
  UUID parentCaseId = spawnAwaitBean.startCase(Map.of("trigger", true))
      .toCompletableFuture().join();

  await()
      .atMost(20, TimeUnit.SECONDS)
      .untilAsserted(() -> {
        assertThat(cache.get(parentCaseId).getState()).isEqualTo(CaseStatus.COMPLETED);
        assertThat(cache.get(parentCaseId).getCaseContext().get("childResult")).isEqualTo("done");
      });
}
```

Add the `SpawnAwaitBean` CaseHub inner class — a parent case whose worker spawns a child case, awaits it, and writes the result to the parent context. The child case is a simple worker that returns `{result: "done"}`. The exact CaseHub definition depends on the existing integration test patterns — follow the `Tier2SequentialBean` pattern.

- [ ] **Step 16: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=HybridOrchestrationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests pass including the new spawnAndAwait test

- [ ] **Step 17: Commit integration test**

```
git add runtime/src/test/java/io/casehub/engine/HybridOrchestrationIntegrationTest.java
git commit -m "test(#636): integration test for WorkerRuntime.spawnCase/awaitCase

Parent case spawns child via WorkerRuntime.spawnAndAwaitCase(), asserts
child result propagates to parent context. Covers the full path:
spawnCase → child case execution → CaseStatusChangedHandler → tracker →
awaitCase resolution.

Closes #636"
```

---

### Task 6: PLATFORM.md + Garden Protocol (#643)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md`
- Create: `/Users/mdproctor/.hortora/garden/docs/protocols/casehub/routing-strategy-convention.md`

**Interfaces:**
- Consumes: Content from engine#634 spec §7
- Produces: Platform documentation and protocol for routing strategy convention

- [ ] **Step 1: Read current PLATFORM.md to find insertion points**

Read the capability ownership table and Step 4 consistency rules sections in `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md` to find exact insertion points.

- [ ] **Step 2: Add routing strategy entry to PLATFORM.md capability ownership table**

Add entry:

> **Routing Strategy Resolution** — `casehub-platform-api` (`io.casehub.platform.api.routing`)
>
> `NamedStrategy` marker interface and `StrategyResolver` CDI bean. All per-case-selectable routing strategies extend `NamedStrategy` and are resolved by `id` via `StrategyResolver`. Resolution order: YAML-specified ID → `@DefaultBean` fallback. Domain-specific strategy interfaces live in their owning module (`engine-api`, `work-api`); the shared convention lives in `platform-api`.

- [ ] **Step 3: Add routing strategy rule to PLATFORM.md Step 4**

Add entry:

> **Routing strategies:** Any SPI where a harness author selects among alternative implementations per case or per binding must extend `NamedStrategy` (platform-api), declare a stable `id()`, and ship a `@DefaultBean` no-op or sensible-default implementation. Resolve via `StrategyResolver`, never via direct `Instance<>` iteration or CDI `@Priority` override.

- [ ] **Step 4: Commit PLATFORM.md changes**

```
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(engine#643): routing strategy convention in PLATFORM.md

Add Routing Strategy Resolution to capability ownership table and
routing strategy rule to Step 4 consistency rules.

Refs casehubio/engine#643"
```

- [ ] **Step 5: Create garden protocol**

Create `/Users/mdproctor/.hortora/garden/docs/protocols/casehub/routing-strategy-convention.md`:

```markdown
---
id: routing-strategy-convention
scope: platform
status: active
created: 2026-07-04
refs:
  - casehubio/engine#634
---

# Routing Strategy Convention

## Rule

Per-case or per-binding selectable strategies extend `NamedStrategy` (`io.casehub.platform.api.routing`), declare a stable `id()`, ship a `@DefaultBean` no-op or sensible-default implementation, and resolve via `StrategyResolver`.

Selection models that bypass this convention are not to be used for new routing strategies:
- CDI `@Priority` override
- `@Named` qualifier
- Config property switch
- Direct `Instance<>` iteration

## Non-Members

These mechanisms are intentionally excluded from the convention — they serve different patterns:

- `ActionRiskClassifier` — chain composition (most-restrictive-wins), not per-case selection
- `@DefaultBean`-only SPIs — single-bean replacement (e.g. `ExclusionPolicy`, `CapabilityHealth`), not per-case selectable
- `ContextDiffStrategy` — deployment-level config switch, stays as-is
- Access control policies — determines what CAN flow, not where
- Data providers — feed INTO routing strategies, not routing decisions themselves
- Delivery infrastructure — direct ID lookup, caller specifies target
```

- [ ] **Step 6: Commit garden protocol**

```
git -C /Users/mdproctor/.hortora/garden add docs/protocols/casehub/routing-strategy-convention.md
git -C /Users/mdproctor/.hortora/garden commit -m "protocol(engine#643): routing strategy convention

Per-case selectable strategies: extend NamedStrategy, declare id(),
ship @DefaultBean default, resolve via StrategyResolver.

Refs casehubio/engine#643"
```

- [ ] **Step 7: Update FOUNDATION-INDEX.md in garden**

Add the new protocol to the index in `/Users/mdproctor/.hortora/garden/docs/protocols/casehub/FOUNDATION-INDEX.md`.

- [ ] **Step 8: Commit index update and close issue**

```
git -C /Users/mdproctor/.hortora/garden add docs/protocols/casehub/FOUNDATION-INDEX.md
git -C /Users/mdproctor/.hortora/garden commit -m "docs(engine#643): add routing-strategy-convention to FOUNDATION-INDEX

Refs casehubio/engine#643"
```

Close the issue:
```
gh issue close 643 --repo casehubio/engine --comment "PLATFORM.md updated (casehubio/parent), garden protocol created and indexed."
```

---

### Task 7: Final Verification and CLAUDE.md Sync

**Files:**
- Modify: `CLAUDE.md` (if needed beyond #629 changes)

- [ ] **Step 1: Full build**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test`
Expected: All tests pass across all modules

- [ ] **Step 2: Coherence review against PLATFORM.md and protocols**

Verify:
- All new SPIs follow `spi-evolution-default-methods` (default methods with no-op returns)
- `CaseCompletionTracker` placement in `runtime/internal/engine/` does not violate `spi-placement-caseinstance-goes-in-common` (it's not an SPI)
- `PlanItem` AtomicReference conversion doesn't break any protocol
- No `@ApplicationScoped` bean added that should be `@DefaultBean`
- No workarounds or backward-compatibility shims

- [ ] **Step 3: Update CLAUDE.md**

Sync any documentation that references changed interfaces:
- `CaseDefinitionRegistry.findByName()` addition
- `CaseCompletionTracker` existence
- `ActionGateScheduleEvent` new field
- `PlanItem.tryMarkRunning()` addition
- `EngineStrategyResolver` @DefaultBean detection
