# Milestone, Goal, and Stage — Full Conceptual Alignment

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close epic #84 — remove Goal.terminal, add Milestone.parentStageId, upgrade milestone lifecycle tracking to MilestoneLifecycleStatus, consolidate milestone evaluation to single path.

**Architecture:** Four changes to the api, runtime, blackboard, and schema modules. Goal.terminal removal is pure deletion. Milestone.parentStageId adds a builder field and CasePlanModel overload. Milestone lifecycle upgrades CasePlanModel from Boolean to MilestoneLifecycleStatus with proper transition guards. Milestone evaluation consolidation removes the old CaseContextChangedEventHandler.milestones() path.

**Tech Stack:** Java 22, Quarkus 3.32, Vert.x event bus, Mutiny (Uni), jackson-databind, JUnit 5, AssertJ

## Global Constraints

- Issue: casehubio/engine#581, epic: casehubio/engine#84
- All commits reference `Refs #581`
- `mvn install -DskipTests -q` before module-specific tests
- `TESTCONTAINERS_RYUK_DISABLED=true` for all test runs
- CaseHubEventType.MILESTONE_REACHED enum value is retained for backwards compatibility with persisted EventLog data

---

### Task 1: Remove Goal.terminal field and schema property

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Goal.java`
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml:429-432`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java:86`
- Create: `api/src/test/java/io/casehub/api/model/GoalTest.java`

**Interfaces:**
- Consumes: nothing (first task)
- Produces: `Goal` without `terminal` — used by all subsequent tasks and all existing code that references `Goal`

- [ ] **Step 1: Write the failing test — Goal has no terminal method**

```java
// api/src/test/java/io/casehub/api/model/GoalTest.java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;

import java.lang.reflect.Method;
import java.util.Arrays;
import org.junit.jupiter.api.Test;

class GoalTest {

  @Test
  void goal_builder_has_no_terminal_method() {
    var methods = Arrays.stream(Goal.Builder.class.getMethods())
        .map(Method::getName)
        .toList();
    assertThat(methods).doesNotContain("terminal");
  }

  @Test
  void goal_has_no_terminal_getter() {
    var methods = Arrays.stream(Goal.class.getMethods())
        .map(Method::getName)
        .toList();
    assertThat(methods).doesNotContain("getTerminal");
    assertThat(methods).doesNotContain("setTerminal");
  }

  @Test
  void goal_equals_ignores_terminal() {
    Goal g1 = Goal.builder()
        .name("approved")
        .condition(".decision == \"approved\"")
        .kind(GoalKind.SUCCESS)
        .build();
    Goal g2 = Goal.builder()
        .name("approved")
        .condition(".decision == \"approved\"")
        .kind(GoalKind.SUCCESS)
        .build();
    assertThat(g1).isEqualTo(g2);
    assertThat(g1.hashCode()).isEqualTo(g2.hashCode());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=GoalTest -q`
Expected: FAIL — `getTerminal` and `terminal` methods still exist

- [ ] **Step 3: Remove terminal from Goal.java**

In `api/src/main/java/io/casehub/api/model/Goal.java`:

1. Remove field `private boolean terminal;` (line 64)
2. Remove getter `getTerminal()` (lines 93-95)
3. Remove setter `setTerminal(boolean)` (lines 97-99)
4. Remove Builder field `private boolean terminal;` (line 110)
5. Remove Builder method `terminal(boolean)` (lines 144-147)
6. Remove `goal.setTerminal(terminal);` from `Builder.build()` (line 160)
7. Remove `&& Objects.equals(terminal, goal.terminal)` from `equals()` (line 172)
8. Remove `terminal` from `Objects.hash()` in `hashCode()` (line 178) → `return Objects.hash(name, condition, kind, description);`
9. Update Javadoc: remove lines 55-57 ("Non-terminal goals — goals not referenced..."). Update line 50-53 to read:

```java
 * <p>When a goal's condition becomes true, a {@code GoalReachedEvent} is published and recorded in
 * the {@link io.casehub.engine.internal.history.EventLog}. If the goal is referenced by a {@link
 * io.casehub.api.model.GoalBasedCompletion}, the engine evaluates whether the case should
 * transition to COMPLETED or FAILED. Goals are always terminal — use {@link Milestone} for
 * non-terminal checkpoints.
```

- [ ] **Step 4: Remove terminal from YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, remove lines 429-432:
```yaml
      terminal:
        type: boolean
        default: false
        description: "If true, achieving this goal can close the case (depending on completion policy)"
```

- [ ] **Step 5: Remove isTerminal from GoalReachedEventHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java`, remove `.put("isTerminal", goal.getTerminal())` from the EventLog metadata construction (line 86). Keep the rest of the metadata node.

- [ ] **Step 6: Run tests to verify everything passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=GoalTest -q`
Expected: PASS

- [ ] **Step 7: Run full api and runtime module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime -q`
Expected: PASS — existing tests that use `Goal.builder()` without `.terminal()` are unaffected. Any test using `.terminal()` will fail to compile — fix by removing the call.

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/Goal.java \
       api/src/test/java/io/casehub/api/model/GoalTest.java \
       schema/src/main/resources/schema/CaseDefinition.yaml \
       runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java
git commit -m "feat(#581): remove Goal.terminal — goals are always terminal

Goals exist to drive case completion. Non-terminal checkpoints with
polarity are Milestones. Removes the terminal field, getter, setter,
builder method, and schema property. Removes isTerminal from EventLog
metadata in GoalReachedEventHandler.

Refs #581"
```

---

### Task 2: Add unreferenced goal registration warning

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryGoalWarningTest.java`

**Interfaces:**
- Consumes: `Goal` (from Task 1 — no `terminal` field)
- Produces: WARNING log during `register()` when a goal is not in any `GoalExpression`

- [ ] **Step 1: Write the failing test — unreferenced goal logs warning**

```java
// runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryGoalWarningTest.java
package io.casehub.engine.internal.engine;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.*;
import io.casehub.api.model.evaluator.JQExpressionEvaluator;
import java.util.List;
import java.util.logging.Handler;
import java.util.logging.Level;
import java.util.logging.LogRecord;
import java.util.logging.Logger;
import org.junit.jupiter.api.Test;

class DefaultCaseDefinitionRegistryGoalWarningTest {

  @Test
  void warns_when_goal_not_referenced_in_any_goal_expression() {
    // Collect log records
    var records = new java.util.ArrayList<LogRecord>();
    var logger = Logger.getLogger(DefaultCaseDefinitionRegistry.class.getName());
    logger.setLevel(Level.WARNING);
    logger.addHandler(new Handler() {
      @Override public void publish(LogRecord record) { records.add(record); }
      @Override public void flush() {}
      @Override public void close() {}
    });

    var unreferencedGoal = Goal.builder()
        .name("orphan-goal")
        .condition(".orphan == true")
        .kind(GoalKind.SUCCESS)
        .build();

    var referencedGoal = Goal.builder()
        .name("real-goal")
        .condition(".done == true")
        .kind(GoalKind.SUCCESS)
        .build();

    var definition = CaseDefinition.builder()
        .namespace("test")
        .name("warn-test")
        .version("1.0")
        .goals(List.of(unreferencedGoal, referencedGoal))
        .completion(new GoalBasedCompletion(
            GoalExpression.allOf(referencedGoal), null))
        .build();

    // The warning check runs at validation time — find the method and call it
    // or test via the full register() path depending on what's available
    assertThat(records).anyMatch(r ->
        r.getMessage().contains("orphan-goal") && r.getMessage().contains("not referenced"));
    assertThat(records).noneMatch(r ->
        r.getMessage().contains("real-goal"));
  }
}
```

Note: The exact test setup depends on how `DefaultCaseDefinitionRegistry.register()` is invoked. The implementer should adapt the test to use the registry's `register()` method with appropriate mocks for its dependencies (`CaseMetaModelRepository`, `ExpressionEngineRegistry`). The key assertions are: the orphaned goal logs a warning, the referenced goal does not.

- [ ] **Step 2: Implement the warning in DefaultCaseDefinitionRegistry**

In `DefaultCaseDefinitionRegistry.registerCaseDefinition()`, after the existing goal validation loop (lines 211-215), add:

```java
if (definition.getGoals() != null && definition.getCompletion() instanceof GoalBasedCompletion gbc) {
  var referencedGoals = new java.util.HashSet<String>();
  if (gbc.getSuccess() != null) {
    gbc.getSuccess().getGoals().forEach(g -> referencedGoals.add(g.getName()));
  }
  if (gbc.getFailure() != null) {
    gbc.getFailure().getGoals().forEach(g -> referencedGoals.add(g.getName()));
  }
  for (Goal goal : definition.getGoals()) {
    if (!referencedGoals.contains(goal.getName())) {
      LOG.warnf("Goal '%s' is not referenced in any GoalExpression. "
          + "Goals should drive case completion — use Milestone for non-terminal checkpoints.",
          goal.getName());
    }
  }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryGoalWarningTest -q`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java \
       runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryGoalWarningTest.java
git commit -m "feat(#581): warn on unreferenced goals at registration

Goals not referenced in any GoalExpression get a WARNING log during
register(). Reinforces 'goals are always terminal' without breaking
existing tests that define goals without completion blocks.

Refs #581"
```

---

### Task 3: Add Milestone.parentStageId

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Milestone.java`
- Modify: `api/src/test/java/io/casehub/api/model/MilestoneTest.java` (or create if absent)

**Interfaces:**
- Consumes: nothing
- Produces: `Milestone.getParentStageId()` (String, nullable), `Milestone.Builder.parentStageId(String)` — used by Task 4

- [ ] **Step 1: Write the failing test**

```java
// api/src/test/java/io/casehub/api/model/MilestoneParentStageTest.java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

class MilestoneParentStageTest {

  @Test
  void milestone_with_parent_stage_id() {
    var milestone = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .parentStageId("kyc-stage")
        .build();
    assertThat(milestone.getParentStageId()).isEqualTo("kyc-stage");
  }

  @Test
  void milestone_without_parent_stage_id_returns_null() {
    var milestone = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .build();
    assertThat(milestone.getParentStageId()).isNull();
  }

  @Test
  void milestone_equality_includes_parent_stage_id() {
    var m1 = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .parentStageId("stage-a")
        .build();
    var m2 = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .parentStageId("stage-b")
        .build();
    assertThat(m1).isNotEqualTo(m2);
  }

  @Test
  void milestone_equality_same_parent_stage_id() {
    var m1 = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .parentStageId("stage-a")
        .build();
    var m2 = Milestone.builder()
        .name("doc-check")
        .completionCriteria(".docsReceived == true")
        .parentStageId("stage-a")
        .build();
    assertThat(m1).isEqualTo(m2);
    assertThat(m1.hashCode()).isEqualTo(m2.hashCode());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=MilestoneParentStageTest -q`
Expected: FAIL — `parentStageId` method doesn't exist on Builder

- [ ] **Step 3: Add parentStageId to Milestone.java**

In `api/src/main/java/io/casehub/api/model/Milestone.java`:

1. Add field after `slaStartFrom` (line 99): `private final String parentStageId;`
2. Add to constructor (after slaStartFrom param): add `String parentStageId` parameter, assign `this.parentStageId = parentStageId;`
3. Add getter: `public String getParentStageId() { return parentStageId; }`
4. Add to Builder: field `private String parentStageId;` and method:
   ```java
   public Builder parentStageId(String parentStageId) {
     this.parentStageId = parentStageId;
     return this;
   }
   ```
5. Pass `parentStageId` in `Builder.build()` constructor call
6. Add to `equals()`: `&& Objects.equals(parentStageId, milestone.parentStageId)`
7. Add to `hashCode()`: include `parentStageId` in `Objects.hash()`

- [ ] **Step 4: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=MilestoneParentStageTest -q`
Expected: PASS

- [ ] **Step 5: Run full api module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q`
Expected: PASS — existing Milestone construction without `parentStageId` passes null by default

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/Milestone.java \
       api/src/test/java/io/casehub/api/model/MilestoneParentStageTest.java
git commit -m "feat(#581): add Milestone.parentStageId for stage containment

Adds parentStageId field, getter, and builder method. Null for
case-level milestones not owned by any stage. Included in
equals/hashCode as structural identity.

Refs #581"
```

---

### Task 4: Upgrade CasePlanModel milestone tracking to MilestoneLifecycleStatus

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/CasePlanModel.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java`

**Interfaces:**
- Consumes: `MilestoneLifecycleStatus` (existing enum in `api/src/main/java/io/casehub/api/model/MilestoneLifecycleStatus.java`)
- Produces: `CasePlanModel.activateMilestone(String)`, `CasePlanModel.completeMilestone(String)`, `CasePlanModel.getMilestoneStatus(String)`, `CasePlanModel.trackMilestone(String, String)` — used by Task 5

- [ ] **Step 1: Write the failing tests**

Add to `blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java`:

```java
@Test
void milestone_tracks_as_pending() {
  plan.trackMilestone("doc-check");
  assertThat(plan.getMilestoneStatus("doc-check"))
      .isPresent()
      .hasValue(MilestoneLifecycleStatus.PENDING);
}

@Test
void activate_milestone_transitions_to_active() {
  plan.trackMilestone("doc-check");
  plan.activateMilestone("doc-check");
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.ACTIVE);
  assertThat(plan.isMilestoneAchieved("doc-check")).isFalse();
}

@Test
void complete_milestone_transitions_to_completed() {
  plan.trackMilestone("doc-check");
  plan.activateMilestone("doc-check");
  plan.completeMilestone("doc-check");
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.COMPLETED);
  assertThat(plan.isMilestoneAchieved("doc-check")).isTrue();
}

@Test
void complete_from_pending_handles_out_of_order_delivery() {
  plan.trackMilestone("doc-check");
  plan.completeMilestone("doc-check");
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.COMPLETED);
}

@Test
void activate_when_already_active_is_noop() {
  plan.trackMilestone("doc-check");
  plan.activateMilestone("doc-check");
  plan.activateMilestone("doc-check"); // second call — no-op
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.ACTIVE);
}

@Test
void activate_when_completed_is_noop() {
  plan.trackMilestone("doc-check");
  plan.completeMilestone("doc-check");
  plan.activateMilestone("doc-check"); // too late — no-op
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.COMPLETED);
}

@Test
void complete_when_already_completed_is_noop() {
  plan.trackMilestone("doc-check");
  plan.completeMilestone("doc-check");
  plan.completeMilestone("doc-check"); // second call — no-op
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.COMPLETED);
}

@Test
void get_milestone_status_unknown_returns_empty() {
  assertThat(plan.getMilestoneStatus("unknown")).isEmpty();
}

@Test
void activate_untracked_milestone_is_noop() {
  plan.activateMilestone("never-tracked"); // no exception
  assertThat(plan.getMilestoneStatus("never-tracked")).isEmpty();
}

@Test
void complete_untracked_milestone_is_noop() {
  plan.completeMilestone("never-tracked"); // no exception
  assertThat(plan.getMilestoneStatus("never-tracked")).isEmpty();
}

@Test
void track_milestone_with_parent_stage() {
  var stage = Stage.of("kyc-stage");
  plan.addStage(stage);
  plan.trackMilestone("doc-check", "kyc-stage");
  assertThat(plan.getMilestoneStatus("doc-check"))
      .hasValue(MilestoneLifecycleStatus.PENDING);
  assertThat(stage.getContainedMilestoneIds()).contains("doc-check");
}

@Test
void track_milestone_with_null_parent_stage() {
  plan.trackMilestone("case-level", null);
  assertThat(plan.getMilestoneStatus("case-level"))
      .hasValue(MilestoneLifecycleStatus.PENDING);
}

@Test
void track_milestone_with_nonexistent_stage_throws() {
  assertThatThrownBy(() -> plan.trackMilestone("doc-check", "nonexistent"))
      .isInstanceOf(IllegalArgumentException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-blackboard -Dtest=DefaultCasePlanModelTest -q`
Expected: FAIL — methods don't exist

- [ ] **Step 3: Update CasePlanModel interface**

In `blackboard/src/main/java/io/casehub/blackboard/plan/CasePlanModel.java`:

1. Add import: `import io.casehub.api.model.MilestoneLifecycleStatus;` and `import java.util.Optional;`
2. Add new methods:
```java
void activateMilestone(String milestoneName);
void completeMilestone(String milestoneName);
Optional<MilestoneLifecycleStatus> getMilestoneStatus(String milestoneName);
void trackMilestone(String milestoneName, String parentStageId);
```
3. Deprecate `achieveMilestone`:
```java
@Deprecated(forRemoval = true)
void achieveMilestone(String milestoneName);
```
4. Update Javadoc (line 33): replace "interim approach" note with: "Milestone lifecycle tracks PENDING → ACTIVE → COMPLETED via {@link MilestoneLifecycleStatus}. See MilestoneLifecycleManager for the event-driven state machine."

- [ ] **Step 4: Implement in DefaultCasePlanModel**

In `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java`:

1. Change field (line 45):
```java
private final ConcurrentHashMap<String, MilestoneLifecycleStatus> milestones = new ConcurrentHashMap<>();
```

2. Update `trackMilestone(String name)`:
```java
@Override
public void trackMilestone(String name) {
  milestones.putIfAbsent(name, MilestoneLifecycleStatus.PENDING);
}
```

3. Add `trackMilestone(String name, String parentStageId)`:
```java
@Override
public void trackMilestone(String name, String parentStageId) {
  milestones.putIfAbsent(name, MilestoneLifecycleStatus.PENDING);
  if (parentStageId != null) {
    Stage stage = getStage(parentStageId);
    if (stage == null) {
      throw new IllegalArgumentException(
          "Stage '%s' not found in plan — register the stage before its milestones".formatted(parentStageId));
    }
    stage.addMilestone(name);
  }
}
```

4. Add `activateMilestone`:
```java
@Override
public void activateMilestone(String name) {
  milestones.compute(name, (k, current) -> {
    if (current == null) {
      LOG.warnf("activateMilestone called for untracked milestone '%s' — ignoring", name);
      return null;
    }
    if (current == MilestoneLifecycleStatus.PENDING) {
      return MilestoneLifecycleStatus.ACTIVE;
    }
    LOG.warnf("activateMilestone called for milestone '%s' in state %s — ignoring", name, current);
    return current;
  });
}
```

5. Add `completeMilestone`:
```java
@Override
public void completeMilestone(String name) {
  milestones.compute(name, (k, current) -> {
    if (current == null) {
      LOG.warnf("completeMilestone called for untracked milestone '%s' — ignoring", name);
      return null;
    }
    if (current == MilestoneLifecycleStatus.PENDING || current == MilestoneLifecycleStatus.ACTIVE) {
      return MilestoneLifecycleStatus.COMPLETED;
    }
    LOG.warnf("completeMilestone called for milestone '%s' in state %s — ignoring", name, current);
    return current;
  });
}
```

6. Add `getMilestoneStatus`:
```java
@Override
public Optional<MilestoneLifecycleStatus> getMilestoneStatus(String name) {
  return Optional.ofNullable(milestones.get(name));
}
```

7. Deprecate and delegate `achieveMilestone`:
```java
@Deprecated(forRemoval = true)
@Override
public void achieveMilestone(String name) {
  completeMilestone(name);
}
```

8. Update `isMilestoneAchieved`:
```java
@Override
public boolean isMilestoneAchieved(String name) {
  return MilestoneLifecycleStatus.COMPLETED.equals(milestones.get(name));
}
```

- [ ] **Step 5: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-blackboard -Dtest=DefaultCasePlanModelTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add blackboard/src/main/java/io/casehub/blackboard/plan/CasePlanModel.java \
       blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java \
       blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java
git commit -m "feat(#581): upgrade CasePlanModel milestone tracking to MilestoneLifecycleStatus

Replaces Boolean milestone map with MilestoneLifecycleStatus. Adds
activateMilestone(), completeMilestone(), getMilestoneStatus(), and
trackMilestone(name, parentStageId) overload. Transition guards use
ConcurrentHashMap.compute() for atomicity. completeMilestone() accepts
PENDING→COMPLETED for out-of-order Vert.x delivery.

Refs #581"
```

---

### Task 5: Consolidate milestone evaluation and update handlers

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java:240-259`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/MilestoneAchievementHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneActivatedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneCompletedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneReachedEventHandler.java`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/handler/MilestoneAchievementHandlerTest.java`

**Interfaces:**
- Consumes: `CasePlanModel.activateMilestone()`, `CasePlanModel.completeMilestone()` (from Task 4)
- Produces: `CaseLifecycleEvent` on milestone activation and completion

- [ ] **Step 1: Remove milestones() from CaseContextChangedEventHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`:

1. Delete the `milestones()` method entirely (lines 240-259)
2. Remove the call to `milestones()` from the handler chain — find where it's called (likely in the `handleContextChanged` chain) and remove that `.chain(() -> milestones(...))` call
3. Remove any unused imports (`MilestoneReachedEvent`, `Milestone` if no longer used)

- [ ] **Step 2: Update MilestoneAchievementHandler to use lifecycle events**

Replace the entire handler body in `blackboard/src/main/java/io/casehub/blackboard/handler/MilestoneAchievementHandler.java`:

```java
@ConsumeEvent(EventBusAddresses.MILESTONE_ACTIVATED)
public Uni<Void> onMilestoneActivated(MilestoneActivatedEvent event) {
  registry.get(event.caseInstance().getUuid())
      .ifPresent(plan -> plan.activateMilestone(event.milestone().getName()));
  return Uni.createFrom().voidItem();
}

@ConsumeEvent(EventBusAddresses.MILESTONE_COMPLETED)
public Uni<Void> onMilestoneCompleted(MilestoneCompletedEvent event) {
  registry.get(event.caseInstance().getUuid())
      .ifPresent(plan -> plan.completeMilestone(event.milestone().getName()));
  return Uni.createFrom().voidItem();
}
```

Remove the old `onMilestoneReached` method. Update imports: add `MilestoneActivatedEvent`, `MilestoneCompletedEvent`; remove `MilestoneReachedEvent`.

- [ ] **Step 3: Add CaseLifecycleEvent to MilestoneActivatedEventHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneActivatedEventHandler.java`:

1. Add injection: `@Inject Event<CaseLifecycleEvent> lifecycleEvents;`
2. Add injection: `@Inject LedgerTraceIdProvider traceIdProvider;`
3. Add to the end of the `onMilestoneActivated` chain (after `scheduleSlaTimeoutJob`):

```java
.chain(() -> {
  String traceId = traceIdProvider.currentTraceId().orElse(null);
  return Uni.createFrom().completionStage(() ->
      lifecycleEvents.fireAsync(new CaseLifecycleEvent(
          caseInstance.getUuid(),
          caseInstance.tenancyId,
          "ActivateMilestone",
          "MilestoneActivated",
          caseInstance.getState().name(),
          null,
          "System",
          traceId)))
      .onFailure().recoverWithItem(t -> {
        LOG.warnf(t, "CaseLifecycleEvent observer failed for caseId=%s event=MilestoneActivated",
            caseInstance.getUuid());
        return null;
      })
      .replaceWithVoid();
})
```

- [ ] **Step 4: Add CaseLifecycleEvent to MilestoneCompletedEventHandler**

Same pattern as Step 3, in `MilestoneCompletedEventHandler.java`:

1. Add injection: `@Inject Event<CaseLifecycleEvent> lifecycleEvents;`
2. Add injection: `@Inject LedgerTraceIdProvider traceIdProvider;`
3. Add to the end of the `onMilestoneCompleted` chain (after `cancelSlaTimeoutJob`):

```java
.chain(() -> {
  String traceId = traceIdProvider.currentTraceId().orElse(null);
  return Uni.createFrom().completionStage(() ->
      lifecycleEvents.fireAsync(new CaseLifecycleEvent(
          caseInstance.getUuid(),
          caseInstance.tenancyId,
          "CompleteMilestone",
          "MilestoneCompleted",
          caseInstance.getState().name(),
          null,
          "System",
          traceId)))
      .onFailure().recoverWithItem(t -> {
        LOG.warnf(t, "CaseLifecycleEvent observer failed for caseId=%s event=MilestoneCompleted",
            caseInstance.getUuid());
        return null;
      })
      .replaceWithVoid();
})
```

- [ ] **Step 5: Deprecate MilestoneReachedEventHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneReachedEventHandler.java`:

Add `@Deprecated(forRemoval = true)` to the class. Add a Javadoc note:
```java
/**
 * @deprecated No remaining publishers fire MILESTONE_REACHED. Milestone lifecycle is handled by
 *     MilestoneActivatedEventHandler and MilestoneCompletedEventHandler. Retained for backwards
 *     compatibility with any external publisher using the address.
 */
```

- [ ] **Step 6: Update MilestoneAchievementHandlerTest**

Replace the existing test in `blackboard/src/test/java/io/casehub/blackboard/handler/MilestoneAchievementHandlerTest.java` to test both new event handlers:

```java
@Test
void activated_event_transitions_milestone_to_active() {
  plan.trackMilestone("docs-received");
  var event = new MilestoneActivatedEvent(caseInstance, milestone, Instant.now(), null);
  handler.onMilestoneActivated(event);
  assertThat(plan.getMilestoneStatus("docs-received"))
      .hasValue(MilestoneLifecycleStatus.ACTIVE);
}

@Test
void completed_event_transitions_milestone_to_completed() {
  plan.trackMilestone("docs-received");
  plan.activateMilestone("docs-received");
  var event = new MilestoneCompletedEvent(caseInstance, milestone, Instant.now());
  handler.onMilestoneCompleted(event);
  assertThat(plan.getMilestoneStatus("docs-received"))
      .hasValue(MilestoneLifecycleStatus.COMPLETED);
  assertThat(plan.isMilestoneAchieved("docs-received")).isTrue();
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl casehub-blackboard,runtime -q`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
       runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneActivatedEventHandler.java \
       runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneCompletedEventHandler.java \
       runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneReachedEventHandler.java \
       blackboard/src/main/java/io/casehub/blackboard/handler/MilestoneAchievementHandler.java \
       blackboard/src/test/java/io/casehub/blackboard/handler/MilestoneAchievementHandlerTest.java
git commit -m "feat(#581): consolidate milestone evaluation to lifecycle path

Removes CaseContextChangedEventHandler.milestones() — MilestoneLifecycleManager
is the sole evaluation path. MilestoneAchievementHandler now listens to
MILESTONE_ACTIVATED and MILESTONE_COMPLETED instead of MILESTONE_REACHED.
Adds CaseLifecycleEvent firing to both lifecycle handlers for audit trail.
Deprecates MilestoneReachedEventHandler.

Refs #581"
```

---

### Task 6: Create follow-on issue for GoalBasedCompletion TODO

**Files:** None (GitHub issue only)

**Interfaces:**
- Consumes: nothing
- Produces: GitHub issue tracking GoalBasedCompletion generalization

- [ ] **Step 1: Create the follow-on issue**

```bash
gh issue create --repo casehubio/engine \
  --title "feat: generalize GoalBasedCompletion to support multiple goal kinds beyond success/failure" \
  --body "GoalBasedCompletion.java:18-19 contains a TODO:

\`\`\`java
// TODO this must be replaced by a more generic implementation that can support
// multiple goals of different kinds, not just success and failure
\`\`\`

This is completion mechanism redesign, not conceptual alignment. Out of scope for epic #84 but tracked here as a follow-on.

Refs #581, #84"
```

- [ ] **Step 2: Commit nothing — issue-only task**

No code changes. The issue is the deliverable.
