# HITL YAML Binding + devtown Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `humanTask` as a first-class YAML binding target in the engine schema and mapper, then wire `casehub-engine-work-adapter` in devtown so HITL cases resume when a human completes a WorkItem.

**Architecture:** Three sequential changes — (1) schema module gains a `HumanTask` JSON Schema `$def` and `humanTask` as a third `Binding.oneOf` branch; (2) `CaseDefinitionYamlMapper` gains a `convertHumanTask` method that produces a `HumanTaskTarget`; (3) devtown's `app/pom.xml` adds `casehub-engine-work-adapter` (which transitively brings `casehub-engine-blackboard`) and `pr-review.yaml` switches the `human-approval` binding from `capability` to `humanTask`.

**Tech Stack:** Java 21+, Quarkus 3.32.2, jsonschema2pojo (schema codegen), AssertJ, JUnit 5, Maven multi-module.

**Repos:** engine (`/Users/mdproctor/claude/casehub/engine`), devtown (`/Users/mdproctor/claude/casehub/devtown`). Both on branch `issue-293-wire-work-adapter-hitl`.

---

## File Map

| File | Repo | Action |
|------|------|--------|
| `schema/src/main/resources/schema/CaseDefinition.yaml` | engine | Modify — add `HumanTask` def + `humanTask` to `Binding` |
| `schema/target/generated-sources/.../model/Binding.java` | engine | Auto-generated — do NOT edit |
| `schema/target/generated-sources/.../model/HumanTask.java` | engine | Auto-generated — do NOT edit |
| `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` | engine | Modify — add `humanTask` branch + `convertHumanTask` |
| `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` | engine | Modify — add humanTask parse tests |
| `review/src/main/resources/devtown/pr-review.yaml` | devtown | Modify — replace capability binding with humanTask |
| `app/pom.xml` | devtown | Modify — add work-adapter dep |
| `app/src/test/resources/application.properties` | devtown | Modify — add quarkus.index-dependency entries |

---

## Task 1: Write failing mapper test for humanTask binding (TDD red)

**Files:**
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`

- [ ] **Step 1: Add failing test — inline humanTask**

Open `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`.

Add these two imports at the top (with existing imports):
```java
import io.casehub.api.model.HumanTaskTarget;
import java.time.Duration;
import java.util.Set;
```

Add these tests to the class body:

```java
@Test
void humanTaskBinding_inline_parsedCorrectly() throws IOException {
    String yaml =
        """
        namespace: test
        name: Human Task Case
        version: 1.0.0
        spec:
          bindings:
            - name: approval
              on: { contextChange: {} }
              when: ".needsApproval == true and .approval == null"
              humanTask:
                title: "PR approval required"
                outputMapping: "{ approval: { status: .decision } }"
        """;

    CaseDefinition def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    assertThat(def.getBindings()).hasSize(1);
    Binding binding = def.getBindings().get(0);
    assertThat(binding.getName()).isEqualTo("approval");
    assertThat(binding.target()).isInstanceOf(HumanTaskTarget.class);

    HumanTaskTarget ht = (HumanTaskTarget) binding.target();
    assertThat(ht.isTemplateMode()).isFalse();
    assertThat(ht.title()).isEqualTo("PR approval required");
    assertThat(ht.outputMapping()).isInstanceOf(JQExpressionEvaluator.class);
    assertThat(ht.inputMapping()).isNull();
    assertThat(ht.candidateGroups()).isNull();
    assertThat(ht.expiresIn()).isNull();
}

@Test
void humanTaskBinding_template_parsedCorrectly() throws IOException {
    String yaml =
        """
        namespace: test
        name: Template Task Case
        version: 1.0.0
        spec:
          bindings:
            - name: review
              on: { contextChange: {} }
              humanTask:
                templateRef: "senior-review"
                outputMapping: "{ review: .outcome }"
        """;

    CaseDefinition def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    HumanTaskTarget ht = (HumanTaskTarget) def.getBindings().get(0).target();
    assertThat(ht.isTemplateMode()).isTrue();
    assertThat(ht.templateRef()).isEqualTo("senior-review");
}

@Test
void humanTaskBinding_withAllOptionalFields_parsedCorrectly() throws IOException {
    String yaml =
        """
        namespace: test
        name: Full Human Task Case
        version: 1.0.0
        spec:
          bindings:
            - name: full-approval
              on: { contextChange: {} }
              humanTask:
                title: "Full approval task"
                inputMapping: "{ pr: .pr }"
                outputMapping: "{ approval: .decision }"
                candidateGroups:
                  - architects
                  - seniors
                candidateUsers:
                  - alice
                expiresIn: "PT24H"
        """;

    CaseDefinition def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    HumanTaskTarget ht = (HumanTaskTarget) def.getBindings().get(0).target();
    assertThat(ht.candidateGroups()).containsExactlyInAnyOrder("architects", "seniors");
    assertThat(ht.candidateUsers()).containsExactly("alice");
    assertThat(ht.expiresIn()).isEqualTo(Duration.parse("PT24H"));
    assertThat(ht.inputMapping()).isInstanceOf(JQExpressionEvaluator.class);
}
```

- [ ] **Step 2: Run the tests — expect FAIL**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest
```

Expected: FAIL — all three new tests fail. The `humanTaskBinding_inline_parsedCorrectly` test fails with a `JsonMappingException` or `IllegalArgumentException("Binding 'approval' must have either capability or subCase")` because the schema doesn't support `humanTask` yet. If Jackson silently ignores unknown fields, the mapper's else branch throws the `IllegalArgumentException`. Either way — red is the goal.

---

## Task 2: Add humanTask to the YAML schema (engine#294)

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`

- [ ] **Step 1: Add the `HumanTask` schema definition**

Open `schema/src/main/resources/schema/CaseDefinition.yaml`. Find the `$defs` section (search for `Binding:`). Add the `HumanTask` definition immediately before the `Binding:` definition:

```yaml
  HumanTask:
    type: object
    description: >-
      A binding target that creates a WorkItem in casehub-work and resumes the case
      when the WorkItem reaches a terminal state. Inline mode requires title;
      template mode requires templateRef. The two modes are mutually exclusive.
    unevaluatedProperties: false
    oneOf:
      - required: [ title ]
        not: { required: [ templateRef ] }
      - required: [ templateRef ]
        not: { required: [ title ] }
    properties:
      title:
        type: string
        description: "WorkItem title — inline mode"
      templateRef:
        type: string
        description: "WorkItemTemplate reference (UUID or name) — template mode"
      inputMapping:
        type: string
        description: "JQ expression: case context → WorkItem payload"
      outputMapping:
        type: string
        description: "JQ expression: WorkItem resolution → case context updates"
      candidateGroups:
        type: array
        items: { type: string }
        description: "Groups eligible to claim this WorkItem"
      candidateUsers:
        type: array
        items: { type: string }
        description: "Users eligible to claim this WorkItem"
      expiresIn:
        type: string
        description: "ISO 8601 duration after which the WorkItem expires (e.g. PT24H)"
```

- [ ] **Step 2: Extend the Binding schema with humanTask as a third oneOf branch**

Find the existing `Binding:` definition. Replace the `oneOf` and `properties` section:

```yaml
  Binding:
    type: object
    required: [ on ]
    unevaluatedProperties: false
    oneOf:
      - required: [ capability ]
        not:
          required: [ subCase, humanTask ]
      - required: [ subCase ]
        not:
          required: [ capability, humanTask ]
      - required: [ humanTask ]
        not:
          required: [ capability, subCase ]
    properties:
      name: { type: string }
      on: { $ref: "#/$defs/Trigger" }
      when: { type: string, description: "JQ over context and/or event" }
      capability: { type: string }
      subCase: { $ref: "#/$defs/SubCase" }
      humanTask: { $ref: "#/$defs/HumanTask" }
      conflictResolverStrategy:
        type: string
        enum: [ LAST_WRITER_WINS, FIRST_WRITER_WINS, FAIL ]
        default: LAST_WRITER_WINS
        description: "Strategy for resolving concurrent writes to the same CaseContext key"
```

- [ ] **Step 3: Rebuild the schema module to regenerate Java models**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q -pl schema
```

Expected: BUILD SUCCESS. The `schema/target/generated-sources/jsonschema2pojo/io/casehub/model/` directory now contains `HumanTask.java` and `Binding.java` has a `getHumanTask()` method returning `io.casehub.model.HumanTask`.

Verify:
```bash
grep -l "HumanTask" /Users/mdproctor/claude/casehub/engine/schema/target/generated-sources/jsonschema2pojo/io/casehub/model/*.java
```
Expected: prints `HumanTask.java` and `Binding.java`.

- [ ] **Step 4: Confirm tests still fail (schema change alone is insufficient)**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest
```

Expected: the three new tests still fail — now Jackson can deserialize `humanTask` into the schema model, but the mapper's `convertBinding` reaches the `else` branch and throws `IllegalArgumentException("Binding 'approval' must have either capability or subCase")`. Existing tests pass.

---

## Task 3: Extend CaseDefinitionYamlMapper to handle humanTask (engine#295)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`

- [ ] **Step 1: Add imports**

At the top of `CaseDefinitionYamlMapper.java`, add with the existing imports:
```java
import io.casehub.api.model.HumanTaskTarget;
import java.time.Duration;
import java.util.LinkedHashSet;
```

- [ ] **Step 2: Add the humanTask branch in convertBinding**

In `convertBinding`, find the `else if (schemaBinding.getSubCase() != null)` block and the `else { throw ... }` after it. Replace that else block:

```java
    } else if (schemaBinding.getHumanTask() != null) {
      builder.humanTask(convertHumanTask(schemaBinding.getHumanTask()));
    } else {
      throw new IllegalArgumentException(
          "Binding '"
              + schemaBinding.getName()
              + "' must have capability, subCase, or humanTask");
    }
```

- [ ] **Step 3: Add the convertHumanTask private method**

Add this method to `CaseDefinitionYamlMapper` (alongside the existing private methods):

```java
  private static HumanTaskTarget convertHumanTask(io.casehub.model.HumanTask schema) {
    HumanTaskTarget.Builder builder =
        schema.getTemplateRef() != null
            ? HumanTaskTarget.template(schema.getTemplateRef())
            : HumanTaskTarget.inline().title(schema.getTitle());

    if (schema.getInputMapping() != null) {
      builder.inputMapping(schema.getInputMapping());
    }
    if (schema.getOutputMapping() != null) {
      builder.outputMapping(schema.getOutputMapping());
    }
    if (schema.getCandidateGroups() != null) {
      builder.candidateGroups(new LinkedHashSet<>(schema.getCandidateGroups()));
    }
    if (schema.getCandidateUsers() != null) {
      builder.candidateUsers(new LinkedHashSet<>(schema.getCandidateUsers()));
    }
    if (schema.getExpiresIn() != null) {
      builder.expiresIn(Duration.parse(schema.getExpiresIn()));
    }
    return builder.build();
  }
```

- [ ] **Step 4: Run the tests — expect PASS**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest
```

Expected: ALL tests pass, including the three new humanTask tests.

- [ ] **Step 5: Run the full api module test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api
```

Expected: BUILD SUCCESS, all tests pass, no regressions.

- [ ] **Step 6: Commit engine changes**

```bash
cd /Users/mdproctor/claude/casehub/engine
git add schema/src/main/resources/schema/CaseDefinition.yaml \
        api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java \
        api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git commit -m "feat: add humanTask binding type to YAML schema and mapper

Closes #294
Closes #295
Refs #293"
```

- [ ] **Step 7: Install engine to local Maven repo (needed by devtown)**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
```

Expected: BUILD SUCCESS. All engine modules including `casehub-engine-schema`, `casehub-engine-api`, and `casehub-engine-work-adapter` are installed to `~/.m2/repository`.

---

## Task 4: Wire casehub-engine-work-adapter in devtown (devtown#33)

**Files:**
- Modify: `app/pom.xml` (devtown)
- Modify: `review/src/main/resources/devtown/pr-review.yaml` (devtown)
- Modify: `app/src/test/resources/application.properties` (devtown)

- [ ] **Step 1: Create devtown branch**

```bash
git -C /Users/mdproctor/claude/casehub/devtown checkout -b issue-293-wire-work-adapter-hitl
```

- [ ] **Step 2: Add casehub-engine-work-adapter to app/pom.xml**

Open `/Users/mdproctor/claude/casehub/devtown/app/pom.xml`.

In the `<dependencies>` section, add after the existing `casehub-engine-scheduler-quartz` entry:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-engine-work-adapter</artifactId>
      <version>${project.version}</version>
    </dependency>
```

Note: `casehub-engine-work-adapter` transitively brings in `casehub-engine-blackboard`. No separate blackboard dependency entry is needed.

- [ ] **Step 3: Update pr-review.yaml — replace capability binding with humanTask**

Open `/Users/mdproctor/claude/casehub/devtown/review/src/main/resources/devtown/pr-review.yaml`.

Remove the unused `"human-decision:pr-approval"` capability from the capabilities list:
```yaml
    # DELETE this entire block:
    - name: "human-decision:pr-approval"
      description: "Human senior architect approval gate"
      inputSchema: "{ pr: .pr }"
      outputSchema: "{ humanApproval: { status: . } }"
```

Change the `human-approval` binding from:
```yaml
    - name: human-approval
      on: { contextChange: {} }
      when: ".pr.linesChanged > .policy.humanApprovalThreshold and .humanApproval == null"
      capability: "human-decision:pr-approval"
```

To:
```yaml
    - name: human-approval
      on: { contextChange: {} }
      when: ".pr.linesChanged > .policy.humanApprovalThreshold and .humanApproval == null"
      humanTask:
        title: "PR approval required"
        outputMapping: "{ humanApproval: { status: .decision } }"
```

- [ ] **Step 4: Add index-dependency entries to test application.properties**

Open `/Users/mdproctor/claude/casehub/devtown/app/src/test/resources/application.properties`.

Add after the existing `engine-scheduler-quartz` index-dependency block:

```properties
# Index work-adapter so HumanTaskScheduleHandler + WorkItemLifecycleAdapter CDI beans are discoverable
quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter

# Index blackboard (transitive via work-adapter) so BlackboardRegistry bean is discoverable
quarkus.index-dependency.engine-blackboard.group-id=io.casehub
quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
```

- [ ] **Step 5: Run devtown tests**

```bash
cd /Users/mdproctor/claude/casehub/devtown
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests -q
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl app
```

Expected: BUILD SUCCESS, all existing tests pass. If a test fails due to a CDI ambiguity or missing bean, diagnose using the error — common fixes are additional `quarkus.index-dependency` entries or `quarkus.arc.exclude-types` entries in `application.properties`.

- [ ] **Step 6: Commit devtown changes**

```bash
cd /Users/mdproctor/claude/casehub/devtown
git add app/pom.xml \
        review/src/main/resources/devtown/pr-review.yaml \
        app/src/test/resources/application.properties
git commit -m "feat: wire casehub-engine-work-adapter for HITL case resumption

Closes #33
Refs casehubio/engine#293"
```

---

## Task 5: Push engine branch and open PR

- [ ] **Step 1: Push engine branch**

```bash
cd /Users/mdproctor/claude/casehub/engine
git push -u origin issue-293-wire-work-adapter-hitl
```

- [ ] **Step 2: Open engine PR**

```bash
gh pr create --repo casehubio/engine \
  --title "feat: add humanTask YAML binding type and wire devtown work-adapter" \
  --body "$(cat <<'EOF'
## Summary

- Adds `humanTask` as a first-class binding target in the YAML case definition schema (`CaseDefinition.yaml`), generating `io.casehub.model.HumanTask` via jsonschema2pojo
- Extends `CaseDefinitionYamlMapper.convertBinding` to produce a `HumanTaskTarget` from `humanTask` schema bindings (both inline and template modes)

## Test plan

- [ ] `CaseDefinitionYamlMapperTest`: inline mode, template mode, all optional fields (candidateGroups, expiresIn, inputMapping) — all passing
- [ ] Full `api` module test suite passes with no regressions

## Closes

Closes #294
Closes #295
Refs #293
EOF
)"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `HumanTask` schema $def with all fields | Task 2 Step 1 |
| `humanTask` as third `Binding.oneOf` branch | Task 2 Step 2 |
| `convertBinding` handles `humanTask` | Task 3 Step 2 |
| `convertHumanTask` maps all fields to `HumanTaskTarget` | Task 3 Step 3 |
| Test: inline mode | Task 1 Step 1 |
| Test: template mode | Task 1 Step 1 |
| Test: optional fields round-trip | Task 1 Step 1 |
| devtown: `humanTask` binding in `pr-review.yaml` | Task 4 Step 3 |
| devtown: `casehub-engine-work-adapter` dep | Task 4 Step 2 |
| devtown: `quarkus.index-dependency` entries | Task 4 Step 4 |
| devtown: remove dead `"human-decision:pr-approval"` capability | Task 4 Step 3 |

**Placeholder scan:** None — all steps contain complete code.

**Type consistency check:**
- `io.casehub.model.HumanTask` (generated) — accessed via `schemaBinding.getHumanTask()` in Task 3 Step 2 ✅
- `HumanTaskTarget.Builder.inputMapping(String)` — matches Task 3 Step 3 ✅
- `HumanTaskTarget.Builder.outputMapping(String)` — matches Task 3 Step 3 ✅
- `HumanTaskTarget.Builder.candidateGroups(Set<String>)` — `new LinkedHashSet<>(schema.getCandidateGroups())` ✅
- `HumanTaskTarget.Builder.expiresIn(Duration)` — `Duration.parse(schema.getExpiresIn())` ✅
