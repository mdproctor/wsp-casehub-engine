# Concurrency Throttle & Watchdog→Recovery Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/parent#468 — Foundation Concurrency Throttle and Watchdog→Recovery Bridge
**Issue group:** engine#1043, engine#1044, qhorus#433

**Goal:** Add case-level binding dispatch throttling with an external budget SPI, and wire qhorus watchdog stall detection to the engine's recovery pipeline via synthetic `Expired` outcomes.

**Architecture:** Two independent mechanisms composing through the existing CONTEXT_CHANGED loop. Budget caps dispatch admission (pre-dispatch gate). Watchdog bridge cancels hung workers via synthetic failure events (post-dispatch intervention). Both reuse existing infrastructure — PlanItem state for budget counting, `WorkflowExecutionCompleted` event for watchdog intervention.

**Tech Stack:** Java 21+, Quarkus 3.32.2, CDI (`@DefaultBean`, `@ObservesAsync`), Vert.x EventBus

## Global Constraints

- Pre-release platform — breaking changes cost nothing
- `@DefaultBean @ApplicationScoped` pattern for all engine SPIs with no-op defaults
- `CaseDefinition` fields are nullable for backward compat (null = disabled/unlimited)
- YAML parsing via `CaseDefinitionYamlMapper`; schema changes in `schema/src/main/resources/schema/CaseDefinition.yaml`
- Tests: `@QuarkusTest` named `*Test.java` (never `*IT.java`), use `casehub-persistence-memory`
- All commits reference an issue: `Refs engine#N` or `Refs qhorus#N`

---

## Batch 1: DispatchBudget SPI + Case-Level Cap (engine#1043)

### Task 1: DispatchBudget SPI and default bean

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/DispatchBudget.java`
- Create: `api/src/main/java/io/casehub/api/spi/DispatchBudgetQuery.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpDispatchBudget.java`
- Test: `api/src/test/java/io/casehub/api/spi/DispatchBudgetTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java` (modify — add NoOp assertion)

**Interfaces:**
- Produces: `DispatchBudget.availableCapacity(DispatchBudgetQuery) → int`, `DispatchBudgetQuery(UUID caseId, String tenancyId)`

- [ ] **Step 1: Write the SPI contract test**

```java
// api/src/test/java/io/casehub/api/spi/DispatchBudgetTest.java
package io.casehub.api.spi;

import static org.assertj.core.api.Assertions.assertThat;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DispatchBudgetTest {

  @Test
  void query_carries_caseId_and_tenancyId() {
    var query = new DispatchBudgetQuery(UUID.randomUUID(), "tenant-1");
    assertThat(query.caseId()).isNotNull();
    assertThat(query.tenancyId()).isEqualTo("tenant-1");
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=DispatchBudgetTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: compilation failure — `DispatchBudget` and `DispatchBudgetQuery` not found

- [ ] **Step 3: Create the SPI interface and query record**

```java
// api/src/main/java/io/casehub/api/spi/DispatchBudget.java
package io.casehub.api.spi;

public interface DispatchBudget {
  int availableCapacity(DispatchBudgetQuery query);
}
```

```java
// api/src/main/java/io/casehub/api/spi/DispatchBudgetQuery.java
package io.casehub.api.spi;

import java.util.UUID;

public record DispatchBudgetQuery(UUID caseId, String tenancyId) {}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=DispatchBudgetTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Create the @DefaultBean no-op implementation**

```java
// runtime/src/main/java/io/casehub/engine/internal/worker/NoOpDispatchBudget.java
package io.casehub.engine.internal.worker;

import io.casehub.api.spi.DispatchBudget;
import io.casehub.api.spi.DispatchBudgetQuery;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpDispatchBudget implements DispatchBudget {
  @Override
  public int availableCapacity(DispatchBudgetQuery query) {
    return Integer.MAX_VALUE;
  }
}
```

- [ ] **Step 6: Add NoOp assertion to DefaultWorkerSpiImplementationsTest**

Add a test method asserting `NoOpDispatchBudget.availableCapacity()` returns `Integer.MAX_VALUE`.

- [ ] **Step 7: Run tests and verify**

Run: `mvn test -pl api,runtime -Dtest=DispatchBudgetTest,DefaultWorkerSpiImplementationsTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add api/src/main/java/io/casehub/api/spi/DispatchBudget.java api/src/main/java/io/casehub/api/spi/DispatchBudgetQuery.java runtime/src/main/java/io/casehub/engine/internal/worker/NoOpDispatchBudget.java api/src/test/java/io/casehub/api/spi/DispatchBudgetTest.java runtime/src/test/java/io/casehub/engine/internal/worker/DefaultWorkerSpiImplementationsTest.java
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "feat(#1043): DispatchBudget SPI and NoOp default bean Refs engine#1043"
```

### Task 2: CaseDefinition.maxConcurrentDispatches + YAML parsing

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add field, getter, setter, Builder method)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (parse `maxConcurrentDispatches` from YAML)
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` (add property)
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperConcurrencyTest.java` (create)

**Interfaces:**
- Consumes: `CaseDefinition` class structure (Task 1 not required — independent)
- Produces: `CaseDefinition.getMaxConcurrentDispatches() → Integer` (nullable), `Builder.maxConcurrentDispatches(Integer)`

- [ ] **Step 1: Write YAML round-trip test**

```java
// api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperConcurrencyTest.java
package io.casehub.api.model.converter;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.api.model.CaseDefinition;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperConcurrencyTest {

  @Test
  void maxConcurrentDispatches_parsed_from_yaml() {
    String yaml = """
        casehub: 1.0
        name: throttled-case
        namespace: test
        spec:
          maxConcurrentDispatches: 5
          capabilities:
            - name: doWork
          workers:
            - name: worker-1
              capabilities: [doWork]
          bindings:
            - capability: doWork
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getMaxConcurrentDispatches()).isEqualTo(5);
  }

  @Test
  void maxConcurrentDispatches_null_when_absent() {
    String yaml = """
        casehub: 1.0
        name: unlimited-case
        namespace: test
        spec:
          capabilities:
            - name: doWork
          workers:
            - name: worker-1
              capabilities: [doWork]
          bindings:
            - capability: doWork
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getMaxConcurrentDispatches()).isNull();
  }

  @Test
  void maxConcurrentDispatches_via_builder() {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("throttled").version("1.0")
        .maxConcurrentDispatches(3)
        .build();
    assertThat(def.getMaxConcurrentDispatches()).isEqualTo(3);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperConcurrencyTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: compilation failure — `getMaxConcurrentDispatches()` not found

- [ ] **Step 3: Add field, getter, setter, Builder method to CaseDefinition**

Use `ide_insert_member` to add to `CaseDefinition.java`:
- Field: `private Integer maxConcurrentDispatches;` (after `recoveryPolicy` field, ~line 207)
- Getter: `public Integer getMaxConcurrentDispatches()`
- Setter: `public void setMaxConcurrentDispatches(Integer maxConcurrentDispatches)`
- Builder field: `private Integer maxConcurrentDispatches;`
- Builder method: `public Builder maxConcurrentDispatches(Integer v) { this.maxConcurrentDispatches = v; return this; }`
- Builder.build(): add `def.setMaxConcurrentDispatches(this.maxConcurrentDispatches);`

- [ ] **Step 4: Add YAML parsing in CaseDefinitionYamlMapper**

In `CaseDefinitionYamlMapper`, find where other `spec:` block fields are parsed (e.g., `maxDecompositionDepth`, `maxAdaptations`). Add:
```java
if (specNode.has("maxConcurrentDispatches")) {
  definition.setMaxConcurrentDispatches(specNode.get("maxConcurrentDispatches").asInt());
}
```

- [ ] **Step 5: Add property to CaseDefinition.yaml schema**

Add `maxConcurrentDispatches` property to the `spec` section of the YAML schema:
```yaml
maxConcurrentDispatches:
  type: integer
  description: Maximum concurrent binding dispatches per case. Null means unlimited.
```

- [ ] **Step 6: Run tests and verify**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperConcurrencyTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add api/src/ schema/src/ 
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "feat(#1043): CaseDefinition.maxConcurrentDispatches field + YAML parsing Refs engine#1043"
```

### Task 3: Case-level throttle + external budget integration in CaseContextChangedEventHandler

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` (inject `DispatchBudget`, add throttle logic in `rules()`)
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandlerConcurrencyTest.java` (create)

**Interfaces:**
- Consumes: `DispatchBudget.availableCapacity(DispatchBudgetQuery)` (Task 1), `CaseDefinition.getMaxConcurrentDispatches()` (Task 2), `BlackboardRegistry` for PlanItem counting
- Produces: throttled dispatch — `rules()` dispatches at most `min(caseBudget, externalBudget)` bindings

- [ ] **Step 1: Write failing test — case-level cap limits dispatch count**

Create `CaseContextChangedEventHandlerConcurrencyTest.java` as a `@QuarkusTest`. Define a `CaseHub` subclass with `maxConcurrentDispatches(2)` and 5 eligible bindings. Assert only 2 are dispatched (verify via `WorkerExecutionManager` or recording spy).

The test should:
1. Create a case definition with `maxConcurrentDispatches(2)` and 5 capability bindings
2. Start a case and signal context to trigger all 5 bindings
3. Assert only 2 worker schedule events are published

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl runtime -Dtest=CaseContextChangedEventHandlerConcurrencyTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: FAIL — all 5 dispatched (no throttle)

- [ ] **Step 3: Implement the throttle in rules()**

In `CaseContextChangedEventHandler`:
1. Inject `DispatchBudget dispatchBudget`
2. After `List<Binding> selected = loopControl.select(planCtx, eligible);`, add:

```java
selected = applyDispatchBudget(caseInstance, definition, selected);
```

3. Add the method:

```java
private List<Binding> applyDispatchBudget(CaseInstance caseInstance, CaseDefinition definition, List<Binding> selected) {
    if (selected.isEmpty()) return selected;
    
    int caseBudget = Integer.MAX_VALUE;
    if (definition.getMaxConcurrentDispatches() != null) {
        int active = countResourceConsumingPlanItems(caseInstance.getUuid());
        caseBudget = definition.getMaxConcurrentDispatches() - active;
        if (caseBudget <= 0) return List.of();
    }
    
    int externalBudget = dispatchBudget.availableCapacity(
        new DispatchBudgetQuery(caseInstance.getUuid(), caseInstance.tenancyId));
    
    int admitted = Math.min(selected.size(), Math.min(caseBudget, externalBudget));
    return admitted >= selected.size() ? selected : selected.subList(0, admitted);
}

private int countResourceConsumingPlanItems(UUID caseId) {
    // Query BlackboardRegistry for PlanItems in RUNNING, DISPATCHING, DELEGATED states
    var planModel = loopControl.getPlanModel(caseId);
    if (planModel == null) return 0;
    return (int) planModel.getAllPlanItems().stream()
        .filter(pi -> {
            var status = pi.status();
            return status == TaskStatus.RUNNING 
                || status == TaskStatus.DISPATCHING 
                || status == TaskStatus.DELEGATED;
        })
        .count();
}
```

Note: The exact method to get PlanItems from `BlackboardRegistry` / `LoopControl` / `CasePlanModel` may differ — check `loopControl` or inject `BlackboardRegistry` and call `get(caseId).getAllPlanItems()`. Use `ide_find_references` on `getAllPlanItems` to find the pattern.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl runtime -Dtest=CaseContextChangedEventHandlerConcurrencyTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write additional tests**

Add tests for:
- `maxConcurrentDispatches` null → unlimited (all bindings dispatched)
- Active PlanItems at max → zero dispatched
- External `DispatchBudget` returns lower than case budget → external wins
- Worker completes → next CONTEXT_CHANGED admits deferred binding (integration-level)

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl runtime -Dtest=CaseContextChangedEventHandlerConcurrencyTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add runtime/src/
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "feat(#1043): case-level dispatch throttle + external budget integration Refs engine#1043"
```

---

## Batch 2: Qhorus WatchdogAlertEvent enrichment (qhorus#433)

### Task 4: Add caseId to WatchdogAlertEvent + resolve in WatchdogEvaluationService

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/qhorus/api/src/main/java/io/casehub/qhorus/api/watchdog/WatchdogAlertEvent.java`
- Modify: `/Users/mdproctor/claude/casehub/qhorus/runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Test: existing qhorus tests (verify backward compat)

**Interfaces:**
- Produces: `WatchdogAlertEvent.caseId()` — `@Nullable UUID`, resolved from channel metadata

- [ ] **Step 1: Add caseId to WatchdogAlertEvent record**

Add 7th component `@Nullable UUID caseId`. Add backward-compatible 6-arg constructor delegating with null:

```java
public record WatchdogAlertEvent(
    UUID watchdogId,
    String targetName,
    String notificationChannel,
    String summary,
    Instant firedAt,
    AlertContext context,
    @jakarta.annotation.Nullable UUID caseId
) {
    public WatchdogAlertEvent(UUID watchdogId, String targetName, String notificationChannel,
                               String summary, Instant firedAt, AlertContext context) {
        this(watchdogId, targetName, notificationChannel, summary, firedAt, context, null);
    }

    public WatchdogConditionType conditionType() {
        return this.context.conditionType();
    }
}
```

- [ ] **Step 2: Thread caseId through WatchdogEvaluationService.fireAlert()**

In `WatchdogEvaluationService`, the `fireAlert()` method needs to resolve caseId from the channel. Channels are resolved per-evaluation method. The channel's metadata (set by engine's `CaseChannelProvider.openChannel()`) may carry `caseId` as a custom attribute. For per-channel evaluations, pass the channel's caseId metadata. For cross-channel evaluations (target="*"), pass null.

Update `fireAlert()` to accept `@Nullable UUID caseId`:
```java
void fireAlert(Watchdog w, String summary, AlertContext context, Instant now, UUID channelId, UUID caseId) {
    this.alertEvents.fireAsync(new WatchdogAlertEvent(w.id(), w.targetName(), w.notificationChannel(), summary, now, context, caseId));
    // ... rest unchanged
}
```

Each evaluation method that has a `Channel ch` extracts caseId from channel metadata and passes it. The exact metadata key depends on how `CaseChannelProvider` stores it — check with `ide_find_references` on `openChannel` to find the metadata format.

- [ ] **Step 3: Run qhorus tests**

Run: `mvn test -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`
Expected: PASS (backward-compatible change)

- [ ] **Step 4: Install qhorus SNAPSHOT**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/qhorus/pom.xml`

- [ ] **Step 5: Commit in qhorus repo**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add api/src/ runtime/src/
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(#433): add caseId to WatchdogAlertEvent for engine recovery bridge Refs qhorus#433"
```

---

## Batch 3: WatchdogRecoveryBridge (engine#1044)

### Task 5: WatchdogAction enum + CaseDefinition.watchdogPolicy

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/WatchdogResponseAction.java` (enum — named to avoid clash with qhorus's `WatchdogAction`)
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` (add `watchdogPolicy` field)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (parse `watchdogPolicy`)
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` (add property)
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperWatchdogTest.java` (create)

**Interfaces:**
- Produces: `WatchdogResponseAction` enum (`CANCEL_AFFECTED`, `IGNORE`), `CaseDefinition.getWatchdogPolicy() → Map<WatchdogConditionType, WatchdogResponseAction>`

- [ ] **Step 1: Write YAML round-trip test**

```java
// api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperWatchdogTest.java
package io.casehub.api.model.converter;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.WatchdogResponseAction;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import org.junit.jupiter.api.Test;

class CaseDefinitionYamlMapperWatchdogTest {

  @Test
  void watchdogPolicy_parsed_from_yaml() {
    String yaml = """
        casehub: 1.0
        name: watchdog-case
        namespace: test
        spec:
          watchdogPolicy:
            AGENT_STALE: CANCEL_AFFECTED
            LOOP_DETECTED: IGNORE
          capabilities:
            - name: doWork
          workers:
            - name: worker-1
              capabilities: [doWork]
          bindings:
            - capability: doWork
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getWatchdogPolicy())
        .containsEntry(WatchdogConditionType.AGENT_STALE, WatchdogResponseAction.CANCEL_AFFECTED)
        .containsEntry(WatchdogConditionType.LOOP_DETECTED, WatchdogResponseAction.IGNORE);
  }

  @Test
  void watchdogPolicy_null_when_absent() {
    String yaml = """
        casehub: 1.0
        name: no-policy
        namespace: test
        spec:
          capabilities:
            - name: doWork
          workers:
            - name: worker-1
              capabilities: [doWork]
          bindings:
            - capability: doWork
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    assertThat(def.getWatchdogPolicy()).isNull();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation failure — `WatchdogResponseAction` and `getWatchdogPolicy()` not found

- [ ] **Step 3: Create WatchdogResponseAction enum**

```java
// api/src/main/java/io/casehub/api/model/WatchdogResponseAction.java
package io.casehub.api.model;

public enum WatchdogResponseAction {
  CANCEL_AFFECTED,
  IGNORE
}
```

- [ ] **Step 4: Add watchdogPolicy to CaseDefinition + Builder + YAML**

Add field `private Map<WatchdogConditionType, WatchdogResponseAction> watchdogPolicy;`, getter, setter, and Builder method to `CaseDefinition.java`. Add YAML parsing in `CaseDefinitionYamlMapper`. Add property to `CaseDefinition.yaml` schema.

- [ ] **Step 5: Run tests and verify**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperWatchdogTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add api/src/ schema/src/
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "feat(#1044): WatchdogResponseAction enum + CaseDefinition.watchdogPolicy Refs engine#1044"
```

### Task 6: WatchdogRecoveryBridge — core bridge with synthetic Expired

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/bridge/WatchdogRecoveryBridge.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/bridge/WatchdogRecoveryBridgeTest.java`

**Interfaces:**
- Consumes: `WatchdogAlertEvent` (CDI event from qhorus, enriched with caseId from Task 4), `CaseDefinition.getWatchdogPolicy()` (Task 5), `CaseInstanceCache`, `CaseDefinitionRegistry`, `PlanItemStore`, `EventBus`
- Produces: publishes `WorkflowExecutionCompleted(Expired("Watchdog: <condition>"))` on `EventBusAddresses.WORKER_EXECUTION_FINISHED` for matched PlanItems

- [ ] **Step 1: Write failing test — AGENT_STALE triggers synthetic Expired**

```java
// runtime/src/test/java/io/casehub/engine/internal/bridge/WatchdogRecoveryBridgeTest.java
package io.casehub.engine.internal.bridge;

import static org.assertj.core.api.Assertions.assertThat;
// ... imports for WatchdogAlertEvent, AlertContext, mocks, etc.

class WatchdogRecoveryBridgeTest {

  // Unit test with mocked dependencies
  @Test
  void agentStale_publishes_synthetic_expired_for_matched_planItem() {
    // Setup: mock CaseInstanceCache, CaseDefinitionRegistry, PlanItemStore
    // Create a WatchdogAlertEvent with AGENT_STALE, caseId, affectedAgentIds=["worker-1"]
    // Mock PlanItemStore to return a RUNNING PlanItem with executorName="worker-1"
    // Call bridge.onWatchdogAlert(event)
    // Verify eventBus.publish(WORKER_EXECUTION_FINISHED, ...) called with Expired outcome
  }

  @Test
  void null_caseId_logs_warning_no_action() {
    // Setup: WatchdogAlertEvent with null caseId
    // Call bridge.onWatchdogAlert(event)
    // Verify no eventBus.publish() called
  }

  @Test
  void no_matching_planItem_logs_warning() {
    // Setup: WatchdogAlertEvent with caseId but affectedAgentIds=["unknown-worker"]
    // Mock PlanItemStore returns no match
    // Call bridge.onWatchdogAlert(event)
    // Verify no eventBus.publish() called
  }

  @Test
  void ignore_policy_skips_cancellation() {
    // Setup: CaseDefinition with watchdogPolicy LOOP_DETECTED=IGNORE
    // WatchdogAlertEvent with LOOP_DETECTED
    // Call bridge.onWatchdogAlert(event)
    // Verify no eventBus.publish() called
  }

  @Test
  void terminal_planItem_skipped() {
    // Setup: PlanItem with COMPLETED status
    // Call bridge.onWatchdogAlert(event)
    // Verify no eventBus.publish() called
  }

  @Test
  void case_level_condition_ignored() {
    // Setup: WatchdogAlertEvent with CONTEXT_PRESSURE (case-level, not worker-hung)
    // Call bridge.onWatchdogAlert(event)
    // Verify no eventBus.publish() called (default IGNORE for case-level)
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation failure — `WatchdogRecoveryBridge` not found

- [ ] **Step 3: Implement WatchdogRecoveryBridge**

```java
// runtime/src/main/java/io/casehub/engine/internal/bridge/WatchdogRecoveryBridge.java
package io.casehub.engine.internal.bridge;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.WatchdogResponseAction;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.casehub.engine.common.internal.event.WorkflowExecutionCompleted;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.internal.engine.CaseInstanceCache;
import io.casehub.engine.spi.CaseDefinitionRegistry;
import io.casehub.engine.spi.PlanItemStore;
import io.casehub.api.model.TaskStatus;
import io.casehub.qhorus.api.watchdog.WatchdogAlertEvent;
import io.casehub.qhorus.api.watchdog.WatchdogConditionType;
import io.casehub.worker.api.WorkerOutcome;
import io.vertx.core.eventbus.EventBus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.util.*;
import org.jboss.logging.Logger;

@ApplicationScoped
public class WatchdogRecoveryBridge {

  private static final Logger LOG = Logger.getLogger(WatchdogRecoveryBridge.class);

  private static final Set<WatchdogConditionType> WORKER_HUNG_CONDITIONS = Set.of(
      WatchdogConditionType.AGENT_STALE,
      WatchdogConditionType.BARRIER_STUCK,
      WatchdogConditionType.LOOP_DETECTED,
      WatchdogConditionType.CONVERSATION_STALL,
      WatchdogConditionType.ECHO_CHAMBER,
      WatchdogConditionType.CIRCULAR_DELEGATION);

  @Inject CaseInstanceCache caseInstanceCache;
  @Inject CaseDefinitionRegistry definitionRegistry;
  @Inject PlanItemStore planItemStore;
  @Inject EventBus eventBus;

  void onWatchdogAlert(@ObservesAsync WatchdogAlertEvent event) {
    if (event.caseId() == null) {
      LOG.debugf("Watchdog alert %s has no caseId — skipping engine recovery", event.conditionType());
      return;
    }

    WatchdogResponseAction action = resolveAction(event);
    if (action == WatchdogResponseAction.IGNORE) {
      LOG.debugf("Watchdog %s on case %s — policy IGNORE", event.conditionType(), event.caseId());
      return;
    }

    CaseInstance instance = caseInstanceCache.get(event.caseId());
    if (instance == null) {
      LOG.warnf("Watchdog alert for unknown case %s — skipping", event.caseId());
      return;
    }

    CaseDefinition definition = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
    if (definition == null) return;

    List<String> agentIds = event.context().affectedAgentIds();
    if (agentIds.isEmpty()) {
      LOG.debugf("Watchdog %s on case %s — no affected agents", event.conditionType(), event.caseId());
      return;
    }

    // Find active PlanItems matching affected agents
    // ... implementation resolves PlanItems and publishes synthetic Expired
    // See spec for exact resolution logic
  }

  private WatchdogResponseAction resolveAction(WatchdogAlertEvent event) {
    CaseInstance instance = caseInstanceCache.get(event.caseId());
    if (instance == null) return WatchdogResponseAction.IGNORE;

    CaseDefinition def = definitionRegistry.getCaseDefinition(instance.getCaseMetaModel());
    if (def == null) return WatchdogResponseAction.IGNORE;

    Map<WatchdogConditionType, WatchdogResponseAction> policy = def.getWatchdogPolicy();
    if (policy != null && policy.containsKey(event.conditionType())) {
      return policy.get(event.conditionType());
    }

    return WORKER_HUNG_CONDITIONS.contains(event.conditionType())
        ? WatchdogResponseAction.CANCEL_AFFECTED
        : WatchdogResponseAction.IGNORE;
  }
}
```

The full implementation of PlanItem matching and synthetic event publishing follows the pattern from `QhorusMessageSignalBridge.handleWorkerOutcome()`. The implementer should:
1. Query `PlanItemStore.findByCase(caseId, tenancyId)` for active PlanItems
2. Filter to non-terminal PlanItems where `executorName()` matches any `agentId`
3. For each match, find the corresponding `Worker` and `Binding` from `CaseDefinition`
4. Build and publish `WorkflowExecutionCompleted` with `WorkerOutcome.expired("Watchdog: " + conditionType)`

- [ ] **Step 4: Run tests and verify**

Run: `mvn test -pl runtime -Dtest=WatchdogRecoveryBridgeTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write integration test — synthetic Expired flows through failure pipeline**

Create a `@QuarkusTest` that:
1. Starts a case with a worker
2. Simulates a `WatchdogAlertEvent` via `Event<WatchdogAlertEvent>.fireAsync()`
3. Verifies the worker's PlanItem transitions to FAULTED
4. Verifies `_diagnostics` failure state is written

- [ ] **Step 6: Run integration test**

Run: `mvn test -pl runtime -Dtest=WatchdogRecoveryBridgeIntegrationTest -f /Users/mdproctor/claude/casehub/slots/174/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add runtime/src/
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "feat(#1044): WatchdogRecoveryBridge — CDI observer with synthetic Expired Refs engine#1044"
```

---

## Batch 4: Documentation + CLAUDE.md update

### Task 7: Update CLAUDE.md and contributor guide

**Files:**
- Modify: `CLAUDE.md` (add Concurrency Budget and Watchdog Bridge sections)
- Modify: `docs/guides/contributor-guide.md` (add concurrency budget SPI docs)

**Interfaces:**
- Consumes: all prior tasks

- [ ] **Step 1: Add Concurrency Budget section to CLAUDE.md**

Document: `DispatchBudget` SPI, `DispatchBudgetQuery`, `NoOpDispatchBudget`, `CaseDefinition.maxConcurrentDispatches`, integration point in `rules()`.

- [ ] **Step 2: Add Watchdog Recovery Bridge section to CLAUDE.md**

Document: `WatchdogRecoveryBridge`, `WatchdogResponseAction`, `CaseDefinition.watchdogPolicy`, supported conditions, synthetic Expired pattern, identity resolution.

- [ ] **Step 3: Update contributor guide**

Add `DispatchBudget` to the SPI table. Add watchdog bridge to the handler wiring table.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/174/engine add CLAUDE.md docs/
git -C /Users/mdproctor/claude/casehub/slots/174/engine commit -m "docs: add concurrency budget and watchdog bridge documentation Refs engine#1043 Refs engine#1044"
```

---

## References

- [2026-09-07-concurrency-throttle-watchdog-recovery-design.md] — design spec
- `CaseContextChangedEventHandler.rules()` (runtime) — dispatch fan-out integration point
- `CaseEvaluationSerializer` (runtime) — per-case serialization guaranteeing no TOCTOU
- `QhorusMessageSignalBridge.handleWorkerOutcome()` (runtime) — established synthetic failure pattern
- `WorkflowExecutionCompleted` (common/internal/event/) — event record for synthetic Expired
- `NoOpWorkerProvisioner` (runtime) — `@DefaultBean` pattern reference
- `CaseDefinition` (api/model/) — field/Builder/setter pattern reference
- `WatchdogAlertEvent` / `AlertContext` (qhorus-api) — watchdog event types
- `WatchdogEvaluationService` (qhorus runtime) — source of CDI events
- engine#1043, engine#1044, qhorus#433 — tracked issues
- engine#1057 — deferred: automated case-level watchdog response
- engine#1066 — deferred: watchdog notification migration
- parent#471 — deferred: platform notification service re-architecture
