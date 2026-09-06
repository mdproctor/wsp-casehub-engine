# Watchdog → Recovery Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1044 — Watchdog → Recovery bridge
**Issue group:** #1044, #1023, #974

**Goal:** Wire qhorus stall detection to engine recovery actions via a new
optional module, fix FlowWorkerFunctionHandler timeout bug, and prepare
engine-side circular dependency fix.

**Architecture:** New `StallRecoveryHandler` SPI in engine-common alongside
existing `RecoveryCoordinator`. SPI types in engine-api, implementation in
new `casehub-engine-watchdog` module. `CaseChannel.parseCaseId()` for case
resolution. Vert.x event bus `@ConsumeEvent(blocking=true)` for handler
serialization.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x event bus, CDI

## Global Constraints

- Module boundary protocol PP-20260727-5267d2: plan-definition types in
  engine-api, execution types in engine-common
- Cross-repo source verification PP-20260722-60e519: foundation tier types
  live in casehubio/worker
- `@DefaultBean @ApplicationScoped` for all no-op SPI defaults
- `NamedStrategy` for all classifier SPIs
- `CaseDefinition` fields: nullable, builder pattern, YAML mapping via
  `CaseDefinitionYamlMapper`

---

## Batch 1: Flow timeout fix (#1023)

### Task 1: Enforce timeout in FlowWorkerFunctionHandler

**Files:**
- Modify: `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java`
- Modify: `flow/src/test/java/io/casehub/engine/flow/YamlSimpleCaseHubBeanTest.java`
- Modify: `flow/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `WorkerFunctionHandler.execute()` contract, `HandlerResult`, `WorkerResult.expired()`
- Produces: No new interfaces — fixes existing handler

- [ ] **Step 1: Write failing test for timeout enforcement**

```java
// In a new test class: FlowWorkerFunctionHandlerTimeoutTest.java
@QuarkusTest
class FlowWorkerFunctionHandlerTimeoutTest {

    @Inject FlowWorkerFunctionHandler handler;

    @Test
    void timeoutProducesExpiredResult() {
        // Create a FlowWorkerFunction with a workflow that sleeps > timeout
        var slowWorkflow = // ... workflow that takes 5s
        var function = new FlowWorkerFunction(slowWorkflow, null);
        var context = new WorkerContext(UUID.randomUUID(), null, null, null, null, null, null, List.of(), List.of());
        var metadata = new ExecutionMetadata("slow-worker", "hash", null, null, null);

        HandlerResult result = handler.execute(function, Map.of(), context, 100, metadata);

        assertInstanceOf(WorkerOutcome.Expired.class, result.result().outcome());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl flow -Dtest=FlowWorkerFunctionHandlerTimeoutTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `.join()` blocks indefinitely, test hangs

- [ ] **Step 3: Fix the handler — replace .join() with .get(timeoutMs, MILLISECONDS)**

In `FlowWorkerFunctionHandler.execute()`, replace the blocking `.join()` call:

```java
@Override
public HandlerResult execute(
    final WorkerFunction<?, ?> function,
    final Object inputData,
    final WorkerContext context,
    final int timeoutMs,
    final ExecutionMetadata metadata) {
  final FlowWorkerFunction flow = (FlowWorkerFunction) function;
  @SuppressWarnings("unchecked")
  final Map<String, Object> mapInput = (Map<String, Object>) inputData;

  final Workflow workflow = flow.workflow();
  final WorkflowInstance wfInstance = app.workflowDefinition(workflow).instance(mapInput);
  final String instanceId = wfInstance.id();

  registry.register(instanceId, context.caseId(), metadata.workerName(), metadata.inputDataHash());
  try {
    final CompletableFuture<WorkflowModel> future = wfInstance.start();
    future.whenComplete((model, ex) -> registry.remove(instanceId));

    final WorkflowModel model;
    try {
      model = future.get(timeoutMs, java.util.concurrent.TimeUnit.MILLISECONDS);
    } catch (java.util.concurrent.TimeoutException e) {
      future.cancel(true);
      registry.remove(instanceId);
      return new HandlerResult(WorkerResult.expired(
          "Flow workflow timed out after " + timeoutMs + "ms"));
    } catch (java.util.concurrent.ExecutionException e) {
      return new HandlerResult(WorkerResult.failed(
          "Flow workflow failed: " + e.getCause().getMessage()));
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      return new HandlerResult(WorkerResult.failed("Flow workflow interrupted"));
    }

    Map<String, Object> outputMap = model.asMap()
        .orElseThrow(() -> new RuntimeException(
            "Workflow produced non-serializable model for worker: " + metadata.workerName()));

    if (flow.plannedActionFn() != null) {
      return new HandlerResult(WorkerResult.of(outputMap, flow.plannedActionFn().apply(mapInput)));
    }
    return new HandlerResult(WorkerResult.of(outputMap));
  } catch (final RuntimeException e) {
    registry.remove(instanceId);
    throw e;
  }
}
```

- [ ] **Step 4: Run timeout test to verify it passes**

Run: `mvn test -pl flow -Dtest=FlowWorkerFunctionHandlerTimeoutTest`
Expected: PASS

- [ ] **Step 5: Clean up test resources**

In `flow/src/test/resources/application.properties`:
- Remove duplicate entries in `quarkus.arc.selected-alternatives` (each repo is listed twice)
- Remove `@Tag("flaky")` from `YamlSimpleCaseHubBeanTest.testExecution()`

- [ ] **Step 6: Run all flow tests**

Run: `mvn test -pl flow`
Expected: All tests PASS including `testExecution` (no longer flaky)

- [ ] **Step 7: Commit**

```bash
git add flow/
git commit -m "fix(#1023): enforce timeout in FlowWorkerFunctionHandler

Replace .join() with .get(timeoutMs, MILLISECONDS) to match the pattern
used by all other WorkerFunctionHandler implementations. The handler was
receiving timeoutMs but ignoring it, causing indefinite blocking under
CI load.

Also clean up duplicate selected-alternatives entries and remove
@Tag(\"flaky\") from YamlSimpleCaseHubBeanTest.

Closes #1023"
```

---

## Batch 2: Stall recovery SPI types (#1044 foundation)

### Task 2: StallRecoveryAction, StallRecoveryPolicy, StallRecoveryContext, StallClassifier

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/StallRecoveryAction.java`
- Create: `api/src/main/java/io/casehub/api/model/StallRecoveryPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/StallRecoveryContext.java`
- Create: `api/src/main/java/io/casehub/api/spi/recovery/StallClassifier.java`
- Create: `api/src/main/java/io/casehub/api/spi/recovery/StallClassificationContext.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/recovery/StallRecoveryHandler.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpStallRecoveryHandler.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/recovery/DefaultStallClassifier.java`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add stallRecoveryPolicy field)
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` (add STALL_DETECTED, STALL_RECOVERY_INITIATED)
- Modify: `api/src/main/java/io/casehub/api/model/converter/yaml/CaseDefinitionYamlMapper.java` (parse stallRecoveryPolicy)
- Create: `api/src/test/java/io/casehub/api/model/StallRecoveryPolicyTest.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/recovery/DefaultStallClassifierTest.java`

**Interfaces:**
- Consumes: `NamedStrategy` (platform-api), `WatchdogConditionType` (qhorus-api), `CaseDefinition` builder pattern
- Produces: `StallRecoveryAction` enum, `StallRecoveryPolicy` record, `StallRecoveryContext` record, `StallClassifier` SPI, `StallRecoveryHandler` SPI — all consumed by Batch 3

- [ ] **Step 1: Write test for StallRecoveryAction enum**

```java
// api/src/test/java/io/casehub/api/model/StallRecoveryPolicyTest.java
class StallRecoveryPolicyTest {

    @Test
    void defaultPolicyHasNotifyDefault() {
        var policy = StallRecoveryPolicy.DEFAULT;
        assertEquals(StallRecoveryAction.NOTIFY, policy.defaultAction());
        assertFalse(policy.enabled());
    }

    @Test
    void conditionActionsOverrideDefault() {
        var policy = new StallRecoveryPolicy(true, "policy-lookup",
            Map.of(WatchdogConditionType.LOOP_DETECTED, StallRecoveryAction.CANCEL),
            StallRecoveryAction.NOTIFY);
        assertEquals(StallRecoveryAction.CANCEL,
            policy.conditionActions().get(WatchdogConditionType.LOOP_DETECTED));
    }
}
```

- [ ] **Step 2: Run test — expected FAIL (types don't exist)**

- [ ] **Step 3: Create StallRecoveryAction enum**

```java
// api/src/main/java/io/casehub/api/model/StallRecoveryAction.java
package io.casehub.api.model;

public enum StallRecoveryAction {
    RETRY, REROUTE, ESCALATE, CANCEL, NOTIFY, EXPIRE, IGNORE;

    public boolean requiresBinding() {
        return this == REROUTE || this == CANCEL || this == EXPIRE;
    }
}
```

- [ ] **Step 4: Create StallRecoveryPolicy record**

```java
// api/src/main/java/io/casehub/api/model/StallRecoveryPolicy.java
package io.casehub.api.model;

import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import java.util.Map;

public record StallRecoveryPolicy(
    boolean enabled,
    String classifierId,
    Map<WatchdogConditionType, StallRecoveryAction> conditionActions,
    StallRecoveryAction defaultAction) {

    public static final StallRecoveryPolicy DEFAULT =
        new StallRecoveryPolicy(false, "policy-lookup", Map.of(), StallRecoveryAction.NOTIFY);

    public StallRecoveryPolicy {
        conditionActions = conditionActions != null ? Map.copyOf(conditionActions) : Map.of();
        if (defaultAction == null) defaultAction = StallRecoveryAction.NOTIFY;
        if (classifierId == null || classifierId.isBlank()) classifierId = "policy-lookup";
    }
}
```

- [ ] **Step 5: Create StallRecoveryContext record**

```java
// api/src/main/java/io/casehub/api/model/StallRecoveryContext.java
package io.casehub.api.model;

import io.casehub.qhorus.api.watchdog.AlertContext;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import jakarta.annotation.Nullable;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record StallRecoveryContext(
    UUID caseId,
    String tenancyId,
    WatchdogConditionType conditionType,
    List<String> affectedAgentIds,
    String alertSummary,
    AlertContext alertContext,
    Instant firedAt,
    @Nullable String resolvedBindingName,
    @Nullable String resolvedPlanItemId) {

    public StallRecoveryContext {
        affectedAgentIds = affectedAgentIds != null ? List.copyOf(affectedAgentIds) : List.of();
    }

    public StallRecoveryContext withBinding(String bindingName, String planItemId) {
        return new StallRecoveryContext(caseId, tenancyId, conditionType, affectedAgentIds,
            alertSummary, alertContext, firedAt, bindingName, planItemId);
    }
}
```

- [ ] **Step 6: Create StallClassifier SPI and StallClassificationContext**

```java
// api/src/main/java/io/casehub/api/spi/recovery/StallClassifier.java
package io.casehub.api.spi.recovery;

import io.casehub.api.model.StallRecoveryAction;
import io.casehub.platform.api.routing.NamedStrategy;

public interface StallClassifier extends NamedStrategy {
    StallRecoveryAction classify(StallClassificationContext context);

    @Override
    default String id() { return "policy-lookup"; }
}

// api/src/main/java/io/casehub/api/spi/recovery/StallClassificationContext.java
package io.casehub.api.spi.recovery;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.StallRecoveryContext;
import io.casehub.api.model.StallRecoveryPolicy;

public record StallClassificationContext(
    StallRecoveryContext recoveryContext,
    CaseDefinition definition,
    StallRecoveryPolicy policy) {}
```

- [ ] **Step 7: Create StallRecoveryHandler SPI + NoOp default**

```java
// common/src/main/java/io/casehub/engine/common/spi/recovery/StallRecoveryHandler.java
package io.casehub.engine.common.spi.recovery;

import io.casehub.api.model.StallRecoveryContext;

public interface StallRecoveryHandler {
    boolean handleStall(StallRecoveryContext context);
}

// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpStallRecoveryHandler.java
package io.casehub.engine.internal.worker;

import io.casehub.api.model.StallRecoveryContext;
import io.casehub.engine.common.spi.recovery.StallRecoveryHandler;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpStallRecoveryHandler implements StallRecoveryHandler {
    @Override
    public boolean handleStall(StallRecoveryContext context) { return false; }
}
```

- [ ] **Step 8: Add event types**

Add to `CaseHubEventType.java`:
```java
STALL_DETECTED,
STALL_RECOVERY_INITIATED,
```

- [ ] **Step 9: Add stallRecoveryPolicy to CaseDefinition**

Use `ide_insert_member` to add:
- Field: `private StallRecoveryPolicy stallRecoveryPolicy;`
- Getter: `public StallRecoveryPolicy getStallRecoveryPolicy() { return stallRecoveryPolicy; }`
- Setter: `public void setStallRecoveryPolicy(StallRecoveryPolicy p) { this.stallRecoveryPolicy = p; }`
- Builder method: `public Builder stallRecoveryPolicy(StallRecoveryPolicy p) { ... }`

- [ ] **Step 10: Write DefaultStallClassifier test**

```java
// runtime/src/test/java/io/casehub/engine/internal/recovery/DefaultStallClassifierTest.java
class DefaultStallClassifierTest {

    @Test
    void looksUpFromConditionActions() {
        var policy = new StallRecoveryPolicy(true, "policy-lookup",
            Map.of(WatchdogConditionType.LOOP_DETECTED, StallRecoveryAction.CANCEL),
            StallRecoveryAction.NOTIFY);
        var ctx = stallContext(WatchdogConditionType.LOOP_DETECTED);
        var classCtx = new StallClassificationContext(ctx, null, policy);

        assertEquals(StallRecoveryAction.CANCEL,
            new DefaultStallClassifier().classify(classCtx));
    }

    @Test
    void fallsBackToDefaultAction() {
        var policy = new StallRecoveryPolicy(true, "policy-lookup",
            Map.of(), StallRecoveryAction.NOTIFY);
        var ctx = stallContext(WatchdogConditionType.QUEUE_DEPTH);
        var classCtx = new StallClassificationContext(ctx, null, policy);

        assertEquals(StallRecoveryAction.NOTIFY,
            new DefaultStallClassifier().classify(classCtx));
    }

    @Test
    void downgradesBindingActionWhenNoBinding() {
        var policy = new StallRecoveryPolicy(true, "policy-lookup",
            Map.of(WatchdogConditionType.AGENT_STALE, StallRecoveryAction.CANCEL),
            StallRecoveryAction.NOTIFY);
        var ctx = stallContext(WatchdogConditionType.AGENT_STALE); // no binding resolved
        var classCtx = new StallClassificationContext(ctx, null, policy);

        assertEquals(StallRecoveryAction.NOTIFY,
            new DefaultStallClassifier().classify(classCtx));
    }

    private StallRecoveryContext stallContext(WatchdogConditionType type) {
        return new StallRecoveryContext(UUID.randomUUID(), "tenant-1", type,
            List.of(), "test", null, Instant.now(), null, null);
    }
}
```

- [ ] **Step 11: Implement DefaultStallClassifier**

```java
// runtime/src/main/java/io/casehub/engine/internal/recovery/DefaultStallClassifier.java
package io.casehub.engine.internal.recovery;

import io.casehub.api.model.StallRecoveryAction;
import io.casehub.api.spi.recovery.StallClassificationContext;
import io.casehub.api.spi.recovery.StallClassifier;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;

@DefaultBean
@ApplicationScoped
public class DefaultStallClassifier implements StallClassifier {

    private static final Logger LOG = Logger.getLogger(DefaultStallClassifier.class);

    @Override
    public StallRecoveryAction classify(StallClassificationContext context) {
        StallRecoveryAction action = context.policy().conditionActions()
            .getOrDefault(context.recoveryContext().conditionType(),
                          context.policy().defaultAction());
        if (action.requiresBinding() && context.recoveryContext().resolvedBindingName() == null) {
            LOG.warnf("Stall action %s requires binding resolution but none available for %s — downgrading to NOTIFY",
                action, context.recoveryContext().conditionType());
            return StallRecoveryAction.NOTIFY;
        }
        return action;
    }

    @Override
    public String id() { return "policy-lookup"; }
}
```

- [ ] **Step 12: Add YAML mapping for stallRecoveryPolicy**

In `CaseDefinitionYamlMapper`, add parsing for the `stallRecoveryPolicy` block
from the `spec:` node. Parse `conditionActions` as a map of string→string,
convert keys via `WatchdogConditionType.valueOf()` and values via
`StallRecoveryAction.valueOf()`.

- [ ] **Step 13: Run all tests**

Run: `mvn install -DskipTests -q && mvn test -pl api,common,runtime`
Expected: All PASS

- [ ] **Step 14: Commit**

```bash
git add api/ common/ runtime/
git commit -m "feat(#1044): stall recovery SPI types — StallRecoveryAction, StallRecoveryPolicy, StallClassifier, StallRecoveryHandler

Foundation types for the watchdog → recovery bridge:
- StallRecoveryAction enum with 7 recovery actions
- StallRecoveryPolicy per-case config on CaseDefinition
- StallRecoveryContext record with case/binding resolution fields
- StallClassifier SPI (NamedStrategy) with DefaultStallClassifier
- StallRecoveryHandler SPI with NoOp default
- STALL_DETECTED and STALL_RECOVERY_INITIATED event types
- YAML mapping for stallRecoveryPolicy block

Refs #1044"
```

---

## Batch 3: Watchdog module implementation (#1044)

### Task 3: Module setup + WatchdogAlertObserver + StallRecoveryDispatchHandler

**Files:**
- Create: `watchdog/pom.xml`
- Create: `watchdog/src/main/java/io/casehub/engine/watchdog/WatchdogAlertObserver.java`
- Create: `watchdog/src/main/java/io/casehub/engine/watchdog/StallRecoveryDispatchHandler.java`
- Modify: `pom.xml` (add `<module>watchdog</module>`)
- Create: `watchdog/src/test/java/io/casehub/engine/watchdog/WatchdogAlertObserverTest.java`
- Create: `watchdog/src/test/java/io/casehub/engine/watchdog/StallRecoveryDispatchHandlerTest.java`
- Create: `watchdog/src/test/resources/application.properties`

**Interfaces:**
- Consumes: `WatchdogAlertEvent` (qhorus-api), `CaseChannel.parseCaseId()`, `CaseInstanceCache`, `CaseDefinitionRegistry`, `StallRecoveryHandler`, `PlanItemStore`
- Produces: `WatchdogAlertObserver` (CDI observer), `StallRecoveryDispatchHandler` (event bus consumer)

- [ ] **Step 1: Create module pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>
    <artifactId>casehub-engine-watchdog</artifactId>
    <name>CaseHub Engine — Watchdog Recovery Bridge</name>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-common</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-worker-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-vertx</artifactId>
        </dependency>
        <!-- test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-persistence-memory</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Add `<module>watchdog</module>` to root `pom.xml`.

- [ ] **Step 2: Write test for case resolution**

```java
@QuarkusTest
class WatchdogAlertObserverTest {

    @Test
    void resolvesFromTargetName() {
        UUID caseId = UUID.randomUUID();
        String channelName = CaseChannel.channelName(caseId, "main");
        var event = new WatchdogAlertEvent(UUID.randomUUID(), channelName,
            null, "test", Instant.now(),
            new ChannelIdleContext(List.of(channelName), 60));

        UUID resolved = CaseChannel.parseCaseId(event.targetName());
        assertEquals(caseId, resolved);
    }

    @Test
    void returnsNullForNonCaseTarget() {
        var event = new WatchdogAlertEvent(UUID.randomUUID(), "*",
            null, "test", Instant.now(),
            new AgentStaleContext(1, List.of("agent-1")));

        UUID resolved = CaseChannel.parseCaseId(event.targetName());
        assertNull(resolved);
    }
}
```

- [ ] **Step 3: Implement WatchdogAlertObserver**

```java
@ApplicationScoped
public class WatchdogAlertObserver {

    @Inject CaseInstanceCache caseInstanceCache;
    @Inject CaseDefinitionRegistry definitionRegistry;
    @Inject EventBus eventBus;

    void onAlert(@ObservesAsync WatchdogAlertEvent event) {
        UUID caseId = resolveCaseId(event);
        if (caseId == null) return;

        var instance = caseInstanceCache.get(caseId);
        if (instance == null || instance.getState().isTerminal()) return;

        var definition = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
        if (definition == null) return;

        var policy = definition.getStallRecoveryPolicy();
        if (policy == null || !policy.enabled()) return;

        var context = new StallRecoveryContext(
            caseId, instance.getTenancyId(), event.conditionType(),
            event.context().affectedAgentIds(), event.summary(),
            event.context(), event.firedAt(), null, null);

        eventBus.publish("casehub.stall.recovery", context);
    }

    private UUID resolveCaseId(WatchdogAlertEvent event) {
        // Primary: parse from targetName
        UUID caseId = CaseChannel.parseCaseId(event.targetName());
        if (caseId != null) return caseId;

        // Fallback: extract channel from AlertContext subtype
        String channelName = extractChannelName(event.context());
        if (channelName != null) {
            return CaseChannel.parseCaseId(channelName);
        }
        return null;
    }

    private String extractChannelName(AlertContext ctx) {
        return switch (ctx) {
            case BarrierStuckContext c -> c.channelName();
            case LoopDetectedContext c -> c.channelName();
            case ContextPressureContext c -> c.channelName();
            case CircularDelegationContext c -> c.channelName();
            case DeliveryLagContext c -> c.channelName();
            case ObligationFanOutContext c -> c.channelName();
            case ConversationStallContext c -> c.channelName();
            case EchoChamberContext c -> c.channelName();
            case QueueDepthContext c -> c.channelName();
            case ChannelIdleContext c ->
                c.channelNames().isEmpty() ? null : c.channelNames().getFirst();
            case AgentStaleContext c -> null;
            case ApprovalPendingContext c -> null;
        };
    }
}
```

- [ ] **Step 4: Write test for binding resolution in dispatch handler**

Test that RUNNING PlanItems are matched by executorName against affectedAgentIds.

- [ ] **Step 5: Implement StallRecoveryDispatchHandler**

```java
@ApplicationScoped
public class StallRecoveryDispatchHandler {

    @Inject StallRecoveryHandler stallRecoveryHandler;
    @Inject CaseInstanceCache caseInstanceCache;
    @Inject CaseDefinitionRegistry definitionRegistry;
    @Inject Instance<PlanItemStore> planItemStore;

    @ConsumeEvent(value = "casehub.stall.recovery", blocking = true)
    void onStallRecovery(StallRecoveryContext context) {
        var instance = caseInstanceCache.get(context.caseId());
        if (instance == null || instance.getState().isTerminal()) return;

        var enriched = resolveBinding(context);
        stallRecoveryHandler.handleStall(enriched);
    }

    private StallRecoveryContext resolveBinding(StallRecoveryContext ctx) {
        if (!planItemStore.isResolvable() || ctx.affectedAgentIds().isEmpty()) return ctx;

        var store = planItemStore.get();
        var items = store.findByCaseId(ctx.caseId(), ctx.tenancyId()).stream()
            .filter(r -> r.status() == TaskStatus.RUNNING)
            .filter(r -> ctx.affectedAgentIds().contains(r.executorName()))
            .sorted(Comparator.comparing(PlanItemRecord::createdAt).reversed())
            .toList();

        if (items.isEmpty()) return ctx;
        var match = items.getFirst();
        return ctx.withBinding(match.bindingName(), match.planItemId());
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -q && mvn test -pl watchdog`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add watchdog/ pom.xml
git commit -m "feat(#1044): casehub-engine-watchdog module — alert observer + dispatch handler

New optional module bridging qhorus watchdog alerts to engine recovery.
WatchdogAlertObserver resolves caseId via CaseChannel.parseCaseId(),
publishes to event bus. StallRecoveryDispatchHandler resolves binding
from PlanItemStore and invokes StallRecoveryHandler SPI.

Refs #1044"
```

### Task 4: DefaultStallRecoveryHandler + integration test

**Files:**
- Create: `watchdog/src/main/java/io/casehub/engine/watchdog/DefaultStallRecoveryHandler.java`
- Create: `watchdog/src/test/java/io/casehub/engine/watchdog/DefaultStallRecoveryHandlerTest.java`
- Create: `watchdog/src/test/java/io/casehub/engine/watchdog/WatchdogIntegrationTest.java`

**Interfaces:**
- Consumes: `StallRecoveryHandler` SPI, `StallClassifier`, `EventBus`, `PlanItemStore`, `EventLogRepository`, `JudgmentScheduler`, `CaseInstanceCache`, `MutableCaseContext`
- Produces: Complete watchdog bridge — alert → classify → action

- [ ] **Step 1: Write tests for each action type**

Test RETRY (publishes CONTEXT_CHANGED), REROUTE (writes excludedAgents),
CANCEL (marks PlanItem CANCELLED), EXPIRE (publishes WORKER_OUTCOME_RESOLVED),
ESCALATE (invokes JudgmentScheduler), NOTIFY (writes EventLog only),
IGNORE (returns false).

- [ ] **Step 2: Write idempotency tests**

Test CANCEL when PlanItem already terminal, REROUTE when agent already excluded,
EXPIRE when PlanItem not RUNNING.

- [ ] **Step 3: Implement DefaultStallRecoveryHandler**

Full implementation with all 7 action handlers, idempotency guards, and
EventLog writing. Each action follows the spec's detailed implementation
description. Uses `EngineStrategyResolver` for StallClassifier resolution.

Key structure:
```java
@ApplicationScoped
public class DefaultStallRecoveryHandler implements StallRecoveryHandler {

    // Inject: StallClassifier (via EngineStrategyResolver), EventBus,
    // PlanItemStore, EventLogRepository, CaseInstanceCache,
    // CaseDefinitionRegistry, JudgmentScheduler

    @Override
    public boolean handleStall(StallRecoveryContext context) {
        var definition = definitionRegistry.getCaseDefinition(...);
        var policy = definition.getStallRecoveryPolicy();
        var classCtx = new StallClassificationContext(context, definition, policy);
        StallRecoveryAction action = classifier.classify(classCtx);

        boolean acted = switch (action) {
            case RETRY -> executeRetry(context);
            case REROUTE -> executeReroute(context);
            case CANCEL -> executeCancel(context);
            case EXPIRE -> executeExpire(context);
            case ESCALATE -> executeEscalate(context);
            case NOTIFY -> executeNotify(context);
            case IGNORE -> false;
        };

        if (acted && action != StallRecoveryAction.NOTIFY) {
            writeAuditLog(context, action);
        }
        return acted;
    }
    // ... per-action methods
}
```

- [ ] **Step 4: Write integration test**

End-to-end: fire a `WatchdogAlertEvent` with a known caseId channel name,
verify the case's `_diagnostics` is updated (for REROUTE) or EventLog is
written (for NOTIFY).

- [ ] **Step 5: Run all tests**

Run: `mvn install -DskipTests -q && mvn test -pl watchdog`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add watchdog/
git commit -m "feat(#1044): DefaultStallRecoveryHandler — all 7 recovery actions

Implements RETRY, REROUTE, ESCALATE, CANCEL, EXPIRE, NOTIFY, IGNORE with
idempotency guards per action. Classifies via StallClassifier, writes
STALL_RECOVERY_INITIATED EventLog for audit.

Refs #1044"
```

---

## Batch 4: Circular dependency engine-side (#974)

### Task 5: SPI extraction + dependency removal

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/InboundWorkItemScheduler.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/InboundWorkItemRequest.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpInboundWorkItemScheduler.java`
- Modify: `casehub-engine-inbound/src/main/java/io/casehub/engine/inbound/InboundWorkItemBridge.java`
- Modify: `casehub-engine-inbound/src/main/java/io/casehub/engine/inbound/InboundWorkItemPolicy.java`
- Modify: `casehub-engine-inbound/pom.xml` (remove casehub-work dependency)
- Delete: `actor-state/src/main/java/io/casehub/actorstate/WorkActorStateContributor.java` (use `ide_refactor_safe_delete`)
- Modify: `actor-state/pom.xml` (remove casehub-work dependency)
- Create: `common/src/test/java/io/casehub/engine/common/spi/InboundWorkItemRequestTest.java`

**Interfaces:**
- Consumes: `MessageObserver`, `MessageReceivedEvent` (qhorus-api)
- Produces: `InboundWorkItemScheduler` SPI — work repo implements in work-engine-adapter

- [ ] **Step 1: Write test for InboundWorkItemRequest**

```java
class InboundWorkItemRequestTest {

    @Test
    void buildsFromMinimalFields() {
        var request = InboundWorkItemRequest.builder()
            .title("Review document")
            .tenancyId("tenant-1")
            .build();
        assertEquals("Review document", request.title());
        assertEquals("tenant-1", request.tenancyId());
    }
}
```

- [ ] **Step 2: Create InboundWorkItemScheduler SPI and request type**

```java
// common/src/main/java/io/casehub/engine/common/spi/InboundWorkItemScheduler.java
package io.casehub.engine.common.spi;

public interface InboundWorkItemScheduler {
    void schedule(InboundWorkItemRequest request);
}

// common/src/main/java/io/casehub/engine/common/spi/InboundWorkItemRequest.java
package io.casehub.engine.common.spi;

// Record with builder — mirrors WorkItemCreateRequest fields that
// InboundWorkItemBridge currently uses: title, description,
// candidateGroups, candidateUsers, callerRef, scope, tenancyId, createdBy
public record InboundWorkItemRequest(
    String title, String description,
    java.util.Set<String> candidateGroups, java.util.Set<String> candidateUsers,
    String callerRef, String scope, String tenancyId, String createdBy) {

    public static Builder builder() { return new Builder(); }

    // Builder class with fluent API
    public static final class Builder { /* ... */ }
}
```

- [ ] **Step 3: Create NoOp default**

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpInboundWorkItemScheduler.java
@DefaultBean
@ApplicationScoped
public class NoOpInboundWorkItemScheduler implements InboundWorkItemScheduler {
    private static final Logger LOG = Logger.getLogger(NoOpInboundWorkItemScheduler.class);

    @Override
    public void schedule(InboundWorkItemRequest request) {
        LOG.warnf("InboundWorkItemScheduler not available — work item '%s' dropped", request.title());
    }
}
```

- [ ] **Step 4: Refactor InboundWorkItemBridge**

Replace `WorkItemService` + `TenantContextRunner` injection with
`InboundWorkItemScheduler`. Change `InboundWorkItemPolicy.decide()` return type
from `Optional<WorkItemCreateRequest>` to `Optional<InboundWorkItemRequest>`.

The bridge becomes:
```java
@Inject InboundWorkItemScheduler scheduler;

@Override
public void onMessage(final MessageReceivedEvent event) {
    if (policy.isUnsatisfied()) return;

    final Optional<InboundWorkItemRequest> decision;
    try {
        decision = policy.get().decide(event);
    } catch (Exception e) {
        LOG.warnf(e, "InboundWorkItemPolicy.decide() threw — message ignored");
        return;
    }
    decision.ifPresent(request -> scheduler.schedule(
        InboundWorkItemRequest.builder()
            .title(request.title())
            // ... map all fields
            .tenancyId(event.tenancyId())
            .createdBy("casehub-engine-inbound")
            .build()));
}
```

- [ ] **Step 5: Remove casehub-work dependencies from pom.xml files**

In `casehub-engine-inbound/pom.xml`: remove `casehub-work` and
`casehub-work-persistence-memory` dependencies.

In `actor-state/pom.xml`: remove `casehub-work` dependency.

- [ ] **Step 6: Delete WorkActorStateContributor**

Use `ide_refactor_safe_delete` to remove
`actor-state/src/main/java/io/casehub/actorstate/WorkActorStateContributor.java`.

- [ ] **Step 7: Verify build**

Run: `mvn install -DskipTests -q && mvn test -pl casehub-engine-inbound,actor-state`
Expected: Build succeeds, tests pass. No work-runtime on engine classpath.

- [ ] **Step 8: File follow-up issues**

File in casehubio/work:
1. "Implement InboundWorkItemScheduler in work-engine-adapter" — provides
   the real implementation injecting WorkItemService + TenantContextRunner
2. "Add WorkActorStateContributor to work-engine-adapter" — relocated from
   engine actor-state module

- [ ] **Step 9: Commit**

```bash
git add common/ runtime/ casehub-engine-inbound/ actor-state/
git commit -m "chore(#974): break engine→work circular dependency

Extract InboundWorkItemScheduler SPI in engine-common. Refactor
InboundWorkItemBridge to use SPI instead of WorkItemService directly.
Remove WorkActorStateContributor (relocated to work repo — follow-up
issues filed).

Remove casehub-work compile dependency from engine-inbound and
actor-state. Engine now depends on work-api at most, never work runtime.
Build order: work-api → engine → work. Clean DAG.

Refs #974"
```

---

## References

- [2026-09-05-watchdog-recovery-bridge-design.md] — design spec this plan implements
- `flow/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:74,86` — timeout bug
- `runtime/src/main/java/io/casehub/engine/internal/executor/SyncAgentWorkerFunctionHandler.java:149` — correct timeout pattern
- `common/src/main/java/io/casehub/engine/common/spi/recovery/RecoveryCoordinator.java` — existing recovery SPI
- `api/src/main/java/io/casehub/api/model/CaseChannel.java:69` — parseCaseId()
- `casehub-engine-inbound/src/main/java/.../InboundWorkItemBridge.java` — boundary violation
- `actor-state/src/main/java/.../WorkActorStateContributor.java` — work concern in wrong repo
- PP-20260727-5267d2 — plan-type module boundary
- PP-20260722-60e519 — cross-repo source verification
- GitHub #1044, #1023, #974
