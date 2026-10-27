# Quorum Coordination Primitive Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1257 — Case DSL: add quorum coordination primitive
**Issue group:** #1257

**Goal:** Add a binding-level `requires` quorum guard that gates dispatch until N of M referenced bindings have COMPLETED, and fix compound-level MofN YAML wiring.

**Architecture:** Two streams. Stream 1 adds `BindingRequires` to the API model, load-time validation, a pre-strategy filter in `PlanningStrategyLoopControl`, a `RequiresThresholdEvaluator` for CANCEL and unreachable-quorum handling via `PlanItemStateChangedEvent`, annotation support, and `CoordinationPipelineProjection` updates. Stream 2 fixes `YamlCaseDefinitionConverter.convertCompound()` to wire `mofnRequired` and adds a `CompoundBlackboardPlanConfigurer`.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JQ expressions

## Global Constraints

- All three pathways (YAML, Java builder, annotations) must expose `requires` — protocol PP-20260915-aa504e
- Definition types in `engine-api`; evaluation logic in `planning-core` — protocol PP-20260727-5267d2
- Test classes must be `*Test.java`, never `*IT.java`
- `TESTCONTAINERS_RYUK_DISABLED=true` for all test runs
- `mvn install -DskipTests -q` before any module-specific test run

---

## Batch 1: API Foundation — BindingRequires record, Binding updates, validation

### Task 1: BindingRequires record and Binding integration

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/BindingRequires.java`
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java:32` — add field, getter, setter, builder method
- Test: `api/src/test/java/io/casehub/api/model/BindingRequiresTest.java`

**Interfaces:**
- Consumes: `OnThresholdReached` enum from `io.casehub.api.model.OnThresholdReached`
- Produces: `BindingRequires(List<String> bindings, int atLeast, OnThresholdReached onThresholdReached)` record; `Binding.getRequires()`, `Binding.requires()`, `Binding.Builder.requires(BindingRequires)`

- [ ] **Step 1: Write failing test for BindingRequires compact constructor validation**

```java
package io.casehub.api.model;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class BindingRequiresTest {

    @Test
    void validRequires() {
        var req = new BindingRequires(List.of("a", "b", "c"), 2, OnThresholdReached.KEEP);
        assertEquals(3, req.bindings().size());
        assertEquals(2, req.atLeast());
        assertEquals(OnThresholdReached.KEEP, req.onThresholdReached());
    }

    @Test
    void defaultsToKeep() {
        var req = new BindingRequires(List.of("a", "b"), 1, null);
        assertEquals(OnThresholdReached.KEEP, req.onThresholdReached());
    }

    @Test
    void rejectsTooFewBindings() {
        assertThrows(IllegalArgumentException.class,
            () -> new BindingRequires(List.of("a"), 1, OnThresholdReached.KEEP));
    }

    @Test
    void rejectsAtLeastGreaterThanBindingsSize() {
        assertThrows(IllegalArgumentException.class,
            () -> new BindingRequires(List.of("a", "b"), 3, OnThresholdReached.KEEP));
    }

    @Test
    void rejectsAtLeastZero() {
        assertThrows(IllegalArgumentException.class,
            () -> new BindingRequires(List.of("a", "b"), 0, OnThresholdReached.KEEP));
    }

    @Test
    void rejectsNullBindings() {
        assertThrows(NullPointerException.class,
            () -> new BindingRequires(null, 1, OnThresholdReached.KEEP));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Create BindingRequires record**

Create `api/src/main/java/io/casehub/api/model/BindingRequires.java`:

```java
package io.casehub.api.model;

import java.util.List;
import java.util.Objects;

public record BindingRequires(
    List<String> bindings,
    int atLeast,
    OnThresholdReached onThresholdReached
) {
    public BindingRequires {
        Objects.requireNonNull(bindings, "bindings must not be null");
        if (bindings.size() < 2)
            throw new IllegalArgumentException("requires.bindings must have at least 2 entries");
        if (atLeast < 1 || atLeast > bindings.size())
            throw new IllegalArgumentException(
                "atLeast must be between 1 and bindings.size(), got " + atLeast + " for " + bindings.size() + " bindings");
        if (onThresholdReached == null) onThresholdReached = OnThresholdReached.KEEP;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresTest -q`
Expected: PASS

- [ ] **Step 5: Add requires field to Binding class**

Use `ide_insert_member` to add to `Binding.java`:
- Private field: `private BindingRequires requires;`
- Getter: `public BindingRequires getRequires() { return requires; }`
- Setter: `public void setRequires(BindingRequires requires) { this.requires = requires; }`
- Convenience accessor: `public BindingRequires requires() { return requires; }`
- Builder method in `Binding.Builder`:
  ```java
  public Builder requires(BindingRequires requires) {
      binding.setRequires(requires);
      return this;
  }
  ```

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/BindingRequires.java api/src/main/java/io/casehub/api/model/Binding.java api/src/test/java/io/casehub/api/model/BindingRequiresTest.java
git commit -m "feat(#1257): add BindingRequires record and Binding.requires field

Refs casehubio/engine#1257"
```

### Task 2: Load-time validation for requires bindings

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java:326` — add `validateRequiresBindings()` method
- Test: `api/src/test/java/io/casehub/api/model/BindingRequiresValidationTest.java`

**Interfaces:**
- Consumes: `BindingRequires` record (Task 1), `CompoundDeclaration` record from `io.casehub.api.model.CompoundDeclaration`
- Produces: `static List<String> validateRequiresBindings(List<Binding> bindings, List<CompoundDeclaration> compounds)`

- [ ] **Step 1: Write failing tests for validation**

```java
package io.casehub.api.model;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class BindingRequiresValidationTest {

    private Binding binding(String name, BindingRequires requires) {
        var b = Binding.builder().name(name).build();
        b.setRequires(requires);
        return b;
    }

    private Binding binding(String name) {
        return Binding.builder().name(name).build();
    }

    @Test
    void noRequiresNoErrors() {
        var errors = Binding.validateRequiresBindings(
            List.of(binding("a"), binding("b")), List.of());
        assertTrue(errors.isEmpty());
    }

    @Test
    void selfReferenceDetected() {
        var errors = Binding.validateRequiresBindings(
            List.of(
                binding("a", new BindingRequires(List.of("a", "b"), 1, null)),
                binding("b")),
            List.of());
        assertTrue(errors.stream().anyMatch(e -> e.contains("self-reference")));
    }

    @Test
    void missingBindingDetected() {
        var errors = Binding.validateRequiresBindings(
            List.of(
                binding("a", new BindingRequires(List.of("b", "missing"), 1, null)),
                binding("b")),
            List.of());
        assertTrue(errors.stream().anyMatch(e -> e.contains("missing")));
    }

    @Test
    void cycleDetected() {
        var errors = Binding.validateRequiresBindings(
            List.of(
                binding("a", new BindingRequires(List.of("b", "c"), 1, null)),
                binding("b", new BindingRequires(List.of("a", "c"), 1, null)),
                binding("c")),
            List.of());
        assertTrue(errors.stream().anyMatch(e -> e.contains("cycle")));
    }

    @Test
    void duplicateBindingNamesDetected() {
        var errors = Binding.validateRequiresBindings(
            List.of(
                binding("a", new BindingRequires(List.of("b", "b"), 1, null)),
                binding("b")),
            List.of());
        assertTrue(errors.stream().anyMatch(e -> e.contains("duplicate")));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresValidationTest -q`
Expected: FAIL — method not found

- [ ] **Step 3: Implement validateRequiresBindings()**

Use `ide_insert_member` to add static method to `Binding.java`, after `validateCompensationBindings()`:

```java
public static List<String> validateRequiresBindings(List<Binding> bindings, List<CompoundDeclaration> compounds) {
    List<String> errors = new ArrayList<>();
    Map<String, Binding> byName = new LinkedHashMap<>();
    for (Binding b : bindings) {
        if (b.getName() != null) byName.put(b.getName(), b);
    }
    Map<String, String> bindingToCompound = new HashMap<>();
    for (CompoundDeclaration c : compounds) {
        if (c.scopedBindings() != null) {
            for (String bName : c.scopedBindings().keySet()) {
                bindingToCompound.put(bName, c.name());
            }
        }
    }
    for (Binding b : bindings) {
        if (b.getRequires() == null) continue;
        BindingRequires req = b.getRequires();
        // Duplicate check
        Set<String> seen = new HashSet<>();
        for (String ref : req.bindings()) {
            if (!seen.add(ref)) {
                errors.add("Binding '" + b.getName() + "' requires: duplicate reference '" + ref + "'");
            }
        }
        // Self-reference check
        if (req.bindings().contains(b.getName())) {
            errors.add("Binding '" + b.getName() + "' requires: self-reference");
        }
        // Existence check
        for (String ref : req.bindings()) {
            if (!byName.containsKey(ref)) {
                errors.add("Binding '" + b.getName() + "' requires: referenced binding '" + ref + "' does not exist");
            }
        }
        // Scope compatibility check
        String myCompound = bindingToCompound.get(b.getName());
        for (String ref : req.bindings()) {
            String refCompound = bindingToCompound.get(ref);
            if (!Objects.equals(myCompound, refCompound)) {
                errors.add("Binding '" + b.getName() + "' requires: referenced binding '" + ref
                    + "' is in a different scope ('" + refCompound + "' vs '" + myCompound + "')");
            }
        }
    }
    // Cycle detection
    for (Binding b : bindings) {
        if (b.getRequires() == null) continue;
        Set<String> visited = new HashSet<>();
        if (hasCycle(b.getName(), byName, visited)) {
            errors.add("Binding '" + b.getName() + "' requires: cycle detected in requires chain");
        }
    }
    return errors;
}

private static boolean hasCycle(String name, Map<String, Binding> byName, Set<String> visited) {
    if (!visited.add(name)) return true;
    Binding b = byName.get(name);
    if (b == null || b.getRequires() == null) return false;
    for (String ref : b.getRequires().bindings()) {
        if (hasCycle(ref, byName, new HashSet<>(visited))) return true;
    }
    return false;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresValidationTest -q`
Expected: PASS

- [ ] **Step 5: Wire validation call**

Find where `validateCompensationBindings()` is called (use `ide_find_references` on `validateCompensationBindings`). Add `validateRequiresBindings()` call at the same validation point.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/Binding.java api/src/test/java/io/casehub/api/model/BindingRequiresValidationTest.java
git commit -m "feat(#1257): add load-time validation for requires bindings

Self-reference, existence, cycle, duplicate, and scope checks.

Refs casehubio/engine#1257"
```

### Task 3: Schema and YAML record mapping for requires

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `BindingRequires` $def and `requires` on Binding
- Modify: `schema/src/main/resources/schema/yaml-record-mappings.yaml` — add `YamlBindingRequires` type and field on Binding
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` — wire requires during conversion
- Test: `api/src/test/java/io/casehub/api/model/converter/BindingRequiresYamlTest.java`

**Interfaces:**
- Consumes: `BindingRequires` record (Task 1), generated `YamlBindingRequires` record (codegen)
- Produces: YAML `requires` property parsed into `Binding.requires` via converter

- [ ] **Step 1: Write failing test for YAML requires parsing**

```java
package io.casehub.api.model.converter;

import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.OnThresholdReached;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class BindingRequiresYamlTest {

    @Test
    void parsesRequiresFromYaml() {
        String yaml = """
            dsl: 0.1.0
            namespace: test
            name: quorum-test
            version: 1.0.0
            spec:
              capabilities:
                - name: extract
                - name: aggregate
              workers:
                - name: extractor
                  capabilities: [extract]
                - name: aggregator
                  capabilities: [aggregate]
              bindings:
                - name: extract-a
                  capability: extract
                  on: { scopeActivated: {} }
                  lifecycleScope: COMPOUND
                - name: extract-b
                  capability: extract
                  on: { scopeActivated: {} }
                  lifecycleScope: COMPOUND
                - name: aggregate
                  capability: aggregate
                  on: { scopeActivated: {} }
                  lifecycleScope: COMPOUND
                  requires:
                    bindings: [extract-a, extract-b]
                    atLeast: 1
                    onThresholdReached: KEEP
            """;
        CaseDefinition def = YamlCaseDefinitionConverter.fromYaml(yaml);
        Binding agg = def.getBindings().stream()
            .filter(b -> "aggregate".equals(b.getName()))
            .findFirst().orElseThrow();
        assertNotNull(agg.getRequires());
        assertEquals(2, agg.getRequires().bindings().size());
        assertEquals(1, agg.getRequires().atLeast());
        assertEquals(OnThresholdReached.KEEP, agg.getRequires().onThresholdReached());
    }

    @Test
    void bindingWithoutRequiresHasNullRequires() {
        String yaml = """
            dsl: 0.1.0
            namespace: test
            name: no-requires
            version: 1.0.0
            spec:
              capabilities:
                - name: extract
              workers:
                - name: extractor
                  capabilities: [extract]
              bindings:
                - name: extract-a
                  capability: extract
                  on: { scopeActivated: {} }
                  lifecycleScope: COMPOUND
            """;
        CaseDefinition def = YamlCaseDefinitionConverter.fromYaml(yaml);
        assertNull(def.getBindings().get(0).getRequires());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresYamlTest -q`
Expected: FAIL — schema rejects `requires` property

- [ ] **Step 3: Add BindingRequires to YAML schema**

Edit `schema/src/main/resources/schema/CaseDefinition.yaml`:

1. Add `BindingRequires` to `$defs` section
2. Add `requires` property to the `Binding` type with `$ref: "#/$defs/BindingRequires"`

- [ ] **Step 4: Add YamlBindingRequires to yaml-record-mappings**

Edit `schema/src/main/resources/schema/yaml-record-mappings.yaml`:

1. Add `requires: { type: YamlBindingRequires }` to `Binding` fields
2. Add new `YamlBindingRequires` type definition with extra fields: `bindings: List<String>`, `atLeast: Integer`, `onThresholdReached: String`

- [ ] **Step 5: Wire requires in YamlCaseDefinitionConverter**

Find the binding conversion method in `YamlCaseDefinitionConverter.java` (use `ide_find_symbol` for `convertBinding`). Add requires conversion:

```java
if (yb.requires() != null) {
    binding.setRequires(new BindingRequires(
        yb.requires().bindings(),
        yb.requires().atLeast(),
        yb.requires().onThresholdReached() != null
            ? OnThresholdReached.valueOf(yb.requires().onThresholdReached())
            : OnThresholdReached.KEEP
    ));
}
```

- [ ] **Step 6: Rebuild and run test**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=BindingRequiresYamlTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add schema/src/main/resources/schema/CaseDefinition.yaml schema/src/main/resources/schema/yaml-record-mappings.yaml api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java api/src/test/java/io/casehub/api/model/converter/BindingRequiresYamlTest.java
git commit -m "feat(#1257): add requires to YAML schema and record mapping

BindingRequires $def, YamlBindingRequires type, converter wiring.

Refs casehubio/engine#1257"
```

---

## Batch 2: Compound MofN YAML Wiring Fix

### Task 4: Fix mofnRequired in converter and add CompoundBlackboardPlanConfigurer

**Files:**
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `mofnRequired` to CompoundDeclaration
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java:1315` — wire mofnRequired
- Create: `planning-core/src/main/java/io/casehub/engine/planning/control/CompoundBlackboardPlanConfigurer.java`
- Test: `planning-core/src/test/java/io/casehub/engine/planning/control/CompoundBlackboardPlanConfigurerTest.java`

**Interfaces:**
- Consumes: `CompoundDeclaration` record (api), `BlackboardPlanConfigurer` SPI (planning-core), `CasePlanModel` interface (planning-core), `CompletionSemantics` sealed interface (planning-core), `PlanItemDefinition.Compound.Builder` (planning-core)
- Produces: `CompoundBlackboardPlanConfigurer` CDI bean; fixed `convertCompound()` that passes `mofnRequired` into `completionSemantics` string

- [ ] **Step 1: Write failing test for CompoundBlackboardPlanConfigurer**

```java
package io.casehub.engine.planning.control;

import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CompoundDeclaration;
import io.casehub.api.model.Participation;
import io.casehub.engine.planning.plan.CompletionSemantics;
import io.casehub.engine.planning.plan.DefaultCasePlanModel;
import io.casehub.engine.planning.plan.PlanItemDefinition;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class CompoundBlackboardPlanConfigurerTest {

    @Test
    void registersCompoundWithMofNSemantics() {
        var compound = new CompoundDeclaration(
            "extraction", "3", null,
            Map.of("a", Participation.PARTICIPANT, "b", Participation.PARTICIPANT, "c", Participation.PARTICIPANT),
            null, null, false, null);

        var def = CaseDefinition.builder()
            .namespace("test").name("test").version("1.0.0").dsl("0.1.0")
            .compound(compound)
            .build();

        var plan = new DefaultCasePlanModel();
        var configurer = new CompoundBlackboardPlanConfigurer();
        configurer.configure(plan, mockContext(def));

        var registered = plan.getCompound("extraction");
        assertNotNull(registered);
        assertInstanceOf(CompletionSemantics.MOfN.class, registered.completion());
        assertEquals(3, ((CompletionSemantics.MOfN) registered.completion()).m());
    }

    @Test
    void defaultsToAllSemantics() {
        var compound = new CompoundDeclaration(
            "phase1", null, null,
            Map.of("a", Participation.PARTICIPANT),
            null, null, false, null);

        var def = CaseDefinition.builder()
            .namespace("test").name("test").version("1.0.0").dsl("0.1.0")
            .compound(compound)
            .build();

        var plan = new DefaultCasePlanModel();
        var configurer = new CompoundBlackboardPlanConfigurer();
        configurer.configure(plan, mockContext(def));

        var registered = plan.getCompound("phase1");
        assertNotNull(registered);
        assertInstanceOf(CompletionSemantics.All.class, registered.completion());
    }
}
```

Note: `mockContext(def)` helper creates a `PlanExecutionContext` stub — check existing test patterns for the correct factory method.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=CompoundBlackboardPlanConfigurerTest -q`
Expected: FAIL — class not found

- [ ] **Step 3: Add mofnRequired to CaseDefinition.yaml CompoundDeclaration**

Edit `schema/src/main/resources/schema/CaseDefinition.yaml`, add to `CompoundDeclaration` properties:

```yaml
mofnRequired:
  type: integer
  minimum: 1
  description: Number of scoped bindings that must complete for MofN completion semantics.
```

- [ ] **Step 4: Fix convertCompound() to wire mofnRequired**

In `YamlCaseDefinitionConverter.convertCompound()` (line ~1315), add before the CompoundDeclaration constructor call:

```java
String resolvedSemantics = yc.completionSemantics();
if (yc.mofnRequired() != null) {
    if (resolvedSemantics == null || "mofn".equalsIgnoreCase(resolvedSemantics)) {
        resolvedSemantics = String.valueOf(yc.mofnRequired());
    }
}
```

Pass `resolvedSemantics` instead of `yc.completionSemantics()` to the constructor.

- [ ] **Step 5: Implement CompoundBlackboardPlanConfigurer**

```java
package io.casehub.engine.planning.control;

import io.casehub.api.model.CompoundDeclaration;
import io.casehub.api.model.Participation;
import io.casehub.engine.planning.plan.*;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CompoundBlackboardPlanConfigurer implements BlackboardPlanConfigurer {

    @Override
    public void configure(CasePlanModel plan, PlanExecutionContext context) {
        if (context.definition().getCompounds() == null) return;
        for (CompoundDeclaration cd : context.definition().getCompounds()) {
            CompletionSemantics completion = parseCompletionSemantics(cd.completionSemantics());
            DispatchMode dispatch = parseDispatchMode(cd.dispatchMode());
            var builder = PlanItemDefinition.Compound.builder()
                .name(cd.name())
                .completion(completion)
                .dispatchMode(dispatch)
                .repeatable(cd.repeatable());
            if (cd.scopedBindings() != null) {
                cd.scopedBindings().forEach((name, participation) ->
                    builder.binding(name, participation));
            }
            plan.registerCompound(builder.build());
        }
    }

    static CompletionSemantics parseCompletionSemantics(String value) {
        if (value == null || value.equalsIgnoreCase("all")) return CompletionSemantics.all();
        if (value.equalsIgnoreCase("firstWins")) return CompletionSemantics.firstWins();
        try {
            int m = Integer.parseInt(value);
            return CompletionSemantics.mOfN(m);
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("Unrecognized completionSemantics: '" + value + "'");
        }
    }

    static DispatchMode parseDispatchMode(String value) {
        if (value == null) return DispatchMode.CHOREOGRAPHED;
        return DispatchMode.valueOf(value.toUpperCase());
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=CompoundBlackboardPlanConfigurerTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add schema/src/main/resources/schema/CaseDefinition.yaml api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java planning-core/src/main/java/io/casehub/engine/planning/control/CompoundBlackboardPlanConfigurer.java planning-core/src/test/java/io/casehub/engine/planning/control/CompoundBlackboardPlanConfigurerTest.java
git commit -m "fix(#1257): wire compound mofnRequired and add CompoundBlackboardPlanConfigurer

Fixes silent drop of mofnRequired during YAML conversion.
Bridges YAML compounds to runtime PlanItemDefinition.Compound.

Refs casehubio/engine#1257"
```

---

## Batch 3: Runtime Quorum — filter, evaluator, CasePlanModel extensions

### Task 5: CasePlanModel requires metadata and RequiresBlackboardPlanConfigurer

**Files:**
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/plan/CasePlanModel.java` — add requires metadata methods
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/plan/DefaultCasePlanModel.java` — implement requires metadata
- Create: `planning-core/src/main/java/io/casehub/engine/planning/control/RequiresBlackboardPlanConfigurer.java`
- Test: `planning-core/src/test/java/io/casehub/engine/planning/control/RequiresBlackboardPlanConfigurerTest.java`

**Interfaces:**
- Consumes: `BlackboardPlanConfigurer` SPI, `CasePlanModel` interface, `BindingRequires` record (Task 1)
- Produces: `CasePlanModel.registerRequiresMetadata()`, `CasePlanModel.getRequiresMetadata()`, `CasePlanModel.isRequiresThresholdTriggered()`, `CasePlanModel.markRequiresThresholdTriggered()`, `RequiresBlackboardPlanConfigurer` bean

- [ ] **Step 1: Write failing test**

```java
package io.casehub.engine.planning.control;

import io.casehub.api.model.Binding;
import io.casehub.api.model.BindingRequires;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.OnThresholdReached;
import io.casehub.engine.planning.plan.DefaultCasePlanModel;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class RequiresBlackboardPlanConfigurerTest {

    @Test
    void registersRequiresMetadataFromBindings() {
        var binding = Binding.builder()
            .name("aggregate")
            .requires(new BindingRequires(List.of("a", "b"), 1, OnThresholdReached.KEEP))
            .build();

        var def = CaseDefinition.builder()
            .namespace("test").name("test").version("1.0.0").dsl("0.1.0")
            .binding(binding)
            .build();

        var plan = new DefaultCasePlanModel();
        var configurer = new RequiresBlackboardPlanConfigurer();
        configurer.configure(plan, mockContext(def));

        var metadata = plan.getRequiresMetadata();
        assertEquals(1, metadata.size());
        assertNotNull(metadata.get("aggregate"));
        assertEquals(List.of("a", "b"), metadata.get("aggregate").bindings());
    }

    @Test
    void thresholdTriggeredFlag() {
        var plan = new DefaultCasePlanModel();
        assertFalse(plan.isRequiresThresholdTriggered("x"));
        plan.markRequiresThresholdTriggered("x");
        assertTrue(plan.isRequiresThresholdTriggered("x"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=RequiresBlackboardPlanConfigurerTest -q`
Expected: FAIL

- [ ] **Step 3: Add requires metadata methods to CasePlanModel interface**

Use `ide_insert_member` to add to `CasePlanModel.java`:

```java
void registerRequiresMetadata(String bindingName, BindingRequires requires);
Map<String, BindingRequires> getRequiresMetadata();
boolean isRequiresThresholdTriggered(String bindingName);
void markRequiresThresholdTriggered(String bindingName);
```

- [ ] **Step 4: Implement in DefaultCasePlanModel**

Use `ide_insert_member` to add fields and methods to `DefaultCasePlanModel.java`:

```java
private final Map<String, BindingRequires> requiresMetadata = new ConcurrentHashMap<>();
private final Set<String> requiresThresholdTriggered = ConcurrentHashMap.newKeySet();

@Override
public void registerRequiresMetadata(String bindingName, BindingRequires requires) {
    requiresMetadata.put(bindingName, requires);
}

@Override
public Map<String, BindingRequires> getRequiresMetadata() {
    return Collections.unmodifiableMap(requiresMetadata);
}

@Override
public boolean isRequiresThresholdTriggered(String bindingName) {
    return requiresThresholdTriggered.contains(bindingName);
}

@Override
public void markRequiresThresholdTriggered(String bindingName) {
    requiresThresholdTriggered.add(bindingName);
}
```

- [ ] **Step 5: Create RequiresBlackboardPlanConfigurer**

```java
package io.casehub.engine.planning.control;

import io.casehub.api.model.Binding;
import io.casehub.engine.planning.plan.CasePlanModel;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class RequiresBlackboardPlanConfigurer implements BlackboardPlanConfigurer {
    @Override
    public void configure(CasePlanModel plan, PlanExecutionContext context) {
        for (Binding b : context.definition().getBindings()) {
            if (b.getRequires() != null) {
                plan.registerRequiresMetadata(b.getName(), b.getRequires());
            }
        }
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=RequiresBlackboardPlanConfigurerTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add planning-core/src/main/java/io/casehub/engine/planning/plan/CasePlanModel.java planning-core/src/main/java/io/casehub/engine/planning/plan/DefaultCasePlanModel.java planning-core/src/main/java/io/casehub/engine/planning/control/RequiresBlackboardPlanConfigurer.java planning-core/src/test/java/io/casehub/engine/planning/control/RequiresBlackboardPlanConfigurerTest.java
git commit -m "feat(#1257): CasePlanModel requires metadata and RequiresBlackboardPlanConfigurer

Stores requires metadata at configuration time.
Provides threshold-triggered one-shot flag.

Refs casehubio/engine#1257"
```

### Task 6: Pre-strategy requires filter in PlanningStrategyLoopControl

**Files:**
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java:112` — add filterByRequires step
- Test: `planning-core/src/test/java/io/casehub/engine/planning/control/RequiresFilterTest.java`

**Interfaces:**
- Consumes: `CasePlanModel.findPlanItemByBindingName()`, `CasePlanModel.getRequiresMetadata()`, `TaskStatus.COMPLETED`, `BindingRequires` record
- Produces: `filterByRequires()` and `isRequiresSatisfied()` private methods in `PlanningStrategyLoopControl`

- [ ] **Step 1: Write failing test**

Test that a binding with an unsatisfied `requires` is filtered from the eligible set, and one with a satisfied `requires` passes through. Use the existing test patterns in `planning-core/src/test/` for `PlanningStrategyLoopControl`.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=RequiresFilterTest -q`
Expected: FAIL

- [ ] **Step 3: Add filterByRequires to PlanningStrategyLoopControl**

Use `ide_insert_member` to add private methods, and `ide_edit_member` to insert the filter call in `select()` at line 112:

```java
private List<Binding> filterByRequires(List<Binding> eligible, CasePlanModel plan) {
    return eligible.stream()
        .filter(b -> isRequiresSatisfied(b, plan))
        .toList();
}

private boolean isRequiresSatisfied(Binding binding, CasePlanModel plan) {
    if (binding.getRequires() == null) return true;
    BindingRequires req = binding.getRequires();
    long completed = req.bindings().stream()
        .map(plan::findPlanItemByBindingName)
        .filter(Optional::isPresent)
        .map(Optional::get)
        .filter(pi -> pi.getStatus() == TaskStatus.COMPLETED)
        .count();
    return completed >= req.atLeast();
}
```

In `select()`, after `gatedEligible` is computed, insert:
```java
List<Binding> requiresSatisfied = filterByRequires(gatedEligible, plan);
```
Then pass `requiresSatisfied` to the next step instead of `gatedEligible`.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -Dtest=RequiresFilterTest -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java planning-core/src/test/java/io/casehub/engine/planning/control/RequiresFilterTest.java
git commit -m "feat(#1257): add requires guard filter to PlanningStrategyLoopControl

Bindings with unsatisfied requires are held from dispatch.

Refs casehubio/engine#1257"
```

### Task 7: RequiresThresholdEvaluator — CANCEL and unreachable quorum

**Files:**
- Create: `planning-core/src/main/java/io/casehub/engine/planning/handler/RequiresThresholdEvaluator.java`
- Test: `planning-core/src/test/java/io/casehub/engine/planning/handler/RequiresThresholdEvaluatorTest.java`

**Interfaces:**
- Consumes: `BlackboardRegistry`, `CompoundCompletionEvaluator`, `Event<PlanItemStateChangedEvent>`, `CasePlanModel` requires metadata methods (Task 5), `PlanItem.markCancelled()`, `PlanItem.markFaulted()`, `TaskStatus`
- Produces: `RequiresThresholdEvaluator` CDI bean that handles CANCEL on threshold reached and FAULTED on unreachable quorum

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.engine.planning.handler;

import io.casehub.api.model.BindingRequires;
import io.casehub.api.model.OnThresholdReached;
import io.casehub.api.model.TaskStatus;
import io.casehub.engine.planning.plan.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class RequiresThresholdEvaluatorTest {

    @Test
    void cancelRemainingOnThresholdReached() {
        // Setup: aggregate requires 2 of [a, b, c] with CANCEL
        // a=COMPLETED, b=COMPLETED, c=RUNNING
        // Expect: c cancelled after threshold met
    }

    @Test
    void keepRemainingOnThresholdReached() {
        // Same setup but KEEP
        // Expect: c still RUNNING
    }

    @Test
    void faultOnUnreachableQuorum() {
        // Setup: aggregate requires 2 of [a, b, c]
        // a=COMPLETED, b=FAULTED, c=FAULTED
        // remaining=0, needed=1 → remaining < needed
        // Expect: aggregate PlanItem created and faulted
    }

    @Test
    void noFaultWhenQuorumStillReachable() {
        // a=COMPLETED, b=FAULTED, c=RUNNING
        // remaining=1, needed=1 → still reachable
        // Expect: no fault
    }

    @Test
    void thresholdTriggeredOnlyOnce() {
        // Verify one-shot flag prevents redundant cancellation
    }
}
```

Fill in test bodies using `DefaultCasePlanModel`, `PlanItem.create()`, and mock/stub the CDI events. Follow existing test patterns in `planning-core/src/test/`.

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement RequiresThresholdEvaluator**

Create the class as specified in the design doc's "Threshold reached handling" section. Key methods: `onPlanItemStateChanged()`, `evaluateThresholdReached()`, `cancelRemaining()`, `evaluateUnreachable()`, `countCompleted()`, `countNonCompletableTerminal()`.

- [ ] **Step 4: Run tests to verify they pass**

- [ ] **Step 5: Run full planning-core tests to check for regressions**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning-core -q`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add planning-core/src/main/java/io/casehub/engine/planning/handler/RequiresThresholdEvaluator.java planning-core/src/test/java/io/casehub/engine/planning/handler/RequiresThresholdEvaluatorTest.java
git commit -m "feat(#1257): RequiresThresholdEvaluator for CANCEL and unreachable quorum

Observes PlanItemStateChangedEvent. CANCEL cancels remaining
plan items on threshold. Unreachable quorum faults gated binding.

Refs casehubio/engine#1257"
```

---

## Batch 4: Surface Parity and Projection

### Task 8: Annotation support for @Requires on @Bind

**Files:**
- Modify: `annotations/runtime/src/main/java/io/casehub/engine/annotations/Bind.java` — add `name()` and `requires()` attributes
- Create: `annotations/runtime/src/main/java/io/casehub/engine/annotations/Requires.java` — new annotation
- Modify: `annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/BindingDescriptor.java` — add requires field
- Modify: `annotations/deployment/src/main/java/io/casehub/engine/annotations/deployment/EngineAnnotationsProcessor.java` — read @Requires
- Test: `annotations/deployment/src/test/java/io/casehub/engine/annotations/deployment/RequiresAnnotationTest.java`

**Interfaces:**
- Consumes: `@Bind` annotation, `BindingDescriptor` record, `EngineAnnotationsProcessor`, `BindingRequires` record (Task 1)
- Produces: `@Requires` annotation, `@Bind.name()`, `@Bind.requires()` attribute

- [ ] **Step 1: Write failing test for annotation processing**

Test that a class with `@Bind(name = "agg", requires = @Requires(bindings = {"a", "b"}, atLeast = 1))` produces a `BindingDescriptor` with a non-null `BindingRequires`.

- [ ] **Step 2: Create @Requires annotation**

```java
package io.casehub.engine.annotations;

import io.casehub.api.model.OnThresholdReached;
import java.lang.annotation.*;

@Target({})
@Retention(RetentionPolicy.RUNTIME)
public @interface Requires {
    String[] bindings();
    int atLeast() default 0;
    OnThresholdReached onThresholdReached() default OnThresholdReached.KEEP;
}
```

- [ ] **Step 3: Add name() and requires() to @Bind**

Use `ide_edit_member` on `Bind.java` to add:

```java
String name() default "";
Requires requires() default @Requires(bindings = {});
```

- [ ] **Step 4: Update BindingDescriptor and EngineAnnotationsProcessor**

Add `BindingRequires requires` field to `BindingDescriptor`. In `EngineAnnotationsProcessor`, read the `@Requires` from `@Bind` and construct `BindingRequires` when `bindings` is non-empty.

- [ ] **Step 5: Run test to verify it passes**

- [ ] **Step 6: Commit**

```bash
git add annotations/runtime/src/main/java/io/casehub/engine/annotations/Requires.java annotations/runtime/src/main/java/io/casehub/engine/annotations/Bind.java annotations/runtime/src/main/java/io/casehub/engine/annotations/runtime/BindingDescriptor.java annotations/deployment/src/main/java/io/casehub/engine/annotations/deployment/EngineAnnotationsProcessor.java annotations/deployment/src/test/java/io/casehub/engine/annotations/deployment/RequiresAnnotationTest.java
git commit -m "feat(#1257): add @Requires annotation support for @Bind

API surface parity: YAML, builder, and annotations all expose requires.

Refs casehubio/engine#1257"
```

### Task 9: CoordinationPipelineProjection quorum display

**Files:**
- Modify: `rest-core/src/main/java/io/casehub/engine/rest/service/CoordinationPipelineProjection.java` — populate required/of on PhaseItemView
- Test: `rest-core/src/test/java/io/casehub/engine/rest/service/CoordinationPipelineQuorumTest.java`

**Interfaces:**
- Consumes: `Binding.getRequires()`, `PhaseItemView` record (required/of fields), `CaseDefinition`
- Produces: `PhaseItemView` entries with `required` and `of` populated for bindings with `requires`

- [ ] **Step 1: Write failing test**

Test that a `CaseDefinition` with a binding that has `requires(bindings=[a,b,c], atLeast=2)` produces a `PhaseItemView` with `required=2` and `of=3`.

- [ ] **Step 2: Update CoordinationPipelineProjection**

In the method that creates `PhaseItemView` for bindings, check `binding.getRequires()` and populate `required` and `of` fields.

- [ ] **Step 3: Run test to verify it passes**

- [ ] **Step 4: Run full module tests**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl rest-core -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add rest-core/src/main/java/io/casehub/engine/rest/service/CoordinationPipelineProjection.java rest-core/src/test/java/io/casehub/engine/rest/service/CoordinationPipelineQuorumTest.java
git commit -m "feat(#1257): populate quorum required/of in CoordinationPipelineProjection

PhaseItemView now carries requires data for blocks-ui rendering.

Refs casehubio/engine#1257"
```

### Task 10: Integration test — end-to-end quorum coordination

**Files:**
- Test: `runtime/src/test/java/io/casehub/engine/planning/RequiresIntegrationTest.java`

**Interfaces:**
- Consumes: Full runtime stack — YAML parsing, planning loop, worker dispatch, context changes

- [ ] **Step 1: Write integration test**

YAML case definition with 3 extractors and 1 aggregator in a compound. `aggregate` requires `atLeast: 2` of the 3 extractors. Verify:
1. All 3 extractors dispatch on scope activation
2. Aggregator does NOT dispatch initially
3. Complete extractor-a → aggregator still held (1 < 2)
4. Complete extractor-b → aggregator dispatches (2 >= 2)

- [ ] **Step 2: Run test**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=RequiresIntegrationTest -q`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/planning/RequiresIntegrationTest.java
git commit -m "test(#1257): end-to-end integration test for requires quorum guard

Verifies full pipeline: YAML → planning → dispatch gating → quorum met.

Closes casehubio/engine#1257"
```

---

## References

- [2026-10-27-quorum-coordination-primitive-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/api/model/Binding.java:326] — validateCompensationBindings() precedent
- [planning-core/src/main/java/io/casehub/engine/planning/control/PlanningStrategyLoopControl.java:112] — requires filter insertion point
- [planning-core/src/main/java/io/casehub/engine/planning/plan/CasePlanModel.java:66] — findPlanItemByBindingName()
- [planning-core/src/main/java/io/casehub/engine/planning/plan/PlanItem.java:315] — markCancelled() CAS pattern
- [planning-core/src/main/java/io/casehub/engine/planning/control/BlackboardPlanConfigurer.java:27] — SPI interface
- [api/src/main/java/io/casehub/api/model/OnThresholdReached.java:18] — shared KEEP/CANCEL enum
- [annotations/runtime/src/main/java/io/casehub/engine/annotations/Bind.java:27] — @Bind annotation
- [rest-core/src/main/java/io/casehub/engine/rest/service/CoordinationPipelineProjection.java:36] — projection class
- [schema/src/main/resources/schema/CaseDefinition.yaml] — JSON Schema
- [schema/src/main/resources/schema/yaml-record-mappings.yaml] — codegen mappings
- [PP-20260915-aa504e] — api-surface-parity protocol
- [PP-20260727-5267d2] — plan-type-module-boundary protocol
- [PP-20260513-ecabd7] — subcase-coordination-strategy protocol
- [GitHub #1257] — focal issue
- [blocks-ui#258] — gallery demo data should come from real YAML
- [blocks-ui#243] — coordination diff view renders quorum changes
