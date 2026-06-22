# Worker Primitives Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate 9 Worker type files from engine-api to casehub-worker-api and casehub-platform-api foundation dependencies.

**Architecture:** Delete engine-local type definitions (Worker, Capability, WorkerFunction, WorkerResult, WorkerOutcome, ExecutionPolicy, RetryPolicy, BackoffStrategy, PlannedAction). Replace with foundation-tier types from `casehub-worker-api` and `casehub-platform-api`. Create engine-owned WorkerFunction variants (AgentWorkerFunction, FlowWorkerFunction) and ClassificationContext. Move AgentDescriptor association from Worker to CaseDefinition.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven multi-module

**Spec:** `docs/specs/2026-06-22-worker-primitives-migration-design.md`

## Global Constraints

- All commits reference `Refs #543` or `Closes #543` (final commit)
- `mvn install -DskipTests -q` before any module-specific test run
- `TESTCONTAINERS_RYUK_DISABLED=true` for all test commands
- Records are final — Mockito cannot mock them. Tests must construct real objects.
- engine-api retains its existing `serverlessworkflow-experimental-fluent-func` dependency
- The YAML mapper (`CaseDefinitionYamlMapper`) is in engine-api and stays there
- `casehub-worker` is a peer repo — do not commit to it directly

---

## Prerequisite: casehub-worker-api Changes

**This must be completed before starting Task 1.** Provide the following briefing to the worker session:

---

**casehub-worker#TBD — Add PlannedAction + enriched WorkerResult factories**

Four changes to `casehub-worker-api` needed for engine#543 migration:

**1. Add `PlannedAction` record** at `api/src/main/java/io/casehub/worker/api/PlannedAction.java`:
```java
package io.casehub.worker.api;

import java.util.Map;
import java.util.Objects;

public record PlannedAction(String action, String actionType, Map<String, Object> parameters) {
    public PlannedAction {
        Objects.requireNonNull(action);
        Objects.requireNonNull(actionType);
        if (parameters == null) parameters = Map.of();
    }
    public static PlannedAction of(String action, String actionType, Map<String, Object> parameters) {
        return new PlannedAction(action, actionType, parameters);
    }
}
```

**2. Change `WorkerOutcome.Success`** from `record Success()` to `record Success(PlannedAction plannedAction)` (nullable). Update `WorkerOutcome.success()` factory to `return new Success(null)`.

**3. Add convenience factory** to `WorkerResult`:
```java
public static WorkerResult of(Map<String, Object> output, PlannedAction action) {
    return new WorkerResult(output, new WorkerOutcome.Success(action));
}
```

**4. Add partial-output factory overloads** to `WorkerResult`:
```java
public static WorkerResult declined(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Declined(reason));
}
public static WorkerResult failed(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Failed(reason));
}
public static WorkerResult expired(String reason, Map<String, Object> partialOutput) {
    return new WorkerResult(partialOutput, new WorkerOutcome.Expired(reason));
}
```

**Why:** Engine#543 migrates Worker primitives to `casehub-worker-api`. PlannedAction is a universal worker concern (action declaration). Placing it on `Success` structurally enforces that only successful workers can declare consequential actions — no runtime validation needed.

After completing, run `mvn install` so the updated 0.2-SNAPSHOT is in the local Maven repo.

---

**Wait for confirmation** that the worker-api changes are done and `mvn install`'d before proceeding.

---

### Task 1: Add worker-api Dependency + Create New Engine Types

**Files:**
- Modify: `api/pom.xml` — add `casehub-worker-api` dependency
- Create: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java`
- Create: `api/src/main/java/io/casehub/api/model/FlowWorkerFunction.java`
- Create: `api/src/main/java/io/casehub/api/spi/ClassificationContext.java`
- Test: `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java`
- Test: `api/src/test/java/io/casehub/api/model/FlowWorkerFunctionTest.java`

**Interfaces:**
- Consumes: `io.casehub.worker.api.WorkerFunction` (from worker-api dependency), `io.casehub.api.model.ai.Agent` (existing), `io.serverlessworkflow.api.types.Workflow` (existing)
- Produces: `AgentWorkerFunction`, `FlowWorkerFunction`, `ClassificationContext` — used by Tasks 2 and 3

- [ ] **Step 1: Add casehub-worker-api dependency to engine-api pom.xml**

Add to `api/pom.xml` `<dependencies>` section:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-worker-api</artifactId>
</dependency>
```
Version managed by parent BOM (`${casehub.version}`).

- [ ] **Step 2: Write failing test for AgentWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.api.model.ai.Agent;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class AgentWorkerFunctionTest {

    @Test
    void implementsWorkerFunction() {
        Agent agent = Agent.builder()
            .systemPrompt("test")
            .build();
        var fn = new AgentWorkerFunction(agent);
        assertInstanceOf(WorkerFunction.class, fn);
        assertSame(agent, fn.agent());
    }

    @Test
    void rejectsNullAgent() {
        assertThrows(NullPointerException.class, () -> new AgentWorkerFunction(null));
    }
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentWorkerFunctionTest -DfailIfNoTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AgentWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.api.model.ai.Agent;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import java.util.Map;
import java.util.Objects;

public record AgentWorkerFunction(Agent agent) implements WorkerFunction {
    public AgentWorkerFunction {
        Objects.requireNonNull(agent);
    }
    @Override
    public WorkerResult execute(Map<String, Object> input) {
        return agent.execute(input);
    }
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=AgentWorkerFunctionTest`
Expected: PASS

- [ ] **Step 4: Write failing test for FlowWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.worker.api.WorkerFunction;
import io.serverlessworkflow.api.types.Workflow;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FlowWorkerFunctionTest {

    @Test
    void implementsWorkerFunction() {
        Workflow workflow = new Workflow();
        var fn = new FlowWorkerFunction(workflow);
        assertInstanceOf(WorkerFunction.class, fn);
        assertSame(workflow, fn.workflow());
    }

    @Test
    void executeThrowsUnsupported() {
        var fn = new FlowWorkerFunction(new Workflow());
        assertThrows(UnsupportedOperationException.class, () -> fn.execute(Map.of()));
    }

    @Test
    void rejectsNullWorkflow() {
        assertThrows(NullPointerException.class, () -> new FlowWorkerFunction(null));
    }
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=FlowWorkerFunctionTest -DfailIfNoTests=false`
Expected: FAIL — class not found

- [ ] **Step 5: Implement FlowWorkerFunction**

```java
package io.casehub.api.model;

import io.casehub.worker.api.WorkerFunction;
import io.casehub.worker.api.WorkerResult;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Map;
import java.util.Objects;

public record FlowWorkerFunction(Workflow workflow) implements WorkerFunction {
    public FlowWorkerFunction {
        Objects.requireNonNull(workflow);
    }
    @Override
    public WorkerResult execute(Map<String, Object> input) {
        throw new UnsupportedOperationException("Flow execution handled by FlowWorkerExecutor");
    }
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=FlowWorkerFunctionTest`
Expected: PASS

- [ ] **Step 6: Create ClassificationContext**

```java
package io.casehub.api.spi;

import java.util.UUID;

public record ClassificationContext(
    String workerId,
    UUID caseId,
    String tenancyId,
    String caseDefinitionName,
    String capabilityName,
    String bindingName
) {}
```

No test needed — pure record with no behaviour.

- [ ] **Step 7: Verify both test classes pass together**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="AgentWorkerFunctionTest,FlowWorkerFunctionTest"`
Expected: 5 tests PASS

- [ ] **Step 8: Commit**

```
feat(#543): add AgentWorkerFunction, FlowWorkerFunction, ClassificationContext

New engine-owned WorkerFunction implementations:
- AgentWorkerFunction(Agent) — delegates to agent.execute()
- FlowWorkerFunction(Workflow) — throws (dispatch by FlowWorkerExecutor)
ClassificationContext record for ActionRiskClassifier SPI.

Refs #543
```

---

### Task 2: Add AgentDescriptor Map to CaseDefinition

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`
- Test: `api/src/test/java/io/casehub/api/model/CaseDefinitionTest.java` (create or modify)

**Interfaces:**
- Consumes: `io.casehub.eidos.api.AgentDescriptor` (existing dependency)
- Produces: `CaseDefinition.agentDescriptorFor(String workerName)` → `Optional<AgentDescriptor>`, `CaseDefinition.Builder.agentDescriptor(String workerName, AgentDescriptor descriptor)` — used by Task 3

- [ ] **Step 1: Write failing test for agentDescriptorFor()**

```java
package io.casehub.api.model;

import io.casehub.eidos.api.AgentDescriptor;
import org.junit.jupiter.api.Test;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class CaseDefinitionAgentDescriptorTest {

    private AgentDescriptor testDescriptor() {
        return AgentDescriptor.builder()
            .agentId("agent-1")
            .name("test-agent")
            .slot("analyst")
            .tenancyId("tenant-1")
            .build();
    }

    @Test
    void agentDescriptorForReturnsDescriptorWhenPresent() {
        var def = CaseDefinition.builder()
            .namespace("ns").name("case").version("1.0")
            .agentDescriptor("worker-a", testDescriptor())
            .build();
        var result = def.agentDescriptorFor("worker-a");
        assertTrue(result.isPresent());
        assertEquals("agent-1", result.get().agentId());
    }

    @Test
    void agentDescriptorForReturnsEmptyWhenAbsent() {
        var def = CaseDefinition.builder()
            .namespace("ns").name("case").version("1.0")
            .build();
        assertEquals(Optional.empty(), def.agentDescriptorFor("worker-a"));
    }

    @Test
    void agentDescriptorForReturnsEmptyForWrongName() {
        var def = CaseDefinition.builder()
            .namespace("ns").name("case").version("1.0")
            .agentDescriptor("worker-a", testDescriptor())
            .build();
        assertEquals(Optional.empty(), def.agentDescriptorFor("worker-b"));
    }
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionAgentDescriptorTest -DfailIfNoTests=false`
Expected: FAIL — method not found

- [ ] **Step 2: Add agentDescriptor map to CaseDefinition**

Add to `CaseDefinition`:
```java
import io.casehub.eidos.api.AgentDescriptor;
import java.util.HashMap;
import java.util.Optional;

// Field (after panelNames):
private Map<String, AgentDescriptor> agentDescriptors = Map.of();

// Method:
public Optional<AgentDescriptor> agentDescriptorFor(String workerName) {
    return Optional.ofNullable(agentDescriptors.get(workerName));
}

// Setter (for builder):
public void setAgentDescriptors(Map<String, AgentDescriptor> agentDescriptors) {
    this.agentDescriptors = agentDescriptors != null ? Map.copyOf(agentDescriptors) : Map.of();
}
```

Add to `CaseDefinition.Builder`:
```java
private Map<String, AgentDescriptor> agentDescriptors = new HashMap<>();

public Builder agentDescriptor(String workerName, AgentDescriptor descriptor) {
    this.agentDescriptors.put(workerName, descriptor);
    return this;
}
```

Update `build()` method to set agentDescriptors on the constructed CaseDefinition:
```java
def.setAgentDescriptors(agentDescriptors);
```

- [ ] **Step 3: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionAgentDescriptorTest`
Expected: 3 tests PASS

- [ ] **Step 4: Commit**

```
feat(#543): add AgentDescriptor map to CaseDefinition

CaseDefinition.agentDescriptorFor(workerName) returns Optional<AgentDescriptor>.
Builder.agentDescriptor(workerName, descriptor) populates the map.
Decouples Worker identity (foundation) from agent identity (eidos).

Refs #543
```

---

### Task 3: Delete Old Types + Full Migration

This is the atomic type swap. Delete 9 old type files, fix all compilation errors across ~90 files. Code will not compile between sub-steps — this is one commit.

**Files:**
- Delete: 9 files from `api/src/main/java/io/casehub/api/model/` and `api/src/main/java/io/casehub/api/spi/`
- Modify: ~60 production files (import changes, accessor renames, builder patterns)
- Modify: ~30 test files (import changes, accessor renames, mock-to-construction rewrites)
- Modify: `api/src/main/java/io/casehub/api/spi/ActionRiskClassifier.java` — signature change
- Modify: `api/src/main/java/io/casehub/api/spi/ReactiveActionRiskClassifier.java` — signature change
- Modify: `api/src/main/java/io/casehub/api/classification/ChainedReactiveActionRiskClassifier.java` — pass ClassificationContext
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/WorkflowExecutionCompleted.java` — remove plannedAction field
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/AgentCandidateFactory.java` — CaseDefinition parameter + descriptor lookup
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — Worker/Capability construction
- Modify: `api/src/main/java/io/casehub/api/plan/PlanElement.java` — Javadoc update

**Interfaces:**
- Consumes: `AgentWorkerFunction`, `FlowWorkerFunction`, `ClassificationContext` (from Task 1), `CaseDefinition.agentDescriptorFor()` (from Task 2), all types from `io.casehub.worker.api.*` and `io.casehub.platform.api.governance.*`
- Produces: fully migrated codebase — all engine code uses foundation-tier types

The approach: use IntelliJ MCP for rename refactoring where possible (`ide_refactor_rename`), then fix remaining compilation errors with targeted edits.

- [ ] **Step 1: Delete 9 old type files**

Delete from `api/src/main/java/io/casehub/api/model/`:
- `Worker.java`
- `WorkerFunction.java`
- `WorkerResult.java`
- `WorkerOutcome.java`
- `Capability.java`
- `ExecutionPolicy.java`
- `RetryPolicy.java`
- `BackoffStrategy.java`

Delete from `api/src/main/java/io/casehub/api/spi/`:
- `PlannedAction.java`

- [ ] **Step 2: Fix engine-api compilation — ActionRiskClassifier SPI**

`ActionRiskClassifier.java` — change `classify(PlannedAction)` to `classify(PlannedAction, ClassificationContext)`:
```java
import io.casehub.worker.api.PlannedAction;
import io.casehub.api.spi.ClassificationContext;

RiskDecision classify(PlannedAction action, ClassificationContext context);
```

`ReactiveActionRiskClassifier.java` — same signature change with `Uni<RiskDecision>`:
```java
Uni<RiskDecision> classify(PlannedAction action, ClassificationContext context);
```

`ChainedReactiveActionRiskClassifier.java` — update to pass context through:
```java
public Uni<RiskDecision> classify(PlannedAction action, ClassificationContext context) {
    // ... pass context to each delegate
}
```

- [ ] **Step 3: Fix engine-api compilation — CaseDefinitionYamlMapper**

Replace all `io.casehub.api.model.Worker` / `Capability` / `ExecutionPolicy` / `RetryPolicy` / `BackoffStrategy` imports with:
```java
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.Capability;
import io.casehub.worker.api.WorkerFunction;
import io.casehub.platform.api.governance.ExecutionPolicy;
import io.casehub.platform.api.governance.RetryPolicy;
import io.casehub.platform.api.governance.BackoffStrategy;
```

Replace Worker construction (`new Worker(name, caps, workflow)` + setters) with builder chains using `FlowWorkerFunction` / `AgentWorkerFunction`.

Replace Capability construction (`new Capability(name, input, output)` + setters) with `Capability.builder()` or `Capability.of()`.

- [ ] **Step 4: Fix engine-api compilation — PlanElement Javadoc**

Update `PlanElement.java` Javadoc to remove Worker reference.

- [ ] **Step 5: Fix engine-api compilation — remaining files**

Fix all other engine-api files: update imports from `io.casehub.api.model.*` to `io.casehub.worker.api.*` / `io.casehub.platform.api.governance.*`. Change getters to record accessors (`getName()` → `name()`, etc.). Includes:
- `CaseDefinition.java` — `List<Worker>`, `List<Capability>` fields
- `Binding.java`, `CapabilityTarget.java` — Capability references
- `Worker.java` references throughout engine-api SPIs and model classes
- `WorkerResult` / `WorkerOutcome` / `PlannedAction` references in SPI interfaces

- [ ] **Step 6: Fix engine-common compilation — WorkflowExecutionCompleted**

Remove `plannedAction` field from `WorkflowExecutionCompleted`:
```java
public record WorkflowExecutionCompleted(
    CaseInstance caseInstance,
    Worker worker,
    String idempotency,
    Map<String, Object> output,
    String bindingName,
    WorkerOutcome outcome) {

  public static WorkflowExecutionCompleted approved(
      final CaseInstance caseInstance,
      final Worker worker,
      final String idempotency,
      final Map<String, Object> output,
      final String bindingName) {
    return new WorkflowExecutionCompleted(
        caseInstance, worker, idempotency, output, bindingName, WorkerOutcome.success());
  }
}
```

Update imports to `io.casehub.worker.api.*`.

Fix all other engine-common files: `WorkerExecutor.java`, `WorkerExecutionConfig.java`, `RetryPolicies.java`, `RetryDecision.java`, `ExecutionMetadata.java`, `WorkerScheduleEvent.java`, etc.

- [ ] **Step 7: Fix runtime compilation — handlers and executor**

Key files with non-trivial changes:
- `DefaultWorkerExecutor.java` — change sealed switch to instanceof dispatch
- `WorkflowExecutionCompletedHandler.java` — extract PlannedAction from `event.outcome()`, construct `ClassificationContext`
- `CaseContextChangedEventHandler.java` — pass CaseDefinition to AgentCandidateFactory
- `DefaultWorkOrchestrator.java` — pass CaseDefinition to AgentCandidateFactory
- `AgentCandidateFactory.java` — add CaseDefinition parameter, use `caseDefinition.agentDescriptorFor()`

All other runtime files: import changes + accessor renames (`getName()` → `name()`, etc.)

- [ ] **Step 8: Fix scheduler-quartz compilation**

`QuartzWorkerExecutionJob.onSuccess()` — remove PlannedAction extraction and `withIdentity()` enrichment. Construct event without plannedAction field:
```java
var event = new WorkflowExecutionCompleted(
    caseInstance, worker, idempotency, result.output(), bindingName, result.outcome());
```

Fix all other scheduler-quartz files: imports + accessor renames.

- [ ] **Step 9: Fix blackboard, resilience, work-adapter, flow, testing, engine-ai compilation**

Module-by-module import changes and accessor renames. No behavioral changes in these modules.

- [ ] **Step 10: Fix test compilation**

Replace Mockito mocks of Worker/Capability with real object construction:
```java
// Old
Worker worker = mock(Worker.class);
when(worker.getName()).thenReturn("worker-a");
when(worker.getCapabilities()).thenReturn(List.of(cap));

// New
Worker worker = Worker.builder()
    .name("worker-a")
    .capabilities(cap)
    .function(input -> WorkerResult.of(Map.of()))
    .build();
```

```java
// Old
Capability cap = mock(Capability.class);
when(cap.getName()).thenReturn("research");

// New
Capability cap = Capability.of("research", "{}", "{}");
```

Fix all test imports and accessor renames.

- [ ] **Step 11: Build and run full test suite**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test
```

Expected: all tests PASS. Fix any remaining compilation or test failures.

- [ ] **Step 12: Commit**

```
refactor(#543): migrate Worker primitives to casehub-worker-api

Delete 9 type files from engine-api:
- Worker, WorkerFunction, WorkerResult, WorkerOutcome, Capability
- ExecutionPolicy, RetryPolicy, BackoffStrategy, PlannedAction

Replace with foundation types from casehub-worker-api and
casehub-platform-api. Create engine-owned AgentWorkerFunction and
FlowWorkerFunction. Move AgentDescriptor from Worker to CaseDefinition.
Update ActionRiskClassifier to use ClassificationContext. Remove
redundant plannedAction from WorkflowExecutionCompleted.

~90 files updated across all engine modules.

Closes #543
```

---

### Task 4: File Follow-Up Issues + Doc Sync

**Files:**
- Modify: `CLAUDE.md` — update Worker type references, package paths

- [ ] **Step 1: File follow-up issues**

```bash
gh issue create --repo casehubio/parent --title "docs: update PLATFORM.MD worker-api type names" \
  --body "PLATFORM.MD says WorkerContext/WorkerSpec/WorkerCapability. Actual types are Worker/Capability/WorkerFunction/WorkerResult/WorkerOutcome. Refs engine#543."

gh issue create --repo casehubio/engine --title "refactor: remove serverlessworkflow SDK from engine-api" \
  --body "After engine#543, FlowWorkerFunction is the sole serverlessworkflow import in engine-api. Removing it requires restructuring CaseDefinitionYamlMapper and YamlCaseHub. Refs #543."
```

- [ ] **Step 2: Update CLAUDE.md**

Update all references to `io.casehub.api.model.Worker`, `io.casehub.api.model.WorkerResult`, etc. to point to the new packages. Update the Worker Provisioner SPIs section, Worker Execution Architecture section, Agent Worker AI Model section, and ActionRiskClassifier SPI section.

- [ ] **Step 3: Run implementation-doc-sync skill**

Invoke `implementation-doc-sync` to check for any other docs that need updating (DESIGN.md, etc.)

- [ ] **Step 4: Commit**

```
docs(#543): update CLAUDE.md for worker-api migration

Update package references, type descriptions, and architectural
documentation to reflect the migration to casehub-worker-api and
casehub-platform-api foundation types.

Refs #543
```

---

## Execution Notes

- **Task 3 is atomic.** The code will not compile between sub-steps. All sub-steps are part of one commit.
- **IntelliJ refactoring** — use `ide_refactor_rename` and `ide_find_references` for mechanical changes. IntelliJ is semantically correct; bash grep is not.
- **Existing tests are the verification.** The migration should preserve all existing test behavior. New tests are only for the new types (Tasks 1 and 2).
- **If a test fails after migration** — investigate whether it's a migration error or a pre-existing issue. The migration should not change behavior, only type sources and accessor syntax.
