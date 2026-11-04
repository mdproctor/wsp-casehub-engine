# CaseCompleted CDI Event Reliability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `CaseLifecycleEvent("CaseCompleted")` reliably delivered to `@ObservesAsync` observers by awaiting the `CompletionStage` returned by `fireAsync()` inside `CaseStatusChangedHandler`.

**Architecture:** Restructure `CaseStatusChangedHandler.onCaseStatusChangedHandler()` to split the terminal `.invoke()` into: (1) a `.invoke()` for fire-and-forget event bus publishes, and (2) a trailing `.chain()` that awaits `lifecycleEvents.fireAsync()` via `Uni.createFrom().completionStage()`. Observer failures are logged and recovered so ledger write errors cannot fail case completion.

**Tech Stack:** Java 21, Quarkus 3.32.x, Mutiny (Uni, CompletionStage bridge), CDI (`Event.fireAsync`, `@ObservesAsync`), JUnit 5, Awaitility, `CopyOnWriteArrayList`.

---

## File Map

| Action | File |
|--------|------|
| Create | `runtime/src/test/java/io/casehub/engine/CaseLifecycleCdiEventTest.java` |
| Modify | `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` |

---

## Task 1: Write the failing test

**File:**
- Create: `runtime/src/test/java/io/casehub/engine/CaseLifecycleCdiEventTest.java`

- [ ] **Step 1.1: Create the test file**

Create `runtime/src/test/java/io/casehub/engine/CaseLifecycleCdiEventTest.java` with this exact content:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine;

import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.model.Binding;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.Goal;
import io.casehub.api.model.GoalExpression;
import io.casehub.api.model.GoalKind;
import io.casehub.api.model.Worker;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies that {@code CaseLifecycleEvent("CaseCompleted")} is reliably delivered to
 * {@code @ObservesAsync} observers before {@code CaseStatusChangedHandler}'s Uni completes.
 *
 * <p>Regression test for engine#393: {@code lifecycleEvents.fireAsync()} was inside
 * {@code .invoke()}, which discarded the CompletionStage. Moving it to {@code .chain()} with
 * {@code Uni.createFrom().completionStage()} makes delivery deterministic.
 */
@QuarkusTest
@TestProfile(CaseLifecycleCdiEventTest.CdiEventProfile.class)
class CaseLifecycleCdiEventTest {

  @Inject LifecycleCapture capture;
  @Inject CompletionCaseHub caseHub;

  @BeforeEach
  void reset() {
    capture.reset();
  }

  @Test
  void caseCompleted_cdiEventDeliveredToObserver() {
    final AtomicReference<UUID> caseIdRef = new AtomicReference<>();

    caseHub
        .startCase(Map.of("trigger", true))
        .thenAccept(caseIdRef::set)
        .toCompletableFuture()
        .join();

    final UUID caseId = caseIdRef.get();

    await()
        .atMost(15, TimeUnit.SECONDS)
        .until(
            () ->
                capture.received().stream()
                    .anyMatch(
                        e ->
                            caseId.equals(e.caseId())
                                && "CaseCompleted".equals(e.eventType())));
  }

  /**
   * Capture bean for {@link CaseLifecycleEvent} delivered via {@code @ObservesAsync}.
   *
   * <p>Uses {@code CopyOnWriteArrayList} — {@code @ObservesAsync} dispatches on a managed
   * executor thread, not the test thread. {@code ArrayList} is a data race (GE-20260522-bc642c).
   */
  @ApplicationScoped
  public static class LifecycleCapture {

    private final CopyOnWriteArrayList<CaseLifecycleEvent> events = new CopyOnWriteArrayList<>();

    public void onEvent(@ObservesAsync CaseLifecycleEvent event) {
      events.add(event);
    }

    public List<CaseLifecycleEvent> received() {
      return Collections.unmodifiableList(events);
    }

    public void reset() {
      events.clear();
    }
  }

  /**
   * Minimal case that completes via a success goal after a single worker execution.
   *
   * <p>Context {@code {trigger: true}} satisfies the ContextChangeTrigger immediately on start.
   * The worker returns {@code {done: true}}, satisfying the goal.
   */
  @ApplicationScoped
  public static class CompletionCaseHub extends CaseHub {

    private final Capability capability =
        Capability.builder()
            .name("do-work")
            .inputSchema(".")
            .outputSchema("{ done: true }")
            .build();

    private final Goal goal =
        Goal.builder()
            .name("all-done")
            .condition(".done == true")
            .kind(GoalKind.SUCCESS)
            .build();

    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test-cdi-event")
          .name("CdiEventCase")
          .version("1.0")
          .capabilities(capability)
          .workers(
              Worker.builder()
                  .name("finisher")
                  .capabilities(capability)
                  .function(input -> Map.of("done", true))
                  .build())
          .bindings(
              Binding.builder()
                  .name("fire-when-triggered")
                  .capability(capability)
                  .on(new ContextChangeTrigger(".trigger == true"))
                  .build())
          .goals(goal)
          .completion(GoalExpression.allOf(goal))
          .build();
    }
  }

  public static class CdiEventProfile implements QuarkusTestProfile {
    @Override
    public Map<String, String> getConfigOverrides() {
      return Map.of("casehub.engine.diff-strategy", "none");
    }
  }
}
```

- [ ] **Step 1.2: Run the test — record result (may pass or fail non-deterministically)**

```bash
cd /Users/mdproctor/claude/casehub/engine && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseLifecycleCdiEventTest" 2>&1 | grep -E "Tests run:|FAIL|ERROR|BUILD" | tail -10
```

Note the result. Without the fix, the CDI event delivery is non-deterministic — the test may pass by luck if the async delivery completes within 15 seconds, or fail if it doesn't. The fix (Task 2) makes it consistently pass.

---

## Task 2: Fix `CaseStatusChangedHandler`

**File:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`

- [ ] **Step 2.1: Replace the terminal `.invoke()` with two chained operations**

The current terminal block (lines 99–121 in the original file):

```java
        .invoke(
            () -> {
              String eventBusAddress = resolveStateAsString(newState);
              if (eventBusAddress != null) {
                eventBus.publish(eventBusAddress, caseInstance);
              }
              // On resume (SUSPENDED → RUNNING), re-evaluate the context so eligible workers fire.
              if (newState == CaseStatus.RUNNING) {
                eventBus.publish(
                    EventBusAddresses.CONTEXT_CHANGED,
                    new CaseContextChangedEvent(
                        caseInstance, caseInstance.getCaseContext().asJsonNode()));
              }
              lifecycleEvents.fireAsync(
                  new CaseLifecycleEvent(
                      caseInstance.getUuid(),
                      resolveCommandType(newState),
                      resolveEventType(newState),
                      newState.name(),
                      null,
                      "System",
                      traceId));
            });
```

Replace with:

```java
        .invoke(
            () -> {
              // Fire-and-forget: downstream event bus consumers (CASE_COMPLETED, CASE_FAULTED,
              // CONTEXT_CHANGED) do not need to complete before this handler returns.
              String eventBusAddress = resolveStateAsString(newState);
              if (eventBusAddress != null) {
                eventBus.publish(eventBusAddress, caseInstance);
              }
              // On resume (SUSPENDED → RUNNING), re-evaluate the context so eligible workers fire.
              if (newState == CaseStatus.RUNNING) {
                eventBus.publish(
                    EventBusAddresses.CONTEXT_CHANGED,
                    new CaseContextChangedEvent(
                        caseInstance, caseInstance.getCaseContext().asJsonNode()));
              }
            })
        .chain(
            () ->
                // Await CDI event delivery so @ObservesAsync observers run before this handler's
                // Uni completes. Failure is logged and recovered — observer errors must not fail
                // case completion (engine#393).
                Uni.createFrom()
                    .completionStage(
                        () ->
                            lifecycleEvents.fireAsync(
                                new CaseLifecycleEvent(
                                    caseInstance.getUuid(),
                                    resolveCommandType(newState),
                                    resolveEventType(newState),
                                    newState.name(),
                                    null,
                                    "System",
                                    traceId)))
                    .onFailure()
                    .invoke(
                        t ->
                            LOG.warnf(
                                t,
                                "CaseLifecycleEvent observer failed for caseId=%s event=%s",
                                caseInstance.getUuid(), resolveEventType(newState)))
                    .recoverWithNull()
                    .replaceWithVoid());
```

No new imports are needed — `Uni` is already imported.

- [ ] **Step 2.2: Run the test — expect consistent green**

```bash
cd /Users/mdproctor/claude/casehub/engine && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseLifecycleCdiEventTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: `Tests run: 1, Failures: 0, Errors: 0, BUILD SUCCESS`.

Run it a second time to confirm it is not flaky:

```bash
cd /Users/mdproctor/claude/casehub/engine && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="CaseLifecycleCdiEventTest" 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

Expected: same result both times.

---

## Task 3: Full test suite verification

- [ ] **Step 3.1: Install dependencies**

```bash
cd /Users/mdproctor/claude/casehub/engine && mvn install -DskipTests -q
```

- [ ] **Step 3.2: Run runtime module (largest — includes integration tests)**

```bash
cd /Users/mdproctor/claude/casehub/engine && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 3.3: Run remaining modules**

```bash
cd /Users/mdproctor/claude/casehub/engine && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,ledger,scheduler-quartz,blackboard 2>&1 | grep -E "Tests run:|BUILD" | tail -10
```

Expected: `BUILD SUCCESS` for all four.

---

## Task 4: File follow-up issue for the other five handlers

Five other production handlers have the identical `.invoke()`-discard pattern for `fireAsync()`. File one issue tracking all five.

- [ ] **Step 4.1: File the issue**

```bash
gh issue create --repo casehubio/engine --title "fix: await fireAsync CompletionStage in all remaining lifecycle event handlers" --body "$(cat <<'EOF'
## Context

engine#393 fixed \`CaseStatusChangedHandler\` to await \`lifecycleEvents.fireAsync()\` by chaining \`Uni.createFrom().completionStage()\` instead of discarding the result inside \`.invoke()\`.

Five other handlers have the identical pattern:

| Handler | Event |
|---------|-------|
| \`GoalReachedEventHandler\` | \`GoalReached\` |
| \`MilestoneReachedEventHandler\` | \`MilestoneReached\` |
| \`SignalReceivedEventHandler\` | \`SignalReceived\` |
| \`CaseStartedEventHandler\` | \`CaseStarted\` |
| \`WorkflowExecutionCompletedHandler\` | \`WorkerCompleted\` |

## Fix

For each handler, apply the same pattern as engine#393:
- Move \`lifecycleEvents.fireAsync()\` from \`.invoke()\` to a trailing \`.chain(() -> Uni.createFrom().completionStage(...))\`
- Add \`.onFailure().invoke(t -> LOG.warnf(...)).recoverWithNull().replaceWithVoid()\`

## Tests

For each handler: add a \`@QuarkusTest\` (or extend the pattern from \`CaseLifecycleCdiEventTest\`) that polls a CDI capture bean for the specific event type.

## Depends on

engine#393 (pattern established — copy it)
EOF
)"
```

Report the issue URL.

---

## Task 5: Commit

- [ ] **Step 5.1: Stage files**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java \
  runtime/src/test/java/io/casehub/engine/CaseLifecycleCdiEventTest.java
```

- [ ] **Step 5.2: Verify diff stat**

```bash
git -C /Users/mdproctor/claude/casehub/engine diff --staged --stat
```

Expected: 2 files changed.

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
fix(engine): await CaseLifecycleEvent CDI delivery in CaseStatusChangedHandler

lifecycleEvents.fireAsync() was inside .invoke(), which discards the
CompletionStage. Observers could receive CaseCompleted after the handler's
Uni resolved, making delivery non-deterministic in test environments.

Move fireAsync() to a trailing .chain() using Uni.createFrom().completionStage()
so the handler completes only after all @ObservesAsync observers have run.
Observer failures are logged and recovered to prevent ledger errors from
failing case completion.

Event bus publishes (CASE_COMPLETED, CASE_FAULTED, CONTEXT_CHANGED) remain
in .invoke() — they are fire-and-forget downstream processing, not audit.

Adds CaseLifecycleCdiEventTest: polls an @ObservesAsync capture bean
(CopyOnWriteArrayList for thread-safety) and asserts CaseCompleted is
received within 15s for a case that completes via goal satisfaction.

Closes #393
EOF
)"
```

- [ ] **Step 5.4: Confirm**

```bash
git -C /Users/mdproctor/claude/casehub/engine log --oneline -1
```
