# Harden CaseRecoveryService.unfault() Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#1183 — harden CaseRecoveryService.unfault()
**Issue group:** #1182, #732, #1183, #1184, #1185

**Goal:** Fix the JPA persistence race in `unfault()`, restore torn-down execution infrastructure (channels, scheduled triggers), and document CANCELLED as non-recoverable.

**Architecture:** `CaseRecoveryService.unfault()` gains three new dependencies (`CaseInstanceRepository`, `CaseChannelProvider`, `SchedulerService`) to persist state synchronously before async event dispatch, re-open the coordination channel, and re-register scheduled triggers. The async event chain (`CaseStatusChanged` → `CaseContextChangedEvent` → binding re-evaluation) handles the remaining infrastructure restoration naturally.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x event bus, JPA/Hibernate

## Global Constraints

- All new dependencies in `CaseRecoveryService` (runtime-core module) must be SPIs from `common-core` or `api` — no Vert.x/Quarkus-specific APIs.
- `CaseRecoveryService` is not CDI-managed — it's constructed via `RuntimeBeans.caseRecoveryService()` producer.
- Test classes must be `*Test.java`, never `*IT.java` (surefire, not failsafe).
- `@ObservesAsync` is unreliable in `@QuarkusTest` — inject beans and call observer methods directly.
- `TESTCONTAINERS_RYUK_DISABLED=true` for test runs.

---

## Batch 1: Harden unfault() — persistence, infrastructure, CANCELLED policy

### Task 1: Add dependencies and fix persistence race

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/internal/recovery/CaseRecoveryService.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java:376-385`
- Create: `runtime-core/src/test/java/io/casehub/engine/internal/recovery/CaseRecoveryServiceTest.java`

**Interfaces:**
- Consumes: `CaseInstanceRepository.update(CaseInstance, String)`, `CaseInstanceCache.put(CaseInstance)`, `CaseChannelProvider.openChannel(UUID, String)`, `SchedulerService.registerScheduledTriggers(CaseInstance)`
- Produces: `CaseRecoveryService(CaseInstanceCache, CrossTenantCaseInstanceRepository, CaseCompletionTracker, EventDispatcher, CaseInstanceRepository, CaseChannelProvider, SchedulerService)` — 7-arg constructor

- [ ] **Step 1: Write failing test — state is persisted synchronously**

Create `CaseRecoveryServiceTest.java` in `runtime-core/src/test/`. This is a unit test with mock/stub dependencies — no `@QuarkusTest` needed.

```java
package io.casehub.engine.internal.recovery;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

import io.casehub.api.model.CaseStatus;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.event.EventDispatcher;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.internal.engine.CaseCompletionTracker;
import io.casehub.engine.internal.scheduler.SchedulerService;
import java.util.Optional;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class CaseRecoveryServiceTest {

  private CaseInstanceCache cache;
  private CrossTenantCaseInstanceRepository crossTenantRepo;
  private CaseCompletionTracker completionTracker;
  private EventDispatcher eventDispatcher;
  private CaseInstanceRepository caseInstanceRepo;
  private CaseChannelProvider channelProvider;
  private SchedulerService schedulerService;
  private CaseRecoveryService service;

  @BeforeEach
  void setUp() {
    cache = mock(CaseInstanceCache.class);
    crossTenantRepo = mock(CrossTenantCaseInstanceRepository.class);
    completionTracker = mock(CaseCompletionTracker.class);
    eventDispatcher = mock(EventDispatcher.class);
    caseInstanceRepo = mock(CaseInstanceRepository.class);
    channelProvider = mock(CaseChannelProvider.class);
    schedulerService = mock(SchedulerService.class);
    service = new CaseRecoveryService(
        cache, crossTenantRepo, completionTracker, eventDispatcher,
        caseInstanceRepo, channelProvider, schedulerService);
  }

  @Test
  void unfault_persistsStateSynchronously_beforeEventDispatch() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createFaultedInstance(caseId);
    when(cache.get(caseId)).thenReturn(instance);

    service.unfault(caseId);

    var inOrder = inOrder(caseInstanceRepo, eventDispatcher);
    inOrder.verify(caseInstanceRepo).update(instance, instance.tenancyId);
    inOrder.verify(eventDispatcher).dispatch(any());
  }

  @Test
  void unfault_putsInstanceInCache() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createFaultedInstance(caseId);
    when(crossTenantRepo.findByUuid(caseId)).thenReturn(Optional.of(instance));

    service.unfault(caseId);

    verify(cache).put(instance);
  }

  private CaseInstance createFaultedInstance(UUID caseId) {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(CaseStatus.FAULTED);
    instance.tenancyId = "test-tenant";
    return instance;
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=CaseRecoveryServiceTest -q`
Expected: FAIL — `CaseRecoveryService` constructor doesn't accept 7 args yet.

- [ ] **Step 3: Update CaseRecoveryService constructor and unfault()**

Modify `CaseRecoveryService.java`. Add three new fields and update the constructor to accept 7 parameters. Update `unfault()` to persist synchronously, cache the instance, re-open the coordination channel, and re-register scheduled triggers.

```java
package io.casehub.engine.internal.recovery;

import io.casehub.api.model.CaseStatus;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.event.EventDispatcher;
import io.casehub.engine.common.internal.event.CaseStatusChanged;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.internal.engine.CaseCompletionTracker;
import io.casehub.engine.internal.scheduler.SchedulerService;
import java.util.Optional;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Administrative recovery operations for cases in terminal states. Provides the {@link
 * #unfault(UUID)} operation that transitions a FAULTED case back to RUNNING, enabling DLQ replay
 * and resumed execution.
 *
 * <p>Only FAULTED cases are recoverable. CANCELLED cases are intentionally excluded — cancellation
 * is an administrative decision and recovery would undermine its semantics. To retry a cancelled
 * case, start a new instance with the same context.
 */
public class CaseRecoveryService {

  private static final Logger LOG = Logger.getLogger(CaseRecoveryService.class);

  private final CaseInstanceCache caseInstanceCache;
  private final CrossTenantCaseInstanceRepository crossTenantRepository;
  private final CaseCompletionTracker caseCompletionTracker;
  private final EventDispatcher eventDispatcher;
  private final CaseInstanceRepository caseInstanceRepository;
  private final CaseChannelProvider caseChannelProvider;
  private final SchedulerService schedulerService;

  public CaseRecoveryService(
      CaseInstanceCache caseInstanceCache,
      CrossTenantCaseInstanceRepository crossTenantRepository,
      CaseCompletionTracker caseCompletionTracker,
      EventDispatcher eventDispatcher,
      CaseInstanceRepository caseInstanceRepository,
      CaseChannelProvider caseChannelProvider,
      SchedulerService schedulerService) {
    this.caseInstanceCache = caseInstanceCache;
    this.crossTenantRepository = crossTenantRepository;
    this.caseCompletionTracker = caseCompletionTracker;
    this.eventDispatcher = eventDispatcher;
    this.caseInstanceRepository = caseInstanceRepository;
    this.caseChannelProvider = caseChannelProvider;
    this.schedulerService = schedulerService;
  }

  /**
   * Transitions a FAULTED case back to RUNNING. Returns the case instance on success, empty if the
   * case is not found or not FAULTED.
   *
   * <p>CANCELLED cases are intentionally non-recoverable — cancellation is an administrative
   * decision. Start a new instance with the same context to retry.
   *
   * <p>The state is persisted synchronously to the database before returning, eliminating the race
   * between this method and DLQ replay that queries the database directly. The coordination channel
   * is re-opened and scheduled triggers are re-registered. An async {@link CaseStatusChanged} event
   * is dispatched to write the EventLog entry and trigger binding re-evaluation.
   */
  public Optional<CaseInstance> unfault(UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
      instance = crossTenantRepository.findByUuid(caseId).orElse(null);
    }
    if (instance == null) {
      LOG.warnf("Unfault: case not found: %s", caseId);
      return Optional.empty();
    }
    if (instance.getState() == CaseStatus.CANCELLED) {
      LOG.warnf("Cannot recover CANCELLED case %s — cancellation is an administrative "
          + "decision. Start a new instance with the same context to retry.", caseId);
      return Optional.empty();
    }
    if (instance.getState() != CaseStatus.FAULTED) {
      LOG.warnf("Unfault: case %s is %s, not FAULTED", caseId, instance.getState());
      return Optional.empty();
    }

    String oldStatus = instance.getState().name();
    instance.setState(CaseStatus.RUNNING);

    caseInstanceRepository.update(instance, instance.tenancyId);
    caseInstanceCache.put(instance);

    caseCompletionTracker.remove(caseId);
    caseCompletionTracker.register(caseId);

    caseChannelProvider.openChannel(caseId, "coordination");
    schedulerService.registerScheduledTriggers(instance);

    eventDispatcher.dispatch(new CaseStatusChanged(instance, oldStatus, CaseStatus.RUNNING.name()));

    LOG.infof("Case unfaulted: caseId=%s (%s → RUNNING)", caseId, oldStatus);
    return Optional.of(instance);
  }
}
```

- [ ] **Step 4: Update RuntimeBeans producer**

Modify `RuntimeBeans.java` to wire the three new dependencies into `caseRecoveryService()`:

```java
  @Produces
  @ApplicationScoped
  io.casehub.engine.internal.recovery.CaseRecoveryService caseRecoveryService(
      io.casehub.engine.common.spi.cache.CaseInstanceCache caseInstanceCache,
      io.casehub.engine.common.spi.CrossTenantCaseInstanceRepository crossTenantRepository,
      CaseCompletionTracker caseCompletionTracker,
      io.casehub.api.spi.event.EventDispatcher eventDispatcher,
      io.casehub.engine.common.spi.CaseInstanceRepository caseInstanceRepository,
      io.casehub.api.spi.CaseChannelProvider caseChannelProvider,
      io.casehub.engine.internal.scheduler.SchedulerService schedulerService) {
    return new io.casehub.engine.internal.recovery.CaseRecoveryService(
        caseInstanceCache, crossTenantRepository, caseCompletionTracker, eventDispatcher,
        caseInstanceRepository, caseChannelProvider, schedulerService);
  }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=CaseRecoveryServiceTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/internal/recovery/CaseRecoveryService.java \
  runtime/src/main/java/io/casehub/engine/internal/quarkus/RuntimeBeans.java \
  runtime-core/src/test/java/io/casehub/engine/internal/recovery/CaseRecoveryServiceTest.java
git commit -m "feat(#1183): persist state synchronously in unfault(), add infrastructure restoration

CaseRecoveryService.unfault() now persists RUNNING to DB before dispatching
the async CaseStatusChanged event, fixing the JPA-specific race where DLQ
replay could see stale FAULTED state. Also re-opens the coordination channel
and re-registers scheduled triggers to restore torn-down infrastructure.

Refs casehubio/engine#1183

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: CANCELLED rejection and comprehensive unit tests

**Files:**
- Modify: `runtime-core/src/test/java/io/casehub/engine/internal/recovery/CaseRecoveryServiceTest.java`

**Interfaces:**
- Consumes: 7-arg `CaseRecoveryService` constructor from Task 1
- Produces: Test coverage for CANCELLED rejection, infrastructure restoration ordering, cache-miss path

- [ ] **Step 1: Write failing test — CANCELLED rejection with specific log**

Add tests to `CaseRecoveryServiceTest.java`:

```java
  @Test
  void unfault_cancelledCase_returnsEmpty() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(CaseStatus.CANCELLED);
    instance.tenancyId = "test-tenant";
    when(cache.get(caseId)).thenReturn(instance);

    Optional<CaseInstance> result = service.unfault(caseId);

    assertThat(result).isEmpty();
    verify(caseInstanceRepo, never()).update(any(), any());
    verify(eventDispatcher, never()).dispatch(any());
  }

  @Test
  void unfault_completedCase_returnsEmpty() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(CaseStatus.COMPLETED);
    instance.tenancyId = "test-tenant";
    when(cache.get(caseId)).thenReturn(instance);

    Optional<CaseInstance> result = service.unfault(caseId);

    assertThat(result).isEmpty();
    verify(caseInstanceRepo, never()).update(any(), any());
  }
```

- [ ] **Step 2: Run test to verify it passes**

These tests should already pass because the CANCELLED check was implemented in Task 1.

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=CaseRecoveryServiceTest -q`
Expected: PASS

- [ ] **Step 3: Write test — infrastructure restoration is called**

```java
  @Test
  void unfault_reopensChannelAndReregistersScheduledTriggers() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createFaultedInstance(caseId);
    when(cache.get(caseId)).thenReturn(instance);

    service.unfault(caseId);

    verify(channelProvider).openChannel(caseId, "coordination");
    verify(schedulerService).registerScheduledTriggers(instance);
  }

  @Test
  void unfault_cacheMiss_loadsFromCrossTenantRepo() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createFaultedInstance(caseId);
    when(cache.get(caseId)).thenReturn(null);
    when(crossTenantRepo.findByUuid(caseId)).thenReturn(Optional.of(instance));

    Optional<CaseInstance> result = service.unfault(caseId);

    assertThat(result).isPresent();
    assertThat(result.get().getState()).isEqualTo(CaseStatus.RUNNING);
    verify(crossTenantRepo).findByUuid(caseId);
    verify(caseInstanceRepo).update(instance, "test-tenant");
    verify(cache).put(instance);
  }

  @Test
  void unfault_fullOrdering_persistBeforeChannelBeforeTriggerBeforeDispatch() {
    UUID caseId = UUID.randomUUID();
    CaseInstance instance = createFaultedInstance(caseId);
    when(cache.get(caseId)).thenReturn(instance);

    service.unfault(caseId);

    var inOrder = inOrder(caseInstanceRepo, cache, channelProvider, schedulerService, eventDispatcher);
    inOrder.verify(caseInstanceRepo).update(instance, "test-tenant");
    inOrder.verify(cache).put(instance);
    inOrder.verify(channelProvider).openChannel(caseId, "coordination");
    inOrder.verify(schedulerService).registerScheduledTriggers(instance);
    inOrder.verify(eventDispatcher).dispatch(any());
  }
```

- [ ] **Step 4: Run all tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -Dtest=CaseRecoveryServiceTest -q`
Expected: PASS

- [ ] **Step 5: Run full module test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime-core -q`
Expected: PASS — no regressions in runtime-core.

- [ ] **Step 6: Commit**

```bash
git add runtime-core/src/test/java/io/casehub/engine/internal/recovery/CaseRecoveryServiceTest.java
git commit -m "test(#1183): comprehensive unit tests for hardened unfault()

Tests: CANCELLED rejection, COMPLETED rejection, cache-miss DB fallback,
infrastructure restoration (channel + triggers), full ordering guarantee
(persist → cache → channel → triggers → dispatch).

Refs casehubio/engine#1183

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Integration verification

### Task 3: Verify integration test and full build

**Files:**
- Modify (if needed): `resilience/src/test/java/io/casehub/resilience/deadletter/DeadLetterReplayIntegrationTest.java`

**Interfaces:**
- Consumes: Hardened `CaseRecoveryService` from Task 1
- Produces: Green integration test suite confirming the full unfault → DLQ replay path

- [ ] **Step 1: Run the existing integration test**

The existing `DeadLetterReplayIntegrationTest.workerFailure_dlqReplay_succeedsAfterRecovery()` already tests the full unfault → replay → worker success path. It should pass because:
- In-memory tests use `InMemoryCaseInstanceRepository` which implements both repo interfaces on the same backing store
- The race only manifests with JPA — the existing test still validates correctness

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl resilience -Dtest=DeadLetterReplayIntegrationTest -q`
Expected: PASS

- [ ] **Step 2: Run full build**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: PASS — no regressions across all modules.

- [ ] **Step 3: Fix any compilation or test failures**

If the RuntimeBeans change causes CDI wiring failures in other test profiles, fix them. Common issues:
- Test profiles that mock `CaseRecoveryService` need to provide the 7-arg constructor
- `DeadLetterReplayIntegrationTest` injects `CaseRecoveryService` — CDI should auto-resolve the new dependencies via the updated producer

- [ ] **Step 4: Commit (if any fixes were needed)**

```bash
git add -A
git commit -m "fix(#1183): resolve integration test wiring for updated CaseRecoveryService

Refs casehubio/engine#1183

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-27-unfault-hardening-design.md] — design spec this plan implements
- [CaseRecoveryService.java:34] — current implementation (4-arg constructor, no sync persist)
- [RuntimeBeans.java:376-385] — CDI producer for CaseRecoveryService
- [CaseStartedEventHandler.java:101-104] — channel + trigger setup pattern to mirror
- [CaseStatusChangedHandler.java:195-243] — terminal cleanup (what unfault reverses)
- [VertxEventDispatcher.java:101] — async dispatch via eventBus.publish()
- [JpaCrossTenantCaseInstanceRepository.java:60] — fromEntity creates NEW object (root cause of race)
- [DeadLetterReplayIntegrationTest.java:72-121] — existing integration test
- [CaseInstanceRepository.java:40] — update(CaseInstance, String) method
- [CaseChannelProvider.java:45] — openChannel(UUID, String) — idempotent
- [SchedulerService.java:56] — registerScheduledTriggers(CaseInstance)
- [decisions.md D5-D7] — design decisions for this issue
- [GitHub casehubio/engine#1183] — focal issue
