# CaseLifecycleEvent Enrichment + Composed GoalExpression Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #571 — enrich CaseLifecycleEvent with case context snapshot
**Issue group:** #571, #548

**Goal:** Enrich `CaseLifecycleEvent` with case identity and context snapshot to eliminate consumer round-trips, and redesign `GoalExpression` as a sealed recursive tree supporting nested `anyOf(allOf(...), ...)` composition.

**Architecture:** Two independent changes on the same branch. #571 adds three fields to the `CaseLifecycleEvent` record with static factories for construction. #548 replaces the flat `GoalExpression` class hierarchy with a sealed interface permitting `AllOfGoalExpression`, `AnyOfGoalExpression`, and `SingleGoalExpression` — evaluation moves from the handler onto the expression tree itself.

**Tech Stack:** Java 21 records, sealed interfaces, Jackson JsonNode, Quarkus CDI events

## Global Constraints

- All types in `io.casehub.api.model` (GoalExpression types) or `io.casehub.engine.common.spi.event` (CaseLifecycleEvent)
- No new module dependencies — `common` already depends on `api`; Jackson already available in `common`
- Pre-release: breaking changes to `GoalExpression` API are free
- Tests: `mvn install -DskipTests -q` before running module-specific tests; always include `TESTCONTAINERS_RYUK_DISABLED=true`
- IntelliJ MCP mandatory for all .java edits — use `project_path=/Users/mdproctor/claude/casehub/engine`

---

### Task 1: CaseLifecycleEvent enrichment (#571)

**Files:**
- Modify: `casehub-engine-common/src/main/java/io/casehub/engine/common/spi/event/CaseLifecycleEvent.java`
- Create: `casehub-engine-common/src/test/java/io/casehub/engine/common/spi/event/CaseLifecycleEventTest.java`
- Modify: 12 handler fire sites in `runtime/src/main/java/io/casehub/engine/internal/engine/handler/`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJobListener.java`
- Modify: ~25 test constructor call sites across 12 test files

**Interfaces:**
- Produces: `CaseLifecycleEvent.of(CaseInstance, String commandType, String eventType, String actorId, String actorRole, String traceId)` — extracts `caseDefinitionName`, `namespace`, `contextSnapshot` from CaseInstance
- Produces: `CaseLifecycleEvent.of(UUID caseId, String tenancyId, String commandType, String eventType, String caseStatus, String actorId, String actorRole, String traceId)` — null enrichment fields
- Produces: `caseDefinitionName()`, `namespace()`, `contextSnapshot()` accessors on the record

- [ ] **Step 1: Write failing test for CaseLifecycleEvent factories**

Create `casehub-engine-common/src/test/java/io/casehub/engine/common/spi/event/CaseLifecycleEventTest.java`:

```java
package io.casehub.engine.common.spi.event;

import static org.junit.jupiter.api.Assertions.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.context.CaseContext;
import io.casehub.api.context.ContextLayer;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.CaseMetaModel;
import java.util.Map;
import java.util.UUID;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

@DisplayName("CaseLifecycleEvent")
class CaseLifecycleEventTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();
  private static final UUID CASE_ID = UUID.randomUUID();
  private static final String TENANCY_ID = "tenant-1";
  private static final String COMMAND_TYPE = "CompleteCase";
  private static final String EVENT_TYPE = "CaseCompleted";
  private static final String ACTOR_ID = "agent:reviewer@v1";
  private static final String ACTOR_ROLE = "System";
  private static final String TRACE_ID = "abc-123";

  @Nested
  @DisplayName("of(CaseInstance, ...) factory")
  class CaseInstanceFactory {

    @Test
    @DisplayName("extracts all fields from a fully populated CaseInstance")
    void fullyPopulated() {
      CaseInstance ci = buildCaseInstance("my-case", "io.casehub", Map.of("key", "value"));
      CaseLifecycleEvent event =
          CaseLifecycleEvent.of(ci, COMMAND_TYPE, EVENT_TYPE, ACTOR_ID, ACTOR_ROLE, TRACE_ID);

      assertEquals(CASE_ID, event.caseId());
      assertEquals(TENANCY_ID, event.tenancyId());
      assertEquals(COMMAND_TYPE, event.commandType());
      assertEquals(EVENT_TYPE, event.eventType());
      assertEquals(CaseStatus.RUNNING.name(), event.caseStatus());
      assertEquals(ACTOR_ID, event.actorId());
      assertEquals(ACTOR_ROLE, event.actorRole());
      assertEquals(TRACE_ID, event.traceId());
      assertEquals("my-case", event.caseDefinitionName());
      assertEquals("io.casehub", event.namespace());
      assertNotNull(event.contextSnapshot());
      assertEquals("value", event.contextSnapshot().get("key").asText());
    }

    @Test
    @DisplayName("handles null CaseMetaModel gracefully")
    void nullMetaModel() {
      CaseInstance ci = buildCaseInstance(null, null, Map.of());
      ci.setCaseMetaModel(null);
      CaseLifecycleEvent event =
          CaseLifecycleEvent.of(ci, COMMAND_TYPE, EVENT_TYPE, null, ACTOR_ROLE, null);

      assertEquals(CASE_ID, event.caseId());
      assertNull(event.caseDefinitionName());
      assertNull(event.namespace());
    }

    @Test
    @DisplayName("handles null CaseContext gracefully")
    void nullContext() {
      CaseInstance ci = buildCaseInstance("my-case", "io.casehub", null);
      ci.setCaseContext(null);
      CaseLifecycleEvent event =
          CaseLifecycleEvent.of(ci, COMMAND_TYPE, EVENT_TYPE, null, ACTOR_ROLE, null);

      assertEquals("my-case", event.caseDefinitionName());
      assertNull(event.contextSnapshot());
    }
  }

  @Nested
  @DisplayName("of(UUID, String, ...) overloaded factory")
  class OverloadedFactory {

    @Test
    @DisplayName("returns null for all enrichment fields")
    void enrichmentFieldsNull() {
      CaseLifecycleEvent event =
          CaseLifecycleEvent.of(
              CASE_ID, TENANCY_ID, COMMAND_TYPE, EVENT_TYPE,
              CaseStatus.COMPLETED.name(), ACTOR_ID, ACTOR_ROLE, TRACE_ID);

      assertEquals(CASE_ID, event.caseId());
      assertEquals(TENANCY_ID, event.tenancyId());
      assertEquals(COMMAND_TYPE, event.commandType());
      assertEquals(EVENT_TYPE, event.eventType());
      assertEquals(CaseStatus.COMPLETED.name(), event.caseStatus());
      assertNull(event.caseDefinitionName());
      assertNull(event.namespace());
      assertNull(event.contextSnapshot());
    }
  }

  private CaseInstance buildCaseInstance(
      String definitionName, String namespace, Map<String, Object> contextData) {
    CaseInstance ci = new CaseInstance();
    ci.setUuid(CASE_ID);
    ci.tenancyId = TENANCY_ID;
    ci.setState(CaseStatus.RUNNING);
    if (definitionName != null) {
      CaseMetaModel mm = new CaseMetaModel();
      mm.setName(definitionName);
      mm.setNamespace(namespace);
      ci.setCaseMetaModel(mm);
    }
    if (contextData != null) {
      CaseContext ctx = CaseContext.create();
      contextData.forEach(ctx::set);
      ci.setCaseContext(ctx);
    }
    return ci;
  }
}
```

- [ ] **Step 2: Run test — verify compile failure**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-engine-common -Dtest=CaseLifecycleEventTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```

Expected: compile error — `CaseLifecycleEvent.of` method does not exist yet.

- [ ] **Step 3: Update CaseLifecycleEvent record + implement factories**

Use `ide_edit_member` on `CaseLifecycleEvent` (member=`CaseLifecycleEvent`, the record declaration itself) to replace the entire record with the enriched version including both factory methods. The new record adds `caseDefinitionName`, `namespace`, `contextSnapshot` fields and two static `of()` factories.

The `of(CaseInstance, ...)` factory extracts:
- `caseId` from `caseInstance.getUuid()`
- `tenancyId` from `caseInstance.tenancyId`
- `caseStatus` from `caseInstance.getState().name()`
- `caseDefinitionName` from `caseInstance.getCaseMetaModel()?.getName()`
- `namespace` from `caseInstance.getCaseMetaModel()?.getNamespace()`
- `contextSnapshot` from `caseInstance.getCaseContext()?.layer(ContextLayer.WORKING).asJsonNode()`

The `of(UUID, String, ...)` overloaded factory passes null for all three enrichment fields.

- [ ] **Step 4: Run test — verify pass**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-engine-common -Dtest=CaseLifecycleEventTest -q
```

Expected: PASS

- [ ] **Step 5: Update all 12 runtime handler fire sites**

Each handler currently constructs `new CaseLifecycleEvent(caseInstance.getUuid(), caseInstance.tenancyId, ...)`. Replace with `CaseLifecycleEvent.of(caseInstance, commandType, eventType, actorId, actorRole, traceId)`.

Handlers to update (all in `runtime/src/main/java/io/casehub/engine/internal/engine/handler/`):
1. `CaseStartedEventHandler` — 1 site
2. `CaseStatusChangedHandler` — 1 site
3. `CaseContextChangedEventHandler` — 1 site
4. `GoalReachedEventHandler` — 1 site
5. `SignalReceivedEventHandler` — 2 sites (applySignalUnderLock + applyBulkSignalUnderLock)
6. `WorkflowExecutionCompletedHandler` — 3 sites
7. `MilestoneActivatedEventHandler` — 1 site
8. `MilestoneReachedEventHandler` — 1 site
9. `MilestoneCompletedEventHandler` — 1 site

Use `ide_replace_member` to replace each method body containing `new CaseLifecycleEvent(` with the factory call.

- [ ] **Step 6: Update QuartzWorkerExecutionJobListener**

`QuartzWorkerExecutionJobListener.jobToBeExecuted()` has no `CaseInstance` — use the overloaded factory:
```java
CaseLifecycleEvent.of(caseId, tenancyId, "ScheduleWorker", "WorkerExecutionStarted",
    null, workerId, "System", traceId)
```

- [ ] **Step 7: Update all test constructor sites**

~25 sites across 12 test files. Each `new CaseLifecycleEvent(uuid, tenancy, cmd, evt, status, actor, role, trace)` gains three trailing null args: `new CaseLifecycleEvent(uuid, tenancy, cmd, evt, status, actor, role, trace, null, null, null)`. Or use the overloaded factory `CaseLifecycleEvent.of(uuid, tenancy, ...)`.

Test files:
- `runtime`: CaseLifecycleCdiEventTest, CaseMemoryObserverTest (3), CaseContextChangedEventHandlerRoutingTest, BulkSignalEventLogAuditTest
- `ledger`: CaseLedgerEventCaptureTest (11), CaseLedgerEventCaptureDisabledTest
- `blackboard`: SubCaseCompletionServiceTest, SubCaseMofNIntegrationTest (2), SubCaseParallelIntegrationTest (3), SubCaseMofNOutputMappingTest

- [ ] **Step 8: Build verification across all modules**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-engine-common,runtime,ledger,blackboard,scheduler-quartz -q
```

Expected: all tests pass.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#571): enrich CaseLifecycleEvent with case identity and context snapshot

Add caseDefinitionName, namespace, and contextSnapshot (JsonNode working layer)
to CaseLifecycleEvent record. Static factories: of(CaseInstance, ...) for enriched
events, of(UUID, String, ...) for sites without CaseInstance (QuartzWorkerExecutionJobListener).
Eliminates consumer round-trip to caseInstanceRepository.

Refs #571

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: GoalExpression sealed type hierarchy + integration (#548)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/SingleGoalExpression.java`
- Modify: `api/src/main/java/io/casehub/api/model/GoalExpression.java`
- Modify: `api/src/main/java/io/casehub/api/model/AllOfGoalExpression.java`
- Modify: `api/src/main/java/io/casehub/api/model/AnyOfGoalExpression.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Create: `api/src/test/java/io/casehub/api/model/GoalExpressionTest.java` (new location in api)
- Modify: `api/src/test/java/io/casehub/api/model/GoalBasedCompletionTest.java`
- Delete: `runtime/src/test/java/io/casehub/engine/model/GoalExpressionTest.java` (old location)

**Interfaces:**
- Consumes: `Goal.getName()` — for backward-compatible factories
- Produces: `GoalExpression` sealed interface with `isSatisfiedBy(Set<String>)`, `goalNames()`, `satisfiedGoalName(Set<String>)`
- Produces: `SingleGoalExpression(String goalName)` record
- Produces: `AllOfGoalExpression(List<GoalExpression> children)` record
- Produces: `AnyOfGoalExpression(List<GoalExpression> children)` record
- Produces: Static factories `GoalExpression.allOf(Goal...)`, `GoalExpression.anyOf(Goal...)`, `GoalExpression.allOf(GoalExpression...)`, `GoalExpression.anyOf(GoalExpression...)`, `GoalExpression.goal(String)`
- Produces: Recursive `parseGoalExpressionFromNode` in CaseDefinitionYamlMapper

- [ ] **Step 1: Write failing tests for GoalExpression types**

Create `api/src/test/java/io/casehub/api/model/GoalExpressionTest.java` with tests covering:

SingleGoalExpression:
- `isSatisfiedBy` with present/absent goal name
- `goalNames` returns singleton set
- `satisfiedGoalName` returns name when present, null when absent

AllOfGoalExpression:
- `isSatisfiedBy` all present → true
- `isSatisfiedBy` partial → false
- `satisfiedGoalName` all present → first child's name
- `satisfiedGoalName` partial → null
- `goalNames` returns union of children
- Constructor rejects empty list

AnyOfGoalExpression:
- `isSatisfiedBy` any present → true
- `isSatisfiedBy` none → false
- `satisfiedGoalName` → first satisfied child's name
- Constructor rejects empty list

Composed expressions:
- `anyOf(allOf(a,b,c), goal(d))` — d satisfied → "d"
- `anyOf(allOf(a,b,c), goal(d))` — a,b,c satisfied → "a"
- `anyOf(allOf(a,b,c), goal(d))` — a,b only → null
- `allOf(anyOf(a,b), goal(c))` — a,c satisfied → "a"

Backward-compatible factories:
- `GoalExpression.allOf(Goal...)` creates correct tree
- `GoalExpression.anyOf(Goal...)` creates correct tree
- `GoalExpression.goal(String)` creates SingleGoalExpression

```java
package io.casehub.api.model;

import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

@DisplayName("GoalExpression")
class GoalExpressionTest {

  @Nested
  @DisplayName("SingleGoalExpression")
  class SingleTests {

    @Test
    @DisplayName("satisfied when goal name is in reached set")
    void satisfied() {
      var expr = GoalExpression.goal("approved");
      assertTrue(expr.isSatisfiedBy(Set.of("approved", "verified")));
    }

    @Test
    @DisplayName("not satisfied when goal name is absent")
    void notSatisfied() {
      var expr = GoalExpression.goal("approved");
      assertFalse(expr.isSatisfiedBy(Set.of("verified")));
    }

    @Test
    @DisplayName("goalNames returns singleton")
    void goalNames() {
      assertEquals(Set.of("approved"), GoalExpression.goal("approved").goalNames());
    }

    @Test
    @DisplayName("satisfiedGoalName returns name when present, null when absent")
    void satisfiedGoalName() {
      var expr = GoalExpression.goal("approved");
      assertEquals("approved", expr.satisfiedGoalName(Set.of("approved")));
      assertNull(expr.satisfiedGoalName(Set.of("other")));
    }
  }

  @Nested
  @DisplayName("AllOfGoalExpression")
  class AllOfTests {

    @Test
    @DisplayName("satisfied when all children are satisfied")
    void allSatisfied() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertTrue(expr.isSatisfiedBy(Set.of("a", "b", "c")));
    }

    @Test
    @DisplayName("not satisfied when only partial children match")
    void partialMatch() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertFalse(expr.isSatisfiedBy(Set.of("a")));
    }

    @Test
    @DisplayName("not satisfied when no children match")
    void noneMatch() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertFalse(expr.isSatisfiedBy(Set.of("x", "y")));
    }

    @Test
    @DisplayName("satisfiedGoalName returns first child name when all satisfied")
    void satisfiedGoalName_allPresent() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertEquals("a", expr.satisfiedGoalName(Set.of("a", "b")));
    }

    @Test
    @DisplayName("satisfiedGoalName returns null when partial")
    void satisfiedGoalName_partial() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertNull(expr.satisfiedGoalName(Set.of("a")));
    }

    @Test
    @DisplayName("goalNames returns union of all children")
    void goalNames() {
      var expr = GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertEquals(Set.of("a", "b"), expr.goalNames());
    }

    @Test
    @DisplayName("rejects empty children")
    void emptyChildren() {
      assertThrows(IllegalArgumentException.class,
          () -> new AllOfGoalExpression(List.of()));
    }
  }

  @Nested
  @DisplayName("AnyOfGoalExpression")
  class AnyOfTests {

    @Test
    @DisplayName("satisfied when any child is satisfied")
    void anySatisfied() {
      var expr = GoalExpression.anyOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertTrue(expr.isSatisfiedBy(Set.of("b")));
    }

    @Test
    @DisplayName("not satisfied when no child matches")
    void noneMatch() {
      var expr = GoalExpression.anyOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertFalse(expr.isSatisfiedBy(Set.of("x")));
    }

    @Test
    @DisplayName("satisfiedGoalName returns first satisfied child name")
    void satisfiedGoalName() {
      var expr = GoalExpression.anyOf(GoalExpression.goal("a"), GoalExpression.goal("b"));
      assertEquals("b", expr.satisfiedGoalName(Set.of("b")));
    }

    @Test
    @DisplayName("rejects empty children")
    void emptyChildren() {
      assertThrows(IllegalArgumentException.class,
          () -> new AnyOfGoalExpression(List.of()));
    }
  }

  @Nested
  @DisplayName("Composed expressions")
  class ComposedTests {

    @Test
    @DisplayName("anyOf(allOf(a,b,c), goal(d)) — d alone satisfies")
    void anyOf_allOf_singleLeaf() {
      var expr = GoalExpression.anyOf(
          GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"), GoalExpression.goal("c")),
          GoalExpression.goal("d"));
      assertTrue(expr.isSatisfiedBy(Set.of("d")));
      assertEquals("d", expr.satisfiedGoalName(Set.of("d")));
    }

    @Test
    @DisplayName("anyOf(allOf(a,b,c), goal(d)) — a,b,c satisfies via allOf branch")
    void anyOf_allOfBranch() {
      var expr = GoalExpression.anyOf(
          GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"), GoalExpression.goal("c")),
          GoalExpression.goal("d"));
      assertTrue(expr.isSatisfiedBy(Set.of("a", "b", "c")));
      assertEquals("a", expr.satisfiedGoalName(Set.of("a", "b", "c")));
    }

    @Test
    @DisplayName("anyOf(allOf(a,b,c), goal(d)) — partial allOf, no d → not satisfied")
    void anyOf_partial() {
      var expr = GoalExpression.anyOf(
          GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b"), GoalExpression.goal("c")),
          GoalExpression.goal("d"));
      assertFalse(expr.isSatisfiedBy(Set.of("a", "b")));
      assertNull(expr.satisfiedGoalName(Set.of("a", "b")));
    }

    @Test
    @DisplayName("allOf(anyOf(a,b), goal(c)) — a and c satisfies")
    void allOf_anyOf() {
      var expr = GoalExpression.allOf(
          GoalExpression.anyOf(GoalExpression.goal("a"), GoalExpression.goal("b")),
          GoalExpression.goal("c"));
      assertTrue(expr.isSatisfiedBy(Set.of("a", "c")));
      assertEquals("a", expr.satisfiedGoalName(Set.of("a", "c")));
    }

    @Test
    @DisplayName("deeply nested goalNames collects all leaves")
    void goalNames_deep() {
      var expr = GoalExpression.anyOf(
          GoalExpression.allOf(GoalExpression.goal("a"), GoalExpression.goal("b")),
          GoalExpression.goal("c"));
      assertEquals(Set.of("a", "b", "c"), expr.goalNames());
    }
  }

  @Nested
  @DisplayName("Backward-compatible factories")
  class FactoryTests {

    @Test
    @DisplayName("allOf(Goal...) extracts names into SingleGoalExpression children")
    void allOfGoals() {
      Goal g1 = Goal.builder().name("a").condition(".a").kind(GoalKind.SUCCESS).build();
      Goal g2 = Goal.builder().name("b").condition(".b").kind(GoalKind.SUCCESS).build();
      var expr = GoalExpression.allOf(g1, g2);
      assertTrue(expr.isSatisfiedBy(Set.of("a", "b")));
      assertFalse(expr.isSatisfiedBy(Set.of("a")));
      assertEquals(Set.of("a", "b"), expr.goalNames());
    }

    @Test
    @DisplayName("anyOf(Goal...) extracts names into SingleGoalExpression children")
    void anyOfGoals() {
      Goal g1 = Goal.builder().name("a").condition(".a").kind(GoalKind.SUCCESS).build();
      var expr = GoalExpression.anyOf(g1);
      assertTrue(expr.isSatisfiedBy(Set.of("a")));
      assertFalse(expr.isSatisfiedBy(Set.of("b")));
    }

    @Test
    @DisplayName("goal(String) creates SingleGoalExpression")
    void goalFactory() {
      var expr = GoalExpression.goal("x");
      assertInstanceOf(SingleGoalExpression.class, expr);
      assertEquals(Set.of("x"), expr.goalNames());
    }
  }
}
```

- [ ] **Step 2: Run test — verify compile failure**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=io.casehub.api.model.GoalExpressionTest -Dsurefire.failIfNoSpecifiedTests=false -q 2>&1 | tail -5
```

Expected: compile error — `SingleGoalExpression` and new factories don't exist yet.

- [ ] **Step 3: Create SingleGoalExpression**

Create `api/src/main/java/io/casehub/api/model/SingleGoalExpression.java`:

```java
package io.casehub.api.model;

import java.util.Objects;
import java.util.Set;

public record SingleGoalExpression(String goalName) implements GoalExpression {

    public SingleGoalExpression {
        Objects.requireNonNull(goalName, "goalName must not be null");
    }

    @Override
    public boolean isSatisfiedBy(Set<String> reachedGoalNames) {
        return reachedGoalNames.contains(goalName);
    }

    @Override
    public Set<String> goalNames() {
        return Set.of(goalName);
    }

    @Override
    public String satisfiedGoalName(Set<String> reachedGoalNames) {
        return reachedGoalNames.contains(goalName) ? goalName : null;
    }
}
```

- [ ] **Step 4: Rewrite AllOfGoalExpression as recursive record**

Use `ide_edit_member` (member=`AllOfGoalExpression`) to replace the entire class:

```java
package io.casehub.api.model;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

public record AllOfGoalExpression(List<GoalExpression> children) implements GoalExpression {

    public AllOfGoalExpression {
        if (children.isEmpty()) {
            throw new IllegalArgumentException("AllOfGoalExpression requires at least one child");
        }
        children = List.copyOf(children);
    }

    @Override
    public boolean isSatisfiedBy(Set<String> reachedGoalNames) {
        return children.stream().allMatch(c -> c.isSatisfiedBy(reachedGoalNames));
    }

    @Override
    public Set<String> goalNames() {
        return children.stream()
            .flatMap(c -> c.goalNames().stream())
            .collect(Collectors.toUnmodifiableSet());
    }

    @Override
    public String satisfiedGoalName(Set<String> reachedGoalNames) {
        String firstName = null;
        for (GoalExpression child : children) {
            String name = child.satisfiedGoalName(reachedGoalNames);
            if (name == null) return null;
            if (firstName == null) firstName = name;
        }
        return firstName;
    }
}
```

- [ ] **Step 5: Rewrite AnyOfGoalExpression as recursive record**

Use `ide_edit_member` (member=`AnyOfGoalExpression`) to replace the entire class:

```java
package io.casehub.api.model;

import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

public record AnyOfGoalExpression(List<GoalExpression> children) implements GoalExpression {

    public AnyOfGoalExpression {
        if (children.isEmpty()) {
            throw new IllegalArgumentException("AnyOfGoalExpression requires at least one child");
        }
        children = List.copyOf(children);
    }

    @Override
    public boolean isSatisfiedBy(Set<String> reachedGoalNames) {
        return children.stream().anyMatch(c -> c.isSatisfiedBy(reachedGoalNames));
    }

    @Override
    public Set<String> goalNames() {
        return children.stream()
            .flatMap(c -> c.goalNames().stream())
            .collect(Collectors.toUnmodifiableSet());
    }

    @Override
    public String satisfiedGoalName(Set<String> reachedGoalNames) {
        for (GoalExpression child : children) {
            String name = child.satisfiedGoalName(reachedGoalNames);
            if (name != null) return name;
        }
        return null;
    }
}
```

- [ ] **Step 6: Update GoalExpression interface**

Use `ide_edit_member` (member=`GoalExpression`) to replace the entire interface:

```java
package io.casehub.api.model;

import java.util.Arrays;
import java.util.Collection;
import java.util.List;
import java.util.Set;

public sealed interface GoalExpression
    permits AllOfGoalExpression, AnyOfGoalExpression, SingleGoalExpression {

  boolean isSatisfiedBy(Set<String> reachedGoalNames);

  Set<String> goalNames();

  String satisfiedGoalName(Set<String> reachedGoalNames);

  static GoalExpression allOf(Goal... goals) {
    return new AllOfGoalExpression(
        Arrays.stream(goals).map(g -> new SingleGoalExpression(g.getName())).toList());
  }

  static GoalExpression allOf(Collection<Goal> goals) {
    return new AllOfGoalExpression(
        goals.stream().map(g -> new SingleGoalExpression(g.getName())).toList());
  }

  static GoalExpression anyOf(Goal... goals) {
    return new AnyOfGoalExpression(
        Arrays.stream(goals).map(g -> new SingleGoalExpression(g.getName())).toList());
  }

  static GoalExpression allOf(GoalExpression... children) {
    return new AllOfGoalExpression(List.of(children));
  }

  static GoalExpression anyOf(GoalExpression... children) {
    return new AnyOfGoalExpression(List.of(children));
  }

  static GoalExpression goal(String name) {
    return new SingleGoalExpression(name);
  }
}
```

- [ ] **Step 7: Run GoalExpression tests — verify pass**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=io.casehub.api.model.GoalExpressionTest -q
```

Expected: PASS

- [ ] **Step 8: Update GoalReachedEventHandler**

Remove `isGoalExpressionSatisfied()` and `findSatisfiedGoalName()` private methods. In `evaluateCompletion()`, replace the two-step dance:

```java
// Before:
if (isGoalExpressionSatisfied(expr, reachedGoals)) {
    String satisfiedGoalName = findSatisfiedGoalName(expr, reachedGoals);
    ...
}

// After:
String satisfiedName = expr.satisfiedGoalName(reachedGoals);
if (satisfiedName != null) {
    ...
}
```

Use `ide_replace_member` on `evaluateCompletion` and `ide_refactor_safe_delete` on the two private methods.

- [ ] **Step 9: Update DefaultCaseDefinitionRegistry**

Replace `expr.getGoals()` usage with `expr.goalNames()`. Build local `goalsByName` map for kind-mismatch check.

In the validation section that checks unreferenced goals and kind mismatches:

```java
// Build local goal name map for kind-mismatch checking
Map<String, Goal> goalsByName = definition.getGoals().stream()
    .collect(Collectors.toMap(Goal::getName, Function.identity()));

var referencedGoals = new HashSet<String>();
for (var entry : gbc.getGoals().entrySet()) {
    GoalExpression expr = entry.getValue();
    if (expr != null) {
        referencedGoals.addAll(expr.goalNames());
    }
}
// Unreferenced check (unchanged logic, different API)
for (Goal goal : definition.getGoals()) {
    if (!referencedGoals.contains(goal.getName())) {
        LOG.warnf("Goal '%s' is not referenced ...", goal.getName());
    }
}
// Kind mismatch check — use goalsByName lookup
for (var entry : gbc.getGoals().entrySet()) {
    String kindValue = entry.getKey().value();
    GoalExpression expr = entry.getValue();
    if (expr != null) {
        for (String goalName : expr.goalNames()) {
            Goal g = goalsByName.get(goalName);
            if (g != null && g.getKind() != null && !g.getKind().equals(kindValue)) {
                LOG.warnf("Goal '%s' has kind '%s' but is referenced in '%s'...",
                    g.getName(), g.getKind(), kindValue);
            }
        }
    }
}
```

- [ ] **Step 10: Write failing test for recursive YAML parsing**

Add tests to `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` for nested goal composition:

```java
@Test
@DisplayName("completion with nested anyOf(allOf(a,b), c) parses correctly")
void nestedComposition() throws IOException {
    String yaml = """
        name: test-case
        namespace: io.casehub.test
        spec:
          goals:
            - name: a
              condition: ".a == true"
              kind: success
            - name: b
              condition: ".b == true"
              kind: success
            - name: c
              condition: ".c == true"
              kind: success
          completion:
            success:
              anyOf:
                - allOf: [a, b]
                - c
        """;
    CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
    GoalExpression expr = ((GoalBasedCompletion<?>) def.getCompletion())
        .getGoals().values().iterator().next();

    assertTrue(expr.isSatisfiedBy(Set.of("a", "b")));
    assertTrue(expr.isSatisfiedBy(Set.of("c")));
    assertFalse(expr.isSatisfiedBy(Set.of("a")));
    assertEquals(Set.of("a", "b", "c"), expr.goalNames());
}

@Test
@DisplayName("completion with invalid goal reference throws at parse time")
void invalidGoalReference() {
    String yaml = """
        name: test-case
        namespace: io.casehub.test
        spec:
          goals:
            - name: a
              condition: ".a == true"
              kind: success
          completion:
            success:
              allOf: [a, nonexistent]
        """;
    assertThrows(IllegalArgumentException.class,
        () -> CaseDefinitionYamlMapper.fromYaml(yaml));
}

@Test
@DisplayName("completion with empty allOf array throws")
void emptyAllOf() {
    String yaml = """
        name: test-case
        namespace: io.casehub.test
        spec:
          goals:
            - name: a
              condition: ".a == true"
              kind: success
          completion:
            success:
              allOf: []
        """;
    assertThrows(IllegalArgumentException.class,
        () -> CaseDefinitionYamlMapper.fromYaml(yaml));
}
```

- [ ] **Step 11: Run test — verify fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#nestedComposition+invalidGoalReference+emptyAllOf -q 2>&1 | tail -5
```

Expected: FAIL — parser doesn't handle nested objects in arrays yet.

- [ ] **Step 12: Update YAML parser — make parseGoalExpressionFromNode recursive**

In `CaseDefinitionYamlMapper`, rewrite `parseGoalExpressionFromNode`:

```java
private static GoalExpression parseGoalExpressionFromNode(
    JsonNode node, Map<String, Goal> goalMap) {
  JsonNode allOfNode = node.get("allOf");
  if (allOfNode != null && allOfNode.isArray()) {
    if (allOfNode.isEmpty()) {
      throw new IllegalArgumentException("allOf array must not be empty");
    }
    List<GoalExpression> children = new java.util.ArrayList<>();
    for (JsonNode element : allOfNode) {
      children.add(parseGoalElement(element, goalMap));
    }
    return new AllOfGoalExpression(children);
  }
  JsonNode anyOfNode = node.get("anyOf");
  if (anyOfNode != null && anyOfNode.isArray()) {
    if (anyOfNode.isEmpty()) {
      throw new IllegalArgumentException("anyOf array must not be empty");
    }
    List<GoalExpression> children = new java.util.ArrayList<>();
    for (JsonNode element : anyOfNode) {
      children.add(parseGoalElement(element, goalMap));
    }
    return new AnyOfGoalExpression(children);
  }
  return null;
}

private static GoalExpression parseGoalElement(JsonNode element, Map<String, Goal> goalMap) {
  if (element.isTextual()) {
    String goalName = element.asText();
    Goal goal = goalMap.get(goalName);
    if (goal == null) {
      throw new IllegalArgumentException(
          "Goal '" + goalName + "' referenced in completion expression is not defined");
    }
    return new SingleGoalExpression(goal.getName());
  }
  if (element.isObject()) {
    GoalExpression nested = parseGoalExpressionFromNode(element, goalMap);
    if (nested == null) {
      throw new IllegalArgumentException(
          "Completion expression object must contain 'allOf' or 'anyOf'");
    }
    return nested;
  }
  throw new IllegalArgumentException(
      "Completion expression element must be a goal name (string) or an object with allOf/anyOf");
}
```

Also update `convertGoalExpression` to use `SingleGoalExpression`:

```java
private static GoalExpression convertGoalExpression(
    final io.casehub.model.GoalExpression expr, final Map<String, Goal> goalMap) {
  if (expr == null) return null;
  if (expr.getAllOf() != null && !expr.getAllOf().isEmpty()) {
    return new AllOfGoalExpression(
        expr.getAllOf().stream()
            .map(name -> (GoalExpression) new SingleGoalExpression(goalMap.get(name).getName()))
            .collect(Collectors.toList()));
  }
  if (expr.getAnyOf() != null && !expr.getAnyOf().isEmpty()) {
    return new AnyOfGoalExpression(
        expr.getAnyOf().stream()
            .map(name -> (GoalExpression) new SingleGoalExpression(goalMap.get(name).getName()))
            .collect(Collectors.toList()));
  }
  return null;
}
```

- [ ] **Step 13: Run YAML parsing tests — verify pass**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest -q
```

Expected: PASS

- [ ] **Step 14: Delete old GoalExpressionTest from runtime**

Use `ide_refactor_safe_delete` on `runtime/src/test/java/io/casehub/engine/model/GoalExpressionTest.java`.

- [ ] **Step 15: Full build verification**

```bash
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,casehub-engine-common,runtime,blackboard,resilience -q
```

Expected: all tests pass — existing integration tests (ChoreographySelectionTest, SignalGoalCompletionTest, CustomGoalKindCompletionTest, etc.) use the backward-compatible `GoalExpression.allOf(Goal...)` factory and should pass without changes.

- [ ] **Step 16: Commit**

```bash
git add -A
git commit -m "feat(#548): composed GoalExpression — sealed recursive tree with nested anyOf/allOf

Replace flat GoalExpression hierarchy with sealed interface permitting
AllOfGoalExpression, AnyOfGoalExpression, SingleGoalExpression. Evaluation
moves from handler instanceof checks to recursive isSatisfiedBy/satisfiedGoalName
on the expression tree. YAML parser supports nested composition:
  anyOf:
    - allOf: [a, b, c]
    - d
Parse-time validation rejects unknown goal references and empty arrays.

Closes #548

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```
