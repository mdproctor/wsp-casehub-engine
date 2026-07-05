# CaseDefinition Types and Labels Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #652 — Add semantic labels/tags to CaseDefinition Java API
**Issue group:** #652

**Goal:** Add `types: Set<Path>` and `labels: Set<Path>` to `CaseDefinition`, with YAML schema support, mapper parsing, and registry queries. Remove dead `tags`/`metadata` from schema.

**Architecture:** `CaseDefinition` gains two `Set<Path>` fields following the existing mutable setter pattern. `CaseDefinitionYamlMapper` parses both from the generated schema model via `Path.parse()`. `DefaultCaseDefinitionRegistry` implements `findByType(Path)` and `findByLabel(Path)` with `Path.isAncestorOf()` ancestor matching. YAML schema adds `types` and `labels` as `type: array` of strings, removes dead `tags` and `metadata`.

**Tech Stack:** Java 21, Quarkus 3.32.2, `io.casehub.platform.api.path.Path`, JUnit 5, AssertJ

**Spec:** `specs/2026-07-05-case-definition-types-labels-design.md`

## Global Constraints

- Use `io.casehub.platform.api.path.Path` — the platform classification primitive
- Follow `CaseDefinition`'s existing mutable pattern (setters + builder delegates)
- SPI evolution protocol for new `default` methods on `CaseDefinitionRegistry`
- `mvn install -DskipTests -q` before module-specific tests; always include `TESTCONTAINERS_RYUK_DISABLED=true`

## File Map

**Created:**
- None

**Modified:**
- `schema/src/main/resources/schema/CaseDefinition.yaml` — add `types`, `labels`; remove `tags`, `metadata`
- `schema/src/main/resources/examples/document-processing.yaml` — remove vestigial `apiVersion`/`kind`/`metadata`; add `namespace`/`dsl`
- `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add fields, setters, getters, builder methods
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse types/labels
- `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java` — add `findByType`, `findByLabel` default methods
- `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` — implement queries
- `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — add types/labels parsing tests
- `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java` — add query tests

---

## Task 1: YAML Schema — add `types` and `labels`, remove `tags` and `metadata`

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml`
- Modify: `schema/src/main/resources/examples/document-processing.yaml`

**Interfaces:**
- Produces: Generated `io.casehub.model.CaseDefinition` with `getTypes(): List<String>` and `getLabels(): List<String>` (jsonschema2pojo generates these from the schema)

- [ ] **Step 1: Update YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`:

Add after the `summary` property (around line 67):

```yaml
  types:
    type: array
    title: CaseHubTypes
    description: >
      Behavioral type classifications for this case definition.
      Each entry is a hierarchical path string parsed via Path.parse().
      Types affect engine behavior — routing, dispatch, completion strategy.
    items:
      type: string
      minLength: 1
  labels:
    type: array
    title: CaseHubLabels
    description: >
      Operational classification labels for this case definition.
      Each entry is a hierarchical path string parsed via Path.parse().
      Labels affect organization — queues, dashboards, analytics.
    items:
      type: string
      minLength: 1
```

Remove the `tags` property block (lines 68-72):

```yaml
  tags:
    type: object
    title: CaseHubTags
    description: A key/value mapping of the CaseHub's tags, if any.
    additionalProperties: true
```

Remove the `metadata` property block (lines 73-77):

```yaml
  metadata:
    type: object
    title: CaseHubMetadata
    description: Holds additional information about the CaseHub.
    additionalProperties: true
```

Change the `required` line (line 16) from:

```yaml
required: [ apiVersion, metadata, version, spec ]
```

to:

```yaml
required: [ apiVersion, version, spec ]
```

- [ ] **Step 2: Update document-processing.yaml example**

Replace lines 16-20:

```yaml
apiVersion: casehub.io/v1
kind: CaseDefinition
metadata:
  name: document-processing
version: "1.0.0"
```

with:

```yaml
dsl: "1.0.0"
namespace: casehub-examples
name: document-processing
version: "1.0.0"
types:
  - processing/document
labels:
  - example/reference
```

- [ ] **Step 3: Rebuild schema module to regenerate classes**

Run: `mvn install -DskipTests -q -pl schema`

Verify: `io.casehub.model.CaseDefinition` now has `getTypes()` and `getLabels()` methods, and no `getTags()` or `getMetadata()`.

- [ ] **Step 4: Commit**

```
chore: update YAML schema — add types/labels, remove dead tags/metadata

Refs #652
```

---

## Task 2: Java API — add `types` and `labels` to `CaseDefinition`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` (contract tests)

**Interfaces:**
- Produces: `CaseDefinition.getTypes(): Set<Path>`, `CaseDefinition.getLabels(): Set<Path>`, `CaseDefinition.setTypes(Set<Path>)`, `CaseDefinition.setLabels(Set<Path>)`, builder methods `type(Path)`, `types(Set<Path>)`, `label(Path)`, `labels(Set<Path>)`

- [ ] **Step 1: Write failing test — types field on CaseDefinition**

In `CaseDefinitionYamlMapperTest.java`, add:

```java
@Test
void types_setViaBuilder_returnedAsUnmodifiableSet() {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("t").version("1.0.0")
      .type(Path.of("situation-response", "replan"))
      .type(Path.of("compliance", "auditable"))
      .build();

  assertThat(def.getTypes()).containsExactlyInAnyOrder(
      Path.of("situation-response", "replan"),
      Path.of("compliance", "auditable"));
  assertThrows(UnsupportedOperationException.class, () -> def.getTypes().add(Path.of("x")));
}
```

- [ ] **Step 2: Run test — expect compile failure**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#types_setViaBuilder_returnedAsUnmodifiableSet`

Expected: compile error — `type(Path)` method not found

- [ ] **Step 3: Implement types/labels on CaseDefinition**

In `CaseDefinition.java`:

Add import: `import io.casehub.platform.api.path.Path;`

Add fields after `candidateMatching` (line 51):

```java
private Set<Path> types = Set.of();
private Set<Path> labels = Set.of();
```

Add import: `import java.util.Set;` and `import java.util.LinkedHashSet;`

Add getters/setters after `setCandidateMatching` (after line 198):

```java
public Set<Path> getTypes() {
  return types;
}

public void setTypes(Set<Path> types) {
  this.types = types != null ? Set.copyOf(types) : Set.of();
}

public Set<Path> getLabels() {
  return labels;
}

public void setLabels(Set<Path> labels) {
  this.labels = labels != null ? Set.copyOf(labels) : Set.of();
}
```

In the `Builder` class, add field after `candidateMatching`:

```java
private Set<Path> types = new LinkedHashSet<>();
private Set<Path> labels = new LinkedHashSet<>();
```

Add builder methods:

```java
public Builder type(Path type) {
  this.types.add(type);
  return this;
}

public Builder types(Set<Path> types) {
  this.types = new LinkedHashSet<>(types);
  return this;
}

public Builder label(Path label) {
  this.labels.add(label);
  return this;
}

public Builder labels(Set<Path> labels) {
  this.labels = new LinkedHashSet<>(labels);
  return this;
}
```

In `build()`, add before `return caseHubDefinition`:

```java
caseHubDefinition.setTypes(types);
caseHubDefinition.setLabels(labels);
```

- [ ] **Step 4: Run test — expect pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#types_setViaBuilder_returnedAsUnmodifiableSet`

Expected: PASS

- [ ] **Step 5: Write and run labels test**

```java
@Test
void labels_setViaBuilder_returnedAsUnmodifiableSet() {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("t").version("1.0.0")
      .label(Path.of("priority", "high"))
      .label(Path.of("team", "infrastructure"))
      .build();

  assertThat(def.getLabels()).containsExactlyInAnyOrder(
      Path.of("priority", "high"),
      Path.of("team", "infrastructure"));
  assertThrows(UnsupportedOperationException.class, () -> def.getLabels().add(Path.of("x")));
}
```

```java
@Test
void types_andLabels_emptyByDefault() {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("t").version("1.0.0")
      .build();

  assertThat(def.getTypes()).isEmpty();
  assertThat(def.getLabels()).isEmpty();
}
```

```java
@Test
void types_setViaSetter_storesUnmodifiableCopy() {
  CaseDefinition def = new CaseDefinition("t", "t", "1.0.0");
  Set<Path> mutable = new LinkedHashSet<>();
  mutable.add(Path.of("a"));
  def.setTypes(mutable);
  mutable.add(Path.of("b"));

  assertThat(def.getTypes()).containsExactly(Path.of("a"));
}
```

```java
@Test
void types_setNull_treatedAsEmpty() {
  CaseDefinition def = new CaseDefinition("t", "t", "1.0.0");
  def.setTypes(null);

  assertThat(def.getTypes()).isEmpty();
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest`

Expected: all tests PASS

- [ ] **Step 6: Commit**

```
feat(api): add types and labels (Set<Path>) to CaseDefinition

Refs #652
```

---

## Task 3: YAML Mapper — parse `types` and `labels`

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java`

**Interfaces:**
- Consumes: `CaseDefinition.setTypes(Set<Path>)`, `CaseDefinition.setLabels(Set<Path>)`, generated `io.casehub.model.CaseDefinition.getTypes(): List<String>`, `getLabels(): List<String>`
- Produces: YAML `types:` and `labels:` fields parsed into `CaseDefinition` API model

- [ ] **Step 1: Write failing test — types parsed from YAML**

```java
@Test
void parseTypes() {
  String yaml = """
      dsl: "1.0.0"
      namespace: test
      name: typed-case
      version: "1.0.0"
      types:
        - situation-response/replan
        - compliance/auditable
      spec:
        capabilities: []
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

  assertThat(def.getTypes()).containsExactlyInAnyOrder(
      Path.parse("situation-response/replan"),
      Path.parse("compliance/auditable"));
}
```

Add import: `import io.casehub.platform.api.path.Path;`

- [ ] **Step 2: Run test — expect failure (types not parsed)**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parseTypes`

Expected: FAIL — `def.getTypes()` is empty

- [ ] **Step 3: Implement parsing in CaseDefinitionYamlMapper**

In `convertToApiModel()`, add after the panels parsing block (around line 245, after `def.setPanelNames(panelNames)`):

```java
    // types — behavioral type classifications
    if (schema.getTypes() != null && !schema.getTypes().isEmpty()) {
      def.setTypes(schema.getTypes().stream()
          .map(io.casehub.platform.api.path.Path::parse)
          .collect(java.util.stream.Collectors.toCollection(java.util.LinkedHashSet::new)));
    }

    // labels — operational classification labels
    if (schema.getLabels() != null && !schema.getLabels().isEmpty()) {
      def.setLabels(schema.getLabels().stream()
          .map(io.casehub.platform.api.path.Path::parse)
          .collect(java.util.stream.Collectors.toCollection(java.util.LinkedHashSet::new)));
    }
```

- [ ] **Step 4: Run test — expect pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parseTypes`

Expected: PASS

- [ ] **Step 5: Write and run labels YAML test**

```java
@Test
void parseLabels() {
  String yaml = """
      dsl: "1.0.0"
      namespace: test
      name: labeled-case
      version: "1.0.0"
      labels:
        - priority/high
        - team/infrastructure
      spec:
        capabilities: []
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

  assertThat(def.getLabels()).containsExactlyInAnyOrder(
      Path.parse("priority/high"),
      Path.parse("team/infrastructure"));
}
```

```java
@Test
void parseTypes_invalidPath_failsFast() {
  String yaml = """
      dsl: "1.0.0"
      namespace: test
      name: bad-type
      version: "1.0.0"
      types:
        - ""
      spec:
        capabilities: []
      """;
  assertThrows(IllegalArgumentException.class,
      () -> CaseDefinitionYamlMapper.load(
          new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8))));
}
```

```java
@Test
void parseTypes_absent_emptySet() {
  String yaml = """
      dsl: "1.0.0"
      namespace: test
      name: no-types
      version: "1.0.0"
      spec:
        capabilities: []
      """;
  CaseDefinition def = CaseDefinitionYamlMapper.load(
      new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

  assertThat(def.getTypes()).isEmpty();
  assertThat(def.getLabels()).isEmpty();
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest`

Expected: all tests PASS (including existing tests — removal of `tags`/`metadata` from schema should not break any mapper test since they were never parsed)

- [ ] **Step 6: Commit**

```
feat(api): parse types and labels from YAML via CaseDefinitionYamlMapper

Refs #652
```

---

## Task 4: Registry — `findByType` and `findByLabel` queries

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/CaseDefinitionRegistry.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTest.java`

**Interfaces:**
- Consumes: `CaseDefinition.getTypes(): Set<Path>`, `CaseDefinition.getLabels(): Set<Path>`
- Produces: `CaseDefinitionRegistry.findByType(Path): List<CaseDefinition>`, `CaseDefinitionRegistry.findByLabel(Path): List<CaseDefinition>`

- [ ] **Step 1: Write failing test — findByType exact match**

In `DefaultCaseDefinitionRegistryTest.java`, add:

```java
@Test
void findByType_exactMatch_returnsDefinition() throws Exception {
  // Use existing test helper pattern to register a definition with a type
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("typed").version("1.0.0")
      .type(Path.of("situation-response", "replan"))
      .build();

  registerDefinition(def);

  List<CaseDefinition> result = registry.findByType(Path.of("situation-response", "replan"));
  assertThat(result).hasSize(1);
  assertThat(result.get(0).getName()).isEqualTo("typed");
}
```

Add import: `import io.casehub.platform.api.path.Path;`

- [ ] **Step 2: Run test — expect compile failure**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTest#findByType_exactMatch_returnsDefinition`

Expected: compile error — `findByType(Path)` not found

- [ ] **Step 3: Add default methods to CaseDefinitionRegistry SPI**

In `CaseDefinitionRegistry.java`, add:

```java
import io.casehub.platform.api.path.Path;
import java.util.List;
```

Add before the closing brace:

```java
  /**
   * Find case definitions by type — matches exact or ancestor path.
   *
   * @param type the type path to match (exact match or ancestor of a definition's type)
   * @return matching definitions, empty list if none match
   */
  default List<CaseDefinition> findByType(Path type) {
    return List.of();
  }

  /**
   * Find case definitions by label — matches exact or ancestor path.
   *
   * @param label the label path to match (exact match or ancestor of a definition's label)
   * @return matching definitions, empty list if none match
   */
  default List<CaseDefinition> findByLabel(Path label) {
    return List.of();
  }
```

- [ ] **Step 4: Implement in DefaultCaseDefinitionRegistry**

Add import: `import io.casehub.platform.api.path.Path;`

Add methods:

```java
@Override
public List<CaseDefinition> findByType(Path type) {
  return registry.values().stream()
      .map(RegistryEntry::definition)
      .filter(def -> def.getTypes().stream()
          .anyMatch(t -> t.equals(type) || type.isAncestorOf(t)))
      .toList();
}

@Override
public List<CaseDefinition> findByLabel(Path label) {
  return registry.values().stream()
      .map(RegistryEntry::definition)
      .filter(def -> def.getLabels().stream()
          .anyMatch(l -> l.equals(label) || label.isAncestorOf(l)))
      .toList();
}
```

- [ ] **Step 5: Run test — expect pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTest#findByType_exactMatch_returnsDefinition`

Expected: PASS

- [ ] **Step 6: Write and run remaining tests**

```java
@Test
void findByType_ancestorMatch_returnsSubtypes() throws Exception {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("replan").version("1.0.0")
      .type(Path.of("situation-response", "replan"))
      .build();
  registerDefinition(def);

  List<CaseDefinition> result = registry.findByType(Path.of("situation-response"));
  assertThat(result).hasSize(1);
  assertThat(result.get(0).getName()).isEqualTo("replan");
}
```

```java
@Test
void findByType_noMatch_returnsEmpty() throws Exception {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("typed").version("1.0.0")
      .type(Path.of("compliance", "auditable"))
      .build();
  registerDefinition(def);

  List<CaseDefinition> result = registry.findByType(Path.of("situation-response"));
  assertThat(result).isEmpty();
}
```

```java
@Test
void findByType_noTypes_notReturned() throws Exception {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("untyped").version("1.0.0")
      .build();
  registerDefinition(def);

  List<CaseDefinition> result = registry.findByType(Path.of("anything"));
  assertThat(result).isEmpty();
}
```

```java
@Test
void findByLabel_exactMatch_returnsDefinition() throws Exception {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("labeled").version("1.0.0")
      .label(Path.of("priority", "high"))
      .build();
  registerDefinition(def);

  List<CaseDefinition> result = registry.findByLabel(Path.of("priority", "high"));
  assertThat(result).hasSize(1);
}
```

```java
@Test
void findByLabel_ancestorMatch_returnsSubLabels() throws Exception {
  CaseDefinition def = CaseDefinition.builder()
      .namespace("t").name("labeled").version("1.0.0")
      .label(Path.of("priority", "high"))
      .build();
  registerDefinition(def);

  List<CaseDefinition> result = registry.findByLabel(Path.of("priority"));
  assertThat(result).hasSize(1);
}
```

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTest`

Expected: all tests PASS

- [ ] **Step 7: Commit**

```
feat: CaseDefinitionRegistry gains findByType/findByLabel with ancestor matching

Refs #652
```

---

## Task 5: Full build verification, PLATFORM.MD update, CLAUDE.md sync

**Files:**
- Modify: `PLATFORM.MD` (parent repo at `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.MD`)
- Modify: `CLAUDE.md` (engine project)

- [ ] **Step 1: Full build**

Run: `mvn install -DskipTests -q` (verify no compile errors across all modules)

Then: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,runtime` (verify all tests pass in the two modified modules)

- [ ] **Step 2: Update PLATFORM.MD**

In `/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.MD`, add a new row to the platform types table (near the existing `Path` row at line ~362):

```
| Types and labels convention | `casehub-platform-api` | Every definable entity carries `types: Set<Path>` (behavioral contracts — affects routing, dispatch, evaluation) and `labels: Set<Path>` (operational classification — affects queues, dashboards, analytics). Both use `io.casehub.platform.api.path.Path`. Types use `implements` semantics (multi-valued). Ancestor matching via `Path.isAncestorOf()`. First adopter: `CaseDefinition` (engine#652). Second adopter: `WorkItem`/`WorkItemTemplate` (work#291/engine#653). |
```

Commit to parent repo:

```
docs: add types/labels platform convention to PLATFORM.MD

Refs casehubio/engine#652
```

- [ ] **Step 3: Sync CLAUDE.md if needed**

Check whether any CLAUDE.md sections reference `tags`, `metadata`, or need to mention the new `types`/`labels` fields. Update if so.

- [ ] **Step 4: Final commit on engine branch**

Verify all changes are committed, branch is clean.
