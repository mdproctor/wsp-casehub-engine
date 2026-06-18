# Expired Outcome Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire engine-internal worker timeouts to `OutcomePolicy.onExpired` via a new `WorkerOutcome.Expired` sealed variant.

**Architecture:** Add `Expired` to `WorkerOutcome` sealed hierarchy. Convert `TimeoutException` to `WorkerResult.expired()` inside `DefaultWorkerExecutor.executeSync()` so the SPI boundary never leaks exceptions. `WorkflowExecutionCompletedHandler.handleSemanticFailure()` branches on the new variant using a consolidated exhaustive switch.

**Tech Stack:** Java 21 sealed types, Quarkus CDI, Mutiny Uni, Vert.x event bus

**Spec:** `docs/specs/2026-06-18-expired-outcome-wiring-design.md`

---

### Task 1: WorkerOutcome.Expired variant + WorkerResult factories

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/WorkerOutcome.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkerResult.java`
- Test: `api/src/test/java/io/casehub/api/model/WorkerResultExpiredTest.java`

- [ ] **Step 1: Write the failing test**

Create `api/src/test/java/io/casehub/api/model/WorkerResultExpiredTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Map;
import org.junit.jupiter.api.Test;

class WorkerResultExpiredTest {

  @Test
  void expired_factory_creates_expired_outcome_with_empty_output() {
    WorkerResult result = WorkerResult.expired("timed out");
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
    assertThat(((WorkerOutcome.Expired) result.outcome()).reason()).isEqualTo("timed out");
    assertThat(result.output()).isEmpty();
    assertThat(result.plannedAction()).isNull();
  }

  @Test
  void expired_factory_with_partial_output() {
    Map<String, Object> partial = Map.of("progress", "50%");
    WorkerResult result = WorkerResult.expired("deadline passed", partial);
    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
    assertThat(result.output()).isEqualTo(partial);
  }

  @Test
  void expired_outcome_rejects_planned_action() {
    assertThatThrownBy(
            () ->
                new WorkerResult(
                    Map.of(),
                    io.casehub.api.spi.PlannedAction.of("action", "type", Map.of()),
                    new WorkerOutcome.Expired("timed out")))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void expired_is_sealed_variant() {
    WorkerOutcome outcome = new WorkerOutcome.Expired("reason");
    assertThat(outcome).isInstanceOf(WorkerOutcome.class);
    String matched =
        switch (outcome) {
          case WorkerOutcome.Success s -> "success";
          case WorkerOutcome.Declined d -> "declined";
          case WorkerOutcome.Failed f -> "failed";
          case WorkerOutcome.Expired e -> "expired:" + e.reason();
        };
    assertThat(matched).isEqualTo("expired:reason");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=WorkerResultExpiredTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode -q`

Expected: Compilation failure — `WorkerOutcome.Expired` does not exist.

- [ ] **Step 3: Add Expired variant to WorkerOutcome**

In `api/src/main/java/io/casehub/api/model/WorkerOutcome.java`, add `Expired` to the permits list and add the record:

```java
public sealed interface WorkerOutcome
    permits WorkerOutcome.Success, WorkerOutcome.Declined, WorkerOutcome.Failed,
        WorkerOutcome.Expired {

  record Success() implements WorkerOutcome {}

  record Declined(String reason) implements WorkerOutcome {}

  record Failed(String reason) implements WorkerOutcome {}

  record Expired(String reason) implements WorkerOutcome {}

  static WorkerOutcome success() {
    return new Success();
  }
}
```

- [ ] **Step 4: Add expired factories to WorkerResult**

In `api/src/main/java/io/casehub/api/model/WorkerResult.java`, add after the `failed` factories (after line 65):

```java
  public static WorkerResult expired(final String reason) {
    return new WorkerResult(Map.of(), null, new WorkerOutcome.Expired(reason));
  }

  public static WorkerResult expired(final String reason, final Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, null, new WorkerOutcome.Expired(reason));
  }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=WorkerResultExpiredTest --batch-mode -q`

Expected: All 4 tests PASS.

- [ ] **Step 6: Commit**

```
git add api/src/main/java/io/casehub/api/model/WorkerOutcome.java \
       api/src/main/java/io/casehub/api/model/WorkerResult.java \
       api/src/test/java/io/casehub/api/model/WorkerResultExpiredTest.java
git commit -m "feat: add WorkerOutcome.Expired sealed variant and WorkerResult.expired factories

Refs #513"
```

---

### Task 2: WorkStatus.EXPIRED + WorkResult.expired factory

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/WorkStatus.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkResult.java`
- Test: `api/src/test/java/io/casehub/api/model/WorkResultExpiredTest.java` (extend)

- [ ] **Step 1: Write the failing test**

Add to `WorkerResultExpiredTest.java`:

```java
  @Test
  void work_result_expired_factory() {
    WorkResult result = WorkResult.expired("hash-123", "worker-1", java.util.UUID.randomUUID());
    assertThat(result.status()).isEqualTo(WorkStatus.EXPIRED);
    assertThat(result.correlationKey()).isEqualTo("hash-123");
    assertThat(result.workerId()).isEqualTo("worker-1");
    assertThat(result.output()).isEmpty();
  }
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=WorkerResultExpiredTest --batch-mode -q`

Expected: Compilation failure — `WorkStatus.EXPIRED` and `WorkResult.expired(String, String, UUID)` do not exist.

- [ ] **Step 3: Add EXPIRED to WorkStatus**

In `api/src/main/java/io/casehub/api/model/WorkStatus.java`, add `EXPIRED` after `FAILED` (line 24):

```java
public enum WorkStatus {
  PENDING,
  RUNNING,
  COMPLETED,
  DECLINED,
  FAILED,
  EXPIRED,
  FAULTED,
  CANCELLED
}
```

- [ ] **Step 4: Add expired factory to WorkResult**

In `api/src/main/java/io/casehub/api/model/WorkResult.java`, add after the `failed` factory (after line 59):

```java
  public static WorkResult expired(String correlationKey, String workerId, UUID caseId) {
    return new WorkResult(correlationKey, WorkStatus.EXPIRED, Map.of(), workerId, caseId);
  }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=WorkerResultExpiredTest --batch-mode -q`

Expected: All 5 tests PASS.

- [ ] **Step 6: Commit**

```
git add api/src/main/java/io/casehub/api/model/WorkStatus.java \
       api/src/main/java/io/casehub/api/model/WorkResult.java \
       api/src/test/java/io/casehub/api/model/WorkerResultExpiredTest.java
git commit -m "feat: add WorkStatus.EXPIRED and WorkResult.expired factory for SPI boundary

Refs #513"
```

---

### Task 3: CaseHubEventType.WORKER_OUTCOME_EXPIRED

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`

- [ ] **Step 1: Add the enum value**

In `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`, add after `WORKER_OUTCOME_FAILED` (line 35):

```java
  WORKER_OUTCOME_DECLINED, // worker ran correctly but declined the work (semantic boundary)
  WORKER_OUTCOME_FAILED, // worker ran correctly but could not complete (semantic failure)
  WORKER_OUTCOME_EXPIRED, // worker timed out (engine-internal or commitment expiration)
```

- [ ] **Step 2: Commit**

```
git add api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java
git commit -m "feat: add CaseHubEventType.WORKER_OUTCOME_EXPIRED

Refs #513"
```

---

### Task 4: Timeout conversion in DefaultWorkerExecutor

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerExecutor.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerExecutorTimeoutTest.java`

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerExecutorTimeoutTest.java`:

```java
package io.casehub.engine.internal.executor;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.WorkerContext;
import io.casehub.api.model.WorkerFunction;
import io.casehub.api.model.WorkerOutcome;
import io.casehub.api.model.WorkerResult;
import io.casehub.engine.common.internal.executor.ExecutionMetadata;
import io.casehub.engine.common.internal.executor.WorkerExecutor;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class DefaultWorkerExecutorTimeoutTest {

  @Inject WorkerExecutor workerExecutor;

  @Test
  void timeout_produces_expired_outcome_not_exception() {
    WorkerFunction.Sync slowWorker =
        new WorkerFunction.Sync(
            input -> {
              try {
                Thread.sleep(5000);
              } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
              }
              return WorkerResult.of(Map.of("result", "late"));
            });

    WorkerContext context =
        new WorkerContext(UUID.randomUUID(), "test-worker", Map.of(), Map.of());

    WorkerResult result =
        workerExecutor
            .execute(
                slowWorker,
                Map.of(),
                context,
                200, // 200ms timeout — worker sleeps 5s
                null,
                new ExecutionMetadata("test-worker", "hash-1"))
            .await()
            .atMost(Duration.ofSeconds(10));

    assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Expired.class);
    assertThat(((WorkerOutcome.Expired) result.outcome()).reason()).contains("200ms");
    assertThat(result.output()).isEmpty();
  }
}
```

- [ ] **Step 2: Install deps then run test to verify it fails**

Run:
```
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultWorkerExecutorTimeoutTest --batch-mode -q
```

Expected: FAIL — `TimeoutException` is thrown instead of returning `WorkerResult.expired()`.

- [ ] **Step 3: Add timeout recovery to executeSync**

In `runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerExecutor.java`, modify `executeSync()` (lines 89–101). Add the `onFailure` recovery after `.fail()`:

```java
  private Uni<WorkerResult> executeSync(
      Function<Map<String, Object>, WorkerResult> fn,
      Map<String, Object> inputData,
      WorkerContext context,
      int timeoutMs) {

    return Uni.createFrom()
        .item(
            () -> {
              WorkerExecutionContext.set(context);
              try {
                return fn.apply(inputData);
              } finally {
                WorkerExecutionContext.clear();
              }
            })
        .runSubscriptionOn(virtualThreads)
        .ifNoItem()
        .after(Duration.ofMillis(timeoutMs))
        .fail()
        .onFailure(java.util.concurrent.TimeoutException.class)
        .recoverWithItem(
            t -> WorkerResult.expired("Worker timed out after " + timeoutMs + "ms"));
  }
```

Add the import at the top of the file:

```java
import java.util.concurrent.TimeoutException;
```

(The import isn't strictly needed since we use the fully qualified name, but add it for consistency — the `onFailure` lambda parameter `t` is typed by the class literal.)

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultWorkerExecutorTimeoutTest --batch-mode -q`

Expected: PASS.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/executor/DefaultWorkerExecutor.java \
       runtime/src/test/java/io/casehub/engine/internal/executor/DefaultWorkerExecutorTimeoutTest.java
git commit -m "feat: convert TimeoutException to WorkerResult.expired in DefaultWorkerExecutor

The executor owns the timeout (.ifNoItem().after().fail()) so it owns the
semantic conversion. The WorkerExecutor SPI contract Uni<WorkerResult> no
longer leaks TimeoutException — adapters only see WorkerResult.

Refs #513"
```

---

### Task 5: WorkflowExecutionCompletedHandler — Expired branch

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`

- [ ] **Step 1: Change the outcome fork to negative check**

At line 97–100, replace the positive instanceof chain:

```java
    // Outcome fork: non-success outcomes route to the semantic failure path.
    if (event.outcome() instanceof WorkerOutcome.Declined
        || event.outcome() instanceof WorkerOutcome.Failed) {
      return handleSemanticFailure(event, traceId);
    }
```

With the negative check:

```java
    // Outcome fork: non-success outcomes route to the semantic failure path.
    if (!(event.outcome() instanceof WorkerOutcome.Success)) {
      return handleSemanticFailure(event, traceId);
    }
```

- [ ] **Step 2: Replace the three separate extractions with a consolidated switch**

In `handleSemanticFailure()`, replace lines 265–273 (the `action`, `outcomeStatus`, and `reason` extractions):

```java
    final OutcomeAction action =
        event.outcome() instanceof WorkerOutcome.Declined ? policy.onDecline() : policy.onFailure();

    final String outcomeStatus =
        event.outcome() instanceof WorkerOutcome.Declined ? "DECLINED" : "FAILED";
    final String reason =
        event.outcome() instanceof WorkerOutcome.Declined d
            ? d.reason()
            : ((WorkerOutcome.Failed) event.outcome()).reason();
```

With the consolidated exhaustive switch:

```java
    final String outcomeStatus;
    final String reason;
    final OutcomeAction action;
    final CaseHubEventType eventType;

    switch (event.outcome()) {
      case WorkerOutcome.Declined d -> {
        outcomeStatus = "DECLINED";
        reason = d.reason();
        action = policy.onDecline();
        eventType = CaseHubEventType.WORKER_OUTCOME_DECLINED;
      }
      case WorkerOutcome.Failed f -> {
        outcomeStatus = "FAILED";
        reason = f.reason();
        action = policy.onFailure();
        eventType = CaseHubEventType.WORKER_OUTCOME_FAILED;
      }
      case WorkerOutcome.Expired e -> {
        outcomeStatus = "EXPIRED";
        reason = e.reason();
        action = policy.onExpired();
        eventType = CaseHubEventType.WORKER_OUTCOME_EXPIRED;
      }
      case WorkerOutcome.Success s ->
          throw new IllegalStateException("Success should not reach handleSemanticFailure");
    }
```

- [ ] **Step 3: Remove the old eventType extraction**

Delete the standalone `eventType` extraction at lines 336–339 (it's now in the consolidated switch):

```java
    // DELETE THIS — now extracted in the consolidated switch above
    final CaseHubEventType eventType =
        event.outcome() instanceof WorkerOutcome.Declined
            ? CaseHubEventType.WORKER_OUTCOME_DECLINED
            : CaseHubEventType.WORKER_OUTCOME_FAILED;
```

- [ ] **Step 4: Replace the WorkResult ternary with exhaustive switch**

Replace lines 368–373:

```java
              final WorkResult workResult =
                  outcomeStatus.equals("DECLINED")
                      ? WorkResult.declined(
                          event.idempotency(), worker.getName(), caseInstance.getUuid())
                      : WorkResult.failed(
                          event.idempotency(), worker.getName(), caseInstance.getUuid());
```

With:

```java
              final WorkResult workResult =
                  switch (event.outcome()) {
                    case WorkerOutcome.Declined d ->
                        WorkResult.declined(
                            event.idempotency(), worker.getName(), caseInstance.getUuid());
                    case WorkerOutcome.Failed f ->
                        WorkResult.failed(
                            event.idempotency(), worker.getName(), caseInstance.getUuid());
                    case WorkerOutcome.Expired e ->
                        WorkResult.expired(
                            event.idempotency(), worker.getName(), caseInstance.getUuid());
                    case WorkerOutcome.Success s ->
                        throw new IllegalStateException(
                            "Success should not reach handleSemanticFailure");
                  };
```

- [ ] **Step 5: Compile check**

Run: `mvn compile -pl runtime -q --batch-mode`

Expected: BUILD SUCCESS (no compilation errors).

- [ ] **Step 6: Commit**

```
git add runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java
git commit -m "feat: wire WorkerOutcome.Expired through handleSemanticFailure

Negative instanceof check for future safety. Consolidated exhaustive switch
extracts outcomeStatus, reason, action, eventType in one place. WorkResult
construction uses exhaustive switch to prevent EXPIRED→FAILED information loss.

Refs #513"
```

---

### Task 6: Integration test — expired outcome with FAULT policy

**Files:**
- Modify: `runtime/src/test/java/io/casehub/engine/FailureCascadeIntegrationTest.java`

- [ ] **Step 1: Write the failing test**

Add to `FailureCascadeIntegrationTest.java` — a new `@Inject` field and test method. Add the import for `WorkerOutcome`:

Add a new `ExpiredFaultPolicyBean` inner class and test:

```java
  @Inject ExpiredFaultPolicyBean expiredFaultPolicyBean;

  @Test
  void expired_with_fault_policy_faults_case() {
    UUID caseId =
        expiredFaultPolicyBean
            .startCase(Map.of("task", "pending"))
            .toCompletableFuture()
            .join();

    await()
        .atMost(30, TimeUnit.SECONDS)
        .untilAsserted(
            () ->
                assertEquals(
                    CaseStatus.FAULTED,
                    caseInstanceCache.get(caseId).getState(),
                    "Case must be FAULTED when worker times out and onExpired is FAULT"));
  }

  @Test
  void expired_produces_worker_outcome_expired_event_log() {
    UUID caseId =
        expiredFaultPolicyBean
            .startCase(Map.of("task", "pending"))
            .toCompletableFuture()
            .join();

    await()
        .atMost(30, TimeUnit.SECONDS)
        .untilAsserted(
            () -> assertEquals(CaseStatus.FAULTED, caseInstanceCache.get(caseId).getState()));

    await()
        .atMost(10, TimeUnit.SECONDS)
        .untilAsserted(
            () -> {
              List<EventLog> expired =
                  findEvents(caseId, CaseHubEventType.WORKER_OUTCOME_EXPIRED);
              assertEquals(
                  1, expired.size(), "Exactly one WORKER_OUTCOME_EXPIRED event log entry");
              assertEquals("FAULT", expired.get(0).getMetadata().get("disposition").asText());
            });
  }
```

Add the `ExpiredFaultPolicyBean` inner class (after `FaultPolicyBean`):

```java
  @ApplicationScoped
  public static class ExpiredFaultPolicyBean extends CaseHub {

    private final Capability cap =
        Capability.builder()
            .name("expired-cap")
            .inputSchema("{ task: .task }")
            .outputSchema(".")
            .build();

    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test-failure-cascade-expired")
          .name("Expired Fault Policy Test")
          .version("1.0.0")
          .capabilities(cap)
          .workers(
              Worker.builder()
                  .name("slow-worker")
                  .capabilities(cap)
                  .function(
                      input -> {
                        try {
                          Thread.sleep(5000);
                        } catch (InterruptedException e) {
                          Thread.currentThread().interrupt();
                        }
                        return WorkerResult.of(Map.of("result", "late"));
                      })
                  .executionPolicy(new ExecutionPolicy(200, new RetryPolicy(1, 100)))
                  .build())
          .bindings(
              Binding.builder()
                  .name("on-task")
                  .capability(cap)
                  .outcomePolicy(
                      new OutcomePolicy(
                          OutcomeAction.REROUTE, OutcomeAction.REROUTE, OutcomeAction.FAULT, 1))
                  .on(new ContextChangeTrigger(".task == \"pending\""))
                  .build())
          .build();
    }
  }
```

Note: The worker has a 200ms timeout (`ExecutionPolicy(200, ...)`) but sleeps for 5 seconds. `onExpired` is set to `FAULT` so the case should reach FAULTED state via the expired outcome path.

- [ ] **Step 2: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=FailureCascadeIntegrationTest --batch-mode -q`

Expected: All tests PASS (including the 2 new expired tests + 3 existing).

- [ ] **Step 3: Commit**

```
git add runtime/src/test/java/io/casehub/engine/FailureCascadeIntegrationTest.java
git commit -m "test: integration test for expired outcome with FAULT policy

Worker with 200ms timeout and 5s sleep triggers WorkerOutcome.Expired.
OutcomePolicy.onExpired=FAULT faults the case immediately. Verifies
WORKER_OUTCOME_EXPIRED event log with FAULT disposition.

Refs #513"
```

---

### Task 7: Javadoc updates and CLAUDE.md

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/OutcomePolicy.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update OutcomePolicy javadoc**

In `api/src/main/java/io/casehub/api/model/OutcomePolicy.java`, remove the "not yet wired" qualifier. Replace lines 23–30:

```java
/**
 * Policy for handling semantic worker outcomes (DECLINED, FAILED, EXPIRED) on a per-binding basis.
 *
 * <p>Each outcome type maps to an {@link OutcomeAction}: {@code REROUTE} writes failure state and
 * re-dispatches to a different agent; {@code FAULT} marks the case FAULTED immediately.
 *
 * @param onDecline action when a worker returns {@link WorkerOutcome.Declined}
 * @param onFailure action when a worker returns {@link WorkerOutcome.Failed}
 * @param onExpired action when a worker times out or its commitment expires
 * @param maxRerouteAttempts maximum dispatch+outcome cycles before writing REROUTES_EXHAUSTED
 */
```

- [ ] **Step 2: Update WorkflowExecutionCompleted javadoc**

In `common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java`, update the javadoc (line 31) to include Expired:

Replace:
```
 * <p>{@code outcome} carries the worker's semantic result: {@link WorkerOutcome.Success}, {@link
 * WorkerOutcome.Declined}, or {@link WorkerOutcome.Failed}. The completion handler branches on this
 * before checking {@code plannedAction}.
```

With:
```
 * <p>{@code outcome} carries the worker's semantic result: {@link WorkerOutcome.Success}, {@link
 * WorkerOutcome.Declined}, {@link WorkerOutcome.Failed}, or {@link WorkerOutcome.Expired}. The
 * completion handler branches on this before checking {@code plannedAction}.
```

- [ ] **Step 3: Update CLAUDE.md Worker Outcome Handling section**

In `CLAUDE.md`, find the `## Worker Outcome Handling` section. Add EXPIRED to the outcome descriptions. After the line about `REROUTE` (default) and `FAULT`, add to the introductory text:

After the sentence "Workers declare semantic outcomes via `WorkerResult`: `Success` (default), `Declined(reason)`, `Failed(reason)`." add `Expired(reason)`:

```
Workers declare semantic outcomes via `WorkerResult`: `Success` (default), `Declined(reason)`, `Failed(reason)`, `Expired(reason)`.
```

And add after the `FAULT` bullet:

```
`Expired` outcomes originate from two sources: engine-internal worker timeout (`DefaultWorkerExecutor` converts `TimeoutException` to `WorkerResult.expired()` — the SPI boundary never leaks exceptions) and Qhorus commitment expiration (future, qhorus#281). Both route through `OutcomePolicy.onExpired` using the same `handleSemanticFailure` path as `Declined` and `Failed`.
```

- [ ] **Step 4: Compile check**

Run: `mvn compile -pl api,common --batch-mode -q`

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```
git add api/src/main/java/io/casehub/api/model/OutcomePolicy.java \
       common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java \
       CLAUDE.md
git commit -m "docs: update javadocs and CLAUDE.md for EXPIRED outcome wiring

OutcomePolicy: remove 'not yet wired' qualifier from onExpired.
WorkflowExecutionCompleted: add Expired to outcome list.
CLAUDE.md: document EXPIRED in Worker Outcome Handling section.

Closes #513"
```

---

### Task 8: Full test suite verification

- [ ] **Step 1: Run full api module tests**

Run: `mvn test -pl api --batch-mode -q`

Expected: All tests PASS (including existing `OutcomePolicyTest` and new `WorkerResultExpiredTest`).

- [ ] **Step 2: Run full runtime module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime --batch-mode -q`

Expected: All tests PASS.

- [ ] **Step 3: Run blackboard module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-blackboard --batch-mode -q`

Expected: All tests PASS. The blackboard `PlanItemCompletionHandler` already gates on `!(instanceof Success)` — no change needed, but verify it handles `Expired` correctly.

- [ ] **Step 4: Run scheduler-quartz module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl scheduler-quartz --batch-mode -q`

Expected: All tests PASS. The Quartz adapter no longer sees `TimeoutException` — only genuine infrastructure failures.
