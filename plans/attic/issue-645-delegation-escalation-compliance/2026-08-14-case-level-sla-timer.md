# Case-Level SLA Timer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #510 — feat: Case-level SLA — time-triggered binding for overall case deadline
**Issue group:** #510

**Goal:** Add `SignalTarget` to the `BindingTarget` sealed hierarchy, implement YAML `ScheduleTrigger` conversion, and wire the dispatch paths so timer-triggered signals write to the case context and enable failure goal evaluation.

**Architecture:** New sealed permit `SignalTarget(Map<String, Object> payload)` on `BindingTarget`. Two dispatch paths: context-change-triggered signals handled inline in `CaseContextChangedEventHandler`, schedule-triggered signals via a new `ScheduledSignalJob` Quartz job that publishes `ContextSignalEvent` to a new `ContextSignalEventHandler`. YAML mapper gains `ScheduleTrigger` conversion (fixing an existing TODO) and `signal:` target parsing.

**Tech Stack:** Java 21, Quarkus, Vert.x event bus, Quartz (RAM store), jackson-jq, jsonschema2pojo

## Global Constraints

- Virtual-thread handler convention (PP-20260723-c4c1cf): all `@ConsumeEvent` handlers return `void` + `@RunOnVirtualThread`
- Plan-type module boundary (PP-20260727-5267d2): definition types in engine-api, execution types in engine-common
- `SignalTarget` requires `LifecycleScope.BINDING` only — no compound or case scope
- No PlanItem created for signal bindings
- Static payload only (v1) — `Map<String, Object>` frozen at definition time
- YAML schema field for duration is `every` (not `delay`) — maps to `ScheduleTrigger.delay(Duration)`
- Use `ide_insert_member` for new methods, `ide_replace_member` for modifying existing implementations
- Use `mcp__intellij-index__*` tools for all code navigation — never bash grep

---

### Task 1: SignalTarget type + BindingTarget sealed permit + Builder

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/SignalTarget.java`
- Modify: `api/src/main/java/io/casehub/api/model/BindingTarget.java:27` — add `SignalTarget` to permits
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java:198-393` — add `.signal()` builder method + validation
- Test: `api/src/test/java/io/casehub/api/model/SignalTargetTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `SignalTarget(Map<String, Object> payload)` record implementing `BindingTarget`. `Binding.Builder.signal(Map<String, Object>)` convenience method.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.time.Duration;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SignalTargetTest {

  @Test
  void signalTarget_createsImmutablePayload() {
    var payload = Map.of("caseSla", Map.of("expired", true));
    var target = new SignalTarget(payload);
    assertThat(target.payload()).isEqualTo(payload);
    assertThatThrownBy(() -> target.payload().put("extra", "value"))
        .isInstanceOf(UnsupportedOperationException.class);
  }

  @Test
  void signalTarget_rejectsNullPayload() {
    assertThatThrownBy(() -> new SignalTarget(null))
        .isInstanceOf(NullPointerException.class);
  }

  @Test
  void signalTarget_rejectsEmptyPayload() {
    assertThatThrownBy(() -> new SignalTarget(Map.of()))
        .isInstanceOf(IllegalArgumentException.class);
  }

  @Test
  void binding_signalBuilder_setsTarget() {
    var binding = Binding.builder()
        .name("case-timeout")
        .signal(Map.of("caseSla", Map.of("expired", true)))
        .on(ScheduleTrigger.delay(Duration.ofHours(48)))
        .build();
    assertThat(binding.target()).isInstanceOf(SignalTarget.class);
    assertThat(((SignalTarget) binding.target()).payload())
        .containsKey("caseSla");
  }

  @Test
  void binding_signalTarget_rejectsNonBindingScope() {
    assertThatThrownBy(() ->
        Binding.builder()
            .name("bad")
            .signal(Map.of("key", "value"))
            .on(ScheduleTrigger.delay(Duration.ofHours(1)))
            .lifecycleScope(LifecycleScope.COMPOUND)
            .build())
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("BINDING");
  }

  @Test
  void binding_signalTarget_rejectsCompanionParticipation() {
    assertThatThrownBy(() ->
        Binding.builder()
            .name("bad")
            .signal(Map.of("key", "value"))
            .on(ScheduleTrigger.delay(Duration.ofHours(1)))
            .participation(Participation.COMPANION)
            .build())
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("COMPANION");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=SignalTargetTest -DfailIfNoTests=false -q`
Expected: FAIL — `SignalTarget` class does not exist

- [ ] **Step 3: Create SignalTarget record**

Create `api/src/main/java/io/casehub/api/model/SignalTarget.java`:

```java
package io.casehub.api.model;

import java.util.Map;
import java.util.Objects;

public record SignalTarget(Map<String, Object> payload) implements BindingTarget {
    public SignalTarget {
        Objects.requireNonNull(payload, "payload must not be null");
        if (payload.isEmpty()) {
            throw new IllegalArgumentException("SignalTarget payload must not be empty");
        }
        payload = Map.copyOf(payload);
    }
}
```

- [ ] **Step 4: Add SignalTarget to BindingTarget permits**

Modify `api/src/main/java/io/casehub/api/model/BindingTarget.java` line 27:

```java
public sealed interface BindingTarget
    permits CapabilityTarget, SubCaseTarget, HumanTaskTarget, SignalTarget, ExtensionTarget {}
```

- [ ] **Step 5: Add signal() builder method to Binding.Builder**

Use `ide_insert_member` to add to `Binding.Builder` (after the `humanTask` method):

```java
public Builder signal(Map<String, Object> payload) {
    this.target = new SignalTarget(payload);
    return this;
}
```

- [ ] **Step 6: Add SignalTarget validation to Binding.Builder.build()**

In `Binding.Builder.build()`, after the existing `ScopeActivatedTrigger` validation (line 361), add:

```java
if (target instanceof SignalTarget && ls != LifecycleScope.BINDING) {
    throw new IllegalArgumentException(
        "SignalTarget requires BINDING scope, got " + ls);
}
if (target instanceof SignalTarget && p == Participation.COMPANION) {
    throw new IllegalArgumentException(
        "SignalTarget cannot use COMPANION participation");
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=SignalTargetTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 8: Fix compilation in all exhaustive switch sites**

Adding `SignalTarget` to the sealed `BindingTarget` will cause compilation failures in all exhaustive `switch` expressions. Fix each with a `case SignalTarget` branch:

| File | Line | Fix |
|------|------|-----|
| `common/.../BindingExecutorResolver.java:47` | After `case ExtensionTarget` | `case SignalTarget st -> ExecutorRef.of("signal");` |
| `runtime/.../SchedulerService.java:101-119` | Target switch expression | See Task 3 (restructure needed) |
| `runtime/.../SchedulerService.java:206-220` | `createJobData` switch | `case SignalTarget ignored -> throw new IllegalStateException("createJobData called with SignalTarget '" + binding.getName() + "'");` |
| `scheduler-quartz/.../QuartzWorkerExecutionManager.java:367-378` | `createJobData` switch | `case SignalTarget ignored -> throw new IllegalStateException("Schedule-triggered binding '" + binding.getName() + "' — use SchedulerService for SignalTarget");` |
| `runtime/.../CaseContextChangedEventHandler.java:366` | `publishByTarget` | See Task 4 (main dispatch) |

The `CbrCaseRetainObserver.buildRoutingKeyMap()` uses `default ->` — already handles `SignalTarget` via the catch-all.

**Cross-repo coordination:** The work repo's `PlanItemCompletionApplier.java` has an exhaustive switch on `BindingTarget`. After publishing the engine artifact with the new `SignalTarget` permit, the work repo will fail to compile. Fix: add `case SignalTarget ignored -> null;` in `casehubio/work`. This must be done immediately after the engine artifact is published — file a work repo issue and coordinate the fix before merging to main.

- [ ] **Step 9: Build to verify compilation**

Run: `/opt/homebrew/bin/mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```
feat(#510): add SignalTarget to BindingTarget sealed hierarchy

Refs casehubio/engine#510
```

---

### Task 2: YAML schema + mapper — ScheduleTrigger and SignalTarget

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml:657-660` — add `signal` to Binding oneOf
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java:1097-1120` — implement `ScheduleTrigger` conversion
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java:927-966` — add `signal:` target handling in `convertBinding()`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperSignalTest.java`

**Interfaces:**
- Consumes: `SignalTarget` record (Task 1), `ScheduleTrigger.delay(Duration)` and `ScheduleTrigger.cron(String)` factory methods (existing)
- Produces: YAML definitions with `schedule:` triggers and `signal:` targets parse into `ScheduleTrigger` and `SignalTarget` instances

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.api.model.converter;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ScheduleTrigger;
import io.casehub.api.model.SignalTarget;
import java.io.InputStream;
import java.time.Duration;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperSignalTest {

  @Test
  void scheduleTrigger_delay_parsesFromYaml() {
    CaseDefinition def = loadDefinition("signal-sla-test.yaml");
    Binding timeout = def.getBindings().stream()
        .filter(b -> "case-timeout".equals(b.getName()))
        .findFirst().orElseThrow();
    assertThat(timeout.getOn()).isInstanceOf(ScheduleTrigger.class);
    ScheduleTrigger trigger = (ScheduleTrigger) timeout.getOn();
    assertThat(trigger.isDelay()).isTrue();
    assertThat(trigger.getDelay()).isEqualTo(Duration.ofHours(48));
  }

  @Test
  void scheduleTrigger_cron_parsesFromYaml() {
    CaseDefinition def = loadDefinition("signal-cron-test.yaml");
    Binding periodic = def.getBindings().stream()
        .filter(b -> "periodic-check".equals(b.getName()))
        .findFirst().orElseThrow();
    assertThat(periodic.getOn()).isInstanceOf(ScheduleTrigger.class);
    ScheduleTrigger trigger = (ScheduleTrigger) periodic.getOn();
    assertThat(trigger.isCron()).isTrue();
    assertThat(trigger.getCron()).isEqualTo("0 0 * * *");
  }

  @Test
  void signalTarget_parsesFromYaml() {
    CaseDefinition def = loadDefinition("signal-sla-test.yaml");
    Binding timeout = def.getBindings().stream()
        .filter(b -> "case-timeout".equals(b.getName()))
        .findFirst().orElseThrow();
    assertThat(timeout.target()).isInstanceOf(SignalTarget.class);
    SignalTarget st = (SignalTarget) timeout.target();
    assertThat(st.payload()).containsKey("caseSla");
  }

  @Test
  void signalTarget_requiresPayload() {
    assertThatThrownBy(() -> loadDefinition("signal-empty-payload-test.yaml"))
        .isInstanceOf(IllegalArgumentException.class);
  }

  private CaseDefinition loadDefinition(String filename) {
    InputStream is = getClass().getClassLoader().getResourceAsStream("yaml/" + filename);
    return CaseDefinitionYamlMapper.fromYaml(is);
  }
}
```

- [ ] **Step 2: Create test YAML fixtures**

Create `api/src/test/resources/yaml/signal-sla-test.yaml`:

```yaml
spec:
  capabilities:
    - name: review-code
      inputSchema: "."
  goals:
    - name: review-complete
      when: ".reviewResult != null"
    - name: review-timed-out
      when: ".caseSla.expired == true"
  completion:
    success:
      allOf: [review-complete]
    failure:
      anyOf: [review-timed-out]
  workers:
    - name: code-reviewer
      capabilities: [review-code]
  bindings:
    - name: do-review
      on:
        contextChange:
          filter: ".reviewRequest != null"
      capability: review-code
    - name: case-timeout
      on:
        schedule:
          every: PT48H
      when: ".caseSla.expired == null"
      signal:
        caseSla:
          expired: true
```

Create `api/src/test/resources/yaml/signal-cron-test.yaml`:

```yaml
spec:
  capabilities:
    - name: check-status
      inputSchema: "."
  workers:
    - name: status-checker
      capabilities: [check-status]
  bindings:
    - name: periodic-check
      on:
        schedule:
          cron: "0 0 * * *"
      signal:
        statusCheck:
          triggered: true
```

Create `api/src/test/resources/yaml/signal-empty-payload-test.yaml`:

```yaml
spec:
  capabilities:
    - name: noop
      inputSchema: "."
  workers:
    - name: noop-worker
      capabilities: [noop]
  bindings:
    - name: bad-signal
      on:
        schedule:
          every: PT1H
      signal: {}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=CaseDefinitionYamlMapperSignalTest -DfailIfNoTests=false -q`
Expected: FAIL — `ScheduleTrigger` conversion throws `UnsupportedOperationException`

- [ ] **Step 4: Update YAML schema — add signal to Binding oneOf**

Modify `schema/src/main/resources/schema/CaseDefinition.yaml` at line 657-661:

```yaml
    oneOf:
      - required: [ capability ]
      - required: [ subCase ]
      - required: [ humanTask ]
      - required: [ signal ]
```

Add the `signal` property to the Binding properties block (after `humanTask` at line 667):

```yaml
      signal:
        type: object
        additionalProperties: true
        description: "Context signal payload. Written to the case context when the binding fires. No worker dispatch — the signal IS the action."
```

- [ ] **Step 5: Regenerate schema classes**

Run: `/opt/homebrew/bin/mvn generate-sources -pl schema -q`

Verify `io.casehub.model.Binding` now has `getSignal()` returning `Object` (or a generated type).

- [ ] **Step 6: Implement ScheduleTrigger conversion in convertTrigger()**

Replace the TODO block in `CaseDefinitionYamlMapper.convertTrigger()` (lines 1116-1119):

```java
if (schemaTrigger.getSchedule() != null) {
    io.casehub.model.ScheduleTrigger st = schemaTrigger.getSchedule();
    if (st.getCron() != null) {
        return io.casehub.api.model.ScheduleTrigger.cron(st.getCron());
    } else if (st.getEvery() != null) {
        return io.casehub.api.model.ScheduleTrigger.delay(
            java.time.Duration.parse(st.getEvery()));
    } else {
        throw new IllegalArgumentException(
            "ScheduleTrigger must have either 'cron' or 'every' set");
    }
}

throw new UnsupportedOperationException(
    "Only ContextChangeTrigger, ScopeActivatedTrigger, and ScheduleTrigger are currently supported. "
        + "CloudEventTrigger conversion not yet implemented.");
```

- [ ] **Step 7: Add signal target handling in convertBinding()**

In `convertBinding()` at line 955, add a `signal:` branch to the target resolution:

```java
} else if (schemaBinding.getSignal() != null) {
    @SuppressWarnings("unchecked")
    Map<String, Object> signalPayload =
        MAPPER.convertValue(schemaBinding.getSignal(), Map.class);
    if (signalPayload.isEmpty()) {
        throw new IllegalArgumentException(
            "Binding '" + schemaBinding.getName() + "' signal payload must not be empty");
    }
    builder.signal(signalPayload);
} else {
    throw new IllegalArgumentException(
        "Binding '" + schemaBinding.getName()
            + "' must have capability, subCase, humanTask, or signal");
}
```

Update the existing `else` error message to include `signal`.

- [ ] **Step 8: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn test -pl api -Dtest=CaseDefinitionYamlMapperSignalTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 9: Commit**

```
feat(#510): YAML parity — ScheduleTrigger conversion + signal target parsing

Refs casehubio/engine#510
```

---

### Task 3: New event type, event bus address, ContextSignalEvent

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java:80` — add `CONTEXT_SIGNAL_APPLIED`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/EventBusAddresses.java:110` — add `CONTEXT_SIGNAL`
- Create: `common/src/main/java/io/casehub/engine/common/internal/event/ContextSignalEvent.java`
- Test: `common/src/test/java/io/casehub/engine/common/internal/event/ContextSignalEventTest.java`

**Interfaces:**
- Consumes: `CaseInstance` (existing)
- Produces: `CaseHubEventType.CONTEXT_SIGNAL_APPLIED`, `EventBusAddresses.CONTEXT_SIGNAL`, `ContextSignalEvent(CaseInstance, String bindingName, Map<String, Object> payload)`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.engine.common.internal.event;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.Map;
import org.junit.jupiter.api.Test;

class ContextSignalEventTest {

  @Test
  void contextSignalEvent_carriesAllFields() {
    CaseInstance instance = new CaseInstance();
    var event = new ContextSignalEvent(
        instance, "case-timeout", Map.of("caseSla", Map.of("expired", true)));
    assertThat(event.caseInstance()).isSameAs(instance);
    assertThat(event.bindingName()).isEqualTo("case-timeout");
    assertThat(event.payload()).containsKey("caseSla");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn test -pl common -Dtest=ContextSignalEventTest -DfailIfNoTests=false -q`
Expected: FAIL — `ContextSignalEvent` does not exist

- [ ] **Step 3: Add CONTEXT_SIGNAL_APPLIED to CaseHubEventType**

Use `ide_insert_member` to add after `PATTERN_CHECKPOINT` (line 80):

```java
CONTEXT_SIGNAL_APPLIED,
```

- [ ] **Step 4: Add CONTEXT_SIGNAL to EventBusAddresses**

Use `ide_insert_member` to add after `COMPOUND_ACTIVATED` (line 110):

```java
public static final String CONTEXT_SIGNAL = "casehub.context.signal";
```

- [ ] **Step 5: Create ContextSignalEvent record**

Create `common/src/main/java/io/casehub/engine/common/internal/event/ContextSignalEvent.java`:

```java
package io.casehub.engine.common.internal.event;

import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.Map;

public record ContextSignalEvent(
    CaseInstance caseInstance,
    String bindingName,
    Map<String, Object> payload
) {}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn test -pl common -Dtest=ContextSignalEventTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#510): add CONTEXT_SIGNAL_APPLIED event type and ContextSignalEvent record

Refs casehubio/engine#510
```

---

### Task 4: Context-change dispatch path — CaseContextChangedEventHandler

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:347-370` — add `case SignalTarget` branch
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerSignalTest.java`

**Interfaces:**
- Consumes: `SignalTarget` (Task 1), `CaseHubEventType.CONTEXT_SIGNAL_APPLIED` (Task 3), `EventBusAddresses.CONTEXT_SIGNAL` (Task 3)
- Produces: Signal payload written to working layer, EventLog with `CONTEXT_SIGNAL_APPLIED`, `CONTEXT_CHANGED` published

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.engine.internal.engine.handler;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.SignalTarget;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CaseContextChangedEventHandlerSignalTest {

  @Test
  void signalTarget_writesPayloadToContext() {
    // Verify that when a SignalTarget binding fires,
    // the payload is written to the case context working layer.
    // This test should call publishByTarget() with a SignalTarget binding
    // and assert the context contains the signal payload after the call.
    // Exact wiring depends on test infrastructure — use the handler's
    // existing test patterns (mock EventBus, mock repositories).
  }

  @Test
  void signalTarget_writesEventLog() {
    // Verify CONTEXT_SIGNAL_APPLIED EventLog entry is created
    // with correct metadata (bindingName, signalKeys, triggerType)
  }

  @Test
  void signalTarget_publishesContextChanged() {
    // Verify CONTEXT_CHANGED is published to the event bus
    // after signal payload is applied
  }
}
```

Note: The test bodies above are skeletal — the implementer must follow the existing test patterns in `CaseContextChangedEventHandlerTest` (mock `EventBus`, mock `EventLogRepository`, construct `CaseInstance` with context). The key assertions are: (1) context contains signal keys, (2) EventLog has `CONTEXT_SIGNAL_APPLIED`, (3) event bus received `CONTEXT_CHANGED`.

- [ ] **Step 2: Implement the SignalTarget branch in publishByTarget()**

In `CaseContextChangedEventHandler.publishByTarget()`, replace the `case ExtensionTarget` warning with `case SignalTarget` before it:

```java
case SignalTarget st -> {
    st.payload().forEach((key, value) ->
        caseInstance.getCaseContext().set(key, value));

    EventLog eventLog = new EventLog();
    eventLog.setCaseId(caseInstance.getUuid());
    eventLog.setEventType(CaseHubEventType.CONTEXT_SIGNAL_APPLIED);
    eventLog.setStreamType(io.casehub.engine.common.internal.history.EventStreamType.CASE);
    eventLog.setTimestamp(java.time.Instant.now());
    eventLog.setMetadata(OBJECT_MAPPER.createObjectNode()
        .put("bindingName", binding.getName())
        .put("triggerType", "contextChange")
        .set("signalKeys", OBJECT_MAPPER.valueToTree(st.payload().keySet())));
    eventLogRepository.append(eventLog, caseInstance.tenancyId);

    eventBus.publish(EventBusAddresses.CONTEXT_CHANGED,
        new io.casehub.engine.common.internal.event.CaseContextChangedEvent(
            caseInstance, signalId, traceId));
}
```

- [ ] **Step 3: Run tests**

Run: `/opt/homebrew/bin/mvn test -pl runtime -Dtest=CaseContextChangedEventHandlerSignalTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 4: Commit**

```
feat(#510): wire SignalTarget dispatch in CaseContextChangedEventHandler

Refs casehubio/engine#510
```

---

### Task 5: Schedule-triggered signals — SchedulerService + ScheduledSignalJob + ContextSignalEventHandler

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/scheduler/SchedulerService.java:80-137` — restructure to handle SignalTarget
- Create: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ScheduledSignalJob.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzJobScheduler.java:113-143` — add `signal` triggerType
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ContextSignalEventHandler.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/ContextSignalEventHandlerTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/scheduler/SchedulerServiceSignalTest.java`

**Interfaces:**
- Consumes: `SignalTarget` (Task 1), `ContextSignalEvent` (Task 3), `EventBusAddresses.CONTEXT_SIGNAL` (Task 3), `JobScheduler` SPI (existing)
- Produces: `ScheduledSignalJob` Quartz job, `ContextSignalEventHandler` event handler, signal bindings scheduled via Quartz

- [ ] **Step 1: Write ContextSignalEventHandler test**

```java
package io.casehub.engine.internal.engine.handler;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.event.ContextSignalEvent;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.Map;
import org.junit.jupiter.api.Test;

class ContextSignalEventHandlerTest {

  @Test
  void onContextSignal_writesPayloadAndPublishesContextChanged() {
    // Construct CaseInstance with empty context
    // Call handler.onContextSignal(event)
    // Assert: context contains signal payload
    // Assert: EventLog with CONTEXT_SIGNAL_APPLIED written
    // Assert: CONTEXT_CHANGED published to event bus
    // Follow existing handler test patterns (mock EventBus, mock repos)
  }
}
```

- [ ] **Step 2: Write SchedulerService signal scheduling test**

```java
package io.casehub.engine.internal.scheduler;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ScheduleTrigger;
import io.casehub.api.model.SignalTarget;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.scheduler.ScheduledJobRequest;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.scheduler.JobScheduler;
import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

class SchedulerServiceSignalTest {

  @Test
  void registerScheduledTriggers_schedulesSignalTarget() {
    JobScheduler scheduler = mock(JobScheduler.class);
    CaseDefinitionRegistry registry = mock(CaseDefinitionRegistry.class);
    SchedulerService service = new SchedulerService(scheduler, registry);

    CaseDefinition def = CaseDefinition.builder()
        .name("test")
        .namespace("test")
        .binding(Binding.builder()
            .name("case-timeout")
            .signal(Map.of("caseSla", Map.of("expired", true)))
            .on(ScheduleTrigger.delay(Duration.ofHours(48)))
            .build())
        .build();

    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    when(registry.getCaseDefinition(any())).thenReturn(def);

    service.registerScheduledTriggers(instance);

    ArgumentCaptor<ScheduledJobRequest> captor =
        ArgumentCaptor.forClass(ScheduledJobRequest.class);
    verify(scheduler).schedule(captor.capture());

    ScheduledJobRequest request = captor.getValue();
    assertThat(request.getData().get("triggerType")).isEqualTo("signal");
    assertThat(request.getData().get("bindingName")).isEqualTo("case-timeout");
    assertThat(request.getData()).containsKey("signalPayload");
  }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `/opt/homebrew/bin/mvn test -pl runtime -Dtest="ContextSignalEventHandlerTest|SchedulerServiceSignalTest" -DfailIfNoTests=false -q`
Expected: FAIL

- [ ] **Step 4: Restructure SchedulerService.registerScheduledTriggers()**

The current method uses a switch expression yielding `CapabilityTarget`. Restructure to a switch statement with early dispatch for `SignalTarget`:

```java
public void registerScheduledTriggers(CaseInstance caseInstance) {
    // ... existing preamble (definition lookup, null checks) ...

    for (Binding binding : bindings) {
        if (!(binding.getOn() instanceof ScheduleTrigger trigger)) {
            continue;
        }

        switch (binding.target()) {
            case SignalTarget st -> scheduleSignal(caseInstance.getUuid(), binding, trigger, st);
            case CapabilityTarget ct -> {
                Worker worker = findWorkerForCapability(definition, ct.capability());
                if (worker == null) {
                    LOG.warnf("No worker found for capability '%s' in binding '%s', skipping",
                        ct.capability().name(), binding.getName());
                    continue;
                }
                if (binding.getWhen() != null) {
                    scheduleConditionalWorker(caseInstance.getUuid(), binding, trigger, worker);
                } else {
                    scheduleWorker(caseInstance.getUuid(), binding, trigger, worker);
                }
            }
            case SubCaseTarget st -> LOG.warnf(
                "Schedule binding '%s' has SubCase target — skipping", binding.getName());
            case HumanTaskTarget ht -> LOG.warnf(
                "Schedule binding '%s' has HumanTask target — skipping", binding.getName());
            case ExtensionTarget et -> LOG.warnf(
                "Schedule binding '%s' has Extension target — skipping", binding.getName());
        }
    }
}
```

Add the `scheduleSignal()` method:

```java
private static final ObjectMapper SIGNAL_MAPPER = new ObjectMapper();

private void scheduleSignal(UUID caseId, Binding binding, ScheduleTrigger trigger, SignalTarget st) {
    JobIdentifier jobId = createJobIdentifier(caseId, binding.getName());
    ScheduleStrategy schedule = toScheduleStrategy(trigger);
    Map<String, Object> data = new HashMap<>();
    data.put("caseId", caseId.toString());
    data.put("bindingName", binding.getName());
    data.put("triggerType", "signal");
    try {
        data.put("signalPayload", SIGNAL_MAPPER.writeValueAsString(
            SIGNAL_MAPPER.valueToTree(st.payload())));
    } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
        throw new IllegalStateException("Failed to serialize signal payload", e);
    }
    if (binding.getWhen() != null) {
        data.put("hasCondition", "true");
    }

    scheduler.schedule(ScheduledJobRequest.builder()
        .jobId(jobId).schedule(schedule).data(data));
    LOG.infof("Scheduled signal trigger: case=%s, binding=%s, trigger=%s",
        caseId, binding.getName(), trigger);
}
```

Note: `OBJECT_MAPPER` needs to be added as a static field if not already present. Use `com.fasterxml.jackson.databind.ObjectMapper`.

- [ ] **Step 5: Add signal triggerType to QuartzJobScheduler.resolveJobClass()**

In `QuartzJobScheduler.resolveJobClass()`, add before the `triggerType == null` branch:

```java
} else if ("signal".equals(triggerType)) {
    jobClass = ScheduledSignalJob.class;
```

- [ ] **Step 6: Create ScheduledSignalJob**

Create `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ScheduledSignalJob.java`:

```java
package io.casehub.engine.scheduler.quartz;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.event.ContextSignalEvent;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.recovery.WorkerExecutionRecoveryService;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import org.jboss.logging.Logger;
import org.quartz.Job;
import org.quartz.JobDataMap;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

@DisallowConcurrentExecution
@ApplicationScoped
public class ScheduledSignalJob implements Job {

    private static final Logger LOG = Logger.getLogger(ScheduledSignalJob.class);
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    @Inject EventBus eventBus;
    @Inject WorkerExecutionRecoveryService recoveryService;
    @Inject CaseDefinitionRegistry caseDefinitionRegistry;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap data = context.getJobDetail().getJobDataMap();
        String caseIdStr = data.getString("caseId");
        String bindingName = data.getString("bindingName");
        String signalPayloadJson = data.getString("signalPayload");

        UUID caseId = UUID.fromString(caseIdStr);
        LOG.infof("Signal trigger fired: case=%s, binding=%s", caseId, bindingName);

        CaseInstance caseInstance;
        try {
            caseInstance = recoveryService.loadOrRestoreCaseInstance(caseId);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to load case %s, skipping signal trigger", caseId);
            return;
        }

        if (caseInstance.getState() != CaseStatus.RUNNING) {
            LOG.infof("Case %s is %s, skipping signal trigger", caseId, caseInstance.getState());
            return;
        }

        if ("true".equals(data.getString("hasCondition"))) {
            CaseDefinition definition =
                caseDefinitionRegistry.getCaseDefinition(caseInstance.getCaseMetaModel());
            if (definition == null) {
                LOG.warnf("CaseDefinition not found for case %s", caseId);
                return;
            }
            Binding binding = definition.getBindings().stream()
                .filter(b -> bindingName.equals(b.getName()))
                .findFirst().orElse(null);
            if (binding == null || binding.getWhen() == null) {
                LOG.warnf("Binding '%s' not found or has no condition", bindingName);
                return;
            }
            try {
                var result = binding.getWhen().evaluate(
                    caseInstance.getCaseContext()
                        .layer(io.casehub.api.context.ContextLayer.WORKING)
                        .asJsonNode());
                if (!result.ok() || !result.isTrue()) {
                    LOG.debugf("Signal condition not met for case=%s binding=%s", caseId, bindingName);
                    return;
                }
            } catch (Exception e) {
                LOG.warnf(e, "Condition evaluation failed for case=%s binding=%s", caseId, bindingName);
                return;
            }
        }

        try {
            Map<String, Object> payload = OBJECT_MAPPER.readValue(
                signalPayloadJson, new TypeReference<Map<String, Object>>() {});
            eventBus.publish(EventBusAddresses.CONTEXT_SIGNAL,
                new ContextSignalEvent(caseInstance, bindingName, payload));
        } catch (Exception e) {
            throw new JobExecutionException("Failed to parse signal payload", e);
        }
    }
}
```

- [ ] **Step 7: Create ContextSignalEventHandler**

Create `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ContextSignalEventHandler.java`:

```java
package io.casehub.engine.internal.engine.handler;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.internal.event.CaseContextChangedEvent;
import io.casehub.engine.common.internal.event.ContextSignalEvent;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.history.EventLog;
import io.casehub.engine.common.internal.history.EventStreamType;
import io.casehub.engine.common.spi.EventLogRepository;
import io.quarkus.virtual.threads.VirtualThreads;
import io.smallrye.common.annotation.RunOnVirtualThread;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import org.jboss.logging.Logger;
import io.quarkus.vertx.ConsumeEvent;

@ApplicationScoped
public class ContextSignalEventHandler {

    private static final Logger LOG = Logger.getLogger(ContextSignalEventHandler.class);
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    @Inject EventBus eventBus;
    @Inject EventLogRepository eventLogRepository;

    @ConsumeEvent(EventBusAddresses.CONTEXT_SIGNAL)
    @RunOnVirtualThread
    public void onContextSignal(ContextSignalEvent event) {
        var caseInstance = event.caseInstance();
        var bindingName = event.bindingName();
        var payload = event.payload();

        LOG.infof("Applying context signal: case=%s, binding=%s, keys=%s",
            caseInstance.getUuid(), bindingName, payload.keySet());

        payload.forEach((key, value) ->
            caseInstance.getCaseContext().set(key, value));

        EventLog eventLog = new EventLog();
        eventLog.setCaseId(caseInstance.getUuid());
        eventLog.setEventType(CaseHubEventType.CONTEXT_SIGNAL_APPLIED);
        eventLog.setStreamType(EventStreamType.CASE);
        eventLog.setTimestamp(Instant.now());
        eventLog.setMetadata(OBJECT_MAPPER.createObjectNode()
            .put("bindingName", bindingName)
            .put("triggerType", "schedule")
            .set("signalKeys", OBJECT_MAPPER.valueToTree(payload.keySet())));
        eventLogRepository.append(eventLog, caseInstance.tenancyId);

        eventBus.publish(EventBusAddresses.CONTEXT_CHANGED,
            new CaseContextChangedEvent(caseInstance, null, null));
    }
}
```

- [ ] **Step 8: Run tests**

Run: `/opt/homebrew/bin/mvn test -pl runtime -Dtest="ContextSignalEventHandlerTest|SchedulerServiceSignalTest" -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 9: Full build**

Run: `/opt/homebrew/bin/mvn install -DskipTests -q`
Expected: BUILD SUCCESS

- [ ] **Step 10: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -q`
Expected: PASS (or only pre-existing failures)

- [ ] **Step 11: Commit**

```
feat(#510): schedule-triggered signals — ScheduledSignalJob + ContextSignalEventHandler

Refs casehubio/engine#510
```

---

### Task 6: Integration test — end-to-end case-level SLA

**Files:**
- Test: `runtime/src/test/java/io/casehub/engine/CaseLevelSlaIntegrationTest.java`

**Interfaces:**
- Consumes: All previous tasks — full stack from YAML definition through Quartz scheduling to goal evaluation

- [ ] **Step 1: Write the integration test**

Create a `@QuarkusTest` that:
1. Defines a CaseHub with a `ScheduleTrigger.delay()` signal binding + a failure goal
2. Starts a case
3. Verifies the signal binding was scheduled (via `SchedulerService`)
4. Manually fires the Quartz job (or uses a short delay like `PT1S` for the test)
5. Asserts the case context contains the SLA expiry signal
6. Asserts the failure goal is reached and the case transitions to FAULTED

Follow the pattern of `ScheduleTriggerSimpleTest` for Quartz job firing mechanics and the `MilestoneLifecycleTest` for SLA verification patterns.

- [ ] **Step 2: Run the integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=CaseLevelSlaIntegrationTest -DfailIfNoTests=false -q`
Expected: PASS

- [ ] **Step 3: Commit**

```
test(#510): end-to-end integration test for case-level SLA timer

Refs casehubio/engine#510
```

---

### Task 7: Update CLAUDE.md + contributor guide

**Files:**
- Modify: `CLAUDE.md` — add SignalTarget to BindingTarget documentation
- Modify: `docs/guides/consumer-guide.md` — add signal target YAML example

- [ ] **Step 1: Add SignalTarget section to CLAUDE.md**

Under the existing `BindingTarget` / `Trigger` documentation sections, add:

```markdown
`SignalTarget(Map<String, Object> payload)` — engine-internal context mutation. When the binding fires, the payload is written to the case context working layer and `CONTEXT_CHANGED` is published. No worker dispatch, no PlanItem. Use for timer-triggered SLA expiry, escalation flags, and other engine-internal state transitions. `LifecycleScope.BINDING` only. YAML: `signal:` block on binding.
```

Document the ScheduleTrigger YAML parity fix:

```markdown
`ScheduleTrigger` — YAML `schedule:` block with `every:` (ISO-8601 Duration, one-shot) or `cron:` (Quartz expression, periodic). Maps to `ScheduleTrigger.delay(Duration)` or `ScheduleTrigger.cron(String)`.
```

- [ ] **Step 2: Add signal target example to consumer guide**

Add a "Case-Level SLA" section with the complete YAML example from the spec.

- [ ] **Step 3: Commit**

```
docs(#510): document SignalTarget and ScheduleTrigger YAML parity

Closes casehubio/engine#510
```
