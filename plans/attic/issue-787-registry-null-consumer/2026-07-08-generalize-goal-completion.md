# Generalize GoalBasedCompletion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #582 — feat: generalize GoalBasedCompletion to support multiple goal kinds beyond success/failure
**Issue group:** #582

**Goal:** Replace the hardcoded two-field `GoalBasedCompletion(success, failure)` with a generic `GoalBasedCompletion<K extends GoalKind>` backed by an insertion-ordered map, enabling domains to define arbitrary terminal goal kinds.

**Architecture:** `GoalKind` becomes an interface with `value()` and `terminalStatus()`. `StandardGoalKind` enum provides SUCCESS/FAILURE built-ins. `GoalBasedCompletion<K>` stores a `LinkedHashMap<K, GoalExpression>` — insertion order is evaluation priority, first satisfied expression wins. `Goal.kind` changes from `GoalKind` (enum) to `String` — audit metadata only, decoupled from the completion-map concern. `CaseStatusChanged` denormalizes `GoalKind` to `String` for event transport.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x EventBus, jsonschema2pojo

## Global Constraints

- **Spec:** `docs/specs/2026-07-08-generalize-goal-completion-design.md` — the adversarially reviewed spec is the source of truth
- **No backward-compat shims** — breaking API changes are intentional; callers must be explicit
- **GoalKind equality contract** — implementations must provide value-based equals/hashCode (they serve as map keys)
- **CANCELLED is not a valid goal terminal status** — only COMPLETED and FAULTED (no CASE_CANCELLED event bus address exists)
- **doneWhen mutual exclusion** — completion block cannot mix `doneWhen` with goal kind entries
- **Reserved kind names** — `"doneWhen"` is reserved; cannot be used as a goal kind name
- **Standard kind status override rejected** — explicit `status` on `success`/`failure` entries is an error
- **Module:** `api` is Tier 1 (pure Java, no Quarkus runtime) — GoalKind, StandardGoalKind, DefaultGoalKind, GoalBasedCompletion, Goal all live here
- **SPI evolution protocol** — new optional SPI capabilities added as Java interface default methods with safe no-op returns
- **IntelliJ MCP** — use `mcp__intellij-index__*` tools for all code navigation and refactoring; never bash grep on source files
- **Maven test workflow** — `mvn install -DskipTests -q` before module tests; include `TESTCONTAINERS_RYUK_DISABLED=true`

---

### Task 1: GoalKind interface + StandardGoalKind enum + DefaultGoalKind record

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/GoalKind.java` — enum → interface
- Create: `api/src/main/java/io/casehub/api/model/StandardGoalKind.java` — built-in enum
- Create: `api/src/main/java/io/casehub/api/model/DefaultGoalKind.java` — package-private record
- Modify: `runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java` — GoalKindFromValueTests → StandardGoalKindFromValueTests + new GoalKind interface tests
- Create: `api/src/test/java/io/casehub/api/model/GoalKindTest.java` — unit tests for GoalKind.of(), GoalKind.fromValue(), DefaultGoalKind equality

**Interfaces:**
- Produces: `GoalKind` interface with `value(): String`, `terminalStatus(): CaseStatus`, `SUCCESS` / `FAILURE` constants, `of(String, CaseStatus)` factory, `fromValue(String)` resolver
- Produces: `StandardGoalKind` enum implementing `GoalKind` with `SUCCESS("success", COMPLETED)`, `FAILURE("failure", FAULTED)`, `fromValue(String)`
- Produces: `DefaultGoalKind` record implementing `GoalKind` — used by YAML mapper for custom kinds

- [ ] **Step 1: Write failing tests for GoalKind interface and StandardGoalKind**

Create `api/src/test/java/io/casehub/api/model/GoalKindTest.java`:

```java
package io.casehub.api.model;

import static org.junit.jupiter.api.Assertions.*;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

class GoalKindTest {

  @Nested
  @DisplayName("GoalKind interface constants")
  class InterfaceConstantsTests {

    @Test
    @DisplayName("GoalKind.SUCCESS delegates to StandardGoalKind.SUCCESS")
    void success_delegatesToStandard() {
      assertSame(StandardGoalKind.SUCCESS, GoalKind.SUCCESS);
      assertEquals("success", GoalKind.SUCCESS.value());
      assertEquals(CaseStatus.COMPLETED, GoalKind.SUCCESS.terminalStatus());
    }

    @Test
    @DisplayName("GoalKind.FAILURE delegates to StandardGoalKind.FAILURE")
    void failure_delegatesToStandard() {
      assertSame(StandardGoalKind.FAILURE, GoalKind.FAILURE);
      assertEquals("failure", GoalKind.FAILURE.value());
      assertEquals(CaseStatus.FAULTED, GoalKind.FAILURE.terminalStatus());
    }
  }

  @Nested
  @DisplayName("GoalKind.of() factory")
  class OfFactoryTests {

    @Test
    @DisplayName("creates custom GoalKind with correct value and status")
    void of_createsCustomKind() {
      GoalKind escalated = GoalKind.of("escalated", CaseStatus.FAULTED);
      assertEquals("escalated", escalated.value());
      assertEquals(CaseStatus.FAULTED, escalated.terminalStatus());
    }

    @Test
    @DisplayName("null value throws NullPointerException")
    void of_nullValue_throws() {
      assertThrows(NullPointerException.class,
          () -> GoalKind.of(null, CaseStatus.FAULTED));
    }

    @Test
    @DisplayName("null terminalStatus throws NullPointerException")
    void of_nullStatus_throws() {
      assertThrows(NullPointerException.class,
          () -> GoalKind.of("escalated", null));
    }

    @Test
    @DisplayName("two custom kinds with same value and status are equal")
    void of_sameValueAndStatus_equal() {
      GoalKind a = GoalKind.of("escalated", CaseStatus.FAULTED);
      GoalKind b = GoalKind.of("escalated", CaseStatus.FAULTED);
      assertEquals(a, b);
      assertEquals(a.hashCode(), b.hashCode());
    }

    @Test
    @DisplayName("custom kinds with different values are not equal")
    void of_differentValues_notEqual() {
      GoalKind a = GoalKind.of("escalated", CaseStatus.FAULTED);
      GoalKind b = GoalKind.of("referred", CaseStatus.FAULTED);
      assertNotEquals(a, b);
    }
  }

  @Nested
  @DisplayName("GoalKind.fromValue()")
  class FromValueTests {

    @Test
    @DisplayName("'success' returns StandardGoalKind.SUCCESS")
    void success_returnsStandard() {
      assertEquals(StandardGoalKind.SUCCESS, GoalKind.fromValue("success"));
    }

    @Test
    @DisplayName("'failure' returns StandardGoalKind.FAILURE")
    void failure_returnsStandard() {
      assertEquals(StandardGoalKind.FAILURE, GoalKind.fromValue("failure"));
    }

    @Test
    @DisplayName("unknown value throws IllegalArgumentException")
    void unknown_throws() {
      var ex = assertThrows(IllegalArgumentException.class,
          () -> GoalKind.fromValue("escalated"));
      assertTrue(ex.getMessage().contains("GoalKind.of"));
    }
  }

  @Nested
  @DisplayName("StandardGoalKind")
  class StandardGoalKindTests {

    @Test
    @DisplayName("fromValue('success') returns SUCCESS")
    void fromValue_success() {
      assertEquals(StandardGoalKind.SUCCESS, StandardGoalKind.fromValue("success"));
    }

    @Test
    @DisplayName("fromValue('failure') returns FAILURE")
    void fromValue_failure() {
      assertEquals(StandardGoalKind.FAILURE, StandardGoalKind.fromValue("failure"));
    }

    @Test
    @DisplayName("fromValue('unknown') throws")
    void fromValue_unknown_throws() {
      assertThrows(IllegalArgumentException.class,
          () -> StandardGoalKind.fromValue("unknown"));
    }

    @Test
    @DisplayName("value() returns lower-case string")
    void value_returnsLowercase() {
      assertEquals("success", StandardGoalKind.SUCCESS.value());
      assertEquals("failure", StandardGoalKind.FAILURE.value());
    }

    @Test
    @DisplayName("toString() matches value()")
    void toString_matchesValue() {
      assertEquals("success", StandardGoalKind.SUCCESS.toString());
      assertEquals("failure", StandardGoalKind.FAILURE.toString());
    }
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=GoalKindTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: compilation errors — `StandardGoalKind` and `GoalKind.of()` don't exist yet

- [ ] **Step 3: Implement GoalKind interface**

Replace the contents of `api/src/main/java/io/casehub/api/model/GoalKind.java`:

```java
package io.casehub.api.model;

/**
 * Classifies a goal kind and maps it to a terminal {@link CaseStatus}.
 *
 * <p>Implementations must provide value-based equals/hashCode — GoalKind instances
 * serve as map keys in {@link GoalBasedCompletion}.
 */
public interface GoalKind {

  String value();

  CaseStatus terminalStatus();

  GoalKind SUCCESS = StandardGoalKind.SUCCESS;
  GoalKind FAILURE = StandardGoalKind.FAILURE;

  static GoalKind of(String value, CaseStatus terminalStatus) {
    return new DefaultGoalKind(value, terminalStatus);
  }

  static GoalKind fromValue(String value) {
    try {
      return StandardGoalKind.fromValue(value);
    } catch (IllegalArgumentException e) {
      throw new IllegalArgumentException(
          "Unknown GoalKind: " + value
              + " — custom kinds must be created with GoalKind.of(value, terminalStatus)");
    }
  }
}
```

- [ ] **Step 4: Implement StandardGoalKind enum**

Create `api/src/main/java/io/casehub/api/model/StandardGoalKind.java`:

```java
package io.casehub.api.model;

public enum StandardGoalKind implements GoalKind {
  SUCCESS("success", CaseStatus.COMPLETED),
  FAILURE("failure", CaseStatus.FAULTED);

  private final String value;
  private final CaseStatus terminalStatus;

  StandardGoalKind(String value, CaseStatus terminalStatus) {
    this.value = value;
    this.terminalStatus = terminalStatus;
  }

  @Override
  public String value() {
    return value;
  }

  @Override
  public CaseStatus terminalStatus() {
    return terminalStatus;
  }

  @Override
  public String toString() {
    return value;
  }

  public static StandardGoalKind fromValue(String value) {
    for (StandardGoalKind kind : values()) {
      if (kind.value.equals(value)) {
        return kind;
      }
    }
    throw new IllegalArgumentException("Unknown StandardGoalKind: " + value);
  }
}
```

- [ ] **Step 5: Implement DefaultGoalKind record**

Create `api/src/main/java/io/casehub/api/model/DefaultGoalKind.java`:

```java
package io.casehub.api.model;

import java.util.Objects;

record DefaultGoalKind(String value, CaseStatus terminalStatus) implements GoalKind {
  DefaultGoalKind {
    Objects.requireNonNull(value, "value must not be null");
    Objects.requireNonNull(terminalStatus, "terminalStatus must not be null");
  }

  @Override
  public String toString() {
    return value;
  }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=GoalKindTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all pass

- [ ] **Step 7: Migrate GoalKindFromValueTests in ModelBuilderTest**

In `runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java`, update the `GoalKindFromValueTests` nested class (lines 507-553):
- Change `GoalKind.fromValue(...)` calls to `StandardGoalKind.fromValue(...)` for the direct enum tests
- Keep `GoalKind.SUCCESS`/`GoalKind.FAILURE` constant tests (they now test the interface delegation)
- Update assertions from `assertEquals(GoalKind.SUCCESS, ...)` to `assertEquals(StandardGoalKind.SUCCESS, ...)`

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/GoalKind.java api/src/main/java/io/casehub/api/model/StandardGoalKind.java api/src/main/java/io/casehub/api/model/DefaultGoalKind.java api/src/test/java/io/casehub/api/model/GoalKindTest.java runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java
git commit -m "feat(#582): GoalKind interface + StandardGoalKind enum + DefaultGoalKind record

GoalKind becomes an interface with value() and terminalStatus().
StandardGoalKind enum provides SUCCESS/FAILURE built-ins.
DefaultGoalKind record backs the GoalKind.of() factory for custom kinds.
GoalKind.SUCCESS and GoalKind.FAILURE constants delegate to the enum.

Refs #582"
```

---

### Task 2: GoalBasedCompletion<K extends GoalKind> — generic class with ordered map

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/GoalBasedCompletion.java` — full rewrite
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — builder completion methods
- Create: `api/src/test/java/io/casehub/api/model/GoalBasedCompletionTest.java` — builder tests
- Modify: `runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java` — update CaseDefinitionBuilderTests

**Interfaces:**
- Consumes: `GoalKind` interface, `StandardGoalKind` enum from Task 1
- Produces: `GoalBasedCompletion<K extends GoalKind>` with `getGoals(): Map<K, GoalExpression>`, `Builder<K>` with `goal(K, GoalExpression)`, duplicate-kind detection

- [ ] **Step 1: Write failing tests for GoalBasedCompletion builder**

Create `api/src/test/java/io/casehub/api/model/GoalBasedCompletionTest.java`:

```java
package io.casehub.api.model;

import static org.junit.jupiter.api.Assertions.*;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

class GoalBasedCompletionTest {

  private final Goal goalA =
      Goal.builder().name("a").condition(".a == true").kind(GoalKind.SUCCESS).build();
  private final Goal goalB =
      Goal.builder().name("b").condition(".b == true").kind(GoalKind.FAILURE).build();

  @Nested
  @DisplayName("Builder")
  class BuilderTests {

    @Test
    @DisplayName("single goal kind creates completion with one entry")
    void singleKind() {
      var completion = GoalBasedCompletion.<StandardGoalKind>builder()
          .goal(StandardGoalKind.SUCCESS, GoalExpression.allOf(goalA))
          .build();
      assertEquals(1, completion.getGoals().size());
      assertNotNull(completion.getGoals().get(StandardGoalKind.SUCCESS));
    }

    @Test
    @DisplayName("multiple goal kinds preserve insertion order")
    void insertionOrderPreserved() {
      var completion = GoalBasedCompletion.<StandardGoalKind>builder()
          .goal(StandardGoalKind.FAILURE, GoalExpression.anyOf(goalB))
          .goal(StandardGoalKind.SUCCESS, GoalExpression.allOf(goalA))
          .build();

      List<GoalKind> keys = new ArrayList<>(completion.getGoals().keySet());
      assertEquals(StandardGoalKind.FAILURE, keys.get(0));
      assertEquals(StandardGoalKind.SUCCESS, keys.get(1));
    }

    @Test
    @DisplayName("duplicate goal kind throws IllegalStateException")
    void duplicateKind_throws() {
      var builder = GoalBasedCompletion.<StandardGoalKind>builder()
          .goal(StandardGoalKind.SUCCESS, GoalExpression.allOf(goalA));
      assertThrows(IllegalStateException.class,
          () -> builder.goal(StandardGoalKind.SUCCESS, GoalExpression.anyOf(goalB)));
    }

    @Test
    @DisplayName("empty builder throws IllegalStateException")
    void emptyBuilder_throws() {
      assertThrows(IllegalStateException.class,
          () -> GoalBasedCompletion.<StandardGoalKind>builder().build());
    }

    @Test
    @DisplayName("null kind throws NullPointerException")
    void nullKind_throws() {
      assertThrows(NullPointerException.class,
          () -> GoalBasedCompletion.<StandardGoalKind>builder()
              .goal(null, GoalExpression.allOf(goalA)));
    }

    @Test
    @DisplayName("null expression throws NullPointerException")
    void nullExpression_throws() {
      assertThrows(NullPointerException.class,
          () -> GoalBasedCompletion.<StandardGoalKind>builder()
              .goal(StandardGoalKind.SUCCESS, null));
    }

    @Test
    @DisplayName("goals map is unmodifiable")
    void goalsMapUnmodifiable() {
      var completion = GoalBasedCompletion.<StandardGoalKind>builder()
          .goal(StandardGoalKind.SUCCESS, GoalExpression.allOf(goalA))
          .build();
      assertThrows(UnsupportedOperationException.class,
          () -> completion.getGoals().put(StandardGoalKind.FAILURE, GoalExpression.anyOf(goalB)));
    }

    @Test
    @DisplayName("custom GoalKind works as map key")
    void customGoalKind() {
      GoalKind escalated = GoalKind.of("escalated", CaseStatus.FAULTED);
      var completion = GoalBasedCompletion.builder()
          .goal(escalated, GoalExpression.allOf(goalA))
          .build();
      assertEquals(1, completion.getGoals().size());
      assertNotNull(completion.getGoals().get(GoalKind.of("escalated", CaseStatus.FAULTED)));
    }
  }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=GoalBasedCompletionTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: compilation errors — GoalBasedCompletion doesn't have the new API yet

- [ ] **Step 3: Rewrite GoalBasedCompletion**

Replace contents of `api/src/main/java/io/casehub/api/model/GoalBasedCompletion.java`:

```java
package io.casehub.api.model;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

public class GoalBasedCompletion<K extends GoalKind> implements CaseCompletion {

  private final LinkedHashMap<K, GoalExpression> goals;

  private GoalBasedCompletion(LinkedHashMap<K, GoalExpression> goals) {
    this.goals = goals;
  }

  public Map<K, GoalExpression> getGoals() {
    return Collections.unmodifiableMap(goals);
  }

  public static <K extends GoalKind> Builder<K> builder() {
    return new Builder<>();
  }

  public static class Builder<K extends GoalKind> {
    private final LinkedHashMap<K, GoalExpression> goals = new LinkedHashMap<>();

    public Builder<K> goal(K kind, GoalExpression expression) {
      Objects.requireNonNull(kind, "kind must not be null");
      Objects.requireNonNull(expression, "expression must not be null");
      if (goals.containsKey(kind)) {
        throw new IllegalStateException("Duplicate goal kind: " + kind.value());
      }
      goals.put(kind, expression);
      return this;
    }

    public GoalBasedCompletion<K> build() {
      if (goals.isEmpty()) {
        throw new IllegalStateException(
            "GoalBasedCompletion requires at least one goal kind");
      }
      return new GoalBasedCompletion<>(new LinkedHashMap<>(goals));
    }
  }
}
```

- [ ] **Step 4: Update CaseDefinition.Builder completion methods**

In `api/src/main/java/io/casehub/api/model/CaseDefinition.java`:
- Remove the `completion(GoalExpression success)` method (line 337-339)
- Rewrite `completion(GoalExpression success, GoalExpression failure)` (line 341-344) to use the builder
- Add `completion(GoalBasedCompletion<?> completion)` method
- Keep `completion(String when)` unchanged

```java
public Builder completion(GoalExpression success, GoalExpression failure) {
  var gbc = GoalBasedCompletion.<StandardGoalKind>builder();
  if (failure != null) gbc.goal(StandardGoalKind.FAILURE, failure);
  if (success != null) gbc.goal(StandardGoalKind.SUCCESS, success);
  this.completion = gbc.build();
  return this;
}

public Builder completion(GoalExpression success) {
  return completion(success, null);
}

public Builder completion(GoalBasedCompletion<?> completion) {
  this.completion = completion;
  return this;
}

public Builder completion(String when) {
  this.completion = new PredicateBasedCompletion(new JQExpressionEvaluator(when));
  return this;
}
```

- [ ] **Step 5: Update ModelBuilderTest CaseDefinitionBuilderTests**

In `runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java`, update:
- `completionGoalExpression_createsGoalBased` — change `gbc.getSuccess()` → iterate `gbc.getGoals()`, assert contains `StandardGoalKind.SUCCESS`
- `completionSuccessFailure_storesBoth` — same pattern, assert both keys present
- Add test: `completionGoalBasedCompletion_storesDirectly` — tests the new `completion(GoalBasedCompletion<?>)` overload

- [ ] **Step 6: Run all api and runtime tests**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all pass

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/GoalBasedCompletion.java api/src/main/java/io/casehub/api/model/CaseDefinition.java api/src/test/java/io/casehub/api/model/GoalBasedCompletionTest.java runtime/src/test/java/io/casehub/engine/model/ModelBuilderTest.java
git commit -m "feat(#582): GoalBasedCompletion<K extends GoalKind> with ordered map builder

Replaces hardcoded success/failure fields with LinkedHashMap<K, GoalExpression>.
Insertion order determines evaluation priority — first match wins.
Builder rejects duplicate kinds and empty completion.
CaseDefinition.Builder gains completion(GoalBasedCompletion<?>) overload.

Refs #582"
```

---

### Task 3: Goal.kind from GoalKind to String + CaseStatusChanged denormalization

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Goal.java` — kind field: `GoalKind` → `String`, builder overloads
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/CaseStatusChanged.java` — `GoalKind` → `String`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java` — metadata uses `goal.getKind()` (now String), evaluateCompletion iterates map
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java` — `fireOutcomeObservers` param type
- Modify: `api/src/test/java/io/casehub/api/model/GoalTest.java` — kind assertion type
- Modify: `runtime/src/test/java/io/casehub/engine/model/GoalExpressionTest.java` — GoalKind references
- Modify: ~40 test files — `GoalKind.SUCCESS` still compiles but assertions for `goal.getKind()` return `String` not `GoalKind`

**Interfaces:**
- Consumes: `GoalKind` interface from Task 1
- Produces: `Goal.getKind(): String`, `Goal.Builder.kind(String)`, `Goal.Builder.kind(GoalKind)` overload
- Produces: `CaseStatusChanged(instance, oldStatus, newStatus, satisfiedGoalName, String satisfiedGoalKind)`

- [ ] **Step 1: Write failing tests for Goal.kind as String**

In `api/src/test/java/io/casehub/api/model/GoalTest.java`, add:

```java
@Test
@DisplayName("kind() returns String value, not GoalKind object")
void kind_returnsString() {
  Goal goal = Goal.builder()
      .name("test")
      .condition(".done == true")
      .kind(GoalKind.SUCCESS)
      .build();
  assertEquals("success", goal.getKind());
}

@Test
@DisplayName("kind(String) builder accepts raw string")
void kind_rawString() {
  Goal goal = Goal.builder()
      .name("test")
      .condition(".done == true")
      .kind("escalated")
      .build();
  assertEquals("escalated", goal.getKind());
}

@Test
@DisplayName("equals uses value equality for kind")
void equals_valueEquality() {
  Goal a = Goal.builder().name("g").condition(".x == true").kind("success").build();
  Goal b = Goal.builder().name("g").condition(".x == true").kind(GoalKind.SUCCESS).build();
  assertEquals(a, b);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=GoalTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: compilation errors — `getKind()` returns `GoalKind` not `String`

- [ ] **Step 3: Modify Goal class**

In `api/src/main/java/io/casehub/api/model/Goal.java`:
- Change `private final GoalKind kind` → `private final String kind` (line 60)
- Change constructor param `GoalKind kind` → `String kind` (line 63)
- Change `getKind()` return type → `String` (line 85)
- In Builder: change `private GoalKind kind` → `private String kind` (line 97)
- Change `kind(GoalKind kind)` to `kind(String kind)` (line 126)
- Add overload `kind(GoalKind kind) { return kind(kind.value()); }`
- In `equals()`: change `kind == goal.kind` → `Objects.equals(kind, goal.kind)` (line 151)

- [ ] **Step 4: Modify CaseStatusChanged record**

In `common/src/main/java/io/casehub/engine/common/internal/event/CaseStatusChanged.java`:
- Change `GoalKind satisfiedGoalKind` → `String satisfiedGoalKind` (line 36)
- Remove `import io.casehub.api.model.GoalKind`

- [ ] **Step 5: Update GoalReachedEventHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/GoalReachedEventHandler.java`:
- Line 85: change `goal.getKind().value()` → `goal.getKind()` (already a String)
- Replace `evaluateCompletion` method body (lines 113-170) with the ordered-map iteration from the spec
- Remove import of `GoalKind` (now unused — the handler uses the `GoalKind` interface via the map keys)

- [ ] **Step 6: Update CaseStatusChangedHandler**

In `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`:
- Line 103: change `event.satisfiedGoalKind().value()` → `event.satisfiedGoalKind()` (already a String)
- `fireOutcomeObservers` method (line 199): change param `GoalKind goalKind` → `String goalKind`
- Line 222: change `goalKind.value()` → `goalKind`

- [ ] **Step 7: Fix compilation across ~40 test files**

Most test files use `GoalKind.SUCCESS` in `Goal.builder().kind(GoalKind.SUCCESS)` — this still compiles via the `kind(GoalKind)` overload.

Files that assert on `goal.getKind()` need updating:
- `CaseDefinitionYamlMapperTest.java` lines 812, 816: `assertThat(goal.getKind()).isEqualTo(GoalKind.SUCCESS)` → `assertThat(goal.getKind()).isEqualTo("success")`
- `GoalExpressionTest.java` lines 49-51, 234, 242, 250, 259: `GoalKind.SUCCESS` / `GoalKind.FAILURE` — these are in `Goal.builder().kind(...)` calls, so they compile unchanged via the overload
- `ModelBuilderTest.java` — Goal equals test (line 463): `GoalKind.FAILURE` in builder is fine; but if there's an assertion on `getKind()`, change to String

Use `ide_find_references` on `Goal.getKind()` to find all assertion sites.

- [ ] **Step 8: Run full build**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,common,runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all pass

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat(#582): Goal.kind → String, CaseStatusChanged denormalized

Goal.kind changes from GoalKind (enum) to String — audit metadata only,
decoupled from the completion-map concern. Builder preserves kind(GoalKind)
overload for compile-time safety. Goal.equals() uses Objects.equals() for
value equality.

CaseStatusChanged carries String satisfiedGoalKind instead of GoalKind —
the event transport layer uses string values, not typed interfaces.

GoalReachedEventHandler.evaluateCompletion() iterates the ordered map
instead of hardcoded success/failure branches.

Refs #582"
```

---

### Task 4: YAML schema + CaseDefinitionYamlMapper — open completion block

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — Goal.kind enum→string, CaseCompletion open structure
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — new completion parsing + Goal kind simplified
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — new tests for custom kinds, validation errors
- Modify: `runtime/src/test/resources/casehub/simple.yaml` — verify existing format still works
- Modify: `flow/src/test/resources/casehub/simple.yaml` — verify existing format still works

**Interfaces:**
- Consumes: `GoalKind.of(String, CaseStatus)` from Task 1, `GoalBasedCompletion.builder()` from Task 2, `Goal.kind` as String from Task 3
- Produces: YAML parsing that creates `GoalBasedCompletion<GoalKind>` with custom kinds resolved from `status:` fields

- [ ] **Step 1: Write failing tests for new YAML completion parsing**

In `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`, add:

```java
@Test
@DisplayName("completion with custom kind and explicit status parsed correctly")
void completion_customKind_parsed() throws IOException {
  String yaml = """
      namespace: test
      name: Custom Goals
      version: 1.0.0
      spec:
        goals:
          - name: fraud-detected
            condition: '.fraudScore > 0.8'
            kind: failure
          - name: review-needed
            condition: '.needsReview == true'
            kind: escalated
          - name: done
            condition: '.decision != null'
            kind: success
        completion:
          failure:
            anyOf: [fraud-detected]
          escalated:
            status: FAULTED
            anyOf: [review-needed]
          success:
            allOf: [done]
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

  assertThat(def.getCompletion()).isInstanceOf(GoalBasedCompletion.class);
  GoalBasedCompletion<?> gbc = (GoalBasedCompletion<?>) def.getCompletion();
  var keys = new java.util.ArrayList<>(gbc.getGoals().keySet());
  assertEquals(3, keys.size());
  assertEquals("failure", keys.get(0).value());
  assertEquals(CaseStatus.FAULTED, keys.get(0).terminalStatus());
  assertEquals("escalated", keys.get(1).value());
  assertEquals(CaseStatus.FAULTED, keys.get(1).terminalStatus());
  assertEquals("success", keys.get(2).value());
  assertEquals(CaseStatus.COMPLETED, keys.get(2).terminalStatus());
}

@Test
@DisplayName("standard kind with explicit status throws")
void completion_standardKindWithStatus_throws() {
  String yaml = """
      namespace: test
      name: Bad Override
      version: 1.0.0
      spec:
        goals:
          - name: done
            condition: '.done == true'
            kind: success
        completion:
          success:
            status: FAULTED
            allOf: [done]
      """;
  assertThrows(IllegalArgumentException.class,
      () -> CaseDefinitionYamlMapper.load(
          new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))));
}

@Test
@DisplayName("doneWhen with goal entries throws mutual exclusion error")
void completion_doneWhenWithGoalEntries_throws() {
  String yaml = """
      namespace: test
      name: Mixed
      version: 1.0.0
      spec:
        goals:
          - name: done
            condition: '.done == true'
            kind: success
        completion:
          doneWhen: '.allDone == true'
          success:
            allOf: [done]
      """;
  assertThrows(IllegalArgumentException.class,
      () -> CaseDefinitionYamlMapper.load(
          new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))));
}

@Test
@DisplayName("custom kind without status throws")
void completion_customKindWithoutStatus_throws() {
  String yaml = """
      namespace: test
      name: Missing Status
      version: 1.0.0
      spec:
        goals:
          - name: review-needed
            condition: '.needsReview == true'
            kind: escalated
        completion:
          escalated:
            anyOf: [review-needed]
      """;
  assertThrows(IllegalArgumentException.class,
      () -> CaseDefinitionYamlMapper.load(
          new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))));
}

@Test
@DisplayName("reserved kind name 'doneWhen' as goal kind throws")
void completion_reservedKindName_throws() {
  String yaml = """
      namespace: test
      name: Reserved
      version: 1.0.0
      spec:
        goals:
          - name: done
            condition: '.done == true'
            kind: doneWhen
        completion:
          doneWhen:
            status: COMPLETED
            allOf: [done]
      """;
  assertThrows(IllegalArgumentException.class,
      () -> CaseDefinitionYamlMapper.load(
          new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))));
}

@Test
@DisplayName("existing success/failure YAML format still works")
void completion_existingFormat_stillWorks() throws IOException {
  String yaml = """
      namespace: test
      name: Legacy
      version: 1.0.0
      spec:
        goals:
          - name: pr-approved
            condition: '.approved == true'
            kind: success
          - name: pr-sla-breached
            condition: '.slaBreached == true'
            kind: failure
        completion:
          failure:
            anyOf: [pr-sla-breached]
          success:
            allOf: [pr-approved]
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
  assertThat(def.getCompletion()).isInstanceOf(GoalBasedCompletion.class);
  GoalBasedCompletion<?> gbc = (GoalBasedCompletion<?>) def.getCompletion();
  assertEquals(2, gbc.getGoals().size());
}

@Test
@DisplayName("goal kind as string in YAML (not enum)")
void goal_kindAsString() throws IOException {
  String yaml = """
      namespace: test
      name: String Kind
      version: 1.0.0
      spec:
        goals:
          - name: done
            condition: '.done == true'
            kind: escalated
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));
  assertEquals("escalated", def.getGoals().get(0).getKind());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: failures — YAML mapper doesn't handle the new completion format yet

- [ ] **Step 3: Update YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`:

1. Goal.kind (line 509-511): change `type: string / enum: [success, failure]` → `type: string` (remove enum constraint)
2. CaseCompletion (lines 518-525): replace the fixed `success`/`failure` properties with open structure — keep `doneWhen` as a named property, use `additionalProperties` for goal kind entries

```yaml
  CaseCompletion:
    type: object
    description: >
      Maps goal kinds to goal expressions. Document order determines evaluation
      priority — first satisfied expression wins. Built-in kinds (success, failure)
      have implicit terminal status mappings. Custom kinds require an explicit
      'status' field.
    properties:
      doneWhen:
        type: string
        description: "Optional JQ predicate over CaseContext as an override/shortcut"
    additionalProperties:
      description: >
        Each additional property is a goal kind name mapped to a GoalExpression.
        Built-in kinds (success, failure) have implicit terminal status.
        Custom kinds must include a 'status' field.
      oneOf:
        - $ref: "#/$defs/GoalExpression"
        - type: object
          properties:
            status:
              type: string
              enum: [COMPLETED, FAULTED]
            allOf:
              type: array
              items: { type: string }
              minItems: 1
            anyOf:
              type: array
              items: { type: string }
              minItems: 1
          required: [status]
```

- [ ] **Step 4: Regenerate schema models**

Run: `mvn generate-sources -pl schema -f /Users/mdproctor/claude/casehub/engine/pom.xml`

Verify generated `io.casehub.model.Goal` no longer has a `Kind` inner enum — `getKind()` returns `String`. Verify generated `io.casehub.model.CaseCompletion` loses fixed `success`/`failure` properties and gains `additionalProperties`.

- [ ] **Step 5: Update CaseDefinitionYamlMapper**

In `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`:

1. **Goal parsing** (lines 436-449): simplify — `Goal.kind` is now String, read directly from `sg.getKind()` (String from generated model). Remove `GoalKind.fromValue()` call.

```java
// Convert goals
final Map<String, Goal> goalMap = new LinkedHashMap<>();
if (schema.getSpec() != null && schema.getSpec().getGoals() != null) {
  for (io.casehub.model.Goal sg : schema.getSpec().getGoals()) {
    final Goal goal = new Goal(
        sg.getName(),
        registry.create(sg.getCondition(), expressionLang),
        sg.getKind() != null ? sg.getKind() : "success");
    goal.setDescription(sg.getDescription());
    goalMap.put(sg.getName(), goal);
    def.getGoals().add(goal);
  }
}
```

2. **Completion parsing** (lines 452-457): replace with raw JSON node iteration from the spec. Read `completionNode` from `rawNode.get("spec").get("completion")`. Iterate fields, skip `doneWhen`, resolve each kind via `resolveGoalKind()`, parse expression via new `parseGoalExpression(JsonNode, goalMap)` method.

3. **Add `resolveGoalKind(String kindValue, JsonNode exprNode)` method:**
   - `"doneWhen"` → throw `IllegalArgumentException` (reserved name)
   - `"success"` or `"failure"` → check for explicit `status` field; if present, throw. Return `StandardGoalKind.fromValue(kindValue)`
   - Anything else → read `status` field from `exprNode`; if missing, throw. Return `GoalKind.of(kindValue, CaseStatus.valueOf(status))`

4. **Add `parseGoalExpression(JsonNode node, Map<String, Goal> goalMap)` method:**
   - Reads `allOf` or `anyOf` string arrays from the JSON node
   - Resolves goal names against `goalMap`
   - Returns `AllOfGoalExpression` or `AnyOfGoalExpression`
   - Ignores `status` field (consumed by `resolveGoalKind`)

5. **Mutual exclusion check:**
   - If `doneWhen` is present AND goal kind entries exist → throw `IllegalArgumentException`

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all pass (including existing tests with the standard success/failure format)

- [ ] **Step 7: Commit**

```bash
git add schema/src/main/resources/schema/CaseDefinition.yaml api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git commit -m "feat(#582): open YAML completion block with custom goal kinds

Goal.kind schema changes from enum to string.
CaseCompletion schema changes from fixed success/failure to additionalProperties.
CaseDefinitionYamlMapper iterates completion entries, resolves custom kinds
via explicit status field. Built-in kinds have implicit mappings.
Mutual exclusion: doneWhen cannot coexist with goal kind entries.
Reserved name check: 'doneWhen' rejected as a goal kind name.
Standard kind status override rejected.

Refs #582"
```

---

### Task 5: DefaultCaseDefinitionRegistry validation + remaining module compilation fixes

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` — generalized goal validation
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryGoalWarningTest.java` — update tests
- Modify: All remaining test files that reference `GoalBasedCompletion` constructor or `gbc.getSuccess()`/`gbc.getFailure()`
- Modify: `flow/src/test/java/io/casehub/engine/flow/YamlSimpleCaseHubBeanTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1-4
- Produces: Generalized validation in the registry; full compilation across all modules

- [ ] **Step 1: Update DefaultCaseDefinitionRegistry validation**

In `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` (lines 307-335):

Replace the GoalBasedCompletion-specific validation block:

```java
// Warn if goals are not referenced in any GoalExpression
if (definition.getGoals() != null
    && definition.getCompletion() instanceof GoalBasedCompletion<?> gbc) {
  var referencedGoals = new HashSet<String>();
  for (var entry : gbc.getGoals().entrySet()) {
    GoalExpression expr = entry.getValue();
    if (expr != null && expr.getGoals() != null) {
      expr.getGoals().forEach(g -> referencedGoals.add(g.getName()));
    }
  }
  for (Goal goal : definition.getGoals()) {
    if (!referencedGoals.contains(goal.getName())) {
      LOG.warnf(
          "Goal '%s' is not referenced in any GoalExpression. "
              + "Goals should drive case completion — use Milestone for non-terminal checkpoints.",
          goal.getName());
    }
  }

  // Kind mismatch warning
  for (var entry : gbc.getGoals().entrySet()) {
    String kindValue = entry.getKey().value();
    GoalExpression expr = entry.getValue();
    if (expr != null && expr.getGoals() != null) {
      for (Goal g : expr.getGoals()) {
        if (g.getKind() != null && !g.getKind().equals(kindValue)) {
          LOG.warnf(
              "Goal '%s' has kind '%s' but is referenced in completion entry '%s'"
                  + " — kind mismatch may indicate a configuration error.",
              g.getName(), g.getKind(), kindValue);
        }
      }
    }
  }
}
```

Also update the `PredicateBasedCompletion` validation — `pbc` check stays unchanged, it doesn't reference `GoalKind`.

- [ ] **Step 2: Fix remaining test compilation errors**

Use `ide_find_references` on `GoalBasedCompletion` constructor to find all remaining call sites:
- `AgentWorkerExecutionTest.java` line 187 — change `new GoalBasedCompletion(...)` → builder
- Any test that calls `gbc.getSuccess()` or `gbc.getFailure()` — change to `gbc.getGoals().get(StandardGoalKind.SUCCESS)` etc.

Fix `YamlSimpleCaseHubBeanTest.java` — update assertions for the new GoalBasedCompletion API.

- [ ] **Step 3: Run full build across all modules**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: all modules compile and tests pass

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat(#582): generalized goal validation + full module compilation

Registry validation generalizes: iterates GoalBasedCompletion map entries
instead of checking fixed success/failure. Adds kind mismatch warning.
All test files updated for the new API.

Refs #582"
```

---

### Task 6: Integration test — end-to-end custom goal kind

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/CustomGoalKindIntegrationTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1-5
- Produces: Integration test proving custom goal kinds reach the correct terminal state

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.engine;

import static org.junit.jupiter.api.Assertions.*;

import io.casehub.api.model.*;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.ReactiveCaseInstanceRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

@QuarkusTest
class CustomGoalKindIntegrationTest {

  @Inject CaseHubRuntime runtime;
  @Inject ReactiveCaseInstanceRepository caseInstanceRepository;

  @Test
  @DisplayName("custom goal kind ESCALATED reaches FAULTED terminal state")
  void customGoalKind_reachesCorrectTerminalState() throws Exception {
    GoalKind escalated = GoalKind.of("escalated", CaseStatus.FAULTED);

    Goal escalationGoal = Goal.builder()
        .name("needs-escalation")
        .condition(".escalate == true")
        .kind("escalated")
        .build();

    Goal successGoal = Goal.builder()
        .name("done")
        .condition(".done == true")
        .kind("success")
        .build();

    CaseDefinition def = CaseDefinition.builder()
        .namespace("test")
        .name("escalation-test")
        .version("1.0.0")
        .goals(escalationGoal, successGoal)
        .completion(GoalBasedCompletion.builder()
            .goal(escalated, GoalExpression.allOf(escalationGoal))
            .goal(StandardGoalKind.SUCCESS, GoalExpression.allOf(successGoal))
            .build())
        .build();

    // Register and start case
    UUID caseId = runtime.createCase(def, Map.of());

    // Signal escalation
    runtime.signal(caseId, Map.of("escalate", true)).toCompletableFuture().get(5, TimeUnit.SECONDS);

    // Await terminal state
    Awaitility.await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
      CaseInstance instance = caseInstanceRepository
          .findByUuid(caseId, /* tenancyId */ "test-tenant")
          .await().atMost(java.time.Duration.ofSeconds(2));
      assertNotNull(instance);
      assertEquals(CaseStatus.FAULTED, instance.getState());
    });
  }
}
```

Note: The exact test setup (CaseHub bean, registration, tenant) will need adapting to match the project's existing integration test patterns — see `SimpleCaseHubBean.java` and `SignalGoalCompletionTest.java` for the pattern. The implementer should read those files and mirror the approach.

- [ ] **Step 2: Run the test**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CustomGoalKindIntegrationTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: passes — custom goal kind reaches FAULTED

- [ ] **Step 3: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/CustomGoalKindIntegrationTest.java
git commit -m "test(#582): integration test for custom goal kind reaching terminal state

Proves that GoalKind.of('escalated', FAULTED) evaluates correctly
and transitions the case to FAULTED when the goal condition is met.

Refs #582"
```

---

### Task 7: CLAUDE.md update + spec commit

**Files:**
- Modify: `CLAUDE.md` — update GoalBasedCompletion documentation section
- Modify: `docs/specs/2026-07-08-generalize-goal-completion-design.md` — final commit with any implementation-time refinements

**Interfaces:**
- Consumes: All completed implementation from Tasks 1-6

- [ ] **Step 1: Update CLAUDE.md**

Remove/update any references to the old `GoalBasedCompletion(success, failure)` constructor or `GoalKind` enum in CLAUDE.md. Add documentation about:
- `GoalKind` as an interface with `StandardGoalKind` built-in enum
- `GoalBasedCompletion<K extends GoalKind>` with ordered map
- `Goal.kind` as String (audit metadata, decoupled from completion)
- YAML completion block: open structure, custom kinds require `status:`
- Evaluation priority: insertion order, first match wins

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md docs/specs/2026-07-08-generalize-goal-completion-design.md
git commit -m "docs(#582): update CLAUDE.md for generalized GoalBasedCompletion

GoalKind is an interface with StandardGoalKind enum.
GoalBasedCompletion<K> uses ordered map, first match wins.
Goal.kind is String (audit metadata only).
YAML completion block is open with custom kind support.

Closes #582"
```
