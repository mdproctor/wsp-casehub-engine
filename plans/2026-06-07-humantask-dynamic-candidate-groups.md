# Dynamic candidateGroups/candidateUsers for humanTask Binding — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow `candidateGroups` and `candidateUsers` in a `humanTask` YAML binding to be JQ expressions evaluated against the case context at runtime, not just static lists.

**Architecture:** A new `ListEvaluator` sealed interface (`StaticList` / `JQList` inner records) replaces the `Set<String>` fields in `HumanTaskTarget`. A new `ListExpressionResolver @ApplicationScoped` (in `runtime`) evaluates `JQList` expressions against the case context at event-publish time. Resolved values are carried as new fields in `HumanTaskScheduleEvent`; `HumanTaskScheduleHandler` is unchanged in structure — it uses already-resolved data.

**Tech Stack:** Java 21 (sealed interfaces, pattern matching switch), jackson-jq via `JQEvaluator`, Quarkus CDI, Vert.x event bus, jsonschema2pojo for YAML schema.

**Spec:** `docs/specs/2026-06-07-humantask-dynamic-candidate-groups-design.md`

**Build commands (always run install first per CLAUDE.md):**
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl <module>
```

---

### Task 1: Create `ListEvaluator` sealed interface

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/evaluator/ListEvaluator.java`

This is a pure type definition — no test needed. All other tasks depend on this type existing.

- [ ] **Step 1: Create `ListEvaluator.java`**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.api.model.evaluator;

import java.util.Set;

/**
 * Specification for a field that resolves to a list of strings at runtime.
 *
 * <p>Two implementations:
 *
 * <ul>
 *   <li>{@link StaticList} — static set of strings, evaluated immediately without case context
 *   <li>{@link JQList} — JQ expression evaluated against the case context at event-publish time
 * </ul>
 *
 * <p>Intentionally separate from {@link ExpressionEvaluator}, which is the input type for {@code
 * ExpressionEngine.evaluate(...): boolean}. {@code ListEvaluator} produces {@code Set<String>},
 * not a boolean — placing it in the {@code ExpressionEvaluator} hierarchy would be type pollution.
 */
public sealed interface ListEvaluator permits ListEvaluator.StaticList, ListEvaluator.JQList {

  /** A literal, pre-defined set of strings — no runtime evaluation. */
  record StaticList(Set<String> values) implements ListEvaluator {}

  /** A JQ expression that resolves to a JSON array of strings at runtime. */
  record JQList(String expression) implements ListEvaluator {}
}
```

- [ ] **Step 2: Verify it compiles**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q -pl api
```

Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/model/evaluator/ListEvaluator.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(api): introduce ListEvaluator sealed interface for runtime list resolution (engine#387)

Refs #387"
```

---

### Task 2: Update `HumanTaskTarget` — replace `Set<String>` with `ListEvaluator`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java`
- Modify: `api/src/test/java/io/casehub/api/model/HumanTaskTargetTest.java`

The accessor return type changes from `Set<String>` to `@Nullable ListEvaluator`. The builder gains `candidateGroupsExpression(String)` and `candidateUsersExpression(String)`. The existing `candidateGroups(Set<String>)` stays — it now wraps in `StaticList`.

- [ ] **Step 1: Write failing tests**

In `api/src/test/java/io/casehub/api/model/HumanTaskTargetTest.java`, replace the existing `inlineMode_candidateGroups_andExpiresIn` test and add the following:

```java
// Replace this test (line ~87):
// void inlineMode_candidateGroups_andExpiresIn() { ... }
// With:

@Test
void candidateGroups_staticSet_wrapsInStaticList() {
  HumanTaskTarget target =
      HumanTaskTarget.inline()
          .title("Review")
          .candidateGroups(Set.of("ethics-committee"))
          .expiresIn(Duration.ofHours(72))
          .build();

  assertThat(target.candidateGroups()).isInstanceOf(ListEvaluator.StaticList.class);
  assertThat(((ListEvaluator.StaticList) target.candidateGroups()).values())
      .containsExactlyInAnyOrder("ethics-committee");
  assertThat(target.expiresIn()).isEqualTo(Duration.ofHours(72));
}

@Test
void candidateGroups_jqExpression_wrapsInJQList() {
  HumanTaskTarget target =
      HumanTaskTarget.inline().title("Review").candidateGroupsExpression(".irb.groups").build();

  assertThat(target.candidateGroups()).isInstanceOf(ListEvaluator.JQList.class);
  assertThat(((ListEvaluator.JQList) target.candidateGroups()).expression())
      .isEqualTo(".irb.groups");
}

@Test
void candidateGroups_absent_returnsNull() {
  HumanTaskTarget target = HumanTaskTarget.inline().title("Review").build();

  assertThat(target.candidateGroups()).isNull();
}

@Test
void candidateUsers_staticSet_wrapsInStaticList() {
  HumanTaskTarget target =
      HumanTaskTarget.inline()
          .title("Review")
          .candidateUsers(Set.of("user-a"))
          .build();

  assertThat(target.candidateUsers()).isInstanceOf(ListEvaluator.StaticList.class);
  assertThat(((ListEvaluator.StaticList) target.candidateUsers()).values())
      .containsExactlyInAnyOrder("user-a");
}

@Test
void candidateUsers_jqExpression_wrapsInJQList() {
  HumanTaskTarget target =
      HumanTaskTarget.inline().title("Review").candidateUsersExpression(".approver.id | [.]").build();

  assertThat(target.candidateUsers()).isInstanceOf(ListEvaluator.JQList.class);
  assertThat(((ListEvaluator.JQList) target.candidateUsers()).expression())
      .isEqualTo(".approver.id | [.]");
}
```

Add imports at the top:
```java
import io.casehub.api.model.evaluator.ListEvaluator;
```

- [ ] **Step 2: Run to verify tests fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=HumanTaskTargetTest
```

Expected: compilation errors — `candidateGroupsExpression` method not found, `candidateGroups()` still returns `Set<String>`.

- [ ] **Step 3: Update `HumanTaskTarget.java`**

Make these changes (show only the changed sections — copy the rest verbatim):

**Field declarations** (replace both `Set<String>` candidateGroups/Users):
```java
// Before:
private final Set<String> candidateGroups;
private final Set<String> candidateUsers;

// After:
private final ListEvaluator candidateGroups;
private final ListEvaluator candidateUsers;
```

**Accessor methods** (replace both):
```java
// Before:
public Set<String> candidateGroups() {
  return candidateGroups;
}

public Set<String> candidateUsers() {
  return candidateUsers;
}

// After:
public ListEvaluator candidateGroups() {
  return candidateGroups;
}

public ListEvaluator candidateUsers() {
  return candidateUsers;
}
```

**Builder fields** (replace both `Set<String>`):
```java
// Before:
private Set<String> candidateGroups;
private Set<String> candidateUsers;

// After:
private ListEvaluator candidateGroups;
private ListEvaluator candidateUsers;
```

**Builder methods** (replace candidateGroups and candidateUsers, add expression variants):
```java
// Before:
public Builder candidateGroups(Set<String> candidateGroups) {
  this.candidateGroups = candidateGroups;
  return this;
}

public Builder candidateUsers(Set<String> candidateUsers) {
  this.candidateUsers = candidateUsers;
  return this;
}

// After:
public Builder candidateGroups(Set<String> candidateGroups) {
  this.candidateGroups = new ListEvaluator.StaticList(candidateGroups);
  return this;
}

public Builder candidateGroupsExpression(String jqExpression) {
  this.candidateGroups = new ListEvaluator.JQList(jqExpression);
  return this;
}

public Builder candidateUsers(Set<String> candidateUsers) {
  this.candidateUsers = new ListEvaluator.StaticList(candidateUsers);
  return this;
}

public Builder candidateUsersExpression(String jqExpression) {
  this.candidateUsers = new ListEvaluator.JQList(jqExpression);
  return this;
}
```

**Add import** (at top with other imports):
```java
import io.casehub.api.model.evaluator.ListEvaluator;
```

**Remove** this import (no longer used directly in HumanTaskTarget):
```java
import java.util.Set;   // keep if still used elsewhere in the file, remove if not
```
Check: `Set` is still used in `Builder.candidateGroups(Set<String>)` signature, so keep it.

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=HumanTaskTargetTest
```

Expected: all tests PASS including the new ones.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/model/HumanTaskTarget.java api/src/test/java/io/casehub/api/model/HumanTaskTargetTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(api): replace candidateGroups/Users Set<String> with ListEvaluator in HumanTaskTarget (engine#387)

- candidateGroups() / candidateUsers() now return @Nullable ListEvaluator
- candidateGroups(Set<String>) wraps in StaticList — existing call sites unchanged
- candidateGroupsExpression(String) / candidateUsersExpression(String) create JQList
- Breaking: all callers reading candidateGroups() as Set<String> must update

Refs #387"
```

---

### Task 3: Update YAML schema and mapper

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`

- [ ] **Step 1: Update `CaseDefinition.yaml`**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, find the `candidateGroups` and `candidateUsers` fields under the `HumanTask` definition and replace them:

```yaml
# Before:
      candidateGroups:
        type: array
        items: { type: string }
        description: "Groups eligible to claim this WorkItem"
      candidateUsers:
        type: array
        items: { type: string }
        description: "Users eligible to claim this WorkItem"

# After:
      candidateGroups:
        oneOf:
          - type: array
            items: { type: string }
            description: "Static list of groups eligible to claim this WorkItem"
          - type: string
            description: "JQ expression resolving to a list of group names from case context (e.g. \".irb.candidateGroups\")"
      candidateUsers:
        oneOf:
          - type: array
            items: { type: string }
            description: "Static list of users eligible to claim this WorkItem"
          - type: string
            description: "JQ expression resolving to a list of user IDs from case context"
```

- [ ] **Step 2: Regenerate the schema model and check the generated type**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn generate-sources -pl schema -q
grep -n "candidateGroups" schema/target/generated-sources/jsonschema2pojo/io/casehub/model/HumanTask.java | head -10
```

Expected: the field should now be `private Object candidateGroups`. Note the exact type — if it is NOT `Object` (e.g. some union wrapper), see the spec implementation note for the `JsonNode`-based fallback.

- [ ] **Step 3: Write failing tests in `CaseDefinitionYamlMapperTest`**

Add these test methods (find the existing humanTask mapper tests and add after them):

```java
@Test
void humanTask_candidateGroups_jqExpression_roundTrips() {
  // language=yaml
  String yaml =
      """
      functions: []
      cases:
        - name: dynamicGroupsCase
          bindings:
            - on:
                contextChange: ".trigger"
              humanTask:
                title: "IRB Review"
                candidateGroups: ".irb.candidateGroups"
      """;

  CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
  HumanTaskTarget ht =
      (HumanTaskTarget)
          def.cases().get(0).bindings().get(0).target();

  assertThat(ht.candidateGroups()).isInstanceOf(ListEvaluator.JQList.class);
  assertThat(((ListEvaluator.JQList) ht.candidateGroups()).expression())
      .isEqualTo(".irb.candidateGroups");
}

@Test
void humanTask_candidateUsers_jqExpression_roundTrips() {
  // language=yaml
  String yaml =
      """
      functions: []
      cases:
        - name: dynamicUsersCase
          bindings:
            - on:
                contextChange: ".trigger"
              humanTask:
                title: "Approval"
                candidateUsers: ".approver.id | [.]"
      """;

  CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
  HumanTaskTarget ht =
      (HumanTaskTarget)
          def.cases().get(0).bindings().get(0).target();

  assertThat(ht.candidateUsers()).isInstanceOf(ListEvaluator.JQList.class);
  assertThat(((ListEvaluator.JQList) ht.candidateUsers()).expression())
      .isEqualTo(".approver.id | [.]");
}

@Test
void humanTask_candidateGroups_staticList_stillWorks() {
  // language=yaml
  String yaml =
      """
      functions: []
      cases:
        - name: staticGroupsCase
          bindings:
            - on:
                contextChange: ".trigger"
              humanTask:
                title: "Review"
                candidateGroups:
                  - architects
                  - seniors
      """;

  CaseDefinition def = CaseDefinitionYamlMapper.fromYaml(yaml);
  HumanTaskTarget ht =
      (HumanTaskTarget)
          def.cases().get(0).bindings().get(0).target();

  assertThat(ht.candidateGroups()).isInstanceOf(ListEvaluator.StaticList.class);
  assertThat(((ListEvaluator.StaticList) ht.candidateGroups()).values())
      .containsExactlyInAnyOrder("architects", "seniors");
}
```

Add imports:
```java
import io.casehub.api.model.evaluator.ListEvaluator;
```

- [ ] **Step 4: Run to verify tests fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest
```

Expected: compilation errors or assertion failures — `candidateGroupsExpression` method not yet called in mapper.

- [ ] **Step 5: Update `CaseDefinitionYamlMapper.convertHumanTask()`**

Find the existing `candidateGroups` and `candidateUsers` mapper blocks (around line 353–358) and replace them:

```java
// Before:
if (schema.getCandidateGroups() != null && !schema.getCandidateGroups().isEmpty()) {
  builder.candidateGroups(new LinkedHashSet<>(schema.getCandidateGroups()));
}
if (schema.getCandidateUsers() != null && !schema.getCandidateUsers().isEmpty()) {
  builder.candidateUsers(new LinkedHashSet<>(schema.getCandidateUsers()));
}

// After:
Object rawGroups = schema.getCandidateGroups();
if (rawGroups instanceof List<?> list && !list.isEmpty()) {
  builder.candidateGroups(new LinkedHashSet<>(castStringList("candidateGroups", list)));
} else if (rawGroups instanceof String expr && !expr.isBlank()) {
  builder.candidateGroupsExpression(expr);
}

Object rawUsers = schema.getCandidateUsers();
if (rawUsers instanceof List<?> list && !list.isEmpty()) {
  builder.candidateUsers(new LinkedHashSet<>(castStringList("candidateUsers", list)));
} else if (rawUsers instanceof String expr && !expr.isBlank()) {
  builder.candidateUsersExpression(expr);
}
```

Add the helper method (private, near the bottom of the class or near other helpers):

```java
@SuppressWarnings("unchecked")
private static List<String> castStringList(String fieldName, List<?> raw) {
  for (Object element : raw) {
    if (!(element instanceof String)) {
      throw new IllegalArgumentException(
          fieldName
              + " list must contain only strings, found: "
              + element.getClass().getSimpleName());
    }
  }
  return (List<String>) raw;
}
```

**If** `schema.getCandidateGroups()` did NOT return `Object` (e.g. still `List<String>`): use the `JsonNode`-based approach instead — read the raw YAML node before schema conversion. Consult the spec implementation note. In practice, jsonschema2pojo 1.x generates `Object` for `oneOf` with mixed types.

- [ ] **Step 6: Also update the existing assertions in `CaseDefinitionYamlMapperTest`**

Find any existing test that asserts on `ht.candidateGroups()` as a `Set<String>` (search for `candidateGroups()` in the test file) and update the assertions to use `ListEvaluator.StaticList`:

```bash
grep -n "candidateGroups()" /Users/mdproctor/claude/casehub/engine/api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
```

For each hit like `assertThat(ht.candidateGroups()).containsExactlyInAnyOrder(...)`, replace with:
```java
assertThat(ht.candidateGroups()).isInstanceOf(ListEvaluator.StaticList.class);
assertThat(((ListEvaluator.StaticList) ht.candidateGroups()).values())
    .containsExactlyInAnyOrder("architects", "seniors");  // use actual expected values from that test
```

For hits like `assertThat(ht.candidateGroups()).isNull()` — these stay correct; `candidateGroups()` returns `null` when absent.

- [ ] **Step 7: Run all api tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api
```

Expected: all tests PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add schema/src/main/resources/schema/CaseDefinition.yaml api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(api): support JQ expression for candidateGroups/Users in YAML humanTask binding (engine#387)

- CaseDefinition.yaml: candidateGroups/Users now accept string (JQ) or array (static)
- CaseDefinitionYamlMapper: branches on Object type; calls candidateGroupsExpression() for string

Refs #387"
```

---

### Task 4: Extend `HumanTaskScheduleEvent` and fix all call sites

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` (many lines)
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java`
- Modify: `runtime/src/test/java/io/casehub/engine/HumanTaskTargetDispatchTest.java` (0 direct constructions — only the recorder reads events)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` (construction site — pass null for now, real values come in Task 6)

The record gains two new fields. All construction sites break — fix by passing `null` for both new fields. The real values are wired in Task 6.

- [ ] **Step 1: Add fields to `HumanTaskScheduleEvent`**

```java
// Before:
public record HumanTaskScheduleEvent(
    UUID caseId,
    String bindingName,
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Instant caseBudgetDeadline,
    String tenancyId) {}

// After:
public record HumanTaskScheduleEvent(
    UUID caseId,
    String bindingName,
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedCandidateGroups,
    Set<String> resolvedCandidateUsers,
    Instant caseBudgetDeadline,
    String tenancyId) {}
```

Add import:
```java
import java.util.Set;
```

- [ ] **Step 2: Find all construction sites**

```bash
grep -rn "new HumanTaskScheduleEvent(" /Users/mdproctor/claude/casehub/engine --include="*.java"
```

Note each file and line. Expected files:
- `runtime/src/main/java/.../CaseContextChangedEventHandler.java` — 1 site
- `work-adapter/src/test/java/.../HumanTaskScheduleHandlerTest.java` — many sites
- `work-adapter/src/test/java/.../HumanTaskScheduleHandlerAtomicityTest.java` — some sites
- Any docs/specs files are not Java and do not need fixing

- [ ] **Step 3: Fix `CaseContextChangedEventHandler.java`**

Find the `new HumanTaskScheduleEvent(` call (around line 362) and add two `null` arguments after `inputData`:

```java
// Before:
new HumanTaskScheduleEvent(
    caseInstance.getUuid(),
    binding.getName(),
    target,
    inputData,
    caseBudgetDeadline,
    caseInstance.tenancyId)

// After:
new HumanTaskScheduleEvent(
    caseInstance.getUuid(),
    binding.getName(),
    target,
    inputData,
    null,   // resolvedCandidateGroups — wired in Task 6
    null,   // resolvedCandidateUsers  — wired in Task 6
    caseBudgetDeadline,
    caseInstance.tenancyId)
```

- [ ] **Step 4: Fix all construction sites in `HumanTaskScheduleHandlerTest.java`**

Every call of the form:
```java
new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of(), null, TENANCY_ID)
```
becomes:
```java
new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of(), null, null, null, TENANCY_ID)
```

And every call of the form:
```java
new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of("key", "val"), null, TENANCY_ID)
```
becomes:
```java
new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of("key", "val"), null, null, null, TENANCY_ID)
```

The pattern is: insert `null, null,` after the `inputData` argument (4th position) in every call. Run `grep -n "new HumanTaskScheduleEvent" HumanTaskScheduleHandlerTest.java` to find every line, then update them all.

- [ ] **Step 5: Fix all construction sites in `HumanTaskScheduleHandlerAtomicityTest.java`**

Same pattern as Step 4 — insert `null, null,` after `inputData` in every call.

- [ ] **Step 6: Build all affected modules**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q
```

Expected: BUILD SUCCESS — no compilation errors.

- [ ] **Step 7: Run existing tests to confirm no regressions**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl work-adapter
```

Expected: all existing tests PASS (handler tests use `null` resolvedGroups — `toCsv(null)` returns null, same behavior as before).

Note: `runtime` tests may fail if integration tests check event fields — that's expected and will be fixed in Task 6.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  common/src/main/java/io/casehub/engine/common/internal/event/HumanTaskScheduleEvent.java \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(common): add resolvedCandidateGroups/Users fields to HumanTaskScheduleEvent (engine#387)

Null-wired in CaseContextChangedEventHandler — real values in next commit.

Refs #387"
```

---

### Task 5: Create `ListExpressionResolver`

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/ListExpressionResolver.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/ListExpressionResolverTest.java`

`ListExpressionResolver` is `@ApplicationScoped` and injects `JQEvaluator`. It has a package-private `resolveJq(JsonNode, JQList, String)` method so tests in the same package can call it directly without needing a `CaseInstance`.

- [ ] **Step 1: Write the test first**

Create `runtime/src/test/java/io/casehub/engine/internal/engine/ListExpressionResolverTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.api.model.evaluator.ListEvaluator.*;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.evaluator.ListEvaluator;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import java.util.Set;
import org.junit.jupiter.api.Test;

/**
 * Unit tests for ListExpressionResolver. Uses @QuarkusTest so JQEvaluator is discovered via CDI
 * (it is in casehub-engine-common, which is indexed in test application.properties). Tests call
 * resolveJq() directly (package-private) to avoid constructing CaseInstance.
 */
@QuarkusTest
class ListExpressionResolverTest {

  @Inject ListExpressionResolver resolver;

  private final ObjectMapper mapper = new ObjectMapper();

  // ── resolve() — top-level dispatch ──────────────────────────────────────────

  @Test
  void resolve_nullSpec_returnsNull() {
    assertThat(resolver.resolve(null, null, "candidateGroups")).isNull();
  }

  @Test
  void resolve_staticList_returnsValuesWithoutInvokingJQ() throws Exception {
    StaticList spec = new StaticList(Set.of("irb-committee", "ethics-board"));
    Set<String> result = resolver.resolve(null, spec, "candidateGroups");
    assertThat(result).containsExactlyInAnyOrder("irb-committee", "ethics-board");
  }

  // ── resolveJq() — JQ evaluation path ────────────────────────────────────────

  @Test
  void resolveJq_validStringArray_returnsStringSet() throws Exception {
    JsonNode context = mapper.readTree("{\"groups\": [\"ethics\", \"irb\"]}");
    Set<String> result = resolver.resolveJq(context, new JQList(".groups"), "candidateGroups");
    assertThat(result).containsExactlyInAnyOrder("ethics", "irb");
    assertThat(ListExpressionResolver.isFailed(result)).isFalse();
  }

  @Test
  void resolveJq_nonArray_returnsResolutionFailed() throws Exception {
    JsonNode context = mapper.readTree("{\"groups\": \"not-an-array\"}");
    Set<String> result = resolver.resolveJq(context, new JQList(".groups"), "candidateGroups");
    assertThat(ListExpressionResolver.isFailed(result)).isTrue();
  }

  @Test
  void resolveJq_emptyArray_returnsResolutionFailed() throws Exception {
    JsonNode context = mapper.readTree("{\"groups\": []}");
    Set<String> result = resolver.resolveJq(context, new JQList(".groups"), "candidateGroups");
    assertThat(ListExpressionResolver.isFailed(result)).isTrue();
  }

  @Test
  void resolveJq_absentField_returnsResolutionFailed() throws Exception {
    JsonNode context = mapper.readTree("{}");
    // .groups on {} produces null in JQ, which is not an array
    Set<String> result = resolver.resolveJq(context, new JQList(".groups"), "candidateGroups");
    assertThat(ListExpressionResolver.isFailed(result)).isTrue();
  }

  @Test
  void resolveJq_invalidJqExpression_returnsResolutionFailed() throws Exception {
    JsonNode context = mapper.readTree("{}");
    Set<String> result =
        resolver.resolveJq(context, new JQList("this is not valid jq !!!"), "candidateGroups");
    assertThat(ListExpressionResolver.isFailed(result)).isTrue();
  }

  @Test
  void resolveJq_arrayWithNonStringElement_returnsResolutionFailed() throws Exception {
    JsonNode context = mapper.readTree("{\"groups\": [\"valid\", 42]}");
    Set<String> result = resolver.resolveJq(context, new JQList(".groups"), "candidateGroups");
    assertThat(ListExpressionResolver.isFailed(result)).isTrue();
  }
}
```

- [ ] **Step 2: Run to verify the test fails (class missing)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=ListExpressionResolverTest
```

Expected: compilation failure — `ListExpressionResolver` not found.

- [ ] **Step 3: Create `ListExpressionResolver.java`**

Create `runtime/src/main/java/io/casehub/engine/internal/engine/ListExpressionResolver.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.engine;

import static io.casehub.api.model.evaluator.ListEvaluator.*;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.evaluator.ListEvaluator;
import io.casehub.engine.common.internal.jq.JQEvaluator;
import io.casehub.engine.common.internal.jq.ValidationResult;
import io.casehub.engine.common.internal.model.CaseInstance;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Collections;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Set;
import org.jboss.logging.Logger;

/**
 * Resolves {@link ListEvaluator} specs to concrete {@code Set<String>} values.
 *
 * <p>Extracted from {@code CaseContextChangedEventHandler} so the six evaluation branches can be
 * unit-tested in isolation via {@link #resolveJq(JsonNode, JQList, String)}.
 */
@ApplicationScoped
public class ListExpressionResolver {

  private static final Logger LOG = Logger.getLogger(ListExpressionResolver.class);

  /**
   * Sentinel returned when JQ resolution fails. Checked with {@link #isFailed(Set)} (== identity,
   * not .equals). The {@code unmodifiableSet} wrapper prevents accidental mutation; only this
   * reference can compare equal via ==.
   */
  static final Set<String> RESOLUTION_FAILED = Collections.unmodifiableSet(new HashSet<>());

  /** Returns {@code true} iff {@code result} is the {@link #RESOLUTION_FAILED} sentinel. */
  public static boolean isFailed(Set<String> result) {
    return result == RESOLUTION_FAILED;
  }

  @Inject JQEvaluator jqEvaluator;

  /**
   * Resolves a {@link ListEvaluator} to a concrete set of strings.
   *
   * @param instance the current case instance — only accessed when {@code spec} is a {@link JQList}
   * @param spec the evaluator spec; null means "no restriction" → returns null
   * @param fieldName for log messages only
   * @return resolved set, null (no restriction), or {@link #RESOLUTION_FAILED}
   */
  public Set<String> resolve(CaseInstance instance, ListEvaluator spec, String fieldName) {
    if (spec == null) return null;
    return switch (spec) {
      case StaticList s -> s.values();
      case JQList jq -> resolveJq(instance.getCaseContext().asJsonNode(), jq, fieldName);
    };
  }

  /**
   * Package-private to allow direct unit testing without constructing a {@link CaseInstance}.
   */
  Set<String> resolveJq(JsonNode context, JQList jq, String fieldName) {
    try {
      ValidationResult vr = jqEvaluator.eval(jq.expression(), context);
      if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) {
        LOG.errorf(
            "'%s' JQ evaluation failed: expression '%s' — %s", fieldName, jq.expression(), vr.error());
        return RESOLUTION_FAILED;
      }
      JsonNode result = vr.output().get(0);
      if (!result.isArray()) {
        LOG.errorf(
            "'%s' JQ expression returned non-array: '%s' produced node type %s",
            fieldName, jq.expression(), result.getNodeType());
        return RESOLUTION_FAILED;
      }
      if (result.size() == 0) {
        LOG.warnf(
            "'%s' JQ expression returned empty array: '%s' — PlanItem stays PENDING",
            fieldName, jq.expression());
        return RESOLUTION_FAILED;
      }
      Set<String> groups = new LinkedHashSet<>();
      for (JsonNode element : result) {
        if (!element.isTextual()) {
          LOG.errorf(
              "'%s' JQ expression returned non-string element in array: node type %s",
              fieldName, element.getNodeType());
          return RESOLUTION_FAILED;
        }
        groups.add(element.asText());
      }
      return Collections.unmodifiableSet(groups);
    } catch (Exception e) {
      LOG.errorf(e, "'%s' JQ evaluation threw: expression '%s'", fieldName, jq.expression());
      return RESOLUTION_FAILED;
    }
  }
}
```

- [ ] **Step 4: Run tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=ListExpressionResolverTest
```

Expected: all 8 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/engine/ListExpressionResolver.java \
  runtime/src/test/java/io/casehub/engine/internal/engine/ListExpressionResolverTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(runtime): add ListExpressionResolver — evaluates ListEvaluator specs against case context (engine#387)

Refs #387"
```

---

### Task 6: Wire `ListExpressionResolver` into `CaseContextChangedEventHandler`

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/test/java/io/casehub/engine/HumanTaskTargetDispatchTest.java`

This is where the null placeholders from Task 4 become real values. Integration tests drive the change.

- [ ] **Step 1: Write new failing integration tests in `HumanTaskTargetDispatchTest.java`**

Add three new inner `CaseHub` subclasses and three test methods. Add after the existing test:

```java
// ── Dynamic candidateGroups ─────────────────────────────────────────────────

@Test
void humanTaskBinding_dynamicCandidateGroups_resolvesFromContext() {
  CompletionStage<UUID> future =
      dynamicGroupsCaseBean.startCase(
          Map.of("stage", "review", "irb", Map.of("candidateGroups", new String[]{"irb-committee"})));
  future.toCompletableFuture().join();

  await()
      .atMost(5, TimeUnit.SECONDS)
      .untilAsserted(() -> assertThat(HumanTaskEventRecorder.events).isNotEmpty());

  HumanTaskScheduleEvent event = HumanTaskEventRecorder.events.get(0);
  assertThat(event.resolvedCandidateGroups()).containsExactlyInAnyOrder("irb-committee");
}

@Test
void humanTaskBinding_dynamicCandidateGroups_jqReturnsNonArray_planItemStaysPending() {
  CompletionStage<UUID> future =
      badGroupsCaseBean.startCase(Map.of("stage", "review", "routing", "not-an-array"));
  future.toCompletableFuture().join();

  // Give the engine time to process the context-changed event
  try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
  assertThat(HumanTaskEventRecorder.events).isEmpty();
}

@Test
void humanTaskBinding_dynamicCandidateGroups_conjunctionFail_bothChecked() {
  // groups is a string (wrong type), users is a valid array — both are checked;
  // groups failing alone is enough to block the event
  CompletionStage<UUID> future =
      conjunctionFailCaseBean.startCase(
          Map.of("stage", "review",
                 "groups", "wrong",
                 "users", new String[]{"user-1"}));
  future.toCompletableFuture().join();

  try { Thread.sleep(500); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
  assertThat(HumanTaskEventRecorder.events).isEmpty();
}

/** Case with candidateGroupsExpression binding. */
@ApplicationScoped
static class DynamicGroupsCaseBean extends CaseHub {
  @Override
  public CaseDefinition getDefinition() {
    HumanTaskTarget target =
        HumanTaskTarget.inline()
            .title("IRB Review")
            .candidateGroupsExpression(".irb.candidateGroups")
            .build();

    return CaseDefinition.builder()
        .namespace("test")
        .name("DynamicGroupsCase")
        .version("1.0.0")
        .bindings(
            Binding.builder()
                .name("review-binding")
                .humanTask(target)
                .on(new ContextChangeTrigger(".stage == \"review\""))
                .build())
        .build();
  }
}

/** Case with candidateGroupsExpression that resolves to a non-array. */
@ApplicationScoped
static class BadGroupsCaseBean extends CaseHub {
  @Override
  public CaseDefinition getDefinition() {
    HumanTaskTarget target =
        HumanTaskTarget.inline()
            .title("Bad Groups")
            .candidateGroupsExpression(".routing")
            .build();

    return CaseDefinition.builder()
        .namespace("test")
        .name("BadGroupsCase")
        .version("1.0.0")
        .bindings(
            Binding.builder()
                .name("bad-binding")
                .humanTask(target)
                .on(new ContextChangeTrigger(".stage == \"review\""))
                .build())
        .build();
  }
}

/** Case where groups expression fails and users expression succeeds — conjunction failure. */
@ApplicationScoped
static class ConjunctionFailCaseBean extends CaseHub {
  @Override
  public CaseDefinition getDefinition() {
    HumanTaskTarget target =
        HumanTaskTarget.inline()
            .title("Conjunction")
            .candidateGroupsExpression(".groups")
            .candidateUsersExpression(".users")
            .build();

    return CaseDefinition.builder()
        .namespace("test")
        .name("ConjunctionFailCase")
        .version("1.0.0")
        .bindings(
            Binding.builder()
                .name("conjunction-binding")
                .humanTask(target)
                .on(new ContextChangeTrigger(".stage == \"review\""))
                .build())
        .build();
  }
}
```

Add `@Inject` fields for the new beans at the top of the test class (next to existing `@Inject HumanTaskCaseBean`):

```java
@Inject DynamicGroupsCaseBean dynamicGroupsCaseBean;
@Inject BadGroupsCaseBean badGroupsCaseBean;
@Inject ConjunctionFailCaseBean conjunctionFailCaseBean;
```

- [ ] **Step 2: Run to verify new tests fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=HumanTaskTargetDispatchTest
```

Expected: the three new tests FAIL — `resolvedCandidateGroups` is null (still passing null in handler).

- [ ] **Step 3: Inject `ListExpressionResolver` into `CaseContextChangedEventHandler`**

Add after the existing `@Inject JQEvaluator jqEvaluator;` line (around line 85):

```java
@Inject ListExpressionResolver listExpressionResolver;
```

Add import:
```java
import io.casehub.engine.internal.engine.ListExpressionResolver;
```

- [ ] **Step 4: Update `publishHumanTaskSchedule()` to resolve and guard**

Find `publishHumanTaskSchedule()` (around line 348). Replace the method body:

```java
private Uni<Void> publishHumanTaskSchedule(
    final CaseInstance caseInstance, final Binding binding, final HumanTaskTarget target) {
  final Map<String, Object> inputData = evaluateInputMapping(caseInstance, target);
  final Set<String> resolvedGroups =
      listExpressionResolver.resolve(caseInstance, target.candidateGroups(), "candidateGroups");
  final Set<String> resolvedUsers =
      listExpressionResolver.resolve(caseInstance, target.candidateUsers(), "candidateUsers");

  if (ListExpressionResolver.isFailed(resolvedGroups)
      || ListExpressionResolver.isFailed(resolvedUsers)) {
    LOG.warnf(
        "HumanTask list expression resolution failed for caseId=%s binding=%s — PlanItem stays PENDING",
        caseInstance.getUuid(), binding.getName());
    return Uni.createFrom().voidItem();
  }

  final java.time.Instant caseBudgetDeadline =
      java.util.Optional.ofNullable(caseInstance.getPropagationContext())
          .flatMap(PropagationContext::getDeadline)
          .orElse(null);

  LOG.infof(
      "Publishing HumanTaskScheduleEvent: caseId=%s binding=%s template=%s deadline=%s",
      caseInstance.getUuid(), binding.getName(), target.templateRef(), caseBudgetDeadline);

  eventBus.publish(
      EventBusAddresses.HUMAN_TASK_SCHEDULE,
      new HumanTaskScheduleEvent(
          caseInstance.getUuid(),
          binding.getName(),
          target,
          inputData,
          resolvedGroups,
          resolvedUsers,
          caseBudgetDeadline,
          caseInstance.tenancyId));

  return Uni.createFrom().voidItem();
}
```

Add imports:
```java
import java.util.Set;
```

Also remove the `import io.casehub.api.model.evaluator.JQExpressionEvaluator;` line from the import block IF it was only used in `evaluateInputMapping()` — check whether it's still used elsewhere. If yes, keep it.

- [ ] **Step 5: Run integration tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=HumanTaskTargetDispatchTest
```

Expected: all tests PASS including the three new ones.

- [ ] **Step 6: Run the full runtime test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java \
  runtime/src/test/java/io/casehub/engine/HumanTaskTargetDispatchTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(runtime): resolve candidateGroups/Users in CaseContextChangedEventHandler (engine#387)

- Inject ListExpressionResolver; resolve both specs before publishing
- Either resolution failure blocks event — PlanItem stays PENDING
- Integration tests confirm dynamic groups resolve correctly

Refs #387"
```

---

### Task 7: Update `HumanTaskScheduleHandler` — use resolved fields

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`

The handler stops reading `target.candidateGroups()`. It uses `event.resolvedCandidateGroups()`. Template mode gains routing override logic.

- [ ] **Step 1: Write new failing tests**

In `HumanTaskScheduleHandlerTest.java`, add the following tests after the existing ones. Note: the existing test `inlineMode_createsWorkItem_withCallerRef_andMarksPlanItemRunning` still passes (it passes `null, null` for resolved groups/users — the handler creates the WorkItem with null candidateGroups, which is fine). Add new explicit tests for the dynamic path:

```java
// ── Dynamic candidateGroups — inline mode ─────────────────────────────────────

@Test
void inlineMode_resolvedCandidateGroups_passedToWorkItem() {
  HumanTaskTarget target = HumanTaskTarget.inline().title("IRB Review").build();

  handler.onHumanTaskSchedule(
      new HumanTaskScheduleEvent(
          caseId, "irb-binding", target, Map.of(),
          Set.of("irb-committee"), null, null, TENANCY_ID));

  WorkItem created =
      workItemStore.scanAll().stream()
          .filter(w -> w.callerRef != null && w.callerRef.startsWith("case:"))
          .findFirst()
          .orElse(null);
  assertThat(created).isNotNull();
  assertThat(created.candidateGroups).isEqualTo("irb-committee");
}

@Test
void inlineMode_nullResolvedCandidateGroups_workItemHasNullGroups() {
  HumanTaskTarget target = HumanTaskTarget.inline().title("Open Review").build();

  handler.onHumanTaskSchedule(
      new HumanTaskScheduleEvent(
          caseId, "irb-binding", target, Map.of(),
          null, null, null, TENANCY_ID));

  WorkItem created =
      workItemStore.scanAll().stream()
          .filter(w -> w.callerRef != null && w.callerRef.startsWith("case:"))
          .findFirst()
          .orElse(null);
  assertThat(created).isNotNull();
  assertThat(created.candidateGroups).isNull();
}

// ── Dynamic candidateGroups — template mode ───────────────────────────────────

@Test
void templateMode_resolvedCandidateGroups_overridesTemplateDefaults() {
  WorkItemTemplate tmpl = persistTemplate("IRB Review Template");

  handler.onHumanTaskSchedule(
      new HumanTaskScheduleEvent(
          caseId,
          "irb-binding",
          HumanTaskTarget.template(tmpl.id.toString()).build(),
          Map.of(),
          Set.of("committee-a", "committee-b"),
          null,
          null,
          TENANCY_ID));

  WorkItem created =
      workItemStore.scanAll().stream()
          .filter(w -> w.callerRef != null && w.callerRef.startsWith("case:"))
          .findFirst()
          .orElse(null);
  assertThat(created).isNotNull();
  // Resolved groups override whatever the template has
  assertThat(created.candidateGroups).contains("committee-a");
  assertThat(created.candidateGroups).contains("committee-b");
}

@Test
void templateMode_nullResolvedCandidateGroups_keepsTemplateDefaults() {
  WorkItemTemplate tmpl = persistTemplate("IRB Review Template");
  // WorkItemTemplate has no candidateGroups set by persistTemplate() — it will be null from the template too.
  // The key assertion is that we do NOT override with non-null when resolvedGroups is null.
  handler.onHumanTaskSchedule(
      new HumanTaskScheduleEvent(
          caseId,
          "irb-binding",
          HumanTaskTarget.template(tmpl.id.toString()).build(),
          Map.of(),
          null,  // resolved groups null = absent spec = do not override
          null,
          null,
          TENANCY_ID));

  WorkItem created =
      workItemStore.scanAll().stream()
          .filter(w -> w.callerRef != null && w.callerRef.startsWith("case:"))
          .findFirst()
          .orElse(null);
  assertThat(created).isNotNull();
  // Template default is null candidateGroups — null because the spec was absent, not overridden
  assertThat(created.candidateGroups).isNull();
}
```

- [ ] **Step 2: Run to verify new tests fail**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl work-adapter -Dtest=HumanTaskScheduleHandlerTest
```

Expected: the four new tests FAIL — handler still uses `target.candidateGroups()` (which no longer exists as `Set<String>`).

- [ ] **Step 3: Update `HumanTaskScheduleHandler.java`**

**Replace `candidateGroupsCsv` and `candidateUsersCsv` helpers** with a single `toCsv(Set<String>)`:

```java
// Remove these two methods:
// private static String candidateGroupsCsv(HumanTaskTarget target) { ... }
// private static String candidateUsersCsv(HumanTaskTarget target) { ... }

// Add:
private static String toCsv(Set<String> values) {
  if (values == null || values.isEmpty()) return null;
  return String.join(",", values);
}
```

**Update `handleInlineMode`** to pass resolved sets:

```java
private void handleInlineMode(PlanItem item, HumanTaskScheduleEvent event) {
  String callerRef = PlanItemCallerRef.encode(event.caseId(), item.getPlanItemId());
  createInline(
      event.target(),
      event.inputData(),
      event.resolvedCandidateGroups(),
      event.resolvedCandidateUsers(),
      callerRef,
      event.caseBudgetDeadline());
  planItemStore.save(
      new PlanItemSaveRequest(
          event.caseId(),
          item.getPlanItemId(),
          item.getBindingName(),
          PlanItemStatus.DELEGATED,
          item.getCreatedAt(),
          TargetType.HUMAN_TASK,
          extractOutputMappingExpression(event.target()),
          event.tenancyId()),
      event.tenancyId());
  item.markDelegated();
}
```

**Update `createInline` signature and body**:

```java
private void createInline(
    HumanTaskTarget target,
    Map<String, Object> inputData,
    Set<String> resolvedGroups,
    Set<String> resolvedUsers,
    String callerRef,
    Instant caseBudgetDeadline) {
  String payload = serializePayload(inputData);
  Instant taskDeadline =
      target.expiresIn() != null ? Instant.now().plus(target.expiresIn()) : null;
  Instant effectiveDeadline = earliestOf(taskDeadline, caseBudgetDeadline);

  WorkItemCreateRequest request =
      WorkItemCreateRequest.builder()
          .title(target.title())
          .candidateGroups(toCsv(resolvedGroups))
          .candidateUsers(toCsv(resolvedUsers))
          .createdBy("casehub-engine")
          .payload(payload)
          .expiresAt(effectiveDeadline)
          .claimDeadlineBusinessHours(target.claimDeadlineHours())
          .callerRef(callerRef)
          .scope(target.scope())
          .build();

  workItemService.create(request);
  LOG.infof(
      "WorkItem created (inline) for binding callerRef=%s title='%s' expiresAt=%s",
      callerRef, target.title(), effectiveDeadline);
}
```

**Update `handleTemplateMode`** to add routing override after `workItem.scope = target.scope()`:

```java
workItem.scope = target.scope();
if (event.resolvedCandidateGroups() != null) {
  workItem.candidateGroups = toCsv(event.resolvedCandidateGroups());
}
if (event.resolvedCandidateUsers() != null) {
  workItem.candidateUsers = toCsv(event.resolvedCandidateUsers());
}
if (event.inputData() != null && !event.inputData().isEmpty()) {
  workItem.payload = serializePayload(event.inputData());
}
workItem.persist();
```

**Add import** for `Set<String>`:
```java
import java.util.Set;
```

- [ ] **Step 4: Run the full work-adapter test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl work-adapter
```

Expected: all tests PASS including the four new ones.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(work-adapter): use resolvedCandidateGroups/Users from event in HumanTaskScheduleHandler (engine#387)

- toCsv(Set<String>) replaces candidateGroupsCsv/candidateUsersCsv(HumanTaskTarget)
- Template mode overrides routing when resolvedCandidateGroups/Users are non-null
- Null = spec absent = template defaults preserved

Refs #387"
```

---

### Task 8: Full build verification

- [ ] **Step 1: Install everything from scratch**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn install -DskipTests -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 2: Run all affected module test suites**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl api,runtime,work-adapter
```

Expected: all tests PASS across all three modules.

- [ ] **Step 3: Check for any remaining callers of the old `candidateGroups()` → `Set<String>` pattern**

```bash
grep -rn "candidateGroups()\." /Users/mdproctor/claude/casehub/engine --include="*.java" | grep -v "test\|spec\|plan"
```

Expected: no hits (all callers should now handle `ListEvaluator`).

```bash
grep -rn "\.candidateGroups()" /Users/mdproctor/claude/casehub/engine --include="*.java" | grep -v "candidateGroupsExpression\|ListEvaluator\|isInstanceOf\|instanceof"
```

Review any hits — confirm they are all either `ListEvaluator`-aware or tests that were intentionally updated.

- [ ] **Step 4: Build casehub-blackboard if present**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -pl casehub-blackboard 2>/dev/null || echo "blackboard not in this build — skip"
```

- [ ] **Step 5: Note for garden protocol update**

After this task is complete, update the garden protocol `PP-20260520-b2a932` (`docs/protocols/casehub/yaml-humantask-binding-type.md` in `casehub/garden`) to document dynamic `candidateGroups`/`candidateUsers`. This is a separate session on the garden repo.
