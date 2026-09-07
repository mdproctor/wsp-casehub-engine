# StepOutcomeObserver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1050 — RoutingOutcomeRecorder: fire per-step, not just at case completion
**Issue group:** #1050

**Goal:** Add a `StepOutcomeObserver` SPI that fires after each worker execution with the working
layer context snapshot, enabling consumers to build per-step CBR corpora.

**Architecture:** New SPI interface + event record in `api/spi/`, `@DefaultBean` no-op in
`runtime/internal/worker/`, two call sites in `WorkflowExecutionCompletedHandler` (success and
failure paths). Symmetric with `CaseOutcomeObserver`. The engine notifies; consumers own the
recording shape.

**Tech Stack:** Java 21, Quarkus (CDI, Vert.x event bus), Jackson

## Global Constraints

- `@DefaultBean @ApplicationScoped` for no-op defaults (PP-20260514-engine-spi-noops-defaultbean)
- `@ConsumeEvent` handlers use `@RunOnVirtualThread` + `void` (PP-20260723-c4c1cf)
- Operational SPIs go in `api/spi/` (SPI placement rule in CLAUDE.md)
- `Instance<>` injection with `isUnsatisfied()` guard for optional SPIs
- Apache 2.0 license header on all new files

---

## Batch 1: StepOutcomeObserver SPI and wiring

### Task 1: StepOutcomeEvent record and StepOutcomeObserver interface

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/StepOutcomeEvent.java`
- Create: `api/src/main/java/io/casehub/api/spi/StepOutcomeObserver.java`

**Interfaces:**
- Consumes: `RoutingOutcome` from `io.casehub.api.spi.routing`
- Produces: `StepOutcomeEvent` record and `StepOutcomeObserver` interface — used by Task 2 (no-op default), Task 3 (handler wiring), Task 4 (test)

- [ ] **Step 1: Create `StepOutcomeEvent` record**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...standard Apache 2.0 header...
 */
package io.casehub.api.spi;

import io.casehub.api.spi.routing.RoutingOutcome;
import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import org.jspecify.annotations.Nullable;

/**
 * Outcome event fired by the engine after each worker execution step completes — on both success
 * and failure paths. Delivered to all {@link StepOutcomeObserver} beans discovered via CDI.
 *
 * <p>{@code contextSnapshot} is the working layer at step execution time — on the success path,
 * captured <em>before</em> output application (the conditions under which the decision was made,
 * not the world after execution). On the failure path, captured at failure handling time (no output
 * was applied).
 *
 * <p>Refs casehubio/engine#1050.
 *
 * @param caseId case instance UUID
 * @param tenancyId tenant identifier owning the case
 * @param caseType case definition name — consumer uses this to find their CaseDefinition/CbrConfig
 * @param bindingName the case definition binding that dispatched the worker
 * @param capabilityName the capability targeted by this binding; nullable for JudgmentTarget traces
 * @param workerName the worker that executed
 * @param outcome the routing outcome (SUCCESS or FAILURE)
 * @param contextSnapshot working-layer context at step execution time; non-null, may be empty
 * @param executionDuration wall-clock duration of the worker execution; nullable
 */
public record StepOutcomeEvent(
    UUID caseId,
    String tenancyId,
    String caseType,
    String bindingName,
    @Nullable String capabilityName,
    String workerName,
    RoutingOutcome outcome,
    Map<String, Object> contextSnapshot,
    @Nullable Duration executionDuration) {}
```

- [ ] **Step 2: Create `StepOutcomeObserver` interface**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...standard Apache 2.0 header...
 */
package io.casehub.api.spi;

/**
 * Lifecycle hook called by the engine after each worker execution step completes.
 *
 * <p>The engine discovers all {@code @ApplicationScoped StepOutcomeObserver} beans via CDI and
 * calls {@link #onStepOutcome(StepOutcomeEvent)} for each when a worker execution finishes —
 * on both success and failure paths. Implementations record per-step CBR cases, update step-level
 * metrics, or perform other step-outcome-based learning operations.
 *
 * <p><strong>Activation:</strong> declare an {@code @ApplicationScoped} bean implementing this
 * interface. No module dependency is required beyond {@code casehub-engine-api}. The engine
 * discovers all implementations automatically.
 *
 * <p><strong>Blocking:</strong> {@code onStepOutcome()} is called on a virtual thread (the handler
 * uses {@code @RunOnVirtualThread}). Implementations may perform blocking work directly, including
 * JPA writes and {@code @Transactional} operations. The engine catches and logs all exceptions
 * thrown by observers without propagating them.
 *
 * <p>Refs casehubio/engine#1050.
 */
public interface StepOutcomeObserver {

  /**
   * Called after a worker execution step completes.
   *
   * @param event structured outcome carrying step identity, context snapshot, and outcome
   */
  void onStepOutcome(StepOutcomeEvent event);
}
```

- [ ] **Step 3: Verify api module compiles**

Run: `mvn compile -pl api -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/api/spi/StepOutcomeEvent.java api/src/main/java/io/casehub/api/spi/StepOutcomeObserver.java
git commit -m "feat(#1050): StepOutcomeObserver SPI — interface and event record

Refs #1050"
```

### Task 2: NoOpStepOutcomeObserver default bean and handler wiring

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpStepOutcomeObserver.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`

**Interfaces:**
- Consumes: `StepOutcomeObserver`, `StepOutcomeEvent` (Task 1)
- Produces: `fireStepOutcomeObserver()` private method on handler — called on success (line ~221) and failure (line ~463) paths

- [ ] **Step 1: Create `NoOpStepOutcomeObserver`**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...standard Apache 2.0 header...
 */
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.StepOutcomeEvent;
import io.casehub.api.spi.StepOutcomeObserver;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

/** Default no-op StepOutcomeObserver. Active when no consumer provides an implementation. */
@DefaultBean
@ApplicationScoped
public class NoOpStepOutcomeObserver implements StepOutcomeObserver {

  @Override
  public void onStepOutcome(StepOutcomeEvent event) {
    // intentional no-op
  }
}
```

- [ ] **Step 2: Add `Instance<StepOutcomeObserver>` injection to `WorkflowExecutionCompletedHandler`**

Add field after the existing `outcomeRecorder` injection (around line 119):

```java
@Inject
jakarta.enterprise.inject.Instance<io.casehub.api.spi.StepOutcomeObserver>
    stepOutcomeObserver;
```

- [ ] **Step 3: Add `MAP_TYPE` constant to `WorkflowExecutionCompletedHandler`**

Add after the existing `OBJECT_MAPPER` constant (line 88):

```java
private static final com.fasterxml.jackson.core.type.TypeReference<Map<String, Object>> MAP_TYPE =
    new com.fasterxml.jackson.core.type.TypeReference<>() {};
```

- [ ] **Step 4: Add `fireStepOutcomeObserver` private method**

Add before the closing brace of the class (before `fireOutcomeRecorder` or after it):

```java
private void fireStepOutcomeObserver(
    CaseInstance caseInstance,
    Worker worker,
    String bindingName,
    io.casehub.api.spi.routing.RoutingOutcome outcome,
    Map<String, Object> contextSnapshot,
    Long executionDurationMs) {
  if (stepOutcomeObserver.isUnsatisfied()) {
    return;
  }
  String capabilityName = extractCapabilityTag(caseInstance, worker, bindingName);
  java.time.Duration duration =
      executionDurationMs != null ? java.time.Duration.ofMillis(executionDurationMs) : null;
  try {
    stepOutcomeObserver.get().onStepOutcome(new io.casehub.api.spi.StepOutcomeEvent(
        caseInstance.getUuid(),
        caseInstance.tenancyId,
        caseInstance.getCaseMetaModel().getName(),
        bindingName,
        capabilityName,
        worker.name(),
        outcome,
        contextSnapshot,
        duration));
  } catch (Exception err) {
    LOG.warnf(err,
        "Step outcome observation failed for caseId=%s worker=%s binding=%s",
        caseInstance.getUuid(), worker.name(), bindingName);
  }
}
```

- [ ] **Step 5: Wire success path — capture working layer before output and fire observer**

In `onWorkflowExecutionCompletedHandler()`, insert a working layer snapshot capture
**before** `contextOutputApplier.apply()` (before line 205). Add alongside the existing
`contextBefore` local variable (line 204):

```java
Map<String, Object> workingLayerBefore = OBJECT_MAPPER.convertValue(
    caseInstance.getCaseContext().layer(ContextLayer.WORKING).asJsonNode(), MAP_TYPE);
```

Then, after the existing `fireOutcomeRecorder(...)` call (line 221-226), insert:

```java
fireStepOutcomeObserver(
    caseInstance, worker, bindingName,
    io.casehub.api.spi.routing.RoutingOutcome.SUCCESS,
    workingLayerBefore,
    extractDurationMs(event));
```

- [ ] **Step 6: Wire failure path — fire observer in `handleSemanticFailure`**

In `handleSemanticFailure()`, after the existing `fireOutcomeRecorder(...)` call (line 463-468),
insert:

```java
fireStepOutcomeObserver(
    caseInstance, worker, bindingName,
    io.casehub.api.spi.routing.RoutingOutcome.FAILURE,
    OBJECT_MAPPER.convertValue(
        caseInstance.getCaseContext().layer(ContextLayer.WORKING).asJsonNode(), MAP_TYPE),
    extractDurationMs(event));
```

- [ ] **Step 7: Verify runtime module compiles**

Run: `mvn install -DskipTests -q && mvn compile -pl runtime -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/worker/NoOpStepOutcomeObserver.java runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java
git commit -m "feat(#1050): wire StepOutcomeObserver into WorkflowExecutionCompletedHandler

NoOpStepOutcomeObserver @DefaultBean. Handler fires on both success
(pre-output working layer snapshot) and failure paths.

Refs #1050"
```

### Task 3: StepOutcomeObserver integration test

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/StepOutcomeObserverTest.java`

**Interfaces:**
- Consumes: `StepOutcomeObserver`, `StepOutcomeEvent` (Task 1), handler wiring (Task 2)
- Produces: test coverage — success/failure/exception-isolation assertions

- [ ] **Step 1: Write the test class**

Pattern follows `CaseOutcomeObserverTest` — inject handler directly, call `@ConsumeEvent` method,
verify observer captures. Uses a recording `@ApplicationScoped` inner class.

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...standard Apache 2.0 header...
 */
package io.casehub.engine;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.Binding;
import io.casehub.api.model.Capability;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.spi.StepOutcomeEvent;
import io.casehub.api.spi.StepOutcomeObserver;
import io.casehub.api.spi.routing.RoutingOutcome;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.event.WorkflowExecutionCompleted;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.internal.context.CaseContextImpl;
import io.casehub.engine.internal.engine.handler.WorkflowExecutionCompletedHandler;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerOutcome;
import io.casehub.worker.api.WorkerResult;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies that StepOutcomeObserver.onStepOutcome() is called after each worker execution.
 * Refs casehubio/engine#1050.
 */
@QuarkusTest
class StepOutcomeObserverTest {

  @Inject WorkflowExecutionCompletedHandler handler;
  @Inject StepCapturingObserver captureObserver;
  @Inject CaseDefinitionRegistry registry;

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @BeforeEach
  void reset() {
    StepCapturingObserver.capturedEvents.clear();
    StepCapturingObserver.shouldThrow = false;
  }

  @Test
  void stepOutcomeObserver_called_on_success() {
    CaseDefinition def = CaseDefinition.builder("step-test")
        .namespace("test").version("1.0.0")
        .binding(Binding.builder("analyse")
            .capability(new Capability("analysis", null, null))
            .build())
        .build();
    registry.register(def);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setName("step-test");
    metaModel.setNamespace("test");
    metaModel.setVersion("1.0.0");

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setCaseMetaModel(metaModel);
    instance.setCaseContext(new CaseContextImpl(Map.of("volatility", 0.85, "hour", 2)));
    instance.tenancyId = "test-tenant";

    Worker worker = Worker.builder()
        .name("momentum-agent")
        .capabilityName("analysis")
        .noFunction()
        .build();

    handler.onWorkflowExecutionCompletedHandler(
        new WorkflowExecutionCompleted(
            instance, worker, WorkerOutcome.success(Map.of("action", "reduce")),
            "analyse", null, null, null, null, null));

    assertThat(StepCapturingObserver.capturedEvents)
        .as("StepOutcomeObserver must fire on worker success — engine#1050")
        .hasSize(1);

    StepOutcomeEvent event = StepCapturingObserver.capturedEvents.get(0);
    assertThat(event.caseId()).isEqualTo(instance.getUuid());
    assertThat(event.tenancyId()).isEqualTo("test-tenant");
    assertThat(event.caseType()).isEqualTo("step-test");
    assertThat(event.bindingName()).isEqualTo("analyse");
    assertThat(event.capabilityName()).isEqualTo("analysis");
    assertThat(event.workerName()).isEqualTo("momentum-agent");
    assertThat(event.outcome()).isEqualTo(RoutingOutcome.SUCCESS);
    assertThat(event.contextSnapshot()).containsEntry("volatility", 0.85);
    assertThat(event.contextSnapshot()).containsEntry("hour", 2);
  }

  @Test
  void stepOutcomeObserver_called_on_failure() {
    CaseDefinition def = CaseDefinition.builder("step-fail-test")
        .namespace("test").version("1.0.0")
        .binding(Binding.builder("assess")
            .capability(new Capability("assessment", null, null))
            .build())
        .build();
    registry.register(def);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setName("step-fail-test");
    metaModel.setNamespace("test");
    metaModel.setVersion("1.0.0");

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setCaseMetaModel(metaModel);
    instance.setCaseContext(new CaseContextImpl(Map.of("error_state", true)));
    instance.tenancyId = "test-tenant";

    Worker worker = Worker.builder()
        .name("risk-agent")
        .capabilityName("assessment")
        .noFunction()
        .build();

    handler.onWorkflowExecutionCompletedHandler(
        new WorkflowExecutionCompleted(
            instance, worker, WorkerOutcome.declined("insufficient data"),
            "assess", null, null, null, null, null));

    assertThat(StepCapturingObserver.capturedEvents)
        .as("StepOutcomeObserver must fire on worker failure — engine#1050")
        .isNotEmpty();

    StepOutcomeEvent event = StepCapturingObserver.capturedEvents.get(0);
    assertThat(event.outcome()).isEqualTo(RoutingOutcome.FAILURE);
    assertThat(event.contextSnapshot()).containsEntry("error_state", true);
  }

  @Test
  void stepOutcomeObserver_exception_does_not_block_case_progression() {
    StepCapturingObserver.shouldThrow = true;

    CaseDefinition def = CaseDefinition.builder("step-err-test")
        .namespace("test").version("1.0.0")
        .binding(Binding.builder("check")
            .capability(new Capability("checking", null, null))
            .build())
        .build();
    registry.register(def);

    CaseMetaModel metaModel = new CaseMetaModel();
    metaModel.setName("step-err-test");
    metaModel.setNamespace("test");
    metaModel.setVersion("1.0.0");

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.setCaseMetaModel(metaModel);
    instance.setCaseContext(new CaseContextImpl(Map.of("val", 1)));
    instance.tenancyId = "test-tenant";

    Worker worker = Worker.builder()
        .name("checker")
        .capabilityName("checking")
        .noFunction()
        .build();

    // Should not throw — exception is caught and logged
    handler.onWorkflowExecutionCompletedHandler(
        new WorkflowExecutionCompleted(
            instance, worker, WorkerOutcome.success(Map.of("ok", true)),
            "check", null, null, null, null, null));
  }

  @ApplicationScoped
  static class StepCapturingObserver implements StepOutcomeObserver {
    static final CopyOnWriteArrayList<StepOutcomeEvent> capturedEvents =
        new CopyOnWriteArrayList<>();
    static volatile boolean shouldThrow = false;

    @Override
    public void onStepOutcome(StepOutcomeEvent event) {
      if (shouldThrow) {
        throw new RuntimeException("Deliberate test exception");
      }
      capturedEvents.add(event);
    }
  }
}
```

**Note:** The test constructs `WorkflowExecutionCompleted` directly and calls the handler method.
The exact constructor args may need adjustment based on the current record signature — check
`WorkflowExecutionCompleted.java` at implementation time. The key fields are `caseInstance`,
`worker`, `outcome`, and `bindingName`; remaining fields can be null.

- [ ] **Step 2: Run the tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=StepOutcomeObserverTest -q`
Expected: 3 tests PASS

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/StepOutcomeObserverTest.java
git commit -m "test(#1050): StepOutcomeObserver integration tests — success, failure, exception isolation

Refs #1050"
```

### Task 4: Update issue description and CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (engine project root) — add StepOutcomeObserver to the CaseOutcomeObserver SPI section

**Interfaces:**
- Consumes: all prior tasks
- Produces: documentation

- [ ] **Step 1: Update CLAUDE.md — add StepOutcomeObserver to the CaseOutcomeObserver SPI section**

After the existing `CaseOutcomeObserver` paragraph in CLAUDE.md, add:

```
`StepOutcomeObserver` — per-step lifecycle hook called by the engine after each worker execution
completes (both success and failure). `StepOutcomeEvent` carries `caseId`, `tenancyId`, `caseType`,
`bindingName`, `capabilityName` (nullable), `workerName`, `outcome` (RoutingOutcome), `contextSnapshot`
(working layer at step execution time — pre-output-application on success, current on failure),
`executionDuration` (nullable). Implementations record per-step CBR cases or step-level metrics.
`NoOpStepOutcomeObserver` (`@DefaultBean`) in `runtime/internal/worker/`. Fired from
`WorkflowExecutionCompletedHandler.fireStepOutcomeObserver()`. Refs engine#1050.
```

- [ ] **Step 2: Update GitHub issue #1050 description**

Run:
```bash
gh issue edit 1050 --repo casehubio/engine --body "## Context

fsitrading C6a (casehubio/fsitrading#36) designed per-step CBR recording — each routing step records its outcome as a CBR case for future retrieval.

**Correction:** \`RoutingOutcomeRecorder\` already fires per-step (both success and failure paths in \`WorkflowExecutionCompletedHandler\`). The blocker was that no SPI existed for step-level outcome observation with sufficient context for CBR recording — \`RoutingOutcomeRecorder\` carries only \`AgentRoutingContext\` (no \`caseType\`, no context snapshot as Map).

## What was needed

A new \`StepOutcomeObserver\` SPI (symmetric with \`CaseOutcomeObserver\`) that fires after each worker execution with the working layer context snapshot at step execution time — enabling consumers to build step-level CBR corpora.

## Consumer

casehubio/fsitrading — \`FsiStepOutcomeObserver\` will implement \`StepOutcomeObserver\`, extract features from \`contextSnapshot\`, and call \`CbrCaseMemoryStore.store()\` per step."
```

- [ ] **Step 3: Commit CLAUDE.md**

```bash
git add CLAUDE.md
git commit -m "docs(#1050): add StepOutcomeObserver to CLAUDE.md SPI documentation

Refs #1050"
```

## References

- [2026-09-07-step-outcome-observer-design.md](../specs/issue-1050-routing-outcome-per-step/2026-09-07-step-outcome-observer-design.md) — design spec this plan implements
- [CaseOutcomeObserver.java](../../api/src/main/java/io/casehub/api/spi/CaseOutcomeObserver.java) — symmetric SPI pattern
- [CaseOutcomeEvent.java](../../api/src/main/java/io/casehub/api/spi/CaseOutcomeEvent.java) — event record pattern
- [NoOpCaseOutcomeObserver.java](../../runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCaseOutcomeObserver.java) — @DefaultBean pattern
- [CaseOutcomeObserverTest.java](../../runtime/src/test/java/io/casehub/engine/CaseOutcomeObserverTest.java) — test pattern
- [WorkflowExecutionCompletedHandler.java](../../runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java) — integration point
- [CaseStatusChangedHandler.java:61](../../runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java) — MAP_TYPE constant pattern
- [PP-20260514-engine-spi-noops-defaultbean] — @DefaultBean convention
- [PP-20260723-c4c1cf] — virtual-thread handler convention
- [GE-20260706-56a75c] — WorkerOutcomeResolvedEvent fires only for non-success
- [GitHub #1050](https://github.com/casehubio/engine/issues/1050)
