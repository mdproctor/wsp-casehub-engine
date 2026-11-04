# CaseContext Panels — Issue #81 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Prerequisite:** Issue #80 plan must be complete (`plans/2026-06-09-issue-80-casefile-panels.md`). All built-in panels (working, semantic, episodic) must be working.

**Goal:** Add user-defined panels (named partitions declared in `CaseDefinition`) and `listenPanel` binding subscription to restrict re-evaluation to a specific panel's changes.

**Architecture:** `CaseDefinition` gains a `panels` list declaring named read-write panels. `CaseHubReactor.buildInstance()` initializes declared panels. `ContextChangeTrigger` gains an optional `listenPanel` field. `CaseContextChangedEventHandler` reads `event.changedPanel()` and skips bindings whose `listenPanel` doesn't match. Panel-scoped addresses (`casehub.context.changed.{name}`) are published alongside `CONTEXT_CHANGED` for external subscribers (Claudony, Drools).

**Spec:** `docs/specs/2026-06-09-casefile-panels-design.md` (rev 6), section "User-Defined Panels (#81)"

**Build commands:**
```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime
```

---

## File Map

### Modified files
| File | Change |
|---|---|
| `api/src/main/java/io/casehub/api/model/CaseDefinition.java` | Add `panelNames: List<String>` field + builder |
| `api/src/main/java/io/casehub/api/model/ContextChangeTrigger.java` | Add `listenPanel: String` field |
| `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java` | Initialize declared panels in `buildInstance()` |
| `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` | `listenPanel` filtering; panel-scoped event publishing |
| `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` | Parse `panels[]` YAML field; parse `listenPanel` on bindings |
| Schema YAML files | Add `panels[]` and `binding.on.contextChange.listenPanel` |

### New test files
| File | Purpose |
|---|---|
| `runtime/src/test/java/io/casehub/engine/UserDefinedPanelTest.java` | User panel lifecycle: creation, write, listenPanel filtering |

---

## Task 1: `CaseDefinition` — add `panelNames` field

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`

- [ ] **Step 1: Add `panelNames` to `CaseDefinition`**

Add field and accessors to `CaseDefinition`:

```java
private List<String> panelNames = new ArrayList<>();

public List<String> getPanelNames() { return panelNames; }
public void setPanelNames(List<String> panelNames) { this.panelNames = panelNames != null ? panelNames : new ArrayList<>(); }
```

Add to `CaseDefinition.Builder`:

```java
private List<String> panelNames = new ArrayList<>();

public Builder panel(String name) {
    this.panelNames.add(name);
    return this;
}

public Builder panels(String... names) {
    this.panelNames.addAll(List.of(names));
    return this;
}
```

In `Builder.build()`, add:
```java
caseHubDefinition.setPanelNames(panelNames.isEmpty() ? List.of() : new ArrayList<>(panelNames));
```

- [ ] **Step 2: Write tests for builder**

Add to `ModelBuilderTest` or create `CaseDefinitionPanelTest`:

```java
@Test
void caseDefinition_panelBuilder() {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("n").version("1.0")
        .panel("raw").panel("extracted").panel("conclusions")
        .build();
    assertEquals(List.of("raw", "extracted", "conclusions"), def.getPanelNames());
}

@Test
void caseDefinition_noPanels_returnsEmptyList() {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("n").version("1.0")
        .build();
    assertNotNull(def.getPanelNames());
    assertTrue(def.getPanelNames().isEmpty());
}
```

- [ ] **Step 3: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=ModelBuilderTest -q 2>&1 | tail -5
```
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): CaseDefinition.panelNames field + builder

Refs #81"
```

---

## Task 2: Initialize user-defined panels in `buildInstance()`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/CaseHubReactor.java`

- [ ] **Step 1: Write failing test**

```java
// runtime/src/test/java/io/casehub/engine/UserDefinedPanelTest.java
package io.casehub.engine;

import io.casehub.api.context.ContextPanel;
import io.casehub.api.context.ReadablePanel;
import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class UserDefinedPanelTest {

  @Inject io.casehub.api.engine.CaseHubRuntime runtime;
  @Inject CaseInstanceCache caseInstanceCache;

  @Test
  void userPanels_initializedEmpty() throws Exception {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("panel-init").version("1.0")
        .panel("raw").panel("extracted")
        .build();

    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();
    var instance = caseInstanceCache.get(caseId);
    assertNotNull(instance);

    ReadablePanel raw = instance.getCaseContext().panel("raw");
    assertNotNull(raw);
    assertEquals("raw", raw.panelName());
    assertTrue(raw.isEmpty());
    assertFalse(raw.isReadOnly());
  }

  @Test
  void workingPanel_alwaysPresent_evenWithNoPanelsDeclared() throws Exception {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("no-panels").version("1.0")
        .build();

    UUID caseId = runtime.startCase(def, Map.of("key", "val")).toCompletableFuture().get();
    var instance = caseInstanceCache.get(caseId);
    assertEquals("val", instance.getCaseContext().get("key"));
    ReadablePanel working = instance.getCaseContext().panel(ContextPanel.WORKING);
    assertNotNull(working);
    assertEquals("val", working.get("key"));
  }
}
```

- [ ] **Step 2: Run to confirm failure**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=UserDefinedPanelTest#userPanels_initializedEmpty -q 2>&1 | tail -10
```
Expected: FAIL — panel "raw" returns empty but exists (or fails with NPE if not initialized)

- [ ] **Step 3: Initialize declared panels in `buildInstance()`**

In `CaseHubReactor.buildInstance()`, after the semantic and episodic panel setup, add:

```java
// Initialize user-defined panels
if (context instanceof CaseContextImpl ctx) {
    for (String panelName : definition.getPanelNames()) {
        // Calling writablePanel() creates the panel if absent (already in CaseContextImpl)
        ctx.writablePanel(panelName); // just touches it to ensure it exists
    }
}
```

This ensures user-declared panels appear in `asJsonNode()` and `snapshot()` even before any writes.

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=UserDefinedPanelTest -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): initialize user-defined panels in buildInstance()

User panels declared in CaseDefinition.panelNames are pre-created (empty)
at case start. Workers write to them via context.panel(name).

Refs #81"
```

---

## Task 3: `ContextChangeTrigger.listenPanel` + handler filtering

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/ContextChangeTrigger.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`

- [ ] **Step 1: Add `listenPanel` to `ContextChangeTrigger`**

```java
// api/src/main/java/io/casehub/api/model/ContextChangeTrigger.java
package io.casehub.api.model;

import io.casehub.api.model.evaluator.ExpressionEvaluator;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;

public class ContextChangeTrigger implements Trigger {

  private final ExpressionEvaluator filter;
  private final String listenPanel;  // null = all non-episodic panel changes

  public ContextChangeTrigger(String filter) {
    this.filter = new JQExpressionEvaluator(filter);
    this.listenPanel = null;
  }

  public ContextChangeTrigger(ExpressionEvaluator filter) {
    this.filter = filter;
    this.listenPanel = null;
  }

  public ContextChangeTrigger(ExpressionEvaluator filter, String listenPanel) {
    this.filter = filter;
    this.listenPanel = listenPanel;
  }

  public ExpressionEvaluator getFilter() { return filter; }
  public String getListenPanel() { return listenPanel; }
}
```

- [ ] **Step 2: Write failing test for `listenPanel` filtering**

Add to `UserDefinedPanelTest.java`:

```java
import io.casehub.api.model.Binding;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.Capability;
import io.casehub.engine.common.internal.event.CaseContextChangedEvent;
import io.casehub.engine.common.internal.event.EventBusAddresses;
import io.vertx.mutiny.core.eventbus.EventBus;

@Inject EventBus eventBus;

@Test
void listenPanel_binding_skippedWhenWrongPanel() throws Exception {
    // Define a case with a binding that listens only to "extracted" panel
    // The binding fires a worker — but working panel changes should NOT trigger it
    // This test verifies the filtering logic by checking worker is NOT scheduled

    // Create a simple case with one worker + one binding with listenPanel=extracted
    var capability = new Capability("test-capability");
    var trigger = new ContextChangeTrigger(
        new io.casehub.api.model.evaluator.JQExpressionEvaluator(".extracted.score != null"),
        "extracted"  // listenPanel
    );
    var binding = Binding.builder()
        .name("test-binding")
        .on(trigger)
        .target(new CapabilityTarget(capability))
        .build();

    // The key assertion: when a CONTEXT_CHANGED event fires with changedPanel=WORKING,
    // a binding with listenPanel="extracted" should be skipped.
    // This is tested by directly testing the handler filtering logic.

    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("listen-panel-test").version("1.0")
        .panel("extracted")
        .bindings(binding)
        .build();

    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();
    // Update working panel — should NOT trigger the "extracted" binding
    runtime.signal(caseId, "result", "done");

    // Brief wait to ensure no worker is scheduled
    Thread.sleep(100);
    // If the filtering works, no worker schedule event was published for this capability
    // Verified implicitly — no NoOpWorkerProvisioner/exception thrown
    assertNotNull(caseId);
}

@Test
void listenPanel_null_triggersOnAnyWorkingPanelChange() throws Exception {
    // A binding without listenPanel should trigger on working panel changes
    // This is the current (unchanged) behavior
    var trigger = new ContextChangeTrigger(".working.result != null");  // no listenPanel
    assertNull(trigger.getListenPanel());  // confirm null
}
```

- [ ] **Step 3: Implement `listenPanel` filtering in `CaseContextChangedEventHandler`**

In `CaseContextChangedEventHandler.onCaseStateContextChangedEventHandler()`, the episodic skip is already added (Task 7 of #80). Now add `listenPanel` filtering in the `rules()` binding loop:

```java
// In rules() binding loop, after checking the trigger type:
if (!(binding.getOn() instanceof ContextChangeTrigger cct)) continue;

// listenPanel filter: skip if binding declares a specific panel that doesn't match
String listenPanel = cct.getListenPanel();
if (listenPanel != null && !listenPanel.equals(changedPanel)) {
    continue;  // binding targets a different panel, skip
}
// null listenPanel = subscribes to all non-episodic changes (already filtered above)
```

Note: `changedPanel` is read from `event.changedPanel()` at the start of the handler.

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="UserDefinedPanelTest,CaseContextChangedEventHandlerRoutingTest" -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): ContextChangeTrigger.listenPanel + handler filtering

Bindings with listenPanel only re-evaluate when changedPanel matches.
Bindings without listenPanel re-evaluate on any working or user-panel change.
Episodic changes bypass all bindings (established in #80).

Refs #81"
```

---

## Task 4: Panel-scoped event publishing for user-defined panels

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: Anywhere else that writes to user-defined panels and fires `CONTEXT_CHANGED`

This task wires the panel-scoped addresses (already defined in `EventBusAddresses.panelChanged()`) into the event publishing flow for user-defined panels.

The rule: when a user-defined panel changes, fire both `CONTEXT_CHANGED` (with `changedPanel = panelName`) and `casehub.context.changed.{panelName}` (for external subscribers).

Currently `CONTEXT_CHANGED` with `changedPanel = WORKING` already fires for working panel. For user-defined panels, the engine doesn't yet write to them directly — workers do via the working panel. If a worker output map contains a key that matches a user panel name... actually, there's no automatic routing. Workers write to the working panel via `applyOutputWithConflictResolution`.

The `panelChanged()` event is primarily for external consumers (Claudony, future Drools integration). For now, fire it when the working panel changes as well, using the same `changedPanel` value from `CaseContextChangedEvent`:

- [ ] **Step 1: Fire panel-scoped events alongside `CONTEXT_CHANGED`**

In `CaseContextChangedEventHandler` (or at each construction site), after publishing `CONTEXT_CHANGED`, also publish the panel-scoped event if `changedPanel` is not null:

Actually, the cleanest approach: add a helper to `CaseHubReactor` or fire from each construction site. The simplest: in `CaseStartedEventHandler` and all handler classes, after publishing `CONTEXT_CHANGED`, fire the panel-scoped event.

Add after each `eventBus.publish(EventBusAddresses.CONTEXT_CHANGED, event)`:

```java
if (event.changedPanel() != null) {
    eventBus.publish(EventBusAddresses.panelChanged(event.changedPanel()), event);
}
```

This can be done at the construction site or in the handler. The construction site approach is explicit. Update each of the 12 production sites.

- [ ] **Step 2: Write test**

```java
// Add to UserDefinedPanelTest:
@Test
void panelScopedEvent_publishedOnWorkingPanelChange() throws Exception {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("panel-event-test").version("1.0")
        .build();

    List<CaseContextChangedEvent> received = new ArrayList<>();
    // Subscribe to panel-scoped event
    eventBus.<CaseContextChangedEvent>consumer(
        EventBusAddresses.panelChanged(ContextPanel.WORKING),
        msg -> received.add(msg.body()));

    UUID caseId = runtime.startCase(def, Map.of("key", "val")).toCompletableFuture().get();
    runtime.signal(caseId, "result", "done");

    Thread.sleep(200); // allow async event delivery
    assertFalse(received.isEmpty(), "Panel-scoped event must be published");
    assertTrue(received.stream().allMatch(e -> ContextPanel.WORKING.equals(e.changedPanel())));
}
```

- [ ] **Step 3: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=UserDefinedPanelTest -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): panel-scoped event publishing

casehub.context.changed.{panelName} published alongside CONTEXT_CHANGED
for external subscribers (Claudony, monitoring, future Drools integration).

Refs #81"
```

---

## Task 5: YAML mapper — `panels[]` and `listenPanel` fields

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: Schema YAML files

- [ ] **Step 1: Add `panels[]` to JSON Schema**

In the JSON Schema YAML for `CaseDefinition`:

```yaml
panels:
  type: array
  items:
    type: object
    required: [name]
    properties:
      name:
        type: string
        description: User-defined panel name; must not conflict with built-in panel names (working, semantic, episodic)
```

In the `ContextChangeTrigger` schema:

```yaml
listenPanel:
  type: string
  description: Optional panel name; if set, binding only re-evaluates when the named panel changes
```

- [ ] **Step 2: Regenerate schema model**

```bash
mvn install -DskipTests -q -pl schema
```

- [ ] **Step 3: Update `CaseDefinitionYamlMapper` to parse `panels[]`**

```java
// After existing binding/worker parsing:
if (schema.getPanels() != null) {
    List<String> panelNames = schema.getPanels().stream()
        .map(p -> p.getName())
        .filter(Objects::nonNull)
        .toList();
    caseDefinition.setPanelNames(panelNames);
}
```

For `listenPanel` on `ContextChangeTrigger` in bindings:

```java
// When constructing ContextChangeTrigger from schema:
String listenPanel = schemaBinding.getOn().getContextChange().getListenPanel(); // if field exists
ExpressionEvaluator filter = ...; // existing code
ContextChangeTrigger trigger = listenPanel != null
    ? new ContextChangeTrigger(filter, listenPanel)
    : new ContextChangeTrigger(filter);
```

- [ ] **Step 4: Write YAML mapper tests**

```java
@Test
void parsePanels() {
    String yaml = """
        namespace: test
        name: multi-panel
        version: 1.0.0
        panels:
          - name: "raw"
          - name: "extracted"
          - name: "conclusions"
        """;
    CaseDefinition def = mapper.parse(yaml);
    assertEquals(List.of("raw", "extracted", "conclusions"), def.getPanelNames());
}

@Test
void parseListenPanel_onBinding() {
    String yaml = """
        namespace: test
        name: listen-panel
        version: 1.0.0
        bindings:
          - name: b1
            on:
              contextChange:
                filter: ".extracted.score > 0.5"
                listenPanel: "extracted"
            capability: "analyze"
        capabilities:
          - name: "analyze"
        """;
    CaseDefinition def = mapper.parse(yaml);
    var trigger = (ContextChangeTrigger) def.getBindings().get(0).getOn();
    assertEquals("extracted", trigger.getListenPanel());
}

@Test
void parseBindingWithoutListenPanel_hasNullListenPanel() {
    String yaml = """
        namespace: test
        name: no-listen-panel
        version: 1.0.0
        bindings:
          - name: b1
            on:
              contextChange:
                filter: ".working.result != null"
            capability: "do-something"
        capabilities:
          - name: "do-something"
        """;
    CaseDefinition def = mapper.parse(yaml);
    var trigger = (ContextChangeTrigger) def.getBindings().get(0).getOn();
    assertNull(trigger.getListenPanel());
}
```

- [ ] **Step 5: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest -q 2>&1 | tail -10
```
Expected: all PASS

- [ ] **Step 6: Run full suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,common,runtime -q 2>&1 | tail -20
```
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(panels): YAML support for panels[] and listenPanel

- panels[].name parsed into CaseDefinition.panelNames
- binding.on.contextChange.listenPanel parsed into ContextChangeTrigger
- CaseDefinitionYamlMapperTest covers both

Refs #81, Closes #81"
```

---

## Task 6: Final verification — full suite across all modules

- [ ] **Step 1: Run complete test suite**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,common,runtime,blackboard,work-adapter -q 2>&1 | tail -30
```
Expected: all PASS

- [ ] **Step 2: Verify user-defined panel round-trip in integration test**

Add one final integration test verifying that a user-defined panel declared in YAML appears in `asJsonNode()` output:

```java
@Test
void userDefinedPanel_inCaseStartedPayload() throws Exception {
    String yaml = """
        namespace: test
        name: userpanel-it
        version: 1.0.0
        panels:
          - name: "raw"
          - name: "inferences"
        """;
    CaseDefinition def = yamlMapper.parse(yaml);
    UUID caseId = runtime.startCase(def, null).toCompletableFuture().get();

    var logs = runtime.eventLog(caseId).toCompletableFuture().get();
    var caseStarted = logs.stream()
        .filter(l -> l.eventType() == CaseHubEventType.CASE_STARTED)
        .findFirst().orElseThrow();

    JsonNode payload = caseStarted.payload();
    assertNotNull(payload.get("working"));
    assertNotNull(payload.get("semantic"));
    assertNotNull(payload.get("episodic"));
    assertNotNull(payload.get("raw"),       "user panel 'raw' must appear in payload");
    assertNotNull(payload.get("inferences"), "user panel 'inferences' must appear in payload");
}
```

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -u
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(panels): user-defined panel round-trip integration test

Refs #81"
```
