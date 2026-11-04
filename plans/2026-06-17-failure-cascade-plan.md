# Failure Cascade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement engine-level semantic failure handling — WorkerOutcome, OutcomePolicy, structured failure state, agent exclusion, and failure goals producing COMPLETED not FAULTED.

**Architecture:** Workers declare outcomes (Success/Declined/Failed) on WorkerResult. The engine writes structured failure state to `_outcomes.<bindingName>` in the working panel, consults OutcomePolicy to decide REROUTE vs FAULT, and re-dispatches via existing CONTEXT_CHANGED + PlanItem lifecycle. Failure goals produce COMPLETED with goal metadata instead of FAULTED.

**Tech Stack:** Java 21, Quarkus 3.32, Vert.x event bus, Jackson, jackson-jq, Quartz RAM store

**Spec:** `docs/specs/2026-06-17-failure-cascade-design.md`

**Issues:** engine#502, #503, #504, #506

**Build before any module-specific tests:**
```bash
mvn install -DskipTests -q
```

**Run module tests with:**
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl <module>
```

---

### Task 1: Thread binding name through dispatch chain (prerequisite)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkerScheduleEventHandler.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java`
- Test: `blackboard/src/test/java/io/casehub/blackboard/handler/PlanItemCompletionHandlerTest.java`

- [ ] **Step 1: Add bindingName to WorkerScheduleEvent**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/WorkerScheduleEvent.java
public record WorkerScheduleEvent(
    CaseInstance caseInstance, Worker worker, Capability capability, String bindingName) {

  /** Backward-compat constructor for call sites that don't have binding name yet. */
  public WorkerScheduleEvent(CaseInstance caseInstance, Worker worker, Capability capability) {
    this(caseInstance, worker, capability, null);
  }
}
```

- [ ] **Step 2: Pass binding name from CaseContextChangedEventHandler.scheduleWorker()**

In `CaseContextChangedEventHandler.scheduleWorker()`, change the `WorkerScheduleEvent` construction to include the binding name:

```java
eventBus.publish(
    EventBusAddresses.WORKER_SCHEDULE,
    new WorkerScheduleEvent(caseInstance, selectedWorker, capability, binding.getName()));
```

This requires adding `Binding binding` as a parameter to `scheduleWorker()` and threading it from `publishWorkerSchedule()`. The binding is already available at the call site in `publishByTarget()`.

- [ ] **Step 3: Store bindingName in Quartz job data map**

In `WorkerScheduleEventHandler.onWorkerScheduleEventHandler()`, add to the job data map building:

The `eventLog.setMetadata()` call already builds a metadata map. Add `bindingName` alongside the existing entries. Also, the Quartz `JobDataMap` in `submitIfNeeded()` → `workflowExecutionManager.submit()` needs the binding name. Check `QuartzWorkerSchedulerService` to find where the JobDataMap is built and add `"bindingName"` to it.

The event log metadata in `buildEventLog()` already has `workerName` and `capabilityName` — add `"bindingName"`:
```java
Map<String, String> metadata = Map.of(
    "workerName", worker.getName(),
    "capabilityName", capability.getName(),
    "inputDataHash", inputDataHash,
    "bindingName", event.bindingName() != null ? event.bindingName() : "");
```

- [ ] **Step 4: Add bindingName to WorkflowExecutionCompleted**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java
public record WorkflowExecutionCompleted(
    CaseInstance caseInstance,
    Worker worker,
    String idempotency,
    Map<String, Object> output,
    PlannedAction plannedAction,
    String bindingName) {

  public static WorkflowExecutionCompleted approved(
      final CaseInstance caseInstance,
      final Worker worker,
      final String idempotency,
      final Map<String, Object> output) {
    return new WorkflowExecutionCompleted(caseInstance, worker, idempotency, output, null, null);
  }
}
```

- [ ] **Step 5: Extract bindingName in QuartzWorkerExecutionJob and include in event**

In `QuartzWorkerExecutionJob.execute()`, extract `bindingName` from the event log metadata (it was stored in step 3):

```java
String bindingName = eventLog.getMetadata().has("bindingName")
    ? eventLog.getMetadata().get("bindingName").asText()
    : null;
```

Pass it to `onSuccess()` and include in the `WorkflowExecutionCompleted`:

```java
private void onSuccess(CaseInstance instance, Worker worker, String inputDataHash,
    WorkerResult workerResult, String bindingName) {
  PlannedAction enrichedAction = workerResult.plannedAction() != null
      ? workerResult.plannedAction().withIdentity(worker.getName(), instance.getUuid())
      : null;
  eventBus.publish(WORKER_EXECUTION_FINISHED,
      new WorkflowExecutionCompleted(
          instance, worker, inputDataHash, workerResult.output(), enrichedAction, bindingName));
}
```

- [ ] **Step 6: Replace findMatchingCapabilityBinding() with direct lookup in WorkflowExecutionCompletedHandler**

Add a private helper that looks up by binding name:

```java
private Binding findBindingByName(final CaseInstance caseInstance, final String bindingName) {
  if (bindingName == null) return null;
  final CaseDefinition definition =
      caseDefinitionRegistry.getCaseDefinition(caseInstance.getCaseMetaModel());
  if (definition == null || definition.getBindings() == null) return null;
  return definition.getBindings().stream()
      .filter(b -> b.getName().equals(bindingName))
      .findFirst().orElse(null);
}
```

Replace the calls to `findMatchingCapabilityBinding()` with `findBindingByName(caseInstance, event.bindingName())` in `extractCapabilityTag()` and `resolveConflictStrategy()`. Keep `findMatchingCapabilityBinding()` as a fallback for events where `bindingName` is null (backward compat during rollout).

- [ ] **Step 7: Update PlanItemCompletionHandler to use bindingName for lookup**

In `PlanItemCompletionHandler.onWorkerFinished()`, use binding name when available:

```java
@ConsumeEvent(value = EventBusAddresses.WORKER_EXECUTION_FINISHED, blocking = true)
public void onWorkerFinished(WorkflowExecutionCompleted event) {
  if (event.bindingName() != null) {
    completePlanItemByBindingName(
        event.caseInstance().getUuid(), event.bindingName(), event.caseInstance().tenancyId);
  } else {
    completePlanItemByKey(
        event.caseInstance().getUuid(), event.worker().getName(), event.caseInstance().tenancyId);
  }
}

private void completePlanItemByBindingName(UUID caseId, String bindingName, String tenancyId) {
  CasePlanModel plan = registry.get(caseId).orElse(null);
  if (plan == null) return;
  plan.getPlanItemByBindingName(bindingName).ifPresent(item -> {
    if (!COMPLETABLE.contains(item.getStatus())) return;
    item.markCompleted();
    stageAutocompleteEvaluator.evaluate(caseId, plan, item.getPlanItemId());
    planItemCompletedEvents.fireAsync(
        new PlanItemCompletedEvent(caseId, item.getPlanItemId(), bindingName, tenancyId));
  });
}
```

- [ ] **Step 8: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl scheduler-quartz
```

All existing tests must pass — the changes are backward compatible (null bindingName falls back to existing behavior).

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "refactor: thread binding name through dispatch chain

WorkerScheduleEvent, WorkerScheduleEventHandler, QuartzWorkerExecutionJob,
WorkflowExecutionCompleted, and PlanItemCompletionHandler now carry
bindingName for precise PlanItem lookup.

Replaces findMatchingCapabilityBinding() fuzzy lookup with direct
binding name match. Fixes multi-worker completion tracking bug
for success path and new semantic failure paths.

Prerequisite for failure cascade (engine#502, #503, #504, #506).

Refs #502"
```

---

### Task 2: WorkerOutcome sealed type + WorkerResult changes (#502 foundation)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/WorkerOutcome.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkerResult.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkStatus.java`
- Modify: `api/src/main/java/io/casehub/api/model/WorkResult.java`
- Test: `api/src/test/java/io/casehub/api/model/WorkerResultTest.java`

- [ ] **Step 1: Write WorkerOutcome tests**

Create `api/src/test/java/io/casehub/api/model/WorkerOutcomeTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.Test;

class WorkerOutcomeTest {

  @Test
  void success_is_sealed_variant() {
    WorkerOutcome outcome = new WorkerOutcome.Success();
    assertThat(outcome).isInstanceOf(WorkerOutcome.class);
  }

  @Test
  void declined_carries_reason() {
    var declined = new WorkerOutcome.Declined("unsupported language");
    assertThat(declined.reason()).isEqualTo("unsupported language");
  }

  @Test
  void failed_carries_reason() {
    var failed = new WorkerOutcome.Failed("compilation error");
    assertThat(failed.reason()).isEqualTo("compilation error");
  }

  @Test
  void exhaustive_switch() {
    WorkerOutcome outcome = new WorkerOutcome.Declined("test");
    String result = switch (outcome) {
      case WorkerOutcome.Success s -> "success";
      case WorkerOutcome.Declined d -> "declined: " + d.reason();
      case WorkerOutcome.Failed f -> "failed: " + f.reason();
    };
    assertThat(result).isEqualTo("declined: test");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=WorkerOutcomeTest
```
Expected: FAIL — `WorkerOutcome` class not found.

- [ ] **Step 3: Create WorkerOutcome sealed interface**

```java
// api/src/main/java/io/casehub/api/model/WorkerOutcome.java
package io.casehub.api.model;

public sealed interface WorkerOutcome
    permits WorkerOutcome.Success, WorkerOutcome.Declined, WorkerOutcome.Failed {

  record Success() implements WorkerOutcome {}

  record Declined(String reason) implements WorkerOutcome {}

  record Failed(String reason) implements WorkerOutcome {}

  static WorkerOutcome success() {
    return new Success();
  }
}
```

- [ ] **Step 4: Run WorkerOutcome test to verify it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=WorkerOutcomeTest
```
Expected: PASS

- [ ] **Step 5: Write WorkerResult tests for new factory methods**

Add to existing `api/src/test/java/io/casehub/api/model/WorkerResultTest.java` (or create if absent):

```java
@Test
void declined_factory_sets_outcome_and_empty_output() {
  WorkerResult result = WorkerResult.declined("unsupported");
  assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Declined.class);
  assertThat(((WorkerOutcome.Declined) result.outcome()).reason()).isEqualTo("unsupported");
  assertThat(result.output()).isEmpty();
  assertThat(result.plannedAction()).isNull();
}

@Test
void failed_factory_with_partial_output() {
  Map<String, Object> partial = Map.of("findings", List.of("a", "b"));
  WorkerResult result = WorkerResult.failed("error", partial);
  assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Failed.class);
  assertThat(result.output()).containsKey("findings");
}

@Test
void of_factory_defaults_to_success() {
  WorkerResult result = WorkerResult.of(Map.of("key", "val"));
  assertThat(result.outcome()).isInstanceOf(WorkerOutcome.Success.class);
}

@Test
void declined_with_planned_action_throws() {
  assertThatThrownBy(() ->
      new WorkerResult(Map.of(), PlannedAction.of("desc", "type", Map.of()),
          new WorkerOutcome.Declined("reason")))
    .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 6: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=WorkerResultTest
```
Expected: FAIL — no `outcome()` method, no `declined()`/`failed()` factories.

- [ ] **Step 7: Update WorkerResult with outcome field and factories**

```java
// api/src/main/java/io/casehub/api/model/WorkerResult.java
public record WorkerResult(
    Map<String, Object> output, PlannedAction plannedAction, WorkerOutcome outcome) {

  public WorkerResult {
    if (!(outcome instanceof WorkerOutcome.Success) && plannedAction != null) {
      throw new IllegalArgumentException(
          "PlannedAction is only valid with Success outcome, got: " + outcome.getClass().getSimpleName());
    }
  }

  public static WorkerResult of(final Map<String, Object> output) {
    return new WorkerResult(output, null, WorkerOutcome.success());
  }

  public static WorkerResult of(final Map<String, Object> output, final PlannedAction action) {
    return new WorkerResult(output, action, WorkerOutcome.success());
  }

  public static WorkerResult declined(final String reason) {
    return new WorkerResult(Map.of(), null, new WorkerOutcome.Declined(reason));
  }

  public static WorkerResult declined(final String reason, final Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, null, new WorkerOutcome.Declined(reason));
  }

  public static WorkerResult failed(final String reason) {
    return new WorkerResult(Map.of(), null, new WorkerOutcome.Failed(reason));
  }

  public static WorkerResult failed(final String reason, final Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, null, new WorkerOutcome.Failed(reason));
  }
}
```

- [ ] **Step 8: Run test to verify it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=WorkerResultTest,WorkerOutcomeTest
```
Expected: PASS

- [ ] **Step 9: Add DECLINED and FAILED to WorkStatus, add WorkResult factories**

```java
// api/src/main/java/io/casehub/api/model/WorkStatus.java — add two values:
DECLINED,
FAILED,
```

```java
// api/src/main/java/io/casehub/api/model/WorkResult.java — add factory methods:
public static WorkResult declined(String correlationKey, String workerId, UUID caseId) {
  return new WorkResult(correlationKey, WorkStatus.DECLINED, Map.of(), workerId, caseId);
}

public static WorkResult failed(String correlationKey, String workerId, UUID caseId) {
  return new WorkResult(correlationKey, WorkStatus.FAILED, Map.of(), workerId, caseId);
}
```

- [ ] **Step 10: Fix compilation — all call sites of WorkerResult constructors**

The canonical constructor now takes 3 args `(output, plannedAction, outcome)`. Search for all direct `new WorkerResult(...)` calls and update them. The factory methods `WorkerResult.of(...)` handle the common case. Agent.execute() returns WorkerResult — verify it uses the factory.

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api
```

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat(api): WorkerOutcome sealed type on WorkerResult

Three outcomes: Success (default), Declined(reason), Failed(reason).
Constructor validates plannedAction is null for non-Success.
WorkStatus gains DECLINED and FAILED. WorkResult gains factories.

Refs #502"
```

---

### Task 3: OutcomePolicy + OutcomeAction model + YAML/DSL support (#504 model)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/OutcomePolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/OutcomeAction.java`
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java`
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Test: `api/src/test/java/io/casehub/api/model/OutcomePolicyTest.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`

- [ ] **Step 1: Write OutcomePolicy tests**

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.Test;

class OutcomePolicyTest {

  @Test
  void default_constructor_all_reroute_max3() {
    OutcomePolicy policy = new OutcomePolicy();
    assertThat(policy.onDecline()).isEqualTo(OutcomeAction.REROUTE);
    assertThat(policy.onFailure()).isEqualTo(OutcomeAction.REROUTE);
    assertThat(policy.onExpired()).isEqualTo(OutcomeAction.REROUTE);
    assertThat(policy.maxRerouteAttempts()).isEqualTo(3);
  }

  @Test
  void custom_policy() {
    OutcomePolicy policy = new OutcomePolicy(OutcomeAction.FAULT, OutcomeAction.REROUTE,
        OutcomeAction.REROUTE, 5);
    assertThat(policy.onDecline()).isEqualTo(OutcomeAction.FAULT);
    assertThat(policy.maxRerouteAttempts()).isEqualTo(5);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=OutcomePolicyTest
```

- [ ] **Step 3: Create OutcomeAction enum and OutcomePolicy record**

```java
// api/src/main/java/io/casehub/api/model/OutcomeAction.java
package io.casehub.api.model;

public enum OutcomeAction { REROUTE, FAULT }
```

```java
// api/src/main/java/io/casehub/api/model/OutcomePolicy.java
package io.casehub.api.model;

public record OutcomePolicy(
    OutcomeAction onDecline,
    OutcomeAction onFailure,
    OutcomeAction onExpired,
    int maxRerouteAttempts) {

  public OutcomePolicy() {
    this(OutcomeAction.REROUTE, OutcomeAction.REROUTE, OutcomeAction.REROUTE, 3);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=OutcomePolicyTest
```

- [ ] **Step 5: Add outcomePolicy to Binding**

Add field and builder method to `api/src/main/java/io/casehub/api/model/Binding.java`:

```java
// Field (alongside existing fields)
private OutcomePolicy outcomePolicy;

// Getter
public OutcomePolicy getOutcomePolicy() { return outcomePolicy; }

// Setter
public void setOutcomePolicy(OutcomePolicy outcomePolicy) { this.outcomePolicy = outcomePolicy; }
```

In `Binding.Builder`:
```java
private OutcomePolicy outcomePolicy;

public Builder outcomePolicy(OutcomePolicy outcomePolicy) {
  this.outcomePolicy = outcomePolicy;
  return this;
}
```

In `Builder.build()`, after constructing the Binding:
```java
b.setOutcomePolicy(outcomePolicy);
```

- [ ] **Step 6: Add outcomePolicy to JSON Schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, under `$defs/Binding/properties`, add:

```yaml
      outcomePolicy:
        type: object
        description: "Policy for handling semantic worker outcomes (DECLINED, FAILED, EXPIRED)"
        unevaluatedProperties: false
        properties:
          onDecline:
            type: string
            enum: [REROUTE, FAULT]
            default: REROUTE
          onFailure:
            type: string
            enum: [REROUTE, FAULT]
            default: REROUTE
          onExpired:
            type: string
            enum: [REROUTE, FAULT]
            default: REROUTE
          maxRerouteAttempts:
            type: integer
            minimum: 1
            default: 3
```

Regenerate schema classes:
```bash
mvn generate-sources -pl schema -q
```

- [ ] **Step 7: Add outcomePolicy mapping to CaseDefinitionYamlMapper.convertBinding()**

In `convertBinding()`, after the `conflictResolverStrategy` handling, add:

```java
if (schemaBinding.getOutcomePolicy() != null) {
  io.casehub.model.OutcomePolicy sp = schemaBinding.getOutcomePolicy();
  OutcomeAction onDecline = sp.getOnDecline() != null
      ? OutcomeAction.valueOf(sp.getOnDecline().value()) : OutcomeAction.REROUTE;
  OutcomeAction onFailure = sp.getOnFailure() != null
      ? OutcomeAction.valueOf(sp.getOnFailure().value()) : OutcomeAction.REROUTE;
  OutcomeAction onExpired = sp.getOnExpired() != null
      ? OutcomeAction.valueOf(sp.getOnExpired().value()) : OutcomeAction.REROUTE;
  int maxAttempts = sp.getMaxRerouteAttempts() != null ? sp.getMaxRerouteAttempts() : 3;
  builder.outcomePolicy(new OutcomePolicy(onDecline, onFailure, onExpired, maxAttempts));
}
```

Note: check the exact getter names after schema regeneration. The generated `OutcomePolicy` class may use enum inner classes with `value()` methods (like `ConflictResolverStrategy`).

- [ ] **Step 8: Write YAML mapper test for outcomePolicy**

Add to `CaseDefinitionYamlMapperTest`:

```java
@Test
void binding_with_outcomePolicy_maps_correctly() throws IOException {
  String yaml = """
      namespace: test
      name: test
      version: "1.0.0"
      dsl: "1.0.0"
      spec:
        capabilities:
          - name: security-review
        bindings:
          - name: review
            capability: security-review
            outcomePolicy:
              onDecline: REROUTE
              onFailure: FAULT
              maxRerouteAttempts: 2
            on:
              contextChange:
                filter: ".review == null"
        workers:
          - name: reviewer
            capabilities: [security-review]
            do: []
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
  Binding binding = def.getBindings().get(0);
  assertThat(binding.getOutcomePolicy()).isNotNull();
  assertThat(binding.getOutcomePolicy().onDecline()).isEqualTo(OutcomeAction.REROUTE);
  assertThat(binding.getOutcomePolicy().onFailure()).isEqualTo(OutcomeAction.FAULT);
  assertThat(binding.getOutcomePolicy().maxRerouteAttempts()).isEqualTo(2);
}
```

- [ ] **Step 9: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=OutcomePolicyTest,CaseDefinitionYamlMapperTest
```

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat(api): OutcomePolicy record on Binding + YAML/DSL support

OutcomeAction enum (REROUTE, FAULT). OutcomePolicy record with defaults
(all REROUTE, max 3). Schema, YAML mapper, and fluent builder support.

Refs #504"
```

---

### Task 4: New event types + WorkflowExecutionCompletedHandler semantic failure path (#503)

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/event/OutcomeDisposition.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/event/WorkerOutcomeResolvedEvent.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/EventBusAddresses.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandlerTest.java`

- [ ] **Step 1: Create OutcomeDisposition enum**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/OutcomeDisposition.java
package io.casehub.engine.common.internal.event;

public enum OutcomeDisposition { REROUTE, EXHAUSTED, FAULT }
```

- [ ] **Step 2: Create WorkerOutcomeResolvedEvent**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/WorkerOutcomeResolvedEvent.java
package io.casehub.engine.common.internal.event;

import io.casehub.engine.common.internal.model.CaseInstance;

public record WorkerOutcomeResolvedEvent(
    CaseInstance caseInstance,
    String workerId,
    String bindingName,
    String capabilityName,
    OutcomeDisposition disposition) {}
```

- [ ] **Step 3: Add event bus address and event types**

In `EventBusAddresses.java`:
```java
public static final String WORKER_OUTCOME_RESOLVED = "casehub.worker.outcome.resolved";
```

In `CaseHubEventType.java`, add:
```java
WORKER_OUTCOME_DECLINED,
WORKER_OUTCOME_FAILED,
```

- [ ] **Step 4: Add WorkerOutcome to WorkflowExecutionCompleted**

The record already has `bindingName` from Task 1. Add `outcome`:

```java
public record WorkflowExecutionCompleted(
    CaseInstance caseInstance,
    Worker worker,
    String idempotency,
    Map<String, Object> output,
    PlannedAction plannedAction,
    String bindingName,
    WorkerOutcome outcome) {

  public static WorkflowExecutionCompleted approved(
      final CaseInstance caseInstance, final Worker worker,
      final String idempotency, final Map<String, Object> output) {
    return new WorkflowExecutionCompleted(
        caseInstance, worker, idempotency, output, null, null, WorkerOutcome.success());
  }
}
```

Update `QuartzWorkerExecutionJob.onSuccess()` to extract the outcome from `WorkerResult` and pass it through:

```java
eventBus.publish(WORKER_EXECUTION_FINISHED,
    new WorkflowExecutionCompleted(
        instance, worker, inputDataHash, workerResult.output(),
        enrichedAction, bindingName, workerResult.outcome()));
```

- [ ] **Step 5: Write test for handleSemanticFailure**

Create a test in `runtime/src/test/java/io/casehub/engine/internal/engine/handler/` that verifies:
1. A DECLINED outcome with REROUTE policy writes `_outcomes.<bindingName>` to the context
2. The event log has type `WORKER_OUTCOME_DECLINED`
3. `WORKER_OUTCOME_RESOLVED` is published with disposition REROUTE
4. Worker status listener receives `WorkResult.declined()`

The test should use `@QuarkusTest` with the in-memory persistence and a mock event bus (or spy), following existing test patterns in `CaseContextChangedEventHandlerRoutingTest`.

- [ ] **Step 6: Implement handleSemanticFailure in WorkflowExecutionCompletedHandler**

Add the outcome check at the TOP of `onWorkflowExecutionCompletedHandler()`, before the PlannedAction check:

```java
if (event.outcome() instanceof WorkerOutcome.Declined
    || event.outcome() instanceof WorkerOutcome.Failed) {
  return handleSemanticFailure(event, traceId);
}
```

Implement `handleSemanticFailure()`:

```java
private Uni<Void> handleSemanticFailure(
    final WorkflowExecutionCompleted event, final String traceId) {
  final CaseInstance caseInstance = event.caseInstance();
  final Worker worker = event.worker();
  final String bindingName = event.bindingName();
  final Instant now = Instant.now();

  // 1. Resolve binding and OutcomePolicy
  final Binding binding = findBindingByName(caseInstance, bindingName);
  final OutcomePolicy policy = binding != null && binding.getOutcomePolicy() != null
      ? binding.getOutcomePolicy() : new OutcomePolicy();

  final OutcomeAction action = event.outcome() instanceof WorkerOutcome.Declined
      ? policy.onDecline() : policy.onFailure();

  final String outcomeStatus = event.outcome() instanceof WorkerOutcome.Declined
      ? "DECLINED" : "FAILED";
  final String reason = event.outcome() instanceof WorkerOutcome.Declined d
      ? d.reason() : ((WorkerOutcome.Failed) event.outcome()).reason();

  // 2. Read/create _outcomes state
  final String capabilityName = extractCapabilityName(binding);
  final ObjectNode outcomesRoot = getOrCreateOutcomesNode(caseInstance);
  final ObjectNode bindingOutcome = getOrCreateBindingOutcome(outcomesRoot, bindingName);

  // 3. Update failure state
  int attempts = bindingOutcome.has("attempts") ? bindingOutcome.get("attempts").asInt() + 1 : 1;
  bindingOutcome.put("attempts", attempts);

  // Check exhaustion before writing status
  final boolean exhausted = action == OutcomeAction.REROUTE
      && attempts >= policy.maxRerouteAttempts();

  bindingOutcome.put("status", exhausted ? "REROUTES_EXHAUSTED" : outcomeStatus);

  // Append history
  ArrayNode history = bindingOutcome.has("history")
      ? (ArrayNode) bindingOutcome.get("history")
      : OBJECT_MAPPER.createArrayNode();
  ObjectNode entry = OBJECT_MAPPER.createObjectNode()
      .put("agent", worker.getName())
      .put("status", outcomeStatus)
      .put("reason", reason)
      .put("timestamp", now.toString());
  if (!event.output().isEmpty()) {
    entry.set("partialOutput", OBJECT_MAPPER.valueToTree(event.output()));
  }
  history.add(entry);
  bindingOutcome.set("history", history);

  // Update excludedAgents
  ArrayNode excluded = bindingOutcome.has("excludedAgents")
      ? (ArrayNode) bindingOutcome.get("excludedAgents")
      : OBJECT_MAPPER.createArrayNode();
  excluded.add(worker.getName());
  bindingOutcome.set("excludedAgents", excluded);

  // 4. Write to context
  caseInstance.getCaseContext().set("_outcomes", OBJECT_MAPPER.convertValue(outcomesRoot, Map.class));

  // 5. Episodic
  if (caseInstance.getCaseContext() instanceof CaseContextImpl ctx) {
    EpisodicPanelUpdater.recordWorkerCompletion(ctx, worker.getName(), outcomeStatus);
  }

  // 6. Persist event log
  final CaseHubEventType eventType = event.outcome() instanceof WorkerOutcome.Declined
      ? CaseHubEventType.WORKER_OUTCOME_DECLINED : CaseHubEventType.WORKER_OUTCOME_FAILED;
  EventLog eventLog = new EventLog();
  eventLog.setCaseId(caseInstance.getUuid());
  eventLog.setWorkerId(worker.getName());
  eventLog.setStreamType(EventStreamType.CASE);
  eventLog.setTimestamp(now);
  eventLog.setEventType(eventType);
  eventLog.setMetadata(OBJECT_MAPPER.createObjectNode()
      .put("bindingName", bindingName)
      .put("reason", reason)
      .put("attempts", attempts)
      .put("disposition", action == OutcomeAction.FAULT ? "FAULT"
          : exhausted ? "EXHAUSTED" : "REROUTE"));

  // 7. Determine disposition
  final OutcomeDisposition disposition = action == OutcomeAction.FAULT
      ? OutcomeDisposition.FAULT
      : exhausted ? OutcomeDisposition.EXHAUSTED : OutcomeDisposition.REROUTE;

  return eventLogRepository.append(eventLog, caseInstance.tenancyId)
      .invoke(() -> {
        // Report to listener
        workerStatusListener.onWorkerCompleted(worker.getName(),
            WorkResult.valueOf(outcomeStatus.equals("DECLINED") ? "DECLINED" : "FAILED",
                event.idempotency(), event.output(), worker.getName(), caseInstance.getUuid()));

        // CDI events
        lifecycleEvents.fireAsync(new CaseLifecycleEvent(
            caseInstance.getUuid(), caseInstance.tenancyId,
            "WorkerOutcome", outcomeStatus + "Outcome",
            caseInstance.getState().name(), worker.getName(), "WORKER", traceId))
            .whenComplete((v, t) -> {
              if (t != null) LOG.warnf(t, "CaseLifecycleEvent observer failed");
            });
      })
      .invoke(() -> {
        // For FAULT: also publish CASE_STATUS_CHANGED
        if (disposition == OutcomeDisposition.FAULT) {
          eventBus.publish(EventBusAddresses.CASE_STATUS_CHANGED,
              new CaseStatusChanged(caseInstance, caseInstance.getState().name(),
                  CaseStatus.FAULTED.name()));
        }
        // Publish for blackboard PlanItem lifecycle
        eventBus.publish(EventBusAddresses.WORKER_OUTCOME_RESOLVED,
            new WorkerOutcomeResolvedEvent(caseInstance, worker.getName(),
                bindingName, capabilityName, disposition));
      })
      .replaceWithVoid();
}
```

Note: the `WorkResult` factory for DECLINED/FAILED needs to be called correctly — use `WorkResult.declined()` or `WorkResult.failed()` from Task 2.

Helper methods:
```java
private ObjectNode getOrCreateOutcomesNode(CaseInstance caseInstance) {
  Object existing = caseInstance.getCaseContext().get("_outcomes");
  if (existing != null) {
    return OBJECT_MAPPER.valueToTree(existing).deepCopy();
  }
  return OBJECT_MAPPER.createObjectNode();
}

private ObjectNode getOrCreateBindingOutcome(ObjectNode outcomesRoot, String bindingName) {
  if (outcomesRoot.has(bindingName)) {
    return (ObjectNode) outcomesRoot.get(bindingName);
  }
  ObjectNode node = OBJECT_MAPPER.createObjectNode();
  outcomesRoot.set(bindingName, node);
  return node;
}

private String extractCapabilityName(Binding binding) {
  if (binding != null && binding.target() instanceof CapabilityTarget ct) {
    return ct.capability().getName();
  }
  return null;
}
```

- [ ] **Step 7: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: structured failure state + handleSemanticFailure path

WorkflowExecutionCompletedHandler branches on WorkerOutcome: DECLINED/FAILED
writes _outcomes.<bindingName> to working panel with status, attempts,
history (including partialOutput), and excludedAgents.

OutcomePolicy consulted for REROUTE vs FAULT. FAULT publishes
CASE_STATUS_CHANGED(FAULTED). All paths publish WORKER_OUTCOME_RESOLVED
for blackboard PlanItem lifecycle.

New CaseHubEventTypes: WORKER_OUTCOME_DECLINED, WORKER_OUTCOME_FAILED.
New EventBusAddress: WORKER_OUTCOME_RESOLVED.

Refs #503, #504"
```

---

### Task 5: PlanItemCompletionHandler outcome gating + WorkerOutcomeResolvedHandler (#504 reroute)

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java`
- Create: `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerOutcomeResolvedHandler.java`
- Test: `blackboard/src/test/java/io/casehub/blackboard/handler/WorkerOutcomeResolvedHandlerTest.java`

- [ ] **Step 1: Add outcome gating to PlanItemCompletionHandler**

In `PlanItemCompletionHandler.onWorkerFinished()`, add at the top:

```java
@ConsumeEvent(value = EventBusAddresses.WORKER_EXECUTION_FINISHED, blocking = true)
public void onWorkerFinished(WorkflowExecutionCompleted event) {
  if (!(event.outcome() instanceof io.casehub.api.model.WorkerOutcome.Success)) {
    return; // WorkerOutcomeResolvedHandler owns PlanItem lifecycle for non-success
  }
  if (event.bindingName() != null) {
    completePlanItemByBindingName(/* ... */);
  } else {
    completePlanItemByKey(/* ... */);
  }
}
```

- [ ] **Step 2: Write WorkerOutcomeResolvedHandler test**

```java
package io.casehub.blackboard.handler;

import static org.assertj.core.api.Assertions.*;
import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.plan.DefaultCasePlanModel;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.common.internal.event.OutcomeDisposition;
import io.casehub.engine.common.internal.event.WorkerOutcomeResolvedEvent;
import io.casehub.engine.common.internal.model.PlanItemStatus;
// ... test infrastructure imports

class WorkerOutcomeResolvedHandlerTest {

  @Test
  void reroute_marks_planItem_faulted_and_publishes_contextChanged() {
    // Setup: CasePlanModel with a RUNNING PlanItem for binding "review"
    UUID caseId = UUID.randomUUID();
    CasePlanModel plan = new DefaultCasePlanModel(caseId);
    PlanItem item = PlanItem.create("review", "worker-a", 0);
    plan.addPlanItem(item);
    item.markRunning();

    // Create handler with mock/spy event bus
    // ...

    WorkerOutcomeResolvedEvent event = new WorkerOutcomeResolvedEvent(
        caseInstance, "worker-a", "review", "security-review", OutcomeDisposition.REROUTE);

    handler.onWorkerOutcomeResolved(event);

    assertThat(item.getStatus()).isEqualTo(PlanItemStatus.FAULTED);
    // Verify CONTEXT_CHANGED was published
    // Verify stageAutocompleteEvaluator was NOT called
  }

  @Test
  void exhausted_marks_faulted_and_calls_stageAutocomplete() {
    // Same setup but with EXHAUSTED disposition
    // Verify stageAutocompleteEvaluator WAS called
    // Verify CONTEXT_CHANGED was published
  }

  @Test
  void fault_marks_faulted_and_calls_stageAutocomplete_no_contextChanged() {
    // FAULT disposition
    // Verify stageAutocompleteEvaluator WAS called
    // Verify CONTEXT_CHANGED was NOT published (case is terminal)
  }
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard -Dtest=WorkerOutcomeResolvedHandlerTest
```

- [ ] **Step 4: Implement WorkerOutcomeResolvedHandler**

```java
package io.casehub.blackboard.handler;

import io.casehub.api.context.ContextPanel;
import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.common.internal.event.CaseContextChangedEvent;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.event.OutcomeDisposition;
import io.casehub.engine.common.internal.event.WorkerOutcomeResolvedEvent;
import io.casehub.engine.common.internal.model.PlanItemStatus;
import io.quarkus.vertx.ConsumeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class WorkerOutcomeResolvedHandler {

  private static final Logger LOG = Logger.getLogger(WorkerOutcomeResolvedHandler.class);

  private final BlackboardRegistry registry;
  private final StageAutocompleteEvaluator stageAutocompleteEvaluator;
  private final io.vertx.mutiny.core.eventbus.EventBus eventBus;

  @Inject
  public WorkerOutcomeResolvedHandler(
      BlackboardRegistry registry,
      StageAutocompleteEvaluator stageAutocompleteEvaluator,
      io.vertx.mutiny.core.eventbus.EventBus eventBus) {
    this.registry = registry;
    this.stageAutocompleteEvaluator = stageAutocompleteEvaluator;
    this.eventBus = eventBus;
  }

  @ConsumeEvent(value = EventBusAddresses.WORKER_OUTCOME_RESOLVED, blocking = true)
  public void onWorkerOutcomeResolved(WorkerOutcomeResolvedEvent event) {
    CasePlanModel plan = registry.get(event.caseInstance().getUuid()).orElse(null);
    if (plan == null) return;

    plan.getPlanItemByBindingName(event.bindingName()).ifPresent(item -> {
      if (item.getStatus() != PlanItemStatus.RUNNING) {
        LOG.debugf("PlanItem for binding '%s' has status %s — not RUNNING, skipping",
            event.bindingName(), item.getStatus());
        return;
      }

      item.markFaulted();

      if (event.disposition() == OutcomeDisposition.EXHAUSTED
          || event.disposition() == OutcomeDisposition.FAULT) {
        stageAutocompleteEvaluator.evaluate(
            event.caseInstance().getUuid(), plan, item.getPlanItemId());
      }

      if (event.disposition() != OutcomeDisposition.FAULT) {
        eventBus.publish(EventBusAddresses.CONTEXT_CHANGED,
            new CaseContextChangedEvent(
                event.caseInstance(),
                event.caseInstance().getCaseContext().snapshot(),
                ContextPanel.WORKING));
      }

      LOG.infof("PlanItem '%s' marked FAULTED for binding '%s' — disposition=%s",
          item.getPlanItemId(), event.bindingName(), event.disposition());
    });
  }
}
```

- [ ] **Step 5: Register codec for WorkerOutcomeResolvedEvent**

Check how other event codecs are registered in the blackboard module (look at `BlackboardEventCodecRegistrar` or similar). Register a local-only codec for `WorkerOutcomeResolvedEvent`. If the engine uses a generic codec registrar for `engine-common` events, add it there instead.

- [ ] **Step 6: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl blackboard
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: WorkerOutcomeResolvedHandler + PlanItem outcome gating

PlanItemCompletionHandler gates on WorkerOutcome.Success — returns
early for DECLINED/FAILED, eliminating the fan-out race.

New WorkerOutcomeResolvedHandler (blackboard, blocking=true) consumes
WORKER_OUTCOME_RESOLVED:
- REROUTE: PlanItem FAULTED, CONTEXT_CHANGED (triggers re-dispatch)
- EXHAUSTED: PlanItem FAULTED, stage autocomplete, CONTEXT_CHANGED
- FAULT: PlanItem FAULTED, stage autocomplete (no CONTEXT_CHANGED)

Refs #504"
```

---

### Task 6: Agent exclusion filter in CaseContextChangedEventHandler (#502)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerRoutingTest.java`

- [ ] **Step 1: Write test for agent exclusion**

Add to `CaseContextChangedEventHandlerRoutingTest`:

```java
@Test
void excludedAgents_filtered_before_routing() {
  // Setup: case with _outcomes containing excludedAgents for the binding
  // Two workers: worker-a (excluded), worker-b (eligible)
  // Verify: routing strategy only receives worker-b as candidate
  // Verify: worker-a is filtered out before select() is called
}

@Test
void all_candidates_excluded_falls_through_to_provision() {
  // Setup: all workers excluded
  // Verify: tryProvision is called
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=CaseContextChangedEventHandlerRoutingTest
```

- [ ] **Step 3: Add exclusion filter to publishWorkerSchedule()**

In `CaseContextChangedEventHandler.publishWorkerSchedule()`, after `AgentCandidateFactory.buildCandidates()` and before `agentRoutingStrategy.select()`:

```java
// Filter excluded agents from _outcomes
final JsonNode workingPanel = caseInstance.getCaseContext()
    .panel(ContextPanel.WORKING).asJsonNode();
final JsonNode outcomeNode = workingPanel.path("_outcomes").path(binding.getName());
if (outcomeNode.has("excludedAgents")) {
  final Set<String> excluded = java.util.stream.StreamSupport.stream(
          outcomeNode.get("excludedAgents").spliterator(), false)
      .map(com.fasterxml.jackson.databind.JsonNode::asText)
      .collect(java.util.stream.Collectors.toSet());
  candidates = candidates.stream()
      .filter(c -> !excluded.contains(c.workerId()))
      .toList();
  if (!excluded.isEmpty()) {
    LOG.debugf("Filtered %d excluded agents for binding '%s': %s",
        excluded.size(), binding.getName(), excluded);
  }
}

if (candidates.isEmpty()) {
  LOG.warnf("All candidates excluded for capability '%s' binding '%s'",
      capability.getName(), binding.getName());
  return tryProvision(caseInstance, capability, triggerChannelId, triggerCorrelationId);
}
```

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: agent exclusion filter before routing strategy

Reads _outcomes.<bindingName>.excludedAgents from working panel and
filters candidates before agentRoutingStrategy.select(). All routing
strategies (LeastLoaded, TrustWeighted, Semantic) benefit automatically.

Empty candidates after filtering falls through to tryProvision().

Closes #502"
```

---

### Task 7: Failure goals → COMPLETED not FAULTED (#506)

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/CaseStatusChanged.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandlerTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/CaseOutcomeObserverTest.java`

- [ ] **Step 1: Write test for failure goal producing COMPLETED**

```java
@Test
void failure_goal_satisfied_produces_COMPLETED_with_goal_metadata() {
  // Setup: CaseInstance with failure goal expression satisfied
  // Act: call evaluateCompletion
  // Assert: CaseStatusChanged published with:
  //   - newStatus = COMPLETED (not FAULTED)
  //   - satisfiedGoalName = "review-blocked"
  //   - satisfiedGoalKind = GoalKind.FAILURE
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=GoalReachedEventHandlerTest
```
Expected: FAIL — CaseStatusChanged has no goal metadata fields.

- [ ] **Step 3: Add goal metadata to CaseStatusChanged**

```java
// common/src/main/java/io/casehub/engine/common/internal/event/CaseStatusChanged.java
public record CaseStatusChanged(
    CaseInstance instance,
    String oldStatus,
    String newStatus,
    String satisfiedGoalName,
    GoalKind satisfiedGoalKind) {

  public CaseStatusChanged(CaseInstance instance, String oldStatus, String newStatus) {
    this(instance, oldStatus, newStatus, null, null);
  }
}
```

Import `io.casehub.api.model.GoalKind`.

- [ ] **Step 4: Update GoalReachedEventHandler — both goal types publish COMPLETED with metadata**

In `evaluateCompletion()`, change the failure goal branch from FAULTED to COMPLETED with metadata:

```java
if (goalBasedCompletion.getFailure() != null
    && isGoalExpressionSatisfied(goalBasedCompletion.getFailure(), reachedGoals)) {
  String failureGoalName = findSatisfiedGoalName(goalBasedCompletion.getFailure(), reachedGoals);
  LOG.infof("Failure goal '%s' satisfied: caseId=%s", failureGoalName, caseInstance.getUuid());
  eventBus.publish(EventBusAddresses.CASE_STATUS_CHANGED,
      new CaseStatusChanged(caseInstance, oldStatus, CaseStatus.COMPLETED.name(),
          failureGoalName, GoalKind.FAILURE));
  return Uni.createFrom().voidItem();
}

if (goalBasedCompletion.getSuccess() != null
    && isGoalExpressionSatisfied(goalBasedCompletion.getSuccess(), reachedGoals)) {
  String successGoalName = findSatisfiedGoalName(goalBasedCompletion.getSuccess(), reachedGoals);
  LOG.infof("Success goal '%s' satisfied: caseId=%s", successGoalName, caseInstance.getUuid());
  eventBus.publish(EventBusAddresses.CASE_STATUS_CHANGED,
      new CaseStatusChanged(caseInstance, oldStatus, CaseStatus.COMPLETED.name(),
          successGoalName, GoalKind.SUCCESS));
  return Uni.createFrom().voidItem();
}
```

Add helper:
```java
private String findSatisfiedGoalName(GoalExpression expression, Set<String> reachedGoals) {
  if (expression == null || expression.getGoals() == null) return null;
  return expression.getGoals().stream()
      .map(Goal::getName)
      .filter(reachedGoals::contains)
      .findFirst().orElse(null);
}
```

- [ ] **Step 5: Update CaseStatusChangedHandler to propagate goal metadata**

In `onCaseStatusChangedHandler()`, extract goal metadata from the event and include in EventLog metadata and CaseOutcomeEvent:

In the EventLog metadata construction:
```java
ObjectNode metadataNode = OBJECT_MAPPER.createObjectNode()
    .put("oldStatus", oldStatus)
    .put("newStatus", event.newStatus());
if (event.satisfiedGoalName() != null) {
  metadataNode.put("goalName", event.satisfiedGoalName());
  metadataNode.put("goalKind", event.satisfiedGoalKind().value());
}
eventLog.setMetadata(metadataNode);
```

In `fireOutcomeObservers()`, build metadata map with goal info:
```java
final Map<String, Object> outcomeMetadata;
if (event.satisfiedGoalName() != null) {
  outcomeMetadata = Map.of(
      "goalName", event.satisfiedGoalName(),
      "goalKind", event.satisfiedGoalKind().value());
} else {
  outcomeMetadata = Map.of();
}
```

Pass `outcomeMetadata` instead of `Map.of()` to `CaseOutcomeEvent`. This requires passing the event (or just the goal fields) to `fireOutcomeObservers()`.

Modify `fireOutcomeObservers` signature:
```java
private void fireOutcomeObservers(CaseInstance caseInstance, CaseStatus newState,
    String goalName, GoalKind goalKind) {
```

Call site:
```java
if (isTerminalState(newState)) {
  fireOutcomeObservers(caseInstance, newState,
      event.satisfiedGoalName(), event.satisfiedGoalKind());
}
```

- [ ] **Step 6: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

- [ ] **Step 7: Update CaseOutcomeObserverTest**

Verify that the observer receives goal metadata:

```java
@Test
void caseOutcomeObserver_receives_goal_metadata_on_failure_completion() {
  // Setup: CaseStatusChanged with goalName="review-blocked", goalKind=FAILURE
  // Act: call onCaseStatusChangedHandler
  // Assert: observer.onOutcome() called with:
  //   - outcomeLabel = "COMPLETED"
  //   - metadata contains goalName and goalKind
}
```

- [ ] **Step 8: Run full test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime
```

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: failure goals produce COMPLETED with goal metadata

GoalReachedEventHandler publishes COMPLETED (not FAULTED) when failure
goals are satisfied. CaseStatusChanged carries satisfiedGoalName and
satisfiedGoalKind. CaseStatusChangedHandler propagates metadata to
EventLog and CaseOutcomeEvent.metadata().

Outcome observers distinguish: outcomeLabel=COMPLETED + goalKind=failure
= process completed with negative outcome. FAULTED reserved for system faults.

Closes #506"
```

---

### Task 8: Integration test — full reroute cycle

**Files:**
- Test: `runtime/src/test/java/io/casehub/engine/FailureCascadeIntegrationTest.java`

- [ ] **Step 1: Write integration test**

A `@QuarkusTest` that exercises the full reroute cycle:

1. Define a CaseHub with two workers (`worker-a`, `worker-b`) for capability `review`
2. `worker-a` returns `WorkerResult.declined("can't handle")`
3. Verify `_outcomes.review` has status DECLINED, excludedAgents [worker-a]
4. Verify CONTEXT_CHANGED triggered binding re-evaluation
5. Verify `worker-b` was dispatched (worker-a excluded)
6. `worker-b` returns `WorkerResult.of(Map.of("result", "done"))`
7. Verify case completes normally

- [ ] **Step 2: Write REROUTES_EXHAUSTED test**

Same setup but both workers decline. Verify:
- `_outcomes.review.status` is `REROUTES_EXHAUSTED`
- PlanItem is FAULTED
- If a failure goal exists, case COMPLETED with goal metadata

- [ ] **Step 3: Write FAULT policy test**

Binding has `outcomePolicy: { onDecline: FAULT }`. Worker declines. Verify:
- Case is FAULTED
- PlanItem is FAULTED
- `_outcomes` contains the declined entry

- [ ] **Step 4: Run tests**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=FailureCascadeIntegrationTest
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "test: failure cascade integration tests

Full reroute cycle: worker-a declines → excluded → worker-b dispatched.
REROUTES_EXHAUSTED: both decline → failure goal fires COMPLETED.
FAULT policy: immediate case FAULTED on decline.

Closes #503, #504"
```

---

### Task 9: CLAUDE.md + DESIGN.md updates

**Files:**
- Modify: engine CLAUDE.md (worker execution architecture, new event types)
- Modify: engine DESIGN.md if applicable

- [ ] **Step 1: Update CLAUDE.md**

Add to the Worker Execution Architecture section:

```markdown
## Worker Outcome Handling

Workers declare semantic outcomes via `WorkerResult`: `Success` (default), `Declined(reason)`, `Failed(reason)`. The engine handles non-success outcomes via `OutcomePolicy` on the `Binding`:

- `REROUTE` (default): writes failure state to `_outcomes.<bindingName>` in the working panel, marks PlanItem FAULTED, publishes CONTEXT_CHANGED. The binding re-fires with excluded agents filtered from candidates.
- `FAULT`: marks both PlanItem and case FAULTED immediately.

Failure state schema at `_outcomes.<bindingName>`: `{status, attempts, history[], excludedAgents[]}`. Status values: `DECLINED`, `FAILED`, `REROUTES_EXHAUSTED`.

`WorkerOutcomeResolvedHandler` (blackboard, `blocking=true`) consumes `WORKER_OUTCOME_RESOLVED` and owns PlanItem lifecycle for non-success outcomes. `PlanItemCompletionHandler` gates on `WorkerOutcome.Success` and returns early for DECLINED/FAILED.

Agent exclusion: `CaseContextChangedEventHandler.publishWorkerSchedule()` filters excluded agents from `_outcomes.<bindingName>.excludedAgents` before calling the routing strategy.

Failure goals: `GoalReachedEventHandler` produces `CaseStatus.COMPLETED` (not `FAULTED`) with goal metadata (`satisfiedGoalName`, `satisfiedGoalKind`). FAULTED is reserved for infrastructure faults.
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "docs: update CLAUDE.md with failure cascade conventions

Worker outcome handling, OutcomePolicy, _outcomes state convention,
agent exclusion filter, failure goal semantics.

Refs #502, #503, #504, #506"
```

---

### Task 10: Run full test suite across all modules

- [ ] **Step 1: Build and test everything**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test
```

All modules must pass. Fix any compilation or test failures.

- [ ] **Step 2: Commit any fixes**

```bash
git add -A
git commit -m "fix: address compilation/test issues from failure cascade

Refs #502, #503, #504, #506"
```
