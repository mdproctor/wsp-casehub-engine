# YAML HTN Decomposition Tree — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #987 — feat: YAML HTN decomposition tree — explicit CompoundTask methods with guards
**Issue group:** #987

**Goal:** Add a `spec.decomposition:` YAML block that declares explicit HTN trees — compound tasks with guard-gated methods decomposing into leaf tasks or nested compounds — mapping directly to the existing `CompoundTask`/`DecompositionMethod`/`LeafTask` type hierarchy.

**Architecture:** Three YAML records (`YamlDecomposition`, `YamlHtnNode`, `YamlHtnMethod`) mirror the YAML shape. `YamlCaseDefinitionConverter` walks the tree recursively, producing `CompoundTask<JsonNode>` stored on `CaseDefinition.decompositionTree`. `ExplicitHtnDecompositionStrategy` (id=`"explicit"`) evaluates method guards and flattens to a `DagPlan<LeafTask<JsonNode>>` at runtime.

**Tech Stack:** Java 21 records, Jackson (existing), JQ expressions (existing `ExpressionEngineRegistry`)

## Global Constraints

- All existing api and planning tests must continue to pass
- Polymorphic deserializers for `ExpressionEvaluator` remain module-registered (guard expressions need it)
- Use `ide_insert_member` / `ide_replace_member` for structural edits
- Build verification: `mvn install -DskipTests -q` then `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dcheckstyle.skip=true`

---

## Batch 1: YAML Records + Converter [foundation — CaseDefinition gains decompositionTree]

### Task 1: YAML Records and CaseDefinition Field

Create the three YAML records and add the `decompositionTree` field to `CaseDefinition`.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlDecomposition.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlHtnNode.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlHtnMethod.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCaseSpec.java` — add `YamlDecomposition decomposition` field
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `decompositionTree` field
- Test: `api/src/test/java/io/casehub/api/model/converter/yaml/YamlHtnDeserializationTest.java`

**Interfaces:**
- Consumes: nothing (additive)
- Produces: `YamlDecomposition`, `YamlHtnNode`, `YamlHtnMethod` consumed by converter (Task 2); `CaseDefinition.getDecompositionTree()` consumed by strategy (Task 3)

- [ ] **Step 1: Create `YamlHtnNode` record**

```java
package io.casehub.api.model.converter.yaml;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.util.List;
import java.util.Map;

@JsonIgnoreProperties(ignoreUnknown = true)
public record YamlHtnNode(
    String name,
    String capability,
    String description,
    String estimatedDuration,
    Map<String, Integer> estimatedCost,
    List<YamlHtnMethod> methods) {

  public YamlHtnNode {
    if (estimatedCost == null) estimatedCost = Map.of();
    if (methods == null) methods = List.of();
  }

  public boolean isLeaf() {
    return capability != null && methods.isEmpty();
  }
}
```

- [ ] **Step 2: Create `YamlHtnMethod` record**

```java
package io.casehub.api.model.converter.yaml;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.List;
import java.util.Map;

@JsonIgnoreProperties(ignoreUnknown = true)
public record YamlHtnMethod(
    String name,
    String guardLabel,
    ExpressionEvaluator guard,
    String estimatedDuration,
    Map<String, Integer> estimatedCost,
    List<YamlHtnNode> tasks) {

  public YamlHtnMethod {
    if (estimatedCost == null) estimatedCost = Map.of();
    if (tasks == null) tasks = List.of();
  }
}
```

- [ ] **Step 3: Create `YamlDecomposition` record**

```java
package io.casehub.api.model.converter.yaml;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public record YamlDecomposition(YamlHtnNode root) {}
```

- [ ] **Step 4: Add `decomposition` field to `YamlCaseSpec`**

Add `YamlDecomposition decomposition` as a new component to the `YamlCaseSpec` record.

- [ ] **Step 5: Add `decompositionTree` field to `CaseDefinition`**

Add to `CaseDefinition`:
```java
private io.casehub.engine.plan.TaskNode.CompoundTask<com.fasterxml.jackson.databind.JsonNode> decompositionTree;

public io.casehub.engine.plan.TaskNode.CompoundTask<com.fasterxml.jackson.databind.JsonNode> getDecompositionTree() {
    return decompositionTree;
}

public void setDecompositionTree(
    io.casehub.engine.plan.TaskNode.CompoundTask<com.fasterxml.jackson.databind.JsonNode> decompositionTree) {
    this.decompositionTree = decompositionTree;
}
```

Add to `CaseDefinition.Builder`:
```java
private io.casehub.engine.plan.TaskNode.CompoundTask<com.fasterxml.jackson.databind.JsonNode> decompositionTree;

public Builder decompositionTree(
    io.casehub.engine.plan.TaskNode.CompoundTask<com.fasterxml.jackson.databind.JsonNode> decompositionTree) {
    this.decompositionTree = decompositionTree;
    return this;
}
```
And in `build()`: `caseHubDefinition.setDecompositionTree(decompositionTree);`

- [ ] **Step 6: Write deserialization test**

```java
@Test
void deserializesHtnTree() throws Exception {
    var yaml = """
        root:
          name: investigate
          methods:
            - guardLabel: "High severity"
              guard: ".severity == \\"high\\""
              tasks:
                - name: triage
                  capability: triage-assessment
                - name: escalate
                  capability: escalation
            - guardLabel: "Low severity"
              tasks:
                - name: auto-resolve
                  capability: auto-resolution
        """;
    ObjectMapper mapper = new ObjectMapper(new YAMLFactory())
        .registerModule(new CaseDefinitionModule(JQ_ONLY))
        .disable(FAIL_ON_UNKNOWN_PROPERTIES);
    var decomp = mapper.readValue(yaml, YamlDecomposition.class);

    assertThat(decomp.root().name()).isEqualTo("investigate");
    assertThat(decomp.root().methods()).hasSize(2);
    assertThat(decomp.root().methods().get(0).tasks()).hasSize(2);
    assertThat(decomp.root().methods().get(0).tasks().get(0).capability()).isEqualTo("triage-assessment");
    assertThat(decomp.root().methods().get(0).tasks().get(0).isLeaf()).isTrue();
}

@Test
void deserializesNestedCompound() throws Exception {
    var yaml = """
        root:
          name: loan
          methods:
            - tasks:
                - name: check
                  capability: credit-check
                - name: decision
                  methods:
                    - guard: ".score > 750"
                      tasks:
                        - name: auto
                          capability: auto-approve
                    - tasks:
                        - name: manual
                          capability: manual-review
        """;
    ObjectMapper mapper = new ObjectMapper(new YAMLFactory())
        .registerModule(new CaseDefinitionModule(JQ_ONLY))
        .disable(FAIL_ON_UNKNOWN_PROPERTIES);
    var decomp = mapper.readValue(yaml, YamlDecomposition.class);

    var decision = decomp.root().methods().get(0).tasks().get(1);
    assertThat(decision.isLeaf()).isFalse();
    assertThat(decision.methods()).hasSize(2);
}
```

- [ ] **Step 7: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="YamlHtnDeserializationTest" -Dcheckstyle.skip=true`

- [ ] **Step 8: Commit**

```
wip: add YAML HTN records + CaseDefinition.decompositionTree field Refs #987
```

### Task 2: Converter — Walk the HTN Tree

Add `convertDecomposition()` to `YamlCaseDefinitionConverter` that recursively converts the YAML tree into `CompoundTask<JsonNode>` and sets it on `CaseDefinition`.

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperHtnTest.java`

**Interfaces:**
- Consumes: `YamlDecomposition`, `YamlHtnNode`, `YamlHtnMethod` (from Task 1); `GoalStep` (existing, `planning/decomposition/`)
- Produces: `CaseDefinition` with populated `decompositionTree`; auto-sets `decompositionStrategy` to `"explicit"` when tree is present and no strategy is explicitly set

- [ ] **Step 1: Write end-to-end mapper test**

```java
@Test
void fullHtnDefinition_producesDecompositionTree() throws Exception {
    var yaml = """
        name: incident-response
        namespace: io.casehub.test
        version: "1.0"
        spec:
          capabilities:
            - name: triage-assessment
            - name: escalation
            - name: auto-resolution
          decomposition:
            root:
              name: investigate
              methods:
                - guardLabel: "High severity"
                  guard: ".severity == \\"high\\""
                  tasks:
                    - name: triage
                      capability: triage-assessment
                    - name: escalate
                      capability: escalation
                - guardLabel: "Low severity"
                  tasks:
                    - name: auto-resolve
                      capability: auto-resolution
        workers:
          - name: triager
            capabilities: [triage-assessment]
        bindings:
          - name: trigger
            capability: triage-assessment
            on:
              contextChange: {}
        """;
    var def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    assertThat(def.getDecompositionTree()).isNotNull();
    assertThat(def.getDecompositionTree().name()).isEqualTo("investigate");
    assertThat(def.getDecompositionTree().methods()).hasSize(2);
    assertThat(def.getDecompositionStrategy()).isEqualTo("explicit");
}

@Test
void explicitStrategy_notOverriddenWhenSet() throws Exception {
    var yaml = """
        name: test
        namespace: io.casehub.test
        version: "1.0"
        spec:
          capabilities:
            - name: analysis
          decompositionStrategy: goap
          decomposition:
            root:
              name: plan
              methods:
                - tasks:
                    - name: analyze
                      capability: analysis
        workers:
          - name: analyzer
            capabilities: [analysis]
        bindings:
          - name: trigger
            capability: analysis
            on:
              contextChange: {}
        """;
    var def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    assertThat(def.getDecompositionTree()).isNotNull();
    assertThat(def.getDecompositionStrategy()).isEqualTo("goap");
}

@Test
void noDecomposition_treeIsNull() throws Exception {
    var yaml = """
        name: test
        namespace: io.casehub.test
        version: "1.0"
        spec:
          capabilities:
            - name: analysis
        workers:
          - name: analyzer
            capabilities: [analysis]
        bindings:
          - name: trigger
            capability: analysis
            on:
              contextChange: {}
        """;
    var def = CaseDefinitionYamlMapper.load(
        new ByteArrayInputStream(yaml.getBytes(StandardCharsets.UTF_8)));

    assertThat(def.getDecompositionTree()).isNull();
}
```

- [ ] **Step 2: Implement `convertDecomposition()` in converter**

Add to `YamlCaseDefinitionConverter`:

```java
private static void convertDecomposition(
    YamlDecomposition decomp, CaseDefinition def, ExpressionEngineRegistry registry) {
  if (decomp == null || decomp.root() == null) return;

  var root = convertCompoundNode(decomp.root(), registry);
  def.setDecompositionTree(root);

  if (def.getDecompositionStrategy() == null) {
    def.setDecompositionStrategy("explicit");
  }
}

private static TaskNode.CompoundTask<JsonNode> convertCompoundNode(
    YamlHtnNode node, ExpressionEngineRegistry registry) {
  List<DecompositionMethod<JsonNode>> methods = node.methods().stream()
      .map(m -> convertMethod(m, registry))
      .toList();
  return new TaskNode.CompoundTask<>(node.name(), methods);
}

private static DecompositionMethod<JsonNode> convertMethod(
    YamlHtnMethod method, ExpressionEngineRegistry registry) {
  Predicate<JsonNode> guard = method.guard() != null
      ? state -> registry.evaluate(method.guard(), state)
      : state -> true;

  DecompositionStrategy<JsonNode> strategy = (task, ctx) -> {
    List<TaskNode.LeafTask<JsonNode>> leaves = new ArrayList<>();
    for (var childNode : method.tasks()) {
      if (childNode.isLeaf()) {
        leaves.add(new GoalStep(
            UUID.randomUUID(), childNode.description() != null ? childNode.description() : childNode.name(),
            childNode.capability(), Instant.now()));
      }
    }
    return DagPlan.sequence(leaves);
  };

  Duration estimatedDuration = method.estimatedDuration() != null
      ? Duration.parse(method.estimatedDuration()) : null;

  return new DecompositionMethod<>(
      method.name(), guard, strategy, method.guardLabel(),
      method.estimatedCost().isEmpty() ? null : method.estimatedCost(),
      estimatedDuration);
}
```

Wire `convertDecomposition()` into the `convert()` method after `convertSpec()`:
```java
if (yaml.spec() != null && yaml.spec().decomposition() != null) {
    convertDecomposition(yaml.spec().decomposition(), def, registry);
}
```

- [ ] **Step 3: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CaseDefinitionYamlMapperHtnTest" -Dcheckstyle.skip=true`

- [ ] **Step 4: Run full api test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dcheckstyle.skip=true`

- [ ] **Step 5: Commit**

```
feat(#987): converter — walk HTN tree, produce CompoundTask<JsonNode> on CaseDefinition
```

## Batch 2: ExplicitHtnDecompositionStrategy [runtime — HTN tree executes]

### Task 3: ExplicitHtnDecompositionStrategy

Create the strategy that evaluates method guards and produces a `DagPlan<LeafTask<JsonNode>>` from the explicit HTN tree.

**Files:**
- Create: `planning/src/main/java/io/casehub/engine/planning/decomposition/ExplicitHtnDecompositionStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java` — add `Instance<ExplicitHtnDecompositionStrategy>` injection
- Test: `planning/src/test/java/io/casehub/engine/planning/decomposition/ExplicitHtnDecompositionStrategyTest.java`

**Interfaces:**
- Consumes: `CaseDefinition.getDecompositionTree()` (from Task 1); `GoalStep` (existing); `DagPlan.sequence()` (existing)
- Produces: `DagPlan<LeafTask<JsonNode>>` — the flattened execution plan from the HTN tree

- [ ] **Step 1: Write failing test — single method, no guards**

```java
@Test
void singleMethod_noGuard_producesSequentialPlan() {
    var leaf1 = new YamlHtnNode("triage", "triage-assessment", null, null, Map.of(), List.of());
    var leaf2 = new YamlHtnNode("resolve", "resolution", null, null, Map.of(), List.of());
    var method = new YamlHtnMethod(null, null, null, null, Map.of(), List.of(leaf1, leaf2));
    var root = new YamlHtnNode("incident", null, null, null, Map.of(), List.of(method));

    var compoundTask = convertCompound(root);
    var strategy = new ExplicitHtnDecompositionStrategy();
    var context = createContext(MAPPER.createObjectNode());

    var plan = strategy.decompose(compoundTask, context);

    assertThat(plan.nodes()).hasSize(2);
}
```

- [ ] **Step 2: Write failing test — guarded methods, first match selected**

```java
@Test
void guardedMethods_firstMatchSelected() {
    // high severity method guards on .severity == "high"
    // low severity method guards on .severity == "low"
    // input state: severity = "high"
    // expect: high severity tasks selected
}
```

- [ ] **Step 3: Write failing test — nested compound recursion**

```java
@Test
void nestedCompound_recursivelyFlattened() {
    // root → method → [leaf, compound → method → [leaf, leaf]]
    // expect: 3 leaf tasks in the flat plan
}
```

- [ ] **Step 4: Implement `ExplicitHtnDecompositionStrategy`**

```java
@ApplicationScoped
public class ExplicitHtnDecompositionStrategy implements DecompositionStrategy<JsonNode> {

    @Override
    public String id() { return "explicit"; }

    @Override
    public DagPlan<LeafTask<JsonNode>> decompose(TaskNode<JsonNode> task, DecompositionContext<JsonNode> context) {
        if (!(task instanceof TaskNode.CompoundTask<JsonNode> compound)) {
            throw new IllegalArgumentException("ExplicitHtnDecompositionStrategy requires a CompoundTask");
        }
        List<LeafTask<JsonNode>> leaves = new ArrayList<>();
        decomposeRecursive(compound, context, leaves);
        return DagPlan.sequence(leaves);
    }

    private void decomposeRecursive(
            TaskNode.CompoundTask<JsonNode> compound,
            DecompositionContext<JsonNode> context,
            List<LeafTask<JsonNode>> accumulator) {
        for (var method : compound.methods()) {
            if (method.guard().test(context.state())) {
                var plan = method.strategy().decompose(compound, context);
                for (var node : plan.nodes()) {
                    accumulator.add(node.task());
                }
                return;
            }
        }
        throw new IllegalStateException("No matching method for compound '" + compound.name() + "'");
    }
}
```

- [ ] **Step 5: Register in `EngineStrategyResolver`**

Add `@Any Instance<ExplicitHtnDecompositionStrategy>` injection and registration.

- [ ] **Step 6: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest="ExplicitHtnDecompositionStrategyTest" -Dcheckstyle.skip=true`

- [ ] **Step 7: Commit**

```
feat(#987): ExplicitHtnDecompositionStrategy — guard evaluation + recursive flattening

Closes #987
```

---

## References

- [2026-08-31-yaml-htn-decomposition-design.md] — design spec this plan implements
- [TaskNode.java] — sealed interface (CompoundTask, LeafTask)
- [DecompositionMethod.java] — guard-gated method record
- [DecompositionStrategy.java] — SPI interface
- [GoalStep.java] — existing LeafTask<JsonNode> implementation
- [YamlCaseDefinitionConverter.java] — converter from #1015
- [GitHub #987] — tracking issue
- [GitHub #978] — parent epic
