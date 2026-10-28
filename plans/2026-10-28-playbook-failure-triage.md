# Playbook Failure Output + RecoveryCoordinator Triage Wiring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1262 — structured playbook failure output + RecoveryCoordinator triage wiring
**Issue group:** #1262, #1263

**Goal:** Enable playbook YAML as worker function definitions, capture per-step execution outcomes, and wire TriageHandler into the recovery coordinator for intelligent post-failure playbook modification and retry.

**Architecture:** Batch 1 introduces `SWFWorkerFunction` sealed interface and `PlaybookWorkerFunctionProvider` (compiles playbook YAML → SWF Workflow). Batch 2 adds `PlaybookExecutionResult` with step-level capture via a CDI `WorkflowExecutionListener` accumulating in `FlowExecutionRegistry`. Batch 3 wires `TriageHandler` inside `DefaultRecoveryCoordinator` after structural recovery exhaustion, using `CaseContextChangedEvent` for re-dispatch. Batch 4 adds dual-granularity CBR recording.

**Tech Stack:** Java 21, Quarkus 3.32.2, Serverless Workflow SDK 7.25.1.Final, casehub-platform-playbook-compiler (7.x), casehub-platform-api 0.4-SNAPSHOT (TriageHandler SPI)

## Global Constraints

- SWF SDK version: 7.x only (no 8.x/Jackson 3 until Quarkus 4.x migration)
- Platform compiler uses `serverlessworkflow-types` 7.35.3.Final — verify binary compat with engine's 7.25.1.Final runtime
- `PlaybookExecutionResult` in engine-api (consumer-visible); execution infrastructure in common-core/runtime
- No `@DefaultBean` on multi-instance SPIs; `@DefaultBean` is OK for 1:1 replacement SPIs like `TriageHandler`
- All new types are pure records/enums/sealed interfaces — no CDI annotations on API types
- Test classes must be `*Test.java`, never `*IT.java`
- Commit messages reference issues: `Refs #1262` or `Refs #1263`
- Use `ide_insert_member` / `ide_replace_member` for code edits; `ide_refactor_rename` for renames; `ide_move_file` for moves
- Build: `mvn install -DskipTests -q` before module tests; `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl <module>`

---

## Batch 1: PlaybookWorkerFunctionProvider (#1263)

### Task 1: SWFWorkerFunction sealed interface + FlowWorkerFunction migration

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/SWFWorkerFunction.java`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java:24`
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java:41`

**Interfaces:**
- Produces: `SWFWorkerFunction` sealed interface with `Workflow workflow()` accessor, permits `FlowWorkerFunction`, `PlaybookWorkerFunction`
- Produces: `FlowWorkerFunction` now implements `SWFWorkerFunction` instead of `WorkerFunction` directly

- [ ] **Step 1: Write the failing test**

Add a test in `FlowWorkerFunctionHandlerTest.java` that verifies `FlowWorkerFunction` is an instance of `SWFWorkerFunction`:

```java
@Test
void flow_worker_function_implements_swf_worker_function() {
    FlowWorkerFunction fn = new FlowWorkerFunction(mock(Workflow.class));
    assertThat(fn).isInstanceOf(SWFWorkerFunction.class);
    assertThat(((SWFWorkerFunction) fn).workflow()).isNotNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest#flow_worker_function_implements_swf_worker_function`
Expected: FAIL — `SWFWorkerFunction` does not exist

- [ ] **Step 3: Create SWFWorkerFunction sealed interface**

Create `engine-support-core/src/main/java/io/casehub/engine/flow/SWFWorkerFunction.java`:

```java
package io.casehub.engine.flow;

import io.casehub.worker.api.WorkerFunction;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Map;

@SuppressWarnings("unchecked")
public sealed interface SWFWorkerFunction
    extends WorkerFunction<Map<String, Object>, Map<String, Object>>
    permits FlowWorkerFunction, PlaybookWorkerFunction {
  Workflow workflow();
}
```

- [ ] **Step 4: Create PlaybookWorkerFunction stub**

Create a minimal `PlaybookWorkerFunction` record so the sealed `permits` clause compiles. Full implementation comes in Task 2.

Create `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunction.java`:

```java
package io.casehub.engine.flow;

import io.casehub.worker.api.PlannedAction;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Map;
import java.util.function.Function;

@SuppressWarnings("unchecked")
public record PlaybookWorkerFunction(
    Workflow workflow,
    Function<Map<String, Object>, PlannedAction> plannedActionFn)
    implements SWFWorkerFunction {

  public PlaybookWorkerFunction(Workflow workflow) {
    this(workflow, null);
  }

  public PlaybookWorkerFunction {
    java.util.Objects.requireNonNull(workflow, "workflow must not be null");
  }

  @Override
  public Class<Map<String, Object>> inputType() {
    return (Class) Map.class;
  }

  @Override
  public Class<Map<String, Object>> outputType() {
    return (Class) Map.class;
  }
}
```

- [ ] **Step 5: Migrate FlowWorkerFunction to implement SWFWorkerFunction**

Use `ide_replace_member` to change `FlowWorkerFunction` record declaration. Replace:

```java
public record FlowWorkerFunction(
    Workflow workflow, Function<java.util.Map<String, Object>, PlannedAction> plannedActionFn)
    implements WorkerFunction<java.util.Map<String, Object>, java.util.Map<String, Object>> {
```

With:

```java
public record FlowWorkerFunction(
    Workflow workflow, Function<java.util.Map<String, Object>, PlannedAction> plannedActionFn)
    implements SWFWorkerFunction {
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/SWFWorkerFunction.java engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunction.java engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java
git commit -m "feat(#1263): add SWFWorkerFunction sealed interface, migrate FlowWorkerFunction

Refs #1263"
```

---

### Task 2: PlaybookWorkerFunctionProvider

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunction.java` (add CompilationResult field)
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunctionProvider.java`
- Modify: `engine-support-core/pom.xml` (add playbook-compiler dependency)
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookWorkerFunctionProviderTest.java`

**Interfaces:**
- Consumes: `WorkerFunctionProvider` SPI (handles/create pattern from FlowWorkerFunctionProvider)
- Consumes: `PlaybookCompiler.compile(Map)` → `CompilationOutcome`; `CompilationResult.workflow()`, `.sourceMap()`, `.coordinationSpec()`
- Produces: `PlaybookWorkerFunctionProvider` that handles `steps:` blocks in YAML worker nodes

- [ ] **Step 1: Add playbook-compiler dependency to engine-support-core pom.xml**

Add to `engine-support-core/pom.xml` `<dependencies>` section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-compiler</artifactId>
</dependency>
```

Add version property to root `pom.xml` `<dependencyManagement>` if not present:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-playbook-compiler</artifactId>
    <version>${version.io.casehub}</version>
</dependency>
```

- [ ] **Step 2: Update PlaybookWorkerFunction with CompilationResult**

Replace the stub `PlaybookWorkerFunction` with the full version that carries `CompilationResult`:

```java
package io.casehub.engine.flow;

import io.casehub.platform.compiler.CompilationResult;
import io.casehub.worker.api.PlannedAction;
import io.serverlessworkflow.api.types.Workflow;
import java.util.Map;
import java.util.function.Function;

@SuppressWarnings("unchecked")
public record PlaybookWorkerFunction(
    Workflow workflow,
    CompilationResult compilationResult,
    Function<Map<String, Object>, PlannedAction> plannedActionFn)
    implements SWFWorkerFunction {

  public PlaybookWorkerFunction(Workflow workflow, CompilationResult compilationResult) {
    this(workflow, compilationResult, null);
  }

  public PlaybookWorkerFunction {
    java.util.Objects.requireNonNull(workflow, "workflow must not be null");
  }

  @Override
  public Class<Map<String, Object>> inputType() {
    return (Class) Map.class;
  }

  @Override
  public Class<Map<String, Object>> outputType() {
    return (Class) Map.class;
  }

  public PlaybookWorkerFunction withPlannedAction(
      Function<Map<String, Object>, PlannedAction> actionFn) {
    return new PlaybookWorkerFunction(workflow, compilationResult, actionFn);
  }
}
```

- [ ] **Step 3: Write the failing test**

Create `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookWorkerFunctionProviderTest.java`:

```java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.junit.jupiter.api.Test;

class PlaybookWorkerFunctionProviderTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void handles_workers_with_steps_block() {
    var provider = new PlaybookWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    node.putArray("steps");
    assertThat(provider.handles(node)).isTrue();
  }

  @Test
  void does_not_handle_workers_without_steps_block() {
    var provider = new PlaybookWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode().put("name", "test");
    assertThat(provider.handles(node)).isFalse();
  }

  @Test
  void does_not_handle_workers_with_do_block() {
    var provider = new PlaybookWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    node.putArray("do");
    assertThat(provider.handles(node)).isFalse();
  }

  @Test
  void creates_playbook_worker_function_from_steps() {
    var provider = new PlaybookWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    ObjectNode step = MAPPER.createObjectNode();
    step.putObject("rest-request")
        .put("method", "GET")
        .put("url", "/api/test");
    node.putArray("steps").add(step);

    var fn = provider.create(node);
    assertThat(fn).isInstanceOf(PlaybookWorkerFunction.class);
    PlaybookWorkerFunction pbFn = (PlaybookWorkerFunction) fn;
    assertThat(pbFn.workflow()).isNotNull();
    assertThat(pbFn.compilationResult()).isNotNull();
    assertThat(pbFn.compilationResult().sourceMap()).isNotNull();
  }

  @Test
  void throws_on_invalid_playbook_syntax() {
    var provider = new PlaybookWorkerFunctionProvider();
    ObjectNode node = MAPPER.createObjectNode();
    node.putArray("steps").add("not-a-valid-step");

    assertThatThrownBy(() -> provider.create(node))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("Failed to compile playbook");
  }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=PlaybookWorkerFunctionProviderTest`
Expected: FAIL — `PlaybookWorkerFunctionProvider` does not exist

- [ ] **Step 5: Implement PlaybookWorkerFunctionProvider**

Create `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunctionProvider.java`:

```java
package io.casehub.engine.flow;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.spi.WorkerFunctionProvider;
import io.casehub.platform.compiler.CompilationOutcome;
import io.casehub.platform.compiler.CompilationResult;
import io.casehub.platform.compiler.PlaybookCompiler;
import io.casehub.worker.api.WorkerFunction;
import java.util.Map;

public class PlaybookWorkerFunctionProvider implements WorkerFunctionProvider {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private final PlaybookCompiler compiler = new PlaybookCompiler();

  @Override
  public boolean handles(final JsonNode rawWorkerNode) {
    return rawWorkerNode.has("steps");
  }

  @SuppressWarnings("unchecked")
  @Override
  public WorkerFunction<?, ?> create(final JsonNode rawWorkerNode) {
    try {
      Map<String, Object> content = MAPPER.convertValue(rawWorkerNode, Map.class);
      CompilationOutcome outcome = compiler.compile(content);
      if (!(outcome instanceof CompilationResult result)) {
        throw new IllegalArgumentException(
            "Playbook compilation produced diagnostic, not a result: " + outcome);
      }
      return new PlaybookWorkerFunction(result.workflow(), result);
    } catch (IllegalArgumentException e) {
      throw e;
    } catch (Exception e) {
      throw new IllegalArgumentException("Failed to compile playbook definition from YAML", e);
    }
  }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=PlaybookWorkerFunctionProviderTest`
Expected: ALL PASS (adjust test expectations if `PlaybookCompiler` error messages differ)

- [ ] **Step 7: Commit**

```bash
git add engine-support-core/pom.xml engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunction.java engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookWorkerFunctionProvider.java engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookWorkerFunctionProviderTest.java
git commit -m "feat(#1263): add PlaybookWorkerFunctionProvider and PlaybookWorkerFunction

Compiles playbook YAML (steps: blocks) via PlaybookCompiler to SWF Workflow.
Carries CompilationResult with SourceMap and CoordinationSpec.

Refs #1263"
```

---

### Task 3: FlowWorkerFunctionHandler update for SWFWorkerFunction

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:43`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java:41`

**Interfaces:**
- Consumes: `SWFWorkerFunction.workflow()` — unified accessor
- Produces: Handler that accepts both `FlowWorkerFunction` and `PlaybookWorkerFunction`

- [ ] **Step 1: Write the failing test**

Add tests to `FlowWorkerFunctionHandlerTest.java`:

```java
@Test
void supports_playbook_worker_function() {
    assertThat(handler.supports(new PlaybookWorkerFunction(mock(Workflow.class)))).isTrue();
}

@Test
void supports_swf_worker_function_interface() {
    SWFWorkerFunction swf = new FlowWorkerFunction(mock(Workflow.class));
    assertThat(handler.supports(swf)).isTrue();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest#supports_playbook_worker_function`
Expected: FAIL — `supports()` returns false for `PlaybookWorkerFunction`

- [ ] **Step 3: Update supports() and execute()**

In `FlowWorkerFunctionHandler.java`, replace `supports()`:

```java
@Override
public boolean supports(final WorkerFunction<?, ?> function) {
    return function instanceof SWFWorkerFunction;
}
```

In `execute()`, replace the cast at line 71:

```java
final FlowWorkerFunction flow = (FlowWorkerFunction) function;
```

With:

```java
final SWFWorkerFunction swf = (SWFWorkerFunction) function;
```

And update workflow access from `flow.workflow()` to `swf.workflow()`. Update the `plannedActionFn` check — only `FlowWorkerFunction` and `PlaybookWorkerFunction` have this, access via cast or check:

```java
Function<Map<String, Object>, PlannedAction> plannedActionFn = null;
if (swf instanceof FlowWorkerFunction flow) {
    plannedActionFn = flow.plannedActionFn();
} else if (swf instanceof PlaybookWorkerFunction pb) {
    plannedActionFn = pb.plannedActionFn();
}
```

Then use `plannedActionFn` instead of `flow.plannedActionFn()` in the result construction.

- [ ] **Step 4: Run full handler test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest`
Expected: ALL PASS

- [ ] **Step 5: Run full engine-support-core test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java
git commit -m "feat(#1263): update FlowWorkerFunctionHandler for SWFWorkerFunction

supports() now accepts any SWFWorkerFunction (Flow or Playbook).
execute() uses SWFWorkerFunction.workflow() for unified access.

Closes #1263"
```

---

## Batch 2: Structured Failure Output (#1262 — step capture)

### Task 4: PlaybookExecutionResult, StepResult, StepStatus types

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/PlaybookExecutionResult.java`
- Create: `api/src/main/java/io/casehub/api/model/StepResult.java`
- Create: `api/src/main/java/io/casehub/api/model/StepStatus.java`
- Test: `api/src/test/java/io/casehub/api/model/PlaybookExecutionResultTest.java`

**Interfaces:**
- Produces: `PlaybookExecutionResult` sealed interface (Success, PartialSuccess, Failure) with `List<StepResult> steps()`
- Produces: `StepResult(String stepId, String stepName, StepStatus status, Duration elapsed, String error)`
- Produces: `StepStatus` enum (SUCCEEDED, FAILED, SKIPPED)

- [ ] **Step 1: Write the failing test**

Create `api/src/test/java/io/casehub/api/model/PlaybookExecutionResultTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Duration;
import java.util.List;
import org.junit.jupiter.api.Test;

class PlaybookExecutionResultTest {

  @Test
  void success_contains_output_and_steps() {
    var step = new StepResult("s1", "fetch-data", StepStatus.SUCCEEDED, Duration.ofMillis(150), null);
    var result = new PlaybookExecutionResult.Success(java.util.Map.of("key", "value"), List.of(step));

    assertThat(result.steps()).hasSize(1);
    assertThat(result.steps().getFirst().stepName()).isEqualTo("fetch-data");
    assertThat(result.output()).isNotNull();
    assertThat(result).isInstanceOf(PlaybookExecutionResult.class);
  }

  @Test
  void failure_contains_root_cause_and_failed_step() {
    var step1 = new StepResult("s1", "fetch-data", StepStatus.SUCCEEDED, Duration.ofMillis(100), null);
    var step2 = new StepResult("s2", "transform", StepStatus.FAILED, Duration.ofMillis(50), "NullPointerException");
    var result = new PlaybookExecutionResult.Failure("transform step NPE", List.of(step1, step2));

    assertThat(result.rootCause()).isEqualTo("transform step NPE");
    assertThat(result.steps()).hasSize(2);
    assertThat(result.steps().get(1).status()).isEqualTo(StepStatus.FAILED);
    assertThat(result.steps().get(1).error()).isEqualTo("NullPointerException");
  }

  @Test
  void partial_success_contains_partial_output_and_mixed_steps() {
    var step1 = new StepResult("s1", "fetch", StepStatus.SUCCEEDED, Duration.ofMillis(200), null);
    var step2 = new StepResult("s2", "process", StepStatus.FAILED, Duration.ofMillis(10), "timeout");
    var result = new PlaybookExecutionResult.PartialSuccess(
        java.util.Map.of("partial", true), List.of(step1, step2));

    assertThat(result.partialOutput()).isNotNull();
    assertThat(result.steps()).hasSize(2);
  }

  @Test
  void step_result_skipped_status() {
    var step = new StepResult("s3", "optional-step", StepStatus.SKIPPED, Duration.ZERO, null);
    assertThat(step.status()).isEqualTo(StepStatus.SKIPPED);
    assertThat(step.error()).isNull();
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=PlaybookExecutionResultTest`
Expected: FAIL — types don't exist

- [ ] **Step 3: Create StepStatus enum**

Create `api/src/main/java/io/casehub/api/model/StepStatus.java`:

```java
package io.casehub.api.model;

public enum StepStatus {
  SUCCEEDED,
  FAILED,
  SKIPPED
}
```

- [ ] **Step 4: Create StepResult record**

Create `api/src/main/java/io/casehub/api/model/StepResult.java`:

```java
package io.casehub.api.model;

import java.time.Duration;

public record StepResult(
    String stepId,
    String stepName,
    StepStatus status,
    Duration elapsed,
    String error) {}
```

- [ ] **Step 5: Create PlaybookExecutionResult sealed interface**

Create `api/src/main/java/io/casehub/api/model/PlaybookExecutionResult.java`:

```java
package io.casehub.api.model;

import java.util.List;

public sealed interface PlaybookExecutionResult {
  List<StepResult> steps();

  record Success(Object output, List<StepResult> steps) implements PlaybookExecutionResult {}

  record PartialSuccess(Object partialOutput, List<StepResult> steps)
      implements PlaybookExecutionResult {}

  record Failure(String rootCause, List<StepResult> steps) implements PlaybookExecutionResult {}
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=PlaybookExecutionResultTest`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/PlaybookExecutionResult.java api/src/main/java/io/casehub/api/model/StepResult.java api/src/main/java/io/casehub/api/model/StepStatus.java api/src/test/java/io/casehub/api/model/PlaybookExecutionResultTest.java
git commit -m "feat(#1262): add PlaybookExecutionResult, StepResult, StepStatus types

Sealed interface for structured step-level playbook execution outcomes.

Refs #1262"
```

---

### Task 5: FlowExecutionRegistry step data storage + find() method

**Files:**
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowExecutionRegistry.java:26`
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowExecutionRegistryTest.java`

**Interfaces:**
- Produces: `find(String instanceId)` → `Optional<FlowExecution>` (non-throwing lookup)
- Produces: `recordStep(String instanceId, StepResult step)` — accumulates step results
- Produces: `getSteps(String instanceId)` → `List<StepResult>` — retrieves accumulated steps
- Produces: `removeSteps(String instanceId)` — cleans up step data

- [ ] **Step 1: Write the failing test**

Add tests to existing `FlowExecutionRegistryTest.java` (or create if needed):

```java
@Test
void find_returns_empty_for_unregistered_instance() {
    var registry = new FlowExecutionRegistry();
    assertThat(registry.find("nonexistent")).isEmpty();
}

@Test
void find_returns_execution_for_registered_instance() {
    var registry = new FlowExecutionRegistry();
    UUID caseId = UUID.randomUUID();
    registry.register("wf-1", caseId, "worker-1", "hash-1");

    var result = registry.find("wf-1");
    assertThat(result).isPresent();
    assertThat(result.get().caseId()).isEqualTo(caseId);
}

@Test
void records_and_retrieves_step_results() {
    var registry = new FlowExecutionRegistry();
    registry.register("wf-1", UUID.randomUUID(), "w", "h");
    var step = new StepResult("s1", "fetch", StepStatus.SUCCEEDED, Duration.ofMillis(100), null);

    registry.recordStep("wf-1", step);

    List<StepResult> steps = registry.getSteps("wf-1");
    assertThat(steps).hasSize(1);
    assertThat(steps.getFirst().stepId()).isEqualTo("s1");
}

@Test
void get_steps_returns_empty_when_no_steps_recorded() {
    var registry = new FlowExecutionRegistry();
    assertThat(registry.getSteps("wf-unknown")).isEmpty();
}

@Test
void remove_clears_step_data() {
    var registry = new FlowExecutionRegistry();
    registry.register("wf-1", UUID.randomUUID(), "w", "h");
    registry.recordStep("wf-1", new StepResult("s1", "s", StepStatus.SUCCEEDED, Duration.ZERO, null));

    registry.remove("wf-1");

    assertThat(registry.find("wf-1")).isEmpty();
    assertThat(registry.getSteps("wf-1")).isEmpty();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowExecutionRegistryTest`
Expected: FAIL — methods don't exist

- [ ] **Step 3: Add find(), recordStep(), getSteps() to FlowExecutionRegistry**

Add a second ConcurrentHashMap for step data and the new methods:

```java
private final ConcurrentHashMap<String, List<StepResult>> stepData = new ConcurrentHashMap<>();

public Optional<FlowExecution> find(final String instanceId) {
    return Optional.ofNullable(executions.get(instanceId));
}

public void recordStep(final String instanceId, final StepResult step) {
    stepData.computeIfAbsent(instanceId, k -> java.util.Collections.synchronizedList(new java.util.ArrayList<>())).add(step);
}

public List<StepResult> getSteps(final String instanceId) {
    List<StepResult> steps = stepData.get(instanceId);
    return steps != null ? List.copyOf(steps) : List.of();
}
```

Update `remove()` to also clear step data:

```java
public void remove(final String instanceId) {
    executions.remove(instanceId);
    stepData.remove(instanceId);
}
```

Add imports for `StepResult`, `Optional`, `List`.

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowExecutionRegistryTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/FlowExecutionRegistry.java engine-support-core/src/test/java/io/casehub/engine/flow/FlowExecutionRegistryTest.java
git commit -m "feat(#1262): add step data storage and find() to FlowExecutionRegistry

Separate ConcurrentHashMap for step results. find() returns Optional
(non-throwing) for use by PlaybookExecutionListener.

Refs #1262"
```

---

### Task 6: PlaybookExecutionListener

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionListener.java`
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookExecutionListenerTest.java`

**Interfaces:**
- Consumes: `FlowExecutionRegistry.find()`, `.recordStep()` — to identify casehub executions and accumulate step results
- Consumes: `WorkflowExecutionListener` callbacks (onTaskStarted, onTaskCompleted, onTaskFailed)
- Consumes: `TaskContext.taskName()`, `.startedAt()`, `.completedAt()`, `TaskFailedEvent.cause()`
- Produces: CDI bean that accumulates `StepResult` entries in `FlowExecutionRegistry` for each workflow task lifecycle event

- [ ] **Step 1: Write the failing test**

Create `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookExecutionListenerTest.java`:

```java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

import io.casehub.api.model.StepResult;
import io.casehub.api.model.StepStatus;
import io.serverlessworkflow.impl.TaskContext;
import io.serverlessworkflow.impl.WorkflowContext;
import io.serverlessworkflow.impl.WorkflowInstance;
import io.serverlessworkflow.impl.lifecycle.TaskCompletedEvent;
import io.serverlessworkflow.impl.lifecycle.TaskFailedEvent;
import io.serverlessworkflow.impl.lifecycle.TaskStartedEvent;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class PlaybookExecutionListenerTest {

  private FlowExecutionRegistry registry;
  private PlaybookExecutionListener listener;

  @BeforeEach
  void setUp() {
    registry = new FlowExecutionRegistry();
    listener = new PlaybookExecutionListener(registry);
  }

  @Test
  void records_completed_step() {
    String instanceId = "wf-1";
    registry.register(instanceId, UUID.randomUUID(), "w", "h");

    TaskContext taskCtx = mockTaskContext("fetch-data", Instant.now().minusMillis(200));
    when(taskCtx.completedAt()).thenReturn(Instant.now());

    listener.onTaskCompleted(new TaskCompletedEvent(mockWorkflowContext(instanceId), taskCtx));

    List<StepResult> steps = registry.getSteps(instanceId);
    assertThat(steps).hasSize(1);
    assertThat(steps.getFirst().stepName()).isEqualTo("fetch-data");
    assertThat(steps.getFirst().status()).isEqualTo(StepStatus.SUCCEEDED);
    assertThat(steps.getFirst().error()).isNull();
  }

  @Test
  void records_failed_step_with_cause() {
    String instanceId = "wf-2";
    registry.register(instanceId, UUID.randomUUID(), "w", "h");

    TaskContext taskCtx = mockTaskContext("transform", Instant.now().minusMillis(50));
    when(taskCtx.completedAt()).thenReturn(Instant.now());

    listener.onTaskFailed(
        new TaskFailedEvent(
            mockWorkflowContext(instanceId), taskCtx, new RuntimeException("NPE in transform")));

    List<StepResult> steps = registry.getSteps(instanceId);
    assertThat(steps).hasSize(1);
    assertThat(steps.getFirst().status()).isEqualTo(StepStatus.FAILED);
    assertThat(steps.getFirst().error()).contains("NPE in transform");
  }

  @Test
  void ignores_non_casehub_executions() {
    TaskContext taskCtx = mockTaskContext("external-step", Instant.now());
    when(taskCtx.completedAt()).thenReturn(Instant.now());

    listener.onTaskCompleted(new TaskCompletedEvent(mockWorkflowContext("unknown-wf"), taskCtx));

    assertThat(registry.getSteps("unknown-wf")).isEmpty();
  }

  private TaskContext mockTaskContext(String name, Instant startedAt) {
    TaskContext ctx = mock(TaskContext.class);
    when(ctx.taskName()).thenReturn(name);
    when(ctx.startedAt()).thenReturn(startedAt);
    when(ctx.position()).thenReturn(mock(io.serverlessworkflow.impl.WorkflowPosition.class));
    when(ctx.position().toString()).thenReturn("/" + name);
    return ctx;
  }

  private WorkflowContext mockWorkflowContext(String instanceId) {
    WorkflowContext ctx = mock(WorkflowContext.class);
    when(ctx.instance()).thenReturn(mock(WorkflowInstance.class));
    when(ctx.instance().id()).thenReturn(instanceId);
    return ctx;
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=PlaybookExecutionListenerTest`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement PlaybookExecutionListener**

Create `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionListener.java`:

```java
package io.casehub.engine.flow;

import io.casehub.api.model.StepResult;
import io.casehub.api.model.StepStatus;
import io.serverlessworkflow.impl.TaskContext;
import io.serverlessworkflow.impl.WorkflowInstance;
import io.serverlessworkflow.impl.lifecycle.TaskCompletedEvent;
import io.serverlessworkflow.impl.lifecycle.TaskFailedEvent;
import io.serverlessworkflow.impl.lifecycle.WorkflowExecutionListener;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Duration;
import java.time.Instant;

@ApplicationScoped
public class PlaybookExecutionListener implements WorkflowExecutionListener {

  private final FlowExecutionRegistry registry;

  @Inject
  public PlaybookExecutionListener(FlowExecutionRegistry registry) {
    this.registry = registry;
  }

  @Override
  public void onTaskCompleted(TaskCompletedEvent event) {
    String instanceId = event.workflowContext().instance().id();
    if (registry.find(instanceId).isEmpty()) {
      return;
    }
    TaskContext task = (TaskContext) event.taskContext();
    registry.recordStep(instanceId, new StepResult(
        task.position().toString(),
        task.taskName(),
        StepStatus.SUCCEEDED,
        elapsed(task.startedAt(), task.completedAt()),
        null));
  }

  @Override
  public void onTaskFailed(TaskFailedEvent event) {
    String instanceId = event.workflowContext().instance().id();
    if (registry.find(instanceId).isEmpty()) {
      return;
    }
    TaskContext task = (TaskContext) event.taskContext();
    registry.recordStep(instanceId, new StepResult(
        task.position().toString(),
        task.taskName(),
        StepStatus.FAILED,
        elapsed(task.startedAt(), task.completedAt()),
        event.cause() != null ? event.cause().getMessage() : "unknown"));
  }

  private Duration elapsed(Instant start, Instant end) {
    if (start == null || end == null) {
      return Duration.ZERO;
    }
    return Duration.between(start, end);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=PlaybookExecutionListenerTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionListener.java engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookExecutionListenerTest.java
git commit -m "feat(#1262): add PlaybookExecutionListener for step-level capture

CDI bean implementing WorkflowExecutionListener. Accumulates StepResult
entries in FlowExecutionRegistry. Auto-discovered by quarkus-flow.

Refs #1262"
```

---

### Task 7: PlaybookExecutionException + failure propagation in handler

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionException.java`
- Modify: `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:43`
- Modify: `engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `FlowExecutionRegistry.getSteps()` — reads accumulated step results after join()
- Produces: `PlaybookExecutionException(message, cause, executionResult)` — wraps failure with structured result
- Produces: Handler catches workflow exceptions and wraps in `PlaybookExecutionException`; success path populates `HandlerResult.executionResult`

- [ ] **Step 1: Write the failing test**

Add to `FlowWorkerFunctionHandlerTest.java`:

```java
@Test
void execute_wraps_failure_in_playbook_execution_exception() {
    final String instanceId = "wf-fail";
    final UUID caseId = UUID.randomUUID();

    final WorkflowInstance wfInstance = mockWorkflowInstance(instanceId);
    final CompletableFuture<WorkflowModel> failed = new CompletableFuture<>();
    failed.completeExceptionally(new RuntimeException("step failed"));
    when(wfInstance.start()).thenReturn(failed);

    stubApp(wfInstance);

    // Use a real registry so step data can be accumulated
    FlowExecutionRegistry realRegistry = new FlowExecutionRegistry();
    FlowWorkerFunctionHandler handlerWithRealRegistry =
        new FlowWorkerFunctionHandler(
            app, realRegistry, java.util.concurrent.Executors.newSingleThreadExecutor());

    final WorkerContext context =
        new WorkerContext("worker-E", caseId, null, null, PropagationContext.createRoot(), null);
    final ExecutionMetadata metadata = new ExecutionMetadata("worker-E", "hash-e");

    try {
      handlerWithRealRegistry.execute(
          new FlowWorkerFunction(mock(Workflow.class)), Map.of(), context, 60000, metadata);
      throw new AssertionError("Should have thrown");
    } catch (PlaybookExecutionException e) {
      assertThat(e.getExecutionResult()).isNotNull();
      assertThat(e.getExecutionResult()).isInstanceOf(PlaybookExecutionResult.Failure.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest#execute_wraps_failure_in_playbook_execution_exception`
Expected: FAIL — `PlaybookExecutionException` doesn't exist

- [ ] **Step 3: Create PlaybookExecutionException**

Create `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionException.java`:

```java
package io.casehub.engine.flow;

import io.casehub.api.model.PlaybookExecutionResult;

public class PlaybookExecutionException extends RuntimeException {

  private final PlaybookExecutionResult executionResult;

  public PlaybookExecutionException(String message, Throwable cause,
      PlaybookExecutionResult executionResult) {
    super(message, cause);
    this.executionResult = executionResult;
  }

  public PlaybookExecutionResult getExecutionResult() {
    return executionResult;
  }
}
```

- [ ] **Step 4: Update FlowWorkerFunctionHandler.execute() to catch and wrap failures**

Modify the `execute()` method. Currently the workflow runs and exceptions propagate directly. Wrap the execution in try-catch and build `PlaybookExecutionResult`:

Replace the body of `execute()` with logic that:
1. Runs the workflow as before
2. On success: reads steps from registry, builds `PlaybookExecutionResult.Success`, includes in `HandlerResult`
3. On failure: catches exception, reads steps from registry, builds `PlaybookExecutionResult.Failure`, throws `PlaybookExecutionException`

```java
@SuppressWarnings("unchecked")
@Override
public HandlerResult execute(
    final WorkerFunction<?, ?> function,
    final Object inputData,
    final WorkerContext context,
    final int timeoutMs,
    final ExecutionMetadata metadata) {
  final SWFWorkerFunction swf = (SWFWorkerFunction) function;
  final Map<String, Object> mapInput = (Map<String, Object>) inputData;

  WorkflowModel model;
  String instanceId;
  try {
    var future = executeWorkflow(
        swf.workflow(), mapInput, context.caseId(), metadata.workerName(), metadata.inputDataHash());
    instanceId = getLastRegisteredInstanceId(swf.workflow(), mapInput, context.caseId(),
        metadata.workerName(), metadata.inputDataHash());
    model = future.join();
  } catch (Exception e) {
    List<StepResult> steps = List.of();
    // Steps would have been accumulated by PlaybookExecutionListener
    // but we need the instanceId to retrieve them — store it during executeWorkflow
    throw new PlaybookExecutionException(
        e.getMessage(), e,
        new PlaybookExecutionResult.Failure(
            e.getCause() != null ? e.getCause().getMessage() : e.getMessage(), steps));
  }

  List<StepResult> steps = List.of(); // retrieved from registry in executeWorkflow cleanup
  Map<String, Object> outputMap = model.asMap()
      .orElseThrow(() -> new RuntimeException(
          "Workflow produced non-serializable model for worker: " + metadata.workerName()));

  Function<Map<String, Object>, PlannedAction> plannedActionFn = null;
  if (swf instanceof FlowWorkerFunction flow) {
    plannedActionFn = flow.plannedActionFn();
  } else if (swf instanceof PlaybookWorkerFunction pb) {
    plannedActionFn = pb.plannedActionFn();
  }

  PlaybookExecutionResult execResult = new PlaybookExecutionResult.Success(outputMap, steps);

  if (plannedActionFn != null) {
    return new HandlerResult(
        WorkerResult.of(outputMap, plannedActionFn.apply(mapInput)), Map.of(), execResult);
  }

  return new HandlerResult(WorkerResult.of(outputMap), Map.of(), execResult);
}
```

Note: The exact implementation will need to capture the `instanceId` from `executeWorkflow()` so steps can be read from the registry. The `executeWorkflow` method already has it — refactor to return both the future and the instanceId, or store the instanceId as a field on the future's metadata. Implementation detail to resolve during coding.

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl engine-support-core -Dtest=FlowWorkerFunctionHandlerTest`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookExecutionException.java engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java engine-support-core/src/test/java/io/casehub/engine/flow/FlowWorkerFunctionHandlerTest.java
git commit -m "feat(#1262): add PlaybookExecutionException and failure propagation

Handler catches workflow failures, builds PlaybookExecutionResult.Failure
from accumulated step data, and wraps in PlaybookExecutionException.
Success path populates HandlerResult.executionResult.

Refs #1262"
```

---

### Task 8: HandlerResult + RecoveryContext updates

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/HandlerResult.java:21`
- Modify: `common-core/src/main/java/io/casehub/engine/common/spi/recovery/RecoveryContext.java:21`
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java:108-117`
- Fix all callers of `RecoveryContext` and `HandlerResult` constructors (use `ide_find_references`)

**Interfaces:**
- Consumes: `PlaybookExecutionException.getExecutionResult()` — extract result from Throwable
- Produces: `HandlerResult(result, protocolMetadata, executionResult)` — 3-arg constructor
- Produces: `RecoveryContext(..., executionResult)` — 9-arg constructor with `PlaybookExecutionResult`

- [ ] **Step 1: Update HandlerResult record**

Add `PlaybookExecutionResult executionResult` field with backward-compatible constructors. Use `ide_find_references` to find all callers and verify they compile with the new field.

```java
public record HandlerResult(
    WorkerResult<?> result,
    Map<String, Object> protocolMetadata,
    PlaybookExecutionResult executionResult) {
  public HandlerResult(WorkerResult<?> result) {
    this(result, Map.of(), null);
  }
  public HandlerResult(WorkerResult<?> result, Map<String, Object> protocolMetadata) {
    this(result, protocolMetadata, null);
  }
}
```

- [ ] **Step 2: Update RecoveryContext record**

Add `PlaybookExecutionResult executionResult` as the 9th field:

```java
public record RecoveryContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    String workerName,
    String capabilityName,
    WorkerOutcome<?> outcome,
    int attemptCount,
    UUID planItemId,
    PlaybookExecutionResult executionResult) {}
```

- [ ] **Step 3: Fix all callers of RecoveryContext constructor**

Use `ide_find_references` on `RecoveryContext` to find all construction sites. Add `null` as the 9th argument to each existing call. The key call site is in `RetryOrchestrator.applyRetryDecision()` (line ~108) — update it to extract `PlaybookExecutionResult` from the `Throwable`:

In `RetryOrchestrator`, add a field for the `Throwable` passed to `handleFailure()`, and in `applyRetryDecision()` extract the result:

```java
PlaybookExecutionResult execResult = null;
if (cause instanceof PlaybookExecutionException pex) {
    execResult = pex.getExecutionResult();
}

RecoveryContext recoveryCtx = new RecoveryContext(
    taskData.caseId(),
    taskData.tenancyId(),
    taskData.bindingName(),
    taskData.workerId(),
    null,
    null,
    (int) failureCount,
    null,
    execResult);
```

Note: The `cause` Throwable needs to be passed from `handleFailure()` to `applyRetryDecision()`. Currently `handleFailure()` receives `Throwable cause` but `applyRetryDecision()` does not. Add the parameter.

- [ ] **Step 4: Run full common-core and runtime test suites**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common-core && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add common-core/src/main/java/io/casehub/engine/common/internal/executor/HandlerResult.java common-core/src/main/java/io/casehub/engine/common/spi/recovery/RecoveryContext.java common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java
git commit -m "feat(#1262): add PlaybookExecutionResult to HandlerResult and RecoveryContext

HandlerResult gains optional executionResult field.
RecoveryContext carries executionResult for triage.
RetryOrchestrator extracts result from PlaybookExecutionException.

Refs #1262"
```

---

## Batch 3: Triage Wiring (#1262 — recovery coordinator)

### Task 9: RecoveryPolicy + RecoveryOverride triage fields

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/RecoveryPolicy.java:18`
- Modify: `api/src/main/java/io/casehub/api/model/RecoveryOverride.java:20`
- Fix all callers via `ide_find_references`
- Test: `api/src/test/java/io/casehub/api/model/RecoveryPolicyTest.java`

**Interfaces:**
- Produces: `RecoveryPolicy.maxTriageAttempts()` (int, default 2)
- Produces: `RecoveryOverride.skipTriage()` (boolean, default false)
- Produces: `RecoveryOverride.effectiveMaxTriageAttempts(RecoveryPolicy)` (Integer → int fallthrough)

- [ ] **Step 1: Write the failing test**

Add to `RecoveryPolicyTest.java`:

```java
@Test
void default_policy_has_max_triage_attempts() {
    assertThat(RecoveryPolicy.DEFAULT.maxTriageAttempts()).isEqualTo(2);
}

@Test
void disabled_policy_has_zero_triage_attempts() {
    assertThat(RecoveryPolicy.DISABLED.maxTriageAttempts()).isEqualTo(0);
}
```

Create or add to `RecoveryOverrideTest.java`:

```java
@Test
void skip_triage_defaults_to_false() {
    var override = new RecoveryOverride(null, null, null, false, Set.of(), false, null);
    assertThat(override.skipTriage()).isFalse();
}

@Test
void effective_max_triage_attempts_falls_through_to_policy() {
    var override = new RecoveryOverride(null, null, null, false, Set.of(), false, null);
    assertThat(override.effectiveMaxTriageAttempts(RecoveryPolicy.DEFAULT)).isEqualTo(2);
}

@Test
void effective_max_triage_attempts_uses_override_when_set() {
    var override = new RecoveryOverride(null, null, null, false, Set.of(), false, 5);
    assertThat(override.effectiveMaxTriageAttempts(RecoveryPolicy.DEFAULT)).isEqualTo(5);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api -Dtest=RecoveryPolicyTest,RecoveryOverrideTest`
Expected: FAIL — fields don't exist

- [ ] **Step 3: Update RecoveryPolicy**

Add `maxTriageAttempts` field:

```java
public record RecoveryPolicy(
    int maxRetries,
    int maxRerouteAttempts,
    String classifierId,
    String revisionStrategyId,
    String replanStrategyId,
    boolean enabled,
    int maxTriageAttempts) {
  public static final RecoveryPolicy DEFAULT =
      new RecoveryPolicy(3, 3, "heuristic", "forward-replan", "llm", true, 2);
  public static final RecoveryPolicy DISABLED =
      new RecoveryPolicy(0, 0, "heuristic", "forward-replan", "llm", false, 0);
}
```

Fix all callers via `ide_find_references` — add the 7th argument to each construction site.

- [ ] **Step 4: Update RecoveryOverride**

Add `skipTriage` and `maxTriageAttempts` fields:

```java
public record RecoveryOverride(
    Integer maxRetries,
    Integer maxRerouteAttempts,
    RecoveryLevel maxLevel,
    boolean skipRecovery,
    Set<OutcomeType> skipRecoveryFor,
    boolean skipTriage,
    Integer maxTriageAttempts) {
  public RecoveryOverride {
    if (skipRecoveryFor == null) skipRecoveryFor = Set.of();
  }

  public static RecoveryOverride skip() {
    return new RecoveryOverride(null, null, null, true, Set.of(), false, null);
  }

  // existing methods unchanged...

  public int effectiveMaxTriageAttempts(RecoveryPolicy policy) {
    return maxTriageAttempts != null ? maxTriageAttempts : policy.maxTriageAttempts();
  }
}
```

Fix all callers via `ide_find_references`.

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api`
Expected: ALL PASS

- [ ] **Step 6: Run downstream modules to catch broken callers**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl common-core && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: ALL PASS (after fixing callers)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/RecoveryPolicy.java api/src/main/java/io/casehub/api/model/RecoveryOverride.java api/src/test/java/io/casehub/api/model/RecoveryPolicyTest.java api/src/test/java/io/casehub/api/model/RecoveryOverrideTest.java
git commit -m "feat(#1262): add maxTriageAttempts to RecoveryPolicy and RecoveryOverride

RecoveryPolicy gains maxTriageAttempts (default 2).
RecoveryOverride gains skipTriage and maxTriageAttempts with fallthrough.

Refs #1262"
```

---

### Task 10: BindingRecoveryState triage tracking

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/spi/recovery/BindingRecoveryState.java:23`
- Test: existing or new unit test for BindingRecoveryState

**Interfaces:**
- Produces: `triageAttemptCount()` → int, `incrementTriageAttemptCount()` → int
- Produces: `previousTriageModifications()` → `List<Map<String, Object>>`
- Produces: `recordTriageModifications(Map<String, Object>)` — appends to history

- [ ] **Step 1: Write the failing test**

```java
@Test
void tracks_triage_attempt_count() {
    var state = new BindingRecoveryState();
    assertThat(state.triageAttemptCount()).isZero();
    state.incrementTriageAttemptCount();
    assertThat(state.triageAttemptCount()).isEqualTo(1);
}

@Test
void records_triage_modification_history() {
    var state = new BindingRecoveryState();
    state.recordTriageModifications(Map.of("playbook.skipSteps", List.of("s1")));
    state.recordTriageModifications(Map.of("playbook.updateParams.timeout", 30000));

    assertThat(state.previousTriageModifications()).hasSize(2);
    assertThat(state.previousTriageModifications().getFirst()).containsKey("playbook.skipSteps");
}
```

- [ ] **Step 2: Add fields and methods to BindingRecoveryState**

```java
private final AtomicInteger triageAttemptCount = new AtomicInteger(0);
private final List<Map<String, Object>> triageModifications =
    java.util.Collections.synchronizedList(new java.util.ArrayList<>());

public int triageAttemptCount() {
    return triageAttemptCount.get();
}

public int incrementTriageAttemptCount() {
    return triageAttemptCount.incrementAndGet();
}

public List<Map<String, Object>> previousTriageModifications() {
    return List.copyOf(triageModifications);
}

public void recordTriageModifications(Map<String, Object> modifications) {
    triageModifications.add(Map.copyOf(modifications));
}
```

- [ ] **Step 3: Run test, verify pass, commit**

```bash
git add common-core/
git commit -m "feat(#1262): add triage attempt tracking to BindingRecoveryState

Refs #1262"
```

---

### Task 11: DefaultRecoveryCoordinator triage integration

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinator.java:61`
- Test: `runtime/src/test/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinatorTriageTest.java`

**Interfaces:**
- Consumes: `TriageHandler.evaluate(TriageContext)` → `TriageDecision`
- Consumes: `RecoveryContext.executionResult()` — structured failure data
- Consumes: `BindingRecoveryState.triageAttemptCount()`, `.previousTriageModifications()`
- Consumes: `RecoveryPolicy.maxTriageAttempts()`, `RecoveryOverride.skipTriage()`
- Produces: Triage flow after structural recovery exhaustion; publishes `CaseContextChangedEvent` on Retry

- [ ] **Step 1: Write the failing test**

Create `runtime/src/test/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinatorTriageTest.java`:

```java
package io.casehub.engine.runtime.recovery;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

import io.casehub.api.model.*;
import io.casehub.api.spi.recovery.ErrorClassifier;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import io.casehub.engine.common.internal.store.InMemoryPlanVersionStore;
import io.casehub.engine.common.spi.*;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.engine.common.spi.recovery.*;
import io.casehub.platform.api.governance.*;
import io.casehub.worker.api.*;
import io.vertx.core.eventbus.EventBus;
import jakarta.enterprise.inject.Instance;
import java.time.Duration;
import java.util.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class DefaultRecoveryCoordinatorTriageTest {

  private DefaultRecoveryCoordinator coordinator;
  private TriageHandler triageHandler;
  private EventBus eventBus;
  private ErrorClassifier classifier;
  private CaseRecoveryStateRegistry stateRegistry;
  private CaseInstanceCache caseInstanceCache;
  private CaseDefinitionRegistry definitionRegistry;

  @BeforeEach
  void setUp() {
    classifier = mock(ErrorClassifier.class);
    stateRegistry = new CaseRecoveryStateRegistry();
    triageHandler = mock(TriageHandler.class);
    eventBus = mock(EventBus.class);
    caseInstanceCache = mock(CaseInstanceCache.class);
    definitionRegistry = mock(CaseDefinitionRegistry.class);

    @SuppressWarnings("unchecked")
    Instance<TriageHandler> triageInstance = mock(Instance.class);
    when(triageInstance.isResolvable()).thenReturn(true);
    when(triageInstance.get()).thenReturn(triageHandler);

    @SuppressWarnings("unchecked")
    Instance<PlanItemStore> planItemStoreInstance = mock(Instance.class);
    when(planItemStoreInstance.isResolvable()).thenReturn(false);

    coordinator = new DefaultRecoveryCoordinator(
        classifier, stateRegistry, new InMemoryPlanVersionStore(),
        mock(PlanAdaptationEvaluator.class), mock(GoalDecomposer.class),
        definitionRegistry, caseInstanceCache,
        mock(EventLogRepository.class), eventBus, new CompoundLockRegistry(),
        planItemStoreInstance, triageInstance);
  }

  @Test
  void triage_fires_after_structural_recovery_exhaustion() {
    when(classifier.classify(any())).thenReturn(RecoveryLevel.FUNDAMENTAL);
    RecoveryContext ctx = buildContextWithFailure();
    setupRecoveryEnabled(ctx);
    stateRegistry.getOrCreate(ctx.caseId()).markReplanAttempted();

    when(triageHandler.evaluate(any())).thenReturn(
        new TriageDecision.Retry(Map.of("playbook.skipSteps", List.of("s2"))));

    boolean handled = coordinator.handleFailure(ctx);

    assertThat(handled).isTrue();
    verify(triageHandler).evaluate(any());
    verify(eventBus).publish(any(), any());
  }

  @Test
  void triage_returns_false_on_abandon() {
    when(classifier.classify(any())).thenReturn(RecoveryLevel.FUNDAMENTAL);
    RecoveryContext ctx = buildContextWithFailure();
    setupRecoveryEnabled(ctx);
    stateRegistry.getOrCreate(ctx.caseId()).markReplanAttempted();

    when(triageHandler.evaluate(any())).thenReturn(
        new TriageDecision.Abandon("no viable modification"));

    boolean handled = coordinator.handleFailure(ctx);

    assertThat(handled).isFalse();
    verify(triageHandler).evaluate(any());
  }

  @Test
  void triage_respects_max_attempts() {
    when(classifier.classify(any())).thenReturn(RecoveryLevel.FUNDAMENTAL);
    RecoveryContext ctx = buildContextWithFailure();
    setupRecoveryEnabled(ctx);
    stateRegistry.getOrCreate(ctx.caseId()).markReplanAttempted();
    var bindingState = stateRegistry.getOrCreate(ctx.caseId()).getOrCreate(ctx.bindingName());
    bindingState.incrementTriageAttemptCount();
    bindingState.incrementTriageAttemptCount();

    boolean handled = coordinator.handleFailure(ctx);

    assertThat(handled).isFalse();
    verify(triageHandler, never()).evaluate(any());
  }

  @Test
  void skip_triage_prevents_triage() {
    when(classifier.classify(any())).thenReturn(RecoveryLevel.FUNDAMENTAL);
    RecoveryContext ctx = buildContextWithFailure();
    setupRecoveryEnabledWithSkipTriage(ctx);
    stateRegistry.getOrCreate(ctx.caseId()).markReplanAttempted();

    boolean handled = coordinator.handleFailure(ctx);

    assertThat(handled).isFalse();
    verify(triageHandler, never()).evaluate(any());
  }

  private RecoveryContext buildContextWithFailure() {
    var step1 = new StepResult("s1", "fetch", StepStatus.SUCCEEDED, Duration.ofMillis(100), null);
    var step2 = new StepResult("s2", "transform", StepStatus.FAILED, Duration.ofMillis(10), "NPE");
    var execResult = new PlaybookExecutionResult.Failure("NPE in transform", List.of(step1, step2));
    return new RecoveryContext(
        UUID.randomUUID(), "tenant-1", "binding-1", "worker-1", "cap-1",
        new WorkerOutcome.Failed<>("failed"), 3, UUID.randomUUID(), execResult);
  }

  private void setupRecoveryEnabled(RecoveryContext ctx) {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("triage").version("1.0")
        .recoveryPolicy(RecoveryPolicy.DEFAULT)
        .capabilities(Capability.of("cap-1", ".", "."))
        .bindings(Binding.builder().name("binding-1")
            .capability(Capability.of("cap-1", ".", "."))
            .on(new ContextChangeTrigger(".ready")).build())
        .build();
    CaseMetaModel meta = new CaseMetaModel();
    meta.setNamespace("test"); meta.setName("triage"); meta.setVersion("1.0");
    CaseInstance instance = new CaseInstance();
    instance.setUuid(ctx.caseId()); instance.tenancyId = ctx.tenancyId();
    instance.setCaseMetaModel(meta);
    var caseContext = mock(io.casehub.api.context.MutableCaseContext.class);
    when(caseContext.snapshot()).thenReturn(caseContext);
    instance.setCaseContext(caseContext);
    when(caseInstanceCache.get(ctx.caseId())).thenReturn(instance);
    when(definitionRegistry.getCaseDefinition(meta)).thenReturn(def);
  }

  private void setupRecoveryEnabledWithSkipTriage(RecoveryContext ctx) {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("triage").version("1.0")
        .recoveryPolicy(RecoveryPolicy.DEFAULT)
        .capabilities(Capability.of("cap-1", ".", "."))
        .bindings(Binding.builder().name("binding-1")
            .capability(Capability.of("cap-1", ".", "."))
            .on(new ContextChangeTrigger(".ready"))
            .recoveryOverride(new RecoveryOverride(null, null, null, false, Set.of(), true, null))
            .build())
        .build();
    // ... same instance/meta setup as above
    CaseMetaModel meta = new CaseMetaModel();
    meta.setNamespace("test"); meta.setName("triage"); meta.setVersion("1.0");
    CaseInstance instance = new CaseInstance();
    instance.setUuid(ctx.caseId()); instance.tenancyId = ctx.tenancyId();
    instance.setCaseMetaModel(meta);
    var caseContext = mock(io.casehub.api.context.MutableCaseContext.class);
    when(caseContext.snapshot()).thenReturn(caseContext);
    instance.setCaseContext(caseContext);
    when(caseInstanceCache.get(ctx.caseId())).thenReturn(instance);
    when(definitionRegistry.getCaseDefinition(meta)).thenReturn(def);
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=DefaultRecoveryCoordinatorTriageTest`
Expected: FAIL — constructor doesn't accept `Instance<TriageHandler>`

- [ ] **Step 3: Implement triage integration in DefaultRecoveryCoordinator**

Key changes:
1. Add `Instance<TriageHandler> triageHandler` to constructor
2. Restructure `handleFailure()`: `skipRecovery` skips structural levels but falls through to triage
3. After structural recovery fails, check `skipTriage` and `maxTriageAttempts`
4. Build `TriageContext` from `RecoveryContext`
5. On `Retry`: apply modifications to context, publish `CaseContextChangedEvent`, return true
6. On `Abandon`: fire `CoordinationExhaustedEvent`, return false

This is the largest single implementation step. Follow the pseudocode in spec §3.6 exactly.

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime -Dtest=DefaultRecoveryCoordinatorTriageTest`
Expected: ALL PASS

- [ ] **Step 5: Run full runtime test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl runtime`
Expected: ALL PASS (existing DefaultRecoveryCoordinatorTest must still pass)

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinator.java runtime/src/test/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinatorTriageTest.java
git commit -m "feat(#1262): wire TriageHandler into DefaultRecoveryCoordinator

Triage fires after structural recovery exhaustion. Builds TriageContext
from RecoveryContext + PlaybookExecutionResult. On Retry: applies
modifications, publishes CaseContextChangedEvent. On Abandon: fires
CoordinationExhaustedEvent.

Refs #1262"
```

---

### Task 12: PlaybookTriageModification + Parser

**Files:**
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModification.java`
- Create: `engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationParser.java`
- Test: `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationParserTest.java`

**Interfaces:**
- Consumes: `Map<String, Object>` from `TriageDecision.Retry.modifications()`
- Produces: `List<PlaybookTriageModification>` — typed sealed hierarchy (SkipStep, ReplaceStepInput, UpdateParameter)

- [ ] **Step 1: Write the failing test**

Create `engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationParserTest.java`:

```java
package io.casehub.engine.flow;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class PlaybookTriageModificationParserTest {

  @Test
  void parses_skip_steps() {
    Map<String, Object> mods = Map.of("playbook.skipSteps", List.of("step-3", "step-5"));
    List<PlaybookTriageModification> result = PlaybookTriageModificationParser.parse(mods);

    assertThat(result).hasSize(2);
    assertThat(result.getFirst()).isInstanceOf(PlaybookTriageModification.SkipStep.class);
    assertThat(((PlaybookTriageModification.SkipStep) result.getFirst()).stepId()).isEqualTo("step-3");
  }

  @Test
  void parses_replace_input() {
    Map<String, Object> mods = Map.of(
        "playbook.replaceInput.step-2", Map.of("url", "/api/v2/contacts"));
    List<PlaybookTriageModification> result = PlaybookTriageModificationParser.parse(mods);

    assertThat(result).hasSize(1);
    assertThat(result.getFirst()).isInstanceOf(PlaybookTriageModification.ReplaceStepInput.class);
    var replace = (PlaybookTriageModification.ReplaceStepInput) result.getFirst();
    assertThat(replace.stepId()).isEqualTo("step-2");
    assertThat(replace.newInput()).containsEntry("url", "/api/v2/contacts");
  }

  @Test
  void parses_update_parameter() {
    Map<String, Object> mods = Map.of("playbook.updateParams.timeout", 30000);
    List<PlaybookTriageModification> result = PlaybookTriageModificationParser.parse(mods);

    assertThat(result).hasSize(1);
    assertThat(result.getFirst()).isInstanceOf(PlaybookTriageModification.UpdateParameter.class);
    var update = (PlaybookTriageModification.UpdateParameter) result.getFirst();
    assertThat(update.key()).isEqualTo("timeout");
    assertThat(update.value()).isEqualTo(30000);
  }

  @Test
  void ignores_unknown_keys() {
    Map<String, Object> mods = Map.of("unknown.key", "value", "playbook.skipSteps", List.of("s1"));
    List<PlaybookTriageModification> result = PlaybookTriageModificationParser.parse(mods);

    assertThat(result).hasSize(1);
  }

  @Test
  void empty_map_returns_empty_list() {
    assertThat(PlaybookTriageModificationParser.parse(Map.of())).isEmpty();
  }
}
```

- [ ] **Step 2: Implement sealed interface and parser**

Create `PlaybookTriageModification.java`:

```java
package io.casehub.engine.flow;

import java.util.Map;

public sealed interface PlaybookTriageModification
    permits PlaybookTriageModification.SkipStep,
            PlaybookTriageModification.ReplaceStepInput,
            PlaybookTriageModification.UpdateParameter {

  record SkipStep(String stepId) implements PlaybookTriageModification {}
  record ReplaceStepInput(String stepId, Map<String, Object> newInput) implements PlaybookTriageModification {}
  record UpdateParameter(String key, Object value) implements PlaybookTriageModification {}
}
```

Create `PlaybookTriageModificationParser.java`:

```java
package io.casehub.engine.flow;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public final class PlaybookTriageModificationParser {

  private static final String PREFIX = "playbook.";
  private static final String SKIP_STEPS = PREFIX + "skipSteps";
  private static final String REPLACE_INPUT_PREFIX = PREFIX + "replaceInput.";
  private static final String UPDATE_PARAMS_PREFIX = PREFIX + "updateParams.";

  private PlaybookTriageModificationParser() {}

  @SuppressWarnings("unchecked")
  public static List<PlaybookTriageModification> parse(Map<String, Object> modifications) {
    List<PlaybookTriageModification> result = new ArrayList<>();

    Object skipSteps = modifications.get(SKIP_STEPS);
    if (skipSteps instanceof List<?> stepIds) {
      for (Object id : stepIds) {
        result.add(new PlaybookTriageModification.SkipStep(id.toString()));
      }
    }

    for (var entry : modifications.entrySet()) {
      String key = entry.getKey();
      if (key.startsWith(REPLACE_INPUT_PREFIX) && entry.getValue() instanceof Map<?, ?> map) {
        String stepId = key.substring(REPLACE_INPUT_PREFIX.length());
        result.add(new PlaybookTriageModification.ReplaceStepInput(stepId, (Map<String, Object>) map));
      } else if (key.startsWith(UPDATE_PARAMS_PREFIX)) {
        String paramKey = key.substring(UPDATE_PARAMS_PREFIX.length());
        result.add(new PlaybookTriageModification.UpdateParameter(paramKey, entry.getValue()));
      }
    }

    return List.copyOf(result);
  }
}
```

- [ ] **Step 3: Run test, verify pass, commit**

```bash
git add engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModification.java engine-support-core/src/main/java/io/casehub/engine/flow/PlaybookTriageModificationParser.java engine-support-core/src/test/java/io/casehub/engine/flow/PlaybookTriageModificationParserTest.java
git commit -m "feat(#1262): add PlaybookTriageModification sealed hierarchy and parser

Converts Map<String, Object> from TriageDecision.Retry into typed
modifications: SkipStep, ReplaceStepInput, UpdateParameter.

Refs #1262"
```

---

### Task 13: Retry count partitioning (triageRound discriminator)

**Files:**
- Modify: `common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java`
- Test: existing RetryOrchestrator tests + new test for triageRound filtering

**Interfaces:**
- Consumes: `RecoveryContext.executionResult()` — indicates whether triage has been involved
- Produces: `triageRound` metadata in event logs, filtering in `countFailedAttempts()`

- [ ] **Step 1: Write the failing test**

Add to RetryOrchestrator test file — a test that verifies failure count partitioning by triage round. The test should verify that after a triage round, the failure count resets to 0 for the new round.

- [ ] **Step 2: Add triageRound to persistFailureEventLog metadata**

In `persistFailureEventLog()`, include a `triageRound` field in the metadata JSON:

```java
.put("triageRound", taskData.triageRound())
```

This requires adding `triageRound` to `WorkerTaskData` — or alternatively, passing it through the event log metadata from `BindingRecoveryState`.

- [ ] **Step 3: Filter countFailedAttempts by triageRound**

In `countFailedAttempts()`, filter event logs to only count failures from the current triage round:

```java
.filter(eventLog -> {
    JsonNode metadata = eventLog.getMetadata();
    JsonNode hashNode = metadata == null ? null : metadata.get("inputDataHash");
    JsonNode roundNode = metadata == null ? null : metadata.get("triageRound");
    int round = roundNode != null ? roundNode.asInt(0) : 0;
    return hashNode != null
        && taskData.inputDataHash().equals(hashNode.asText())
        && round == currentTriageRound;
})
```

- [ ] **Step 4: Run test, verify pass, commit**

```bash
git add common-core/
git commit -m "feat(#1262): partition retry counts by triage round

Prevents re-dispatched workers from immediately exhausting after triage
modifies context. Each triage round starts with a fresh failure count.

Refs #1262"
```

---

## Batch 4: CBR Recording (#1262)

### Task 14: PlaybookOutcomeRecorder

**Files:**
- Create: `yaml-cbr/src/main/java/io/casehub/engine/yamlcbr/PlaybookOutcomeRecorder.java`
- Test: `yaml-cbr/src/test/java/io/casehub/engine/yamlcbr/PlaybookOutcomeRecorderTest.java`

**Interfaces:**
- Consumes: `PlaybookExecutionResult` — step-level execution outcomes
- Consumes: `CbrRecordStore` SPI — same store used by `StepExecutionCbrBridge`
- Produces: Per-execution CBR records with full step trace and triage chain context

- [ ] **Step 1: Write the failing test**

Create `yaml-cbr/src/test/java/io/casehub/engine/yamlcbr/PlaybookOutcomeRecorderTest.java` following the `StepExecutionCbrBridgeTest` pattern — use a `RecordingCbrStore` stub, construct `PlaybookExecutionResult.Success` and `.Failure`, verify records are stored with correct features.

- [ ] **Step 2: Implement PlaybookOutcomeRecorder**

CDI bean that observes playbook execution events and stores per-execution records in `CbrRecordStore`. Records include:
- All step results (names, statuses, durations)
- Overall outcome (success/partial/failure)
- Triage chain context (modifications applied, attempt count) when present
- `executionChainId` feature linking triage attempts together

- [ ] **Step 3: Run test, verify pass, commit**

```bash
git add yaml-cbr/
git commit -m "feat(#1262): add PlaybookOutcomeRecorder for per-execution CBR recording

Dual-granularity alongside StepExecutionCbrBridge. Records full execution
arcs with step trace and triage chain context.

Refs #1262"
```

---

## References

- [2026-10-28-playbook-failure-triage-design.md] — design spec this plan implements
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:43` — handler execution entry point
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java:24` — existing function type
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionProvider.java:34` — existing provider pattern
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowExecutionRegistry.java:26` — step data storage target
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java:44` — failure handling, triage round partitioning
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/HandlerResult.java:21` — gains executionResult
- `common-core/src/main/java/io/casehub/engine/common/spi/recovery/RecoveryContext.java:21` — gains executionResult
- `common-core/src/main/java/io/casehub/engine/common/spi/recovery/BindingRecoveryState.java:23` — triage tracking
- `runtime/src/main/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinator.java:61` — triage insertion point
- `api/src/main/java/io/casehub/api/model/RecoveryPolicy.java:18` — maxTriageAttempts
- `api/src/main/java/io/casehub/api/model/RecoveryOverride.java:20` — skipTriage, maxTriageAttempts
- `yaml-cbr/src/main/java/io/casehub/engine/yamlcbr/StepExecutionCbrBridge.java:34` — CBR pattern to follow
- Protocol: `api-surface-parity.md` — PlaybookExecutionResult in engine-api
- Protocol: `plan-type-module-boundary.md` — definition types in api, execution in common/runtime
- GitHub #1262, #1263, parent#536, platform#668
