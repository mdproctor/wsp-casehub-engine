# Binding Graph Projection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1047 — saga: extend CompensationGraphProjection with data flow edges for design-time viewer
**Issue group:** #1047, #1048

**Goal:** Replace the compensation-specific graph projection with a generalized binding graph that includes compensation, data flow, and trigger dependency edges.

**Architecture:** Add `requiredKeys` to `Binding` (symmetric counterpart to `producedKeys`). Replace `CompensationGraphProjection` and its types with `BindingGraphProjection` that computes three edge types: COMPENSATION (from `compensateRef`), DATA_FLOW (from `produces`/`consumes` channel matching), and TRIGGER_DEPENDENCY (from `producedKeys ∩ requiredKeys`). Update `CaseDefinitionType` GraphQL record to expose `bindingGraph` instead of `compensationGraph`.

**Tech Stack:** Java 21, Quarkus, SmallRye GraphQL, AssertJ

## Global Constraints

- All edits use IntelliJ MCP tools (`ide_insert_member`, `ide_replace_member`, `ide_edit_member`)
- `producedKeys` pattern is the template for `requiredKeys` — same field type, builder, YAML, converter
- Codegen regenerates `YamlBinding` from schema — run `mvn compile -pl schema,codegen,api` after schema changes
- Existing `CompensationTimeline` and `CompensationChain` queries are UNCHANGED — they are runtime queries

---

## Batch 1: Foundation — requiredKeys on Binding + generalized graph types

### Task 1: Add requiredKeys to Binding and YAML schema

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java` — add field, setter, getter, builder field + method, wire in build()
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `requiredKeys` property
- Modify: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` — add requiredKeys conversion
- Test: `api/src/test/java/io/casehub/api/model/BindingRequiredKeysTest.java`

**Interfaces:**
- Produces: `Binding.getRequiredKeys(): Set<String>`, `Binding.setRequiredKeys(Set<String>)`, `Binding.Builder.requiredKeys(Set<String>)`

- [ ] **Step 1: Write failing test for requiredKeys builder round-trip**

Create `api/src/test/java/io/casehub/api/model/BindingRequiredKeysTest.java`:

```java
package io.casehub.api.model;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.worker.api.Capability;
import java.util.Set;
import org.junit.jupiter.api.Test;

class BindingRequiredKeysTest {

    @Test
    void requiredKeys_roundTrip() {
        Binding binding = Binding.builder()
            .name("fraud-check")
            .capability(Capability.of("fraud.check", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .requiredKeys(Set.of("entityResolution", "transactionAmount"))
            .build();

        assertThat(binding.getRequiredKeys())
            .containsExactlyInAnyOrder("entityResolution", "transactionAmount");
    }

    @Test
    void requiredKeys_defaultsToNull() {
        Binding binding = Binding.builder()
            .name("simple")
            .capability(Capability.of("simple.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .build();

        assertThat(binding.getRequiredKeys()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=BindingRequiredKeysTest -DfailIfNoTests=false -q`
Expected: compilation failure — `requiredKeys` method doesn't exist on Builder

- [ ] **Step 3: Add requiredKeys to Binding class**

In `Binding.java`, use `ide_insert_member` to add:

Field (after `producedKeys` at line 64):
```java
  @com.fasterxml.jackson.annotation.JsonPropertyDescription(
      "Context keys this binding requires to fire meaningfully — symmetric to producedKeys.")
  private Set<String> requiredKeys;
```

Setter (after `setProducedKeys`):
```java
  public void setRequiredKeys(Set<String> requiredKeys) {
    this.requiredKeys = requiredKeys;
  }
```

Getter (after `getProducedKeys`):
```java
  public Set<String> getRequiredKeys() {
    return requiredKeys;
  }
```

Builder field (after `producedKeys` field in Builder):
```java
    private Set<String> requiredKeys;
```

Builder method (after `producedKeys` builder method):
```java
    public Builder requiredKeys(Set<String> requiredKeys) {
      this.requiredKeys = requiredKeys;
      return this;
    }
```

In `build()` method, add after `b.setProducedKeys(producedKeys);`:
```java
      b.setRequiredKeys(requiredKeys);
```

- [ ] **Step 4: Add requiredKeys to YAML schema**

In `schema/src/main/resources/schema/CaseDefinition.yaml`, add after the `producedKeys` block (after line 73):

```yaml
      requiredKeys:
        description: Context keys this binding requires to fire meaningfully — symmetric
          to producedKeys, used for trigger dependency edges.
        type: array
        items:
          type: string
```

- [ ] **Step 5: Regenerate YamlBinding and add converter logic**

Run: `mvn compile -pl schema,codegen,api -q`

This regenerates `YamlBinding` with a `requiredKeys()` accessor.

Then in `YamlCaseDefinitionConverter.java`, add after the `producedKeys` block (after line 786):

```java
      if (!yb.requiredKeys().isEmpty()) {
        builder.requiredKeys(new LinkedHashSet<>(yb.requiredKeys()));
      }
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=BindingRequiredKeysTest -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/Binding.java \
       api/src/test/java/io/casehub/api/model/BindingRequiredKeysTest.java \
       api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java \
       schema/src/main/resources/schema/CaseDefinition.yaml
git commit -m "feat(#1047): add requiredKeys to Binding — symmetric to producedKeys

Refs #1047"
```

### Task 2: Create generalized graph types and BindingGraphProjection

**Files:**
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/EdgeKind.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingNodeType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingEdgeType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingGraphType.java`
- Create: `graphql/src/main/java/io/casehub/engine/graphql/BindingGraphProjection.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationGraphType.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationNodeType.java`
- Delete: `graphql/src/main/java/io/casehub/engine/graphql/dto/CompensationEdgeType.java`
- Modify: `graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java`
- Delete: `graphql/src/test/java/io/casehub/engine/graphql/CompensationGraphProjectionTest.java`
- Create: `graphql/src/test/java/io/casehub/engine/graphql/BindingGraphProjectionTest.java`

**Interfaces:**
- Consumes: `Binding.getCompensateRef()`, `Binding.isCompensation()`, `Binding.getProduces()`, `Binding.getConsumes()`, `Binding.getProducedKeys()`, `Binding.getRequiredKeys()` (from Task 1), `Binding.target()`, `Binding.getName()`
- Produces: `BindingGraphProjection.project(List<Binding>): BindingGraphType`

- [ ] **Step 1: Write failing tests for BindingGraphProjection**

Create `graphql/src/test/java/io/casehub/engine/graphql/BindingGraphProjectionTest.java`:

```java
package io.casehub.engine.graphql;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.api.model.Binding;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.api.model.JudgmentTarget;
import io.casehub.engine.graphql.dto.BindingGraphType;
import io.casehub.engine.graphql.dto.EdgeKind;
import io.casehub.worker.api.Capability;
import java.util.List;
import java.util.Set;
import org.junit.jupiter.api.Test;

class BindingGraphProjectionTest {

    @Test
    void compensationEdges() {
        Binding forward = Binding.builder()
            .name("irb-review")
            .judgment(JudgmentTarget.builder().prompt("IRB Review").title("IRB Review").build())
            .on(new ContextChangeTrigger("$"))
            .compensateRef("irb-reversal")
            .build();
        Binding compensating = Binding.builder()
            .name("irb-reversal")
            .judgment(JudgmentTarget.builder().prompt("Reverse IRB").title("Reverse IRB").build())
            .on(new ContextChangeTrigger("$"))
            .compensation(true)
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(forward, compensating));

        assertThat(graph.nodes()).hasSize(2);
        assertThat(graph.edges()).hasSize(1);
        assertThat(graph.edges().get(0).edgeType()).isEqualTo(EdgeKind.COMPENSATION);
        assertThat(graph.edges().get(0).sourceBinding()).isEqualTo("irb-review");
        assertThat(graph.edges().get(0).targetBinding()).isEqualTo("irb-reversal");
        assertThat(graph.edges().get(0).label()).isNull();
        assertThat(graph.compensationGaps()).isEmpty();
    }

    @Test
    void compensationGaps() {
        Binding noCompensation = Binding.builder()
            .name("data-export")
            .capability(Capability.of("exportService.export", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(noCompensation));

        assertThat(graph.nodes()).hasSize(1);
        assertThat(graph.nodes().get(0).targetType()).isEqualTo("capability");
        assertThat(graph.edges()).isEmpty();
        assertThat(graph.compensationGaps()).containsExactly("data-export");
    }

    @Test
    void compensationOnlyBindingNotAGap() {
        Binding compensatingOnly = Binding.builder()
            .name("cleanup")
            .capability(Capability.of("cleanupService.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .compensation(true)
            .build();
        Binding forward = Binding.builder()
            .name("process")
            .capability(Capability.of("processService.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .compensateRef("cleanup")
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(compensatingOnly, forward));

        assertThat(graph.compensationGaps()).isEmpty();
        assertThat(graph.nodes()).extracting("compensationOnly").containsExactly(true, false);
    }

    @Test
    void dataFlowEdges() {
        Binding producer = Binding.builder()
            .name("ingestion")
            .capability(Capability.of("ingest.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .produces("orders-channel")
            .build();
        Binding consumer = Binding.builder()
            .name("processing")
            .capability(Capability.of("process.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .consumes("orders-channel")
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(producer, consumer));

        assertThat(graph.edges()).hasSize(1);
        assertThat(graph.edges().get(0).edgeType()).isEqualTo(EdgeKind.DATA_FLOW);
        assertThat(graph.edges().get(0).sourceBinding()).isEqualTo("ingestion");
        assertThat(graph.edges().get(0).targetBinding()).isEqualTo("processing");
        assertThat(graph.edges().get(0).label()).isEqualTo("orders-channel");
    }

    @Test
    void dataFlowNoMatchingConsumer() {
        Binding producer = Binding.builder()
            .name("orphan-producer")
            .capability(Capability.of("orphan.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .produces("nowhere")
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(producer));

        assertThat(graph.edges()).isEmpty();
    }

    @Test
    void triggerDependencyEdges() {
        Binding writer = Binding.builder()
            .name("entity-resolver")
            .capability(Capability.of("resolve.entity", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .producedKeys(Set.of("entityResolution", "confidence"))
            .build();
        Binding reader = Binding.builder()
            .name("fraud-check")
            .capability(Capability.of("fraud.check", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .requiredKeys(Set.of("entityResolution"))
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(writer, reader));

        var triggerEdges = graph.edges().stream()
            .filter(e -> e.edgeType() == EdgeKind.TRIGGER_DEPENDENCY)
            .toList();
        assertThat(triggerEdges).hasSize(1);
        assertThat(triggerEdges.get(0).sourceBinding()).isEqualTo("entity-resolver");
        assertThat(triggerEdges.get(0).targetBinding()).isEqualTo("fraud-check");
        assertThat(triggerEdges.get(0).label()).isEqualTo("entityResolution");
    }

    @Test
    void triggerDependencyNoSelfEdge() {
        Binding selfRef = Binding.builder()
            .name("self-ref")
            .capability(Capability.of("self.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .producedKeys(Set.of("key"))
            .requiredKeys(Set.of("key"))
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(selfRef));

        assertThat(graph.edges().stream()
            .filter(e -> e.edgeType() == EdgeKind.TRIGGER_DEPENDENCY))
            .isEmpty();
    }

    @Test
    void mixedEdgeTypes() {
        Binding a = Binding.builder()
            .name("step-a")
            .capability(Capability.of("a.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .produces("data-pipe")
            .producedKeys(Set.of("resultA"))
            .compensateRef("undo-a")
            .build();
        Binding b = Binding.builder()
            .name("step-b")
            .capability(Capability.of("b.run", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .consumes("data-pipe")
            .requiredKeys(Set.of("resultA"))
            .build();
        Binding undoA = Binding.builder()
            .name("undo-a")
            .capability(Capability.of("a.undo", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .compensation(true)
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(a, b, undoA));

        assertThat(graph.nodes()).hasSize(3);
        assertThat(graph.edges()).hasSize(3);
        assertThat(graph.edges().stream().map(e -> e.edgeType()))
            .containsExactlyInAnyOrder(
                EdgeKind.COMPENSATION, EdgeKind.DATA_FLOW, EdgeKind.TRIGGER_DEPENDENCY);
        assertThat(graph.compensationGaps()).containsExactly("step-b");
    }

    @Test
    void emptyBindings() {
        BindingGraphType graph = BindingGraphProjection.project(List.of());

        assertThat(graph.nodes()).isEmpty();
        assertThat(graph.edges()).isEmpty();
        assertThat(graph.compensationGaps()).isEmpty();
    }

    @Test
    void targetTypeMapping() {
        Binding cap = Binding.builder()
            .name("a")
            .capability(Capability.of("x", ".", "."))
            .on(new ContextChangeTrigger("$"))
            .build();
        Binding jt = Binding.builder()
            .name("b")
            .judgment(JudgmentTarget.builder().prompt("p").title("t").build())
            .on(new ContextChangeTrigger("$"))
            .build();

        BindingGraphType graph = BindingGraphProjection.project(List.of(cap, jt));

        assertThat(graph.nodes()).extracting("targetType")
            .containsExactly("capability", "judgment");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl graphql -Dtest=BindingGraphProjectionTest -DfailIfNoTests=false -q`
Expected: compilation failure — types don't exist yet

- [ ] **Step 3: Create EdgeKind enum**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/EdgeKind.java`:

```java
package io.casehub.engine.graphql.dto;

public enum EdgeKind {
    COMPENSATION,
    DATA_FLOW,
    TRIGGER_DEPENDENCY
}
```

- [ ] **Step 4: Create BindingNodeType record**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingNodeType.java`:

```java
package io.casehub.engine.graphql.dto;

import org.eclipse.microprofile.graphql.Type;

@Type("BindingNode")
public record BindingNodeType(String bindingName, String targetType, boolean compensationOnly) {}
```

- [ ] **Step 5: Create BindingEdgeType record**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingEdgeType.java`:

```java
package io.casehub.engine.graphql.dto;

import org.eclipse.microprofile.graphql.Type;

@Type("BindingEdge")
public record BindingEdgeType(
    String sourceBinding, String targetBinding, EdgeKind edgeType, String label) {}
```

- [ ] **Step 6: Create BindingGraphType record**

Create `graphql/src/main/java/io/casehub/engine/graphql/dto/BindingGraphType.java`:

```java
package io.casehub.engine.graphql.dto;

import java.util.List;
import org.eclipse.microprofile.graphql.Type;

@Type("BindingGraph")
public record BindingGraphType(
    List<BindingNodeType> nodes,
    List<BindingEdgeType> edges,
    List<String> compensationGaps) {}
```

- [ ] **Step 7: Create BindingGraphProjection**

Create `graphql/src/main/java/io/casehub/engine/graphql/BindingGraphProjection.java`:

```java
package io.casehub.engine.graphql;

import io.casehub.api.model.Binding;
import io.casehub.api.model.BindingTarget;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.ExtensionTarget;
import io.casehub.api.model.JudgmentTarget;
import io.casehub.api.model.SignalTarget;
import io.casehub.api.model.SubCaseTarget;
import io.casehub.engine.graphql.dto.BindingEdgeType;
import io.casehub.engine.graphql.dto.BindingGraphType;
import io.casehub.engine.graphql.dto.BindingNodeType;
import io.casehub.engine.graphql.dto.EdgeKind;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class BindingGraphProjection {

    private BindingGraphProjection() {}

    public static BindingGraphType project(List<Binding> bindings) {
        List<BindingNodeType> nodes = new ArrayList<>();
        List<BindingEdgeType> edges = new ArrayList<>();
        List<String> compensationGaps = new ArrayList<>();

        Map<String, String> channelProducers = new HashMap<>();
        Map<String, Set<String>> producedKeysIndex = new HashMap<>();

        for (Binding b : bindings) {
            nodes.add(new BindingNodeType(
                b.getName(), targetTypeName(b.target()), b.isCompensation()));

            if (b.getCompensateRef() != null) {
                edges.add(new BindingEdgeType(
                    b.getName(), b.getCompensateRef(), EdgeKind.COMPENSATION, null));
            } else if (!b.isCompensation()) {
                compensationGaps.add(b.getName());
            }

            if (b.getProduces() != null) {
                channelProducers.put(b.getProduces(), b.getName());
            }

            if (b.getProducedKeys() != null) {
                for (String key : b.getProducedKeys()) {
                    producedKeysIndex
                        .computeIfAbsent(key, k -> new LinkedHashSet<>())
                        .add(b.getName());
                }
            }
        }

        for (Binding b : bindings) {
            if (b.getConsumes() != null) {
                String producer = channelProducers.get(b.getConsumes());
                if (producer != null) {
                    edges.add(new BindingEdgeType(
                        producer, b.getName(), EdgeKind.DATA_FLOW, b.getConsumes()));
                }
            }

            if (b.getRequiredKeys() != null) {
                for (String requiredKey : b.getRequiredKeys()) {
                    Set<String> producers = producedKeysIndex.get(requiredKey);
                    if (producers != null) {
                        for (String producerName : producers) {
                            if (!producerName.equals(b.getName())) {
                                edges.add(new BindingEdgeType(
                                    producerName, b.getName(),
                                    EdgeKind.TRIGGER_DEPENDENCY, requiredKey));
                            }
                        }
                    }
                }
            }
        }

        return new BindingGraphType(nodes, edges, compensationGaps);
    }

    static String targetTypeName(BindingTarget target) {
        return switch (target) {
            case CapabilityTarget ignored -> "capability";
            case JudgmentTarget ignored -> "judgment";
            case SubCaseTarget ignored -> "sub-case";
            case SignalTarget ignored -> "signal";
            case ExtensionTarget ignored -> "extension";
        };
    }
}
```

- [ ] **Step 8: Delete old compensation types and projection**

Use `ide_refactor_safe_delete` for:
- `CompensationGraphProjection.java`
- `CompensationGraphType.java`
- `CompensationNodeType.java`
- `CompensationEdgeType.java`
- `CompensationGraphProjectionTest.java`

- [ ] **Step 9: Update CaseDefinitionType**

Replace the `CaseDefinitionType` record in `graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java`:

```java
package io.casehub.engine.graphql.dto;

import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.graphql.BindingGraphProjection;
import java.util.List;
import org.eclipse.microprofile.graphql.Type;

@Type("CaseDefinitionResponse")
public record CaseDefinitionType(
    String namespace,
    String name,
    String version,
    String title,
    String summary,
    List<String> capabilities,
    BindingGraphType bindingGraph) {

  public static CaseDefinitionType from(CaseDefinition def) {
    List<String> capNames =
        def.getCapabilities() != null
            ? def.getCapabilities().stream().map(c -> c.name()).toList()
            : List.of();
    BindingGraphType graph =
        def.getBindings() != null && !def.getBindings().isEmpty()
            ? BindingGraphProjection.project(def.getBindings())
            : null;
    return new CaseDefinitionType(
        def.getNamespace(),
        def.getName(),
        def.getVersion(),
        def.getTitle(),
        def.getSummary(),
        capNames,
        graph);
  }
}
```

- [ ] **Step 10: Run tests to verify all pass**

Run: `mvn test -pl graphql -Dtest=BindingGraphProjectionTest -q`
Expected: PASS (all 9 tests)

- [ ] **Step 11: Run full graphql module tests**

Run: `mvn test -pl graphql -q`
Expected: PASS — no remaining references to deleted types

- [ ] **Step 12: Commit**

```bash
git add graphql/
git commit -m "feat(#1047): generalize CompensationGraph to BindingGraph with typed edges

Replace CompensationGraphProjection with BindingGraphProjection supporting
three edge types: COMPENSATION, DATA_FLOW, TRIGGER_DEPENDENCY.
CaseDefinitionType.compensationGraph replaced by bindingGraph.

Refs #1047"
```

## Batch 2: Integration verification

### Task 3: Cross-module compilation and CLAUDE.md update

**Files:**
- Verify: full project compilation
- Modify: `CLAUDE.md` — document new types in graphql section (if section exists)

**Interfaces:**
- Consumes: all types from Tasks 1-2

- [ ] **Step 1: Verify full project compiles**

Run: `mvn compile -q`
Expected: BUILD SUCCESS — no cross-module reference breaks

- [ ] **Step 2: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api,graphql -q`
Expected: PASS

- [ ] **Step 3: Check for stale references to old types**

Use `ide_search_text` for `CompensationGraphProjection`, `CompensationGraphType`, `CompensationNodeType`, `CompensationEdgeType` across the project. Fix any remaining references.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore(#1047): verify cross-module compilation after binding graph generalization

Refs #1047 Closes #1047"
```

## References

- [2026-09-06-binding-graph-projection-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/api/model/Binding.java:64] — producedKeys field (pattern for requiredKeys)
- [graphql/src/main/java/io/casehub/engine/graphql/CompensationGraphProjection.java] — existing projection (replaced)
- [graphql/src/main/java/io/casehub/engine/graphql/dto/CaseDefinitionType.java] — GraphQL record (modified)
- [api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java:785] — producedKeys conversion pattern
- [schema/src/main/resources/schema/CaseDefinition.yaml:68] — producedKeys schema (pattern for requiredKeys)
- [schema/src/main/resources/schema/yaml-record-mappings.yaml:76] — Binding record mapping
- [GitHub #1047] — focal issue
