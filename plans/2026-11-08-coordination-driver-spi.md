# CoordinationDriver SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1297 — CoordinationDriver SPI — COORDINATOR behavioral contract
**Issue group:** #1297

**Goal:** Define the engine-api behavioral contract for PlatformRole.COORDINATOR with vocabulary types, marker interface, SPI, and default implementation.

**Architecture:** Two-layer pattern (CoordinatorBehavior marker in model.coordination + CoordinationDriver SPI in spi.coordination) with five supporting vocabulary types (CoordinationTask, DelegationAssignment, DelegationOutcome, CoordinationResult, CoordinationLifecycleEvent). NoOpCoordinationDriver default in runtime-core, wired via SpiDefaultBeans.

**Tech Stack:** Java 21 (sealed interfaces, records, pattern matching), JUnit 5, AssertJ

## Global Constraints

- Marker interface in `io.casehub.api.model.coordination` (api module)
- SPI in `io.casehub.api.spi.coordination` (api module)
- Vocabulary types in `io.casehub.api.model.coordination` (api module)
- NoOp default in `io.casehub.engine.runtime.coordination` (runtime-core module)
- Wiring in `SpiDefaultBeans` (runtime module) and Spring auto-config (generated)
- Follow existing test conventions: package-private class, `snake_case` methods, AssertJ
- All commits reference `Refs #1297` or `Closes #1297`
- Run spotless before committing: `mvn spotless:apply -pl api -q` (or `-pl runtime-core` / `-pl runtime`)

---

## Batch 1: Vocabulary Types

### Task 1: CoordinationTask + DelegationAssignment + DelegationOutcome

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/CoordinationTask.java`
- Create: `api/src/main/java/io/casehub/api/model/coordination/DelegationAssignment.java`
- Create: `api/src/main/java/io/casehub/api/model/coordination/DelegationOutcome.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/CoordinationTaskTest.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/DelegationAssignmentTest.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/DelegationOutcomeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `CoordinationTask(String taskId, String description, @Nullable String targetAgentConstraint, @Nullable String aggregationHint)`, `DelegationAssignment(String agentId, CoordinationTask task)`, `DelegationOutcome(String agentId, DelegationStatus status, @Nullable String payload)` with `DelegationStatus` enum

- [ ] **Step 1: Write the failing tests**

Create `api/src/test/java/io/casehub/api/model/coordination/CoordinationTaskTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatIllegalArgumentException;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import org.junit.jupiter.api.Test;

class CoordinationTaskTest {

  @Test
  void valid_construction_all_fields() {
    var task = new CoordinationTask("t-1", "analyse input", "reviewer", "merge");
    assertThat(task.taskId()).isEqualTo("t-1");
    assertThat(task.description()).isEqualTo("analyse input");
    assertThat(task.targetAgentConstraint()).isEqualTo("reviewer");
    assertThat(task.aggregationHint()).isEqualTo("merge");
  }

  @Test
  void valid_construction_optional_fields_null() {
    var task = new CoordinationTask("t-2", "summarise", null, null);
    assertThat(task.targetAgentConstraint()).isNull();
    assertThat(task.aggregationHint()).isNull();
  }

  @Test
  void null_taskId_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationTask(null, "desc", null, null))
        .withMessageContaining("taskId");
  }

  @Test
  void blank_taskId_rejected() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new CoordinationTask("  ", "desc", null, null))
        .withMessageContaining("taskId");
  }

  @Test
  void null_description_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationTask("t-1", null, null, null))
        .withMessageContaining("description");
  }
}
```

Create `api/src/test/java/io/casehub/api/model/coordination/DelegationAssignmentTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import org.junit.jupiter.api.Test;

class DelegationAssignmentTest {

  private static final CoordinationTask TASK =
      new CoordinationTask("t-1", "analyse", null, null);

  @Test
  void valid_construction() {
    var assignment = new DelegationAssignment("agent-a", TASK);
    assertThat(assignment.agentId()).isEqualTo("agent-a");
    assertThat(assignment.task()).isEqualTo(TASK);
  }

  @Test
  void null_agentId_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new DelegationAssignment(null, TASK))
        .withMessageContaining("agentId");
  }

  @Test
  void null_task_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new DelegationAssignment("agent-a", null))
        .withMessageContaining("task");
  }
}
```

Create `api/src/test/java/io/casehub/api/model/coordination/DelegationOutcomeTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import org.junit.jupiter.api.Test;

class DelegationOutcomeTest {

  @Test
  void valid_construction_completed() {
    var outcome =
        new DelegationOutcome("agent-a", DelegationOutcome.DelegationStatus.COMPLETED, "result");
    assertThat(outcome.agentId()).isEqualTo("agent-a");
    assertThat(outcome.status()).isEqualTo(DelegationOutcome.DelegationStatus.COMPLETED);
    assertThat(outcome.payload()).isEqualTo("result");
  }

  @Test
  void valid_construction_failed_null_payload() {
    var outcome =
        new DelegationOutcome("agent-b", DelegationOutcome.DelegationStatus.FAILED, null);
    assertThat(outcome.payload()).isNull();
  }

  @Test
  void valid_construction_partial() {
    var outcome =
        new DelegationOutcome("agent-c", DelegationOutcome.DelegationStatus.PARTIAL, "partial");
    assertThat(outcome.status()).isEqualTo(DelegationOutcome.DelegationStatus.PARTIAL);
  }

  @Test
  void null_agentId_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () ->
                new DelegationOutcome(null, DelegationOutcome.DelegationStatus.COMPLETED, "data"))
        .withMessageContaining("agentId");
  }

  @Test
  void null_status_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new DelegationOutcome("agent-a", null, "data"))
        .withMessageContaining("status");
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CoordinationTaskTest,DelegationAssignmentTest,DelegationOutcomeTest" -q`
Expected: compilation error — types do not exist

- [ ] **Step 3: Implement the three types**

Create `api/src/main/java/io/casehub/api/model/coordination/CoordinationTask.java`:

```java
package io.casehub.api.model.coordination;

import jakarta.annotation.Nullable;
import java.util.Objects;

public record CoordinationTask(
    String taskId,
    String description,
    @Nullable String targetAgentConstraint,
    @Nullable String aggregationHint) {

  public CoordinationTask {
    Objects.requireNonNull(taskId, "taskId must not be null");
    if (taskId.isBlank()) throw new IllegalArgumentException("taskId must not be blank");
    Objects.requireNonNull(description, "description must not be null");
  }
}
```

Create `api/src/main/java/io/casehub/api/model/coordination/DelegationAssignment.java`:

```java
package io.casehub.api.model.coordination;

import java.util.Objects;

public record DelegationAssignment(String agentId, CoordinationTask task) {

  public DelegationAssignment {
    Objects.requireNonNull(agentId, "agentId must not be null");
    Objects.requireNonNull(task, "task must not be null");
  }
}
```

Create `api/src/main/java/io/casehub/api/model/coordination/DelegationOutcome.java`:

```java
package io.casehub.api.model.coordination;

import jakarta.annotation.Nullable;
import java.util.Objects;

public record DelegationOutcome(
    String agentId, DelegationStatus status, @Nullable String payload) {

  public DelegationOutcome {
    Objects.requireNonNull(agentId, "agentId must not be null");
    Objects.requireNonNull(status, "status must not be null");
  }

  public enum DelegationStatus {
    COMPLETED,
    FAILED,
    PARTIAL
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CoordinationTaskTest,DelegationAssignmentTest,DelegationOutcomeTest" -q`
Expected: all tests PASS

- [ ] **Step 5: Run spotless and commit**

```bash
mvn spotless:apply -pl api -q
git add api/src/main/java/io/casehub/api/model/coordination/CoordinationTask.java \
        api/src/main/java/io/casehub/api/model/coordination/DelegationAssignment.java \
        api/src/main/java/io/casehub/api/model/coordination/DelegationOutcome.java \
        api/src/test/java/io/casehub/api/model/coordination/CoordinationTaskTest.java \
        api/src/test/java/io/casehub/api/model/coordination/DelegationAssignmentTest.java \
        api/src/test/java/io/casehub/api/model/coordination/DelegationOutcomeTest.java
git commit -m "feat: add CoordinationTask, DelegationAssignment, DelegationOutcome records

Refs #1297"
```

### Task 2: CoordinationResult + CoordinationLifecycleEvent sealed interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/CoordinationResult.java`
- Create: `api/src/main/java/io/casehub/api/model/coordination/CoordinationLifecycleEvent.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/CoordinationResultTest.java`
- Test: `api/src/test/java/io/casehub/api/model/coordination/CoordinationLifecycleEventTest.java`

**Interfaces:**
- Consumes: `TopologyDescriptor` (existing)
- Produces: `CoordinationResult` sealed interface (Aggregated, Partial, Failed), `CoordinationLifecycleEvent` sealed interface (Started, Stopped, Reconfigured)

- [ ] **Step 1: Write the failing tests**

Create `api/src/test/java/io/casehub/api/model/coordination/CoordinationResultTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.util.List;
import org.junit.jupiter.api.Test;

class CoordinationResultTest {

  @Test
  void sealed_interface_permits_three_variants() {
    assertThat(CoordinationResult.class.getPermittedSubclasses())
        .hasSize(3)
        .extracting(Class::getSimpleName)
        .containsExactlyInAnyOrder("Aggregated", "Partial", "Failed");
  }

  @Test
  void aggregated_valid_construction() {
    var result = new CoordinationResult.Aggregated("merged output", List.of("a-1", "a-2"));
    assertThat(result.payload()).isEqualTo("merged output");
    assertThat(result.contributors()).containsExactly("a-1", "a-2");
  }

  @Test
  void aggregated_null_payload_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationResult.Aggregated(null, List.of()))
        .withMessageContaining("payload");
  }

  @Test
  void aggregated_defensive_copy() {
    var list = new java.util.ArrayList<>(List.of("a-1"));
    var result = new CoordinationResult.Aggregated("output", list);
    list.add("a-2");
    assertThat(result.contributors()).hasSize(1);
  }

  @Test
  void partial_valid_construction() {
    var result = new CoordinationResult.Partial("interim", List.of("a-1"), List.of("a-2"));
    assertThat(result.payload()).isEqualTo("interim");
    assertThat(result.completed()).containsExactly("a-1");
    assertThat(result.pending()).containsExactly("a-2");
  }

  @Test
  void failed_valid_construction() {
    var result = new CoordinationResult.Failed("timeout", List.of("a-3"));
    assertThat(result.reason()).isEqualTo("timeout");
    assertThat(result.failedAgents()).containsExactly("a-3");
  }

  @Test
  void failed_null_reason_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationResult.Failed(null, List.of()))
        .withMessageContaining("reason");
  }
}
```

Create `api/src/test/java/io/casehub/api/model/coordination/CoordinationLifecycleEventTest.java`:

```java
package io.casehub.api.model.coordination;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatNullPointerException;

import java.util.Map;
import org.junit.jupiter.api.Test;

class CoordinationLifecycleEventTest {

  private static final TopologyDescriptor TOPOLOGY =
      new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());

  @Test
  void sealed_interface_permits_three_variants() {
    assertThat(CoordinationLifecycleEvent.class.getPermittedSubclasses())
        .hasSize(3)
        .extracting(Class::getSimpleName)
        .containsExactlyInAnyOrder("Started", "Stopped", "Reconfigured");
  }

  @Test
  void started_valid_construction() {
    var event = new CoordinationLifecycleEvent.Started("coord-1", TOPOLOGY);
    assertThat(event.coordinatorId()).isEqualTo("coord-1");
    assertThat(event.topology()).isEqualTo(TOPOLOGY);
  }

  @Test
  void started_null_coordinatorId_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationLifecycleEvent.Started(null, TOPOLOGY));
  }

  @Test
  void started_null_topology_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationLifecycleEvent.Started("coord-1", null));
  }

  @Test
  void stopped_valid_construction() {
    var event = new CoordinationLifecycleEvent.Stopped("coord-1", "case completed");
    assertThat(event.coordinatorId()).isEqualTo("coord-1");
    assertThat(event.reason()).isEqualTo("case completed");
  }

  @Test
  void stopped_null_reason_rejected() {
    assertThatNullPointerException()
        .isThrownBy(() -> new CoordinationLifecycleEvent.Stopped("coord-1", null));
  }

  @Test
  void reconfigured_valid_construction() {
    var newTopology = new TopologyDescriptor(TopologyType.DISTRIBUTED, Map.of());
    var event =
        new CoordinationLifecycleEvent.Reconfigured("coord-1", TOPOLOGY, newTopology);
    assertThat(event.previousTopology()).isEqualTo(TOPOLOGY);
    assertThat(event.newTopology()).isEqualTo(newTopology);
  }

  @Test
  void reconfigured_null_previous_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () -> new CoordinationLifecycleEvent.Reconfigured("coord-1", null, TOPOLOGY));
  }

  @Test
  void reconfigured_null_new_rejected() {
    assertThatNullPointerException()
        .isThrownBy(
            () -> new CoordinationLifecycleEvent.Reconfigured("coord-1", TOPOLOGY, null));
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CoordinationResultTest,CoordinationLifecycleEventTest" -q`
Expected: compilation error — types do not exist

- [ ] **Step 3: Implement the sealed interfaces**

Create `api/src/main/java/io/casehub/api/model/coordination/CoordinationResult.java`:

```java
package io.casehub.api.model.coordination;

import java.util.List;
import java.util.Objects;

public sealed interface CoordinationResult
    permits CoordinationResult.Aggregated,
        CoordinationResult.Partial,
        CoordinationResult.Failed {

  record Aggregated(String payload, List<String> contributors) implements CoordinationResult {
    public Aggregated {
      Objects.requireNonNull(payload, "payload must not be null");
      contributors = contributors != null ? List.copyOf(contributors) : List.of();
    }
  }

  record Partial(String payload, List<String> completed, List<String> pending)
      implements CoordinationResult {
    public Partial {
      Objects.requireNonNull(payload, "payload must not be null");
      completed = completed != null ? List.copyOf(completed) : List.of();
      pending = pending != null ? List.copyOf(pending) : List.of();
    }
  }

  record Failed(String reason, List<String> failedAgents) implements CoordinationResult {
    public Failed {
      Objects.requireNonNull(reason, "reason must not be null");
      failedAgents = failedAgents != null ? List.copyOf(failedAgents) : List.of();
    }
  }
}
```

Create `api/src/main/java/io/casehub/api/model/coordination/CoordinationLifecycleEvent.java`:

```java
package io.casehub.api.model.coordination;

import java.util.Objects;

public sealed interface CoordinationLifecycleEvent
    permits CoordinationLifecycleEvent.Started,
        CoordinationLifecycleEvent.Stopped,
        CoordinationLifecycleEvent.Reconfigured {

  record Started(String coordinatorId, TopologyDescriptor topology)
      implements CoordinationLifecycleEvent {
    public Started {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(topology, "topology must not be null");
    }
  }

  record Stopped(String coordinatorId, String reason) implements CoordinationLifecycleEvent {
    public Stopped {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(reason, "reason must not be null");
    }
  }

  record Reconfigured(
      String coordinatorId, TopologyDescriptor previousTopology, TopologyDescriptor newTopology)
      implements CoordinationLifecycleEvent {
    public Reconfigured {
      Objects.requireNonNull(coordinatorId, "coordinatorId must not be null");
      Objects.requireNonNull(previousTopology, "previousTopology must not be null");
      Objects.requireNonNull(newTopology, "newTopology must not be null");
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CoordinationResultTest,CoordinationLifecycleEventTest" -q`
Expected: all tests PASS

- [ ] **Step 5: Run spotless and commit**

```bash
mvn spotless:apply -pl api -q
git add api/src/main/java/io/casehub/api/model/coordination/CoordinationResult.java \
        api/src/main/java/io/casehub/api/model/coordination/CoordinationLifecycleEvent.java \
        api/src/test/java/io/casehub/api/model/coordination/CoordinationResultTest.java \
        api/src/test/java/io/casehub/api/model/coordination/CoordinationLifecycleEventTest.java
git commit -m "feat: add CoordinationResult and CoordinationLifecycleEvent sealed interfaces

Refs #1297"
```

## Batch 2: Marker + SPI + Default + Wiring

### Task 3: CoordinatorBehavior marker + CoordinationDriver SPI + NoOp + wiring

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/coordination/CoordinatorBehavior.java`
- Create: `api/src/main/java/io/casehub/api/spi/coordination/CoordinationDriver.java`
- Create: `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpCoordinationDriver.java`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java:305-310` — add factory method
- Modify: `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:25` — add Javadoc to COORDINATOR
- Test: `api/src/test/java/io/casehub/api/spi/coordination/CoordinationDriverTest.java`

**Interfaces:**
- Consumes: `CoordinationTask`, `DelegationAssignment`, `DelegationOutcome`, `CoordinationResult`, `CoordinationLifecycleEvent` (Task 1-2), `TopologyDescriptor` (existing)
- Produces: `CoordinatorBehavior` marker, `CoordinationDriver` SPI (delegate, aggregate, allocate, onLifecycleEvent), `NoOpCoordinationDriver`

- [ ] **Step 1: Write the failing tests**

Create `api/src/test/java/io/casehub/api/spi/coordination/CoordinationDriverTest.java`:

```java
package io.casehub.api.spi.coordination;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.coordination.CoordinationLifecycleEvent;
import io.casehub.api.model.coordination.CoordinationResult;
import io.casehub.api.model.coordination.CoordinationTask;
import io.casehub.api.model.coordination.CoordinatorBehavior;
import io.casehub.api.model.coordination.DelegationOutcome;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.model.coordination.TopologyType;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class CoordinationDriverTest {

  private static final TopologyDescriptor TOPOLOGY =
      new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
  private static final CoordinationTask TASK =
      new CoordinationTask("t-1", "analyse input", null, null);

  @Test
  void coordination_driver_extends_coordinator_behavior() {
    assertThat(CoordinatorBehavior.class).isAssignableFrom(CoordinationDriver.class);
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CoordinationDriverTest -q`
Expected: compilation error — `CoordinationDriver` does not exist

- [ ] **Step 3: Create CoordinatorBehavior marker**

Create `api/src/main/java/io/casehub/api/model/coordination/CoordinatorBehavior.java`:

```java
package io.casehub.api.model.coordination;

/**
 * Marker for components that implement {@link PlatformRole#COORDINATOR} behavior — the ability to
 * delegate tasks, allocate agents, aggregate results, and manage coordination lifecycle.
 *
 * @see PlatformRole#COORDINATOR
 * @see io.casehub.api.spi.coordination.CoordinationDriver
 */
public interface CoordinatorBehavior {}
```

- [ ] **Step 4: Create CoordinationDriver SPI**

Create `api/src/main/java/io/casehub/api/spi/coordination/CoordinationDriver.java`:

```java
package io.casehub.api.spi.coordination;

import io.casehub.api.model.coordination.CoordinationLifecycleEvent;
import io.casehub.api.model.coordination.CoordinationResult;
import io.casehub.api.model.coordination.CoordinationTask;
import io.casehub.api.model.coordination.CoordinatorBehavior;
import io.casehub.api.model.coordination.DelegationAssignment;
import io.casehub.api.model.coordination.DelegationOutcome;
import io.casehub.api.model.coordination.TopologyDescriptor;
import java.util.List;

public interface CoordinationDriver extends CoordinatorBehavior {

  List<DelegationAssignment> delegate(CoordinationTask task, TopologyDescriptor topology);

  CoordinationResult aggregate(List<DelegationOutcome> outcomes, CoordinationTask originalTask);

  List<String> allocate(CoordinationTask task, TopologyDescriptor topology);

  void onLifecycleEvent(CoordinationLifecycleEvent event);
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CoordinationDriverTest -q`
Expected: PASS

- [ ] **Step 6: Create NoOpCoordinationDriver**

Create `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpCoordinationDriver.java`:

```java
package io.casehub.engine.runtime.coordination;

import io.casehub.api.model.coordination.CoordinationLifecycleEvent;
import io.casehub.api.model.coordination.CoordinationResult;
import io.casehub.api.model.coordination.CoordinationTask;
import io.casehub.api.model.coordination.DelegationAssignment;
import io.casehub.api.model.coordination.DelegationOutcome;
import io.casehub.api.model.coordination.TopologyDescriptor;
import io.casehub.api.spi.coordination.CoordinationDriver;
import java.util.List;

public class NoOpCoordinationDriver implements CoordinationDriver {

  @Override
  public List<DelegationAssignment> delegate(CoordinationTask task, TopologyDescriptor topology) {
    return List.of();
  }

  @Override
  public CoordinationResult aggregate(
      List<DelegationOutcome> outcomes, CoordinationTask originalTask) {
    return new CoordinationResult.Failed("no coordinator configured", List.of());
  }

  @Override
  public List<String> allocate(CoordinationTask task, TopologyDescriptor topology) {
    return List.of();
  }

  @Override
  public void onLifecycleEvent(CoordinationLifecycleEvent event) {}
}
```

- [ ] **Step 7: Wire NoOp in SpiDefaultBeans**

In `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java`, add a new factory method after the existing `noOpAgentWorkStateNotifier` method (end of file, before the closing `}`). Use `ide_insert_member` or `ide_replace_text_in_file`:

```java
  @Produces
  @DefaultBean
  @ApplicationScoped
  CoordinationDriver noOpCoordinationDriver() {
    return new NoOpCoordinationDriver();
  }
```

Add the required imports at the top of the file:
```java
import io.casehub.api.spi.coordination.CoordinationDriver;
import io.casehub.engine.runtime.coordination.NoOpCoordinationDriver;
```

- [ ] **Step 8: Add Javadoc to PlatformRole.COORDINATOR**

In `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java`, add Javadoc to the `COORDINATOR` enum constant. Use `ide_replace_text_in_file`:

Replace:
```java
  COORDINATOR(StructuralRole.INITIATOR),
```

With:
```java
  /**
   * Orchestrates, delegates tasks, allocates agents, and aggregates results.
   *
   * @see CoordinatorBehavior
   * @see io.casehub.api.spi.coordination.CoordinationDriver
   */
  COORDINATOR(StructuralRole.INITIATOR),
```

- [ ] **Step 9: Install all modules and run full api test suite**

First install:
Run: `mvn install -DskipTests -q`

Then run all api tests:
Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q`
Expected: all tests PASS

- [ ] **Step 10: Run spotless and commit**

```bash
mvn spotless:apply -pl api,runtime-core,runtime -q
git add api/src/main/java/io/casehub/api/model/coordination/CoordinatorBehavior.java \
        api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java \
        api/src/main/java/io/casehub/api/spi/coordination/CoordinationDriver.java \
        api/src/test/java/io/casehub/api/spi/coordination/CoordinationDriverTest.java \
        runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpCoordinationDriver.java \
        runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java
git commit -m "feat: add CoordinatorBehavior marker, CoordinationDriver SPI, NoOp default

Two-layer pattern: CoordinatorBehavior marker in model.coordination,
CoordinationDriver SPI in spi.coordination extending it.
NoOpCoordinationDriver wired as @DefaultBean in SpiDefaultBeans.
PlatformRole.COORDINATOR gains Javadoc cross-references.

Closes #1297"
```

---

## References

- [2026-11-08-coordination-driver-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/model/coordination/SupervisorGateBehavior.java` — marker interface pattern
- `api/src/main/java/io/casehub/api/spi/OversightGateService.java` — SPI extending marker pattern
- `api/src/main/java/io/casehub/api/model/coordination/PlatformRole.java:25` — COORDINATOR value to annotate
- `runtime-core/src/main/java/io/casehub/engine/runtime/coordination/NoOpAgentCapabilityGuard.java` — NoOp default pattern
- `runtime/src/main/java/io/casehub/engine/runtime/quarkus/SpiDefaultBeans.java:288-310` — default bean wiring pattern
- GitHub #1297, #1296, #1282
