# CaseDefinitionYamlMapper Error Handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three input validations to `CaseDefinitionYamlMapper.convertHumanTask` so malformed humanTask bindings produce actionable `IllegalArgumentException`s with binding name context.

**Architecture:** All validation goes directly in `convertHumanTask`. Errors thrown as `IllegalArgumentException`; the existing catch block in `convertBinding` wraps them with the binding name (`"Binding 'X' has invalid humanTask: ..."`). No structural changes outside these two files.

**Tech Stack:** Java 17+, JUnit 5, AssertJ — `casehub-engine-api` module (no Quarkus, no containers).

---

## Files

- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`

---

### Task 1: Write failing tests for all three validations

- [ ] **Step 1: Add three test methods to `CaseDefinitionYamlMapperTest`**

Add at the end of the class, before the closing `}`:

```java
@Test
void humanTaskBinding_withBothTitleAndTemplateRef_throwsIllegalArgument() {
  String yaml =
      """
      namespace: test
      name: Conflict Case
      version: 1.0.0
      spec:
        bindings:
          - name: conflict-binding
            on: { contextChange: {} }
            humanTask:
              title: "Inline title"
              templateRef: "some-template"
      """;

  assertThatThrownBy(
          () ->
              CaseDefinitionYamlMapper.load(
                  new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))))
      .isInstanceOf(IllegalArgumentException.class)
      .hasMessageContaining("conflict-binding")
      .hasMessageContaining("cannot specify both");
}

@Test
void humanTaskBinding_withInvalidExpiresInFormat_throwsIllegalArgument() {
  String yaml =
      """
      namespace: test
      name: Bad Expires Case
      version: 1.0.0
      spec:
        bindings:
          - name: expires-binding
            on: { contextChange: {} }
            humanTask:
              title: "Review"
              expiresIn: "P1D"
      """;

  assertThatThrownBy(
          () ->
              CaseDefinitionYamlMapper.load(
                  new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))))
      .isInstanceOf(IllegalArgumentException.class)
      .hasMessageContaining("expires-binding")
      .hasMessageContaining("P1D");
}

@Test
void humanTaskBinding_withNonPositiveExpiresIn_throwsIllegalArgument() {
  String yaml =
      """
      namespace: test
      name: Zero Expires Case
      version: 1.0.0
      spec:
        bindings:
          - name: zero-binding
            on: { contextChange: {} }
            humanTask:
              title: "Review"
              expiresIn: "PT0S"
      """;

  assertThatThrownBy(
          () ->
              CaseDefinitionYamlMapper.load(
                  new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))))
      .isInstanceOf(IllegalArgumentException.class)
      .hasMessageContaining("zero-binding")
      .hasMessageContaining("must be positive");
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#humanTaskBinding_withBothTitleAndTemplateRef_throwsIllegalArgument+humanTaskBinding_withInvalidExpiresInFormat_throwsIllegalArgument+humanTaskBinding_withNonPositiveExpiresIn_throwsIllegalArgument
```

Expected: 3 FAILED — the first silently returns a result (no exception), the second and third throw `DateTimeParseException` or nothing (wrong type/no exception).

---

### Task 2: Add validation 1 — conflicting fields

- [ ] **Step 3: Add the both-fields check at the top of `convertHumanTask`**

In `CaseDefinitionYamlMapper.java`, `convertHumanTask` currently starts:

```java
private static HumanTaskTarget convertHumanTask(io.casehub.model.HumanTask schema) {
  HumanTaskTarget.Builder builder =
      schema.getTemplateRef() != null
          ? HumanTaskTarget.template(schema.getTemplateRef())
          : HumanTaskTarget.inline().title(schema.getTitle());
```

Replace with:

```java
private static HumanTaskTarget convertHumanTask(io.casehub.model.HumanTask schema) {
  if (schema.getTitle() != null && schema.getTemplateRef() != null) {
    throw new IllegalArgumentException(
        "humanTask cannot specify both title and templateRef"
            + " — use inline mode (title) or template mode (templateRef), not both");
  }
  HumanTaskTarget.Builder builder =
      schema.getTemplateRef() != null
          ? HumanTaskTarget.template(schema.getTemplateRef())
          : HumanTaskTarget.inline().title(schema.getTitle());
```

- [ ] **Step 4: Run the conflicting-fields test to confirm it passes**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#humanTaskBinding_withBothTitleAndTemplateRef_throwsIllegalArgument
```

Expected: PASS.

---

### Task 3: Add validations 2 & 3 — `expiresIn` format and positivity

- [ ] **Step 5: Add `DateTimeParseException` import to `CaseDefinitionYamlMapper.java`**

Add to the import block (keep imports alphabetically ordered):

```java
import java.time.format.DateTimeParseException;
```

- [ ] **Step 6: Replace the bare `Duration.parse` call**

Find in `convertHumanTask`:

```java
    if (schema.getExpiresIn() != null) {
      builder.expiresIn(Duration.parse(schema.getExpiresIn()));
    }
```

Replace with:

```java
    if (schema.getExpiresIn() != null) {
      Duration duration;
      try {
        duration = Duration.parse(schema.getExpiresIn());
      } catch (DateTimeParseException e) {
        throw new IllegalArgumentException(
            "invalid expiresIn '"
                + schema.getExpiresIn()
                + "' — must be ISO-8601 duration (e.g. PT24H, PT1H30M)",
            e);
      }
      if (duration.isNegative() || duration.isZero()) {
        throw new IllegalArgumentException(
            "expiresIn must be positive, got '" + schema.getExpiresIn() + "'");
      }
      builder.expiresIn(duration);
    }
```

- [ ] **Step 7: Run all three new tests to confirm they pass**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#humanTaskBinding_withBothTitleAndTemplateRef_throwsIllegalArgument+humanTaskBinding_withInvalidExpiresInFormat_throwsIllegalArgument+humanTaskBinding_withNonPositiveExpiresIn_throwsIllegalArgument
```

Expected: 3 PASSED.

- [ ] **Step 8: Run the full `CaseDefinitionYamlMapperTest` to confirm no regressions**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest
```

Expected: all tests PASSED (11 total after additions).

---

### Task 4: Commit

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java \
  api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix(api): validate humanTask conflicting fields and expiresIn in CaseDefinitionYamlMapper (Closes #297)"
```
