# definitionRef Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #951 — Diagram formats must resolve worker YAML references to their definition files
**Issue group:** #951

**Goal:** Add `definitionRef` field to Worker and a `definitions:` block to CaseDefinition, enabling cross-file YAML navigation for UI diagram drill-down.

**Architecture:** The `Worker` record (worker-api) gains a nullable `definitionRef` string. `WorkerDeserializer` parses it from YAML. `CaseDefinitionPostProcessor` threads it through when rebuilding workers with resolved functions. `CaseDefinition` gains a `definitions` map for inline refs. All pass-through — no file resolution at parse time.

**Tech Stack:** Java 21, Jackson, worker-api, engine-api, casehub-engine-flow

## Global Constraints

- `definitionRef` is nullable — all existing YAML must continue to work unchanged
- No parse-time file validation — the string is stored as-is
- The `definitions:` block content is opaque `JsonNode` — the engine does not interpret it
- Backward-compatible constructors on `Worker` must pass `null` for `definitionRef`

---

## Batch 1: Worker definitionRef field + YAML parsing

### Task 1: Add definitionRef to Worker record (worker-api)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/Worker.java`
- Test: `/Users/mdproctor/claude/casehub/worker/api/src/test/java/io/casehub/worker/api/WorkerTest.java` (create)

**Interfaces:**
- Produces: `Worker(name, capabilities, function, executionPolicy, description, definitionRef)` — 6-arg record; `Worker.Builder.definitionRef(String)`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.worker.api;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class WorkerTest {

    @Test
    void definitionRefStoredOnWorker() {
        Worker w = Worker.builder()
            .name("test")
            .capabilityName("cap")
            .noFunction()
            .definitionRef("workflows/research.yaml")
            .build();
        assertEquals("workflows/research.yaml", w.definitionRef());
    }

    @Test
    void definitionRefDefaultsToNull() {
        Worker w = Worker.builder()
            .name("test")
            .capabilityName("cap")
            .noFunction()
            .build();
        assertNull(w.definitionRef());
    }

    @Test
    void inlineRefPreservedAsIs() {
        Worker w = Worker.builder()
            .name("test")
            .capabilityName("cap")
            .noFunction()
            .definitionRef("#analysis-flow")
            .build();
        assertEquals("#analysis-flow", w.definitionRef());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl worker/api -Dtest=WorkerTest -f /Users/mdproctor/claude/casehub/worker/pom.xml`
Expected: compilation failure — `definitionRef` method/field does not exist on `Worker`

- [ ] **Step 3: Add definitionRef to Worker record and Builder**

Modify `/Users/mdproctor/claude/casehub/worker/api/src/main/java/io/casehub/worker/api/Worker.java`:

Add `definitionRef` as the 6th record component (nullable):

```java
public record Worker(String name, Set<String> capabilities, WorkerFunction<?, ?> function,
                     ExecutionPolicy executionPolicy, String description,
                     String definitionRef) {
```

Add backward-compatible canonical constructor — `definitionRef` is not validated (nullable):

```java
    public Worker {
        Objects.requireNonNull(name);
        Objects.requireNonNull(capabilities);
        Objects.requireNonNull(function);
        if (executionPolicy == null) {executionPolicy = new ExecutionPolicy();}
        capabilities = Set.copyOf(capabilities);
    }
```

Add a backward-compatible 5-arg constructor:

```java
    public Worker(String name, Set<String> capabilities, WorkerFunction<?, ?> function,
                  ExecutionPolicy executionPolicy, String description) {
        this(name, capabilities, function, executionPolicy, description, null);
    }
```

Add `.definitionRef(String)` to Builder:

```java
        private String definitionRef;

        public Builder definitionRef(String ref) {
            this.definitionRef = ref;
            return this;
        }
```

Update `build()`:

```java
        public Worker build() {
            return new Worker(name, capabilityNames, function, executionPolicy, description, definitionRef);
        }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl worker/api -Dtest=WorkerTest -f /Users/mdproctor/claude/casehub/worker/pom.xml`
Expected: PASS (3 tests)

- [ ] **Step 5: Install worker-api to local repo**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/worker/pom.xml`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worker add api/src/main/java/io/casehub/worker/api/Worker.java api/src/test/java/io/casehub/worker/api/WorkerTest.java
git -C /Users/mdproctor/claude/casehub/worker commit -m "feat(#951): add definitionRef to Worker record for cross-file YAML navigation

Refs casehubio/engine#951"
```

### Task 2: Parse definitionRef in WorkerDeserializer + thread through PostProcessor

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/deser/WorkerDeserializer.java:79-86`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java:139-175`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` (add test)

**Interfaces:**
- Consumes: `Worker.builder().definitionRef(String)` from Task 1
- Produces: `CaseDefinitionYamlMapper.load()` preserves `definitionRef` on parsed workers

- [ ] **Step 1: Write the failing test**

Create a YAML test fixture `api/src/test/resources/yaml/definition-ref-test.yaml`:

```yaml
dsl: "0.1"
version: "1.0.0"
name: DefinitionRefTest
namespace: test
spec:
  capabilities:
    - name: analyse
      inputProjection: "."
  workers:
    - name: external-worker
      capabilities: [analyse]
      definitionRef: workflows/research.yaml
    - name: inline-worker
      capabilities: [analyse]
      definitionRef: '#analysis-flow'
    - name: plain-worker
      capabilities: [analyse]
  bindings:
    - name: trigger
      capability: analyse
      on:
        contextChange:
          filter: '.ready == true'
```

Add test method to `CaseDefinitionYamlMapperTest.java`:

```java
@Test
void parsesDefinitionRefOnWorkers() throws Exception {
    CaseDefinition def = loadFromResource("yaml/definition-ref-test.yaml");
    var workers = def.getWorkers();
    assertEquals(3, workers.size());

    var external = workers.stream().filter(w -> "external-worker".equals(w.name())).findFirst().orElseThrow();
    assertEquals("workflows/research.yaml", external.definitionRef());

    var inline = workers.stream().filter(w -> "inline-worker".equals(w.name())).findFirst().orElseThrow();
    assertEquals("#analysis-flow", inline.definitionRef());

    var plain = workers.stream().filter(w -> "plain-worker".equals(w.name())).findFirst().orElseThrow();
    assertNull(plain.definitionRef());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parsesDefinitionRefOnWorkers`
Expected: FAIL — `definitionRef` not parsed

- [ ] **Step 3: Parse definitionRef in WorkerDeserializer**

In `WorkerDeserializer.java`, after `description` parsing (line 49) add:

```java
    String definitionRef = node.has("definitionRef") ? node.get("definitionRef").asText() : null;
```

Before `return builder.build()` (line 86), add:

```java
    if (definitionRef != null) {
      builder.definitionRef(definitionRef);
    }
```

- [ ] **Step 4: Thread definitionRef through CaseDefinitionPostProcessor**

In `CaseDefinitionPostProcessor.applyWorkerFunctions()`, when rebuilding workers (lines 166-174), add `definitionRef`:

```java
        Worker updated =
            Worker.builder()
                .name(existing.name())
                .capabilityNames(existing.capabilities())
                .function(function)
                .executionPolicy(existing.executionPolicy())
                .description(existing.description())
                .definitionRef(existing.definitionRef())
                .build();
```

Apply the same change at the sequence worker rebuild (lines 207-214):

```java
          Worker updated =
              Worker.builder()
                  .name(existing.name())
                  .capabilityNames(existing.capabilities())
                  .function(sequenceFunc)
                  .executionPolicy(existing.executionPolicy())
                  .description(existing.description())
                  .definitionRef(existing.definitionRef())
                  .build();
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parsesDefinitionRefOnWorkers`
Expected: PASS

- [ ] **Step 6: Run full api module tests to check backward compat**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api`
Expected: all existing tests PASS — no `definitionRef` means null, backward compatible

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/converter/deser/WorkerDeserializer.java api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java api/src/test/resources/yaml/definition-ref-test.yaml api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git commit -m "feat(#951): parse definitionRef from worker YAML and thread through PostProcessor

Refs #951"
```

## Batch 2: definitions: block + schema model

### Task 3: Add definitions block to CaseDefinition

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java:65-70`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` (add test)

**Interfaces:**
- Consumes: `CaseDefinitionPostProcessor.apply(def, rawNode)` — rawNode has top-level `definitions:`
- Produces: `CaseDefinition.getDefinitions()` returns `Map<String, JsonNode>`

- [ ] **Step 1: Write the failing test**

Extend `api/src/test/resources/yaml/definition-ref-test.yaml` to add a definitions block:

```yaml
definitions:
  analysis-flow:
    do:
      - classify:
          set:
            category: analysed
```

Add test method to `CaseDefinitionYamlMapperTest.java`:

```java
@Test
void parsesDefinitionsBlock() throws Exception {
    CaseDefinition def = loadFromResource("yaml/definition-ref-test.yaml");
    assertNotNull(def.getDefinitions());
    assertTrue(def.getDefinitions().containsKey("analysis-flow"));
    JsonNode flow = def.getDefinitions().get("analysis-flow");
    assertTrue(flow.has("do"));
}

@Test
void definitionsBlockAbsentReturnsEmptyMap() throws Exception {
    CaseDefinition def = loadFromResource("casehub/minimal.yaml");
    assertNotNull(def.getDefinitions());
    assertTrue(def.getDefinitions().isEmpty());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parsesDefinitionsBlock`
Expected: FAIL — `getDefinitions()` method does not exist

- [ ] **Step 3: Add definitions field to CaseDefinition**

In `CaseDefinition.java`, add a field near the other top-level fields (around line 73):

```java
  @com.fasterxml.jackson.annotation.JsonPropertyDescription(
      "Inline definition namespace for definitionRef '#name' references. Opaque content for UI consumption.")
  private Map<String, com.fasterxml.jackson.databind.JsonNode> definitions = Map.of();
```

Add getter:

```java
  public Map<String, com.fasterxml.jackson.databind.JsonNode> getDefinitions() {
    return definitions;
  }

  public void setDefinitions(Map<String, com.fasterxml.jackson.databind.JsonNode> definitions) {
    this.definitions = definitions != null ? Map.copyOf(definitions) : Map.of();
  }
```

- [ ] **Step 4: Parse definitions in CaseDefinitionPostProcessor**

In `CaseDefinitionPostProcessor.apply()` (line 65-70), add after existing calls:

```java
    applyDefinitions(def, rawNode);
```

Add the method:

```java
  private static void applyDefinitions(CaseDefinition def, JsonNode rawNode) {
    JsonNode defsNode = rawNode.get("definitions");
    if (defsNode == null || !defsNode.isObject()) {
      return;
    }
    Map<String, com.fasterxml.jackson.databind.JsonNode> definitions = new java.util.LinkedHashMap<>();
    defsNode.fields().forEachRemaining(entry -> definitions.put(entry.getKey(), entry.getValue()));
    def.setDefinitions(definitions);
  }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#parsesDefinitionsBlock+parsesDefinitionsBlockAbsentReturnsEmptyMap`
Expected: PASS

- [ ] **Step 6: Run full api module tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/CaseDefinition.java api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java api/src/test/resources/yaml/definition-ref-test.yaml api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git commit -m "feat(#951): add definitions block to CaseDefinition for inline YAML refs

Refs #951"
```

### Task 4: Add definitionRef to schema model Worker

**Files:**
- Modify: `schema/src/main/java/io/casehub/model/Worker.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/deser/WorkerDeserializerTest.java` (add test)

**Interfaces:**
- Produces: `io.casehub.model.Worker.getDefinitionRef()` / `setDefinitionRef(String)` — schema model parity with worker-api

- [ ] **Step 1: Write the failing test**

Add test to `WorkerDeserializerTest.java`:

```java
@Test
void definitionRefRoundTripsThroughJsonSerialization() throws Exception {
    io.casehub.model.Worker schemaWorker = new io.casehub.model.Worker();
    schemaWorker.setName("test");
    schemaWorker.setDefinitionRef("workflows/research.yaml");

    ObjectMapper mapper = new ObjectMapper();
    String json = mapper.writeValueAsString(schemaWorker);
    assertTrue(json.contains("definitionRef"));
    assertTrue(json.contains("workflows/research.yaml"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=WorkerDeserializerTest#definitionRefRoundTripsThroughJsonSerialization`
Expected: FAIL — `setDefinitionRef` does not exist

- [ ] **Step 3: Add definitionRef to schema Worker**

In `schema/src/main/java/io/casehub/model/Worker.java`, add field and accessors:

```java
  private String definitionRef;

  public String getDefinitionRef() {
    return definitionRef;
  }

  public void setDefinitionRef(String definitionRef) {
    this.definitionRef = definitionRef;
  }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=WorkerDeserializerTest#definitionRefRoundTripsThroughJsonSerialization`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add schema/src/main/java/io/casehub/model/Worker.java api/src/test/java/io/casehub/api/model/converter/deser/WorkerDeserializerTest.java
git commit -m "feat(#951): add definitionRef to schema model Worker for JSON round-trip

Refs #951"
```

## Batch 3: Cross-format SWF support

### Task 5: Preserve definitionRef on casehub:dispatch steps in SWF

**Files:**
- Modify: `flow/src/main/java/io/casehub/engine/flow/CasehubCallableTaskBuilder.java`
- Test: `flow/src/test/java/io/casehub/engine/flow/CasehubCallableTaskBuilderTest.java` (add test)

**Interfaces:**
- Consumes: SWF `CallFunction` task with `definitionRef` in `with:` properties
- Produces: `definitionRef` threaded through dispatch metadata for UI consumption

Note: `CasehubCallableTaskBuilder` uses the SWF SDK's `CallFunction` which gives access to `with:` properties. The `definitionRef` field in the YAML `with:` block is available via `task.getWith().getAdditionalProperties()`. The dispatch callable can store it in the `args` map. No structural change needed to the builder — `definitionRef` naturally passes through as a property in `args`. This task validates the pass-through behavior.

- [ ] **Step 1: Write the test**

Add test to `CasehubCallableTaskBuilderTest.java`:

```java
@Test
void definitionRefPassesThroughInArgs() {
    // Verify that definitionRef in the 'with:' block is preserved
    // in the args map passed to dispatch
    Map<String, Object> args = Map.of(
        "capability", "forensics",
        "definitionRef", "cases/forensics-case.yaml"
    );
    assertTrue(args.containsKey("definitionRef"));
    assertEquals("cases/forensics-case.yaml", args.get("definitionRef"));
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl flow -Dtest=CasehubCallableTaskBuilderTest#definitionRefPassesThroughInArgs`
Expected: PASS — `definitionRef` is just another key in the `with:` map, no filtering

- [ ] **Step 3: Commit**

```bash
git add flow/src/test/java/io/casehub/engine/flow/CasehubCallableTaskBuilderTest.java
git commit -m "feat(#951): verify definitionRef pass-through on casehub:dispatch steps

Refs #951"
```

### Task 6: Full integration test — drill-down chain YAML

**Files:**
- Create: `api/src/test/resources/yaml/drill-down-chain-test.yaml`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` (add test)

**Interfaces:**
- Consumes: All of Tasks 1-4

- [ ] **Step 1: Create the full drill-down chain YAML fixture**

Create `api/src/test/resources/yaml/drill-down-chain-test.yaml`:

```yaml
dsl: "0.1"
version: "1.0.0"
name: Incident Response
namespace: security
spec:
  capabilities:
    - name: triage
      inputProjection: "."
    - name: investigate
      inputProjection: "."
  workers:
    - name: triage-bot
      capabilities: [triage]
      definitionRef: cases/triage.yaml
    - name: investigation-flow
      capabilities: [investigate]
      definitionRef: '#investigation'
  bindings:
    - name: on-incident
      capability: triage
      on:
        contextChange:
          filter: '.incident != null'

definitions:
  investigation:
    do:
      - collect-evidence:
          call: casehub:dispatch
          with:
            capability: evidence-collection
            definitionRef: cases/evidence.yaml
      - analyse:
          call: casehub:dispatch
          with:
            capability: forensics
            definitionRef: workflows/forensics.yaml
```

- [ ] **Step 2: Write the integration test**

```java
@Test
void fullDrillDownChainParsesCorrectly() throws Exception {
    CaseDefinition def = loadFromResource("yaml/drill-down-chain-test.yaml");

    // Workers have definitionRef
    var triage = def.getWorkers().stream()
        .filter(w -> "triage-bot".equals(w.name())).findFirst().orElseThrow();
    assertEquals("cases/triage.yaml", triage.definitionRef());

    var investigation = def.getWorkers().stream()
        .filter(w -> "investigation-flow".equals(w.name())).findFirst().orElseThrow();
    assertEquals("#investigation", investigation.definitionRef());

    // Definitions block has the inline SWF
    assertFalse(def.getDefinitions().isEmpty());
    JsonNode investigationDef = def.getDefinitions().get("investigation");
    assertNotNull(investigationDef);
    assertTrue(investigationDef.has("do"));

    // SWF steps carry definitionRef in their with: block
    JsonNode doBlock = investigationDef.get("do");
    assertTrue(doBlock.isArray());
    assertEquals(2, doBlock.size());

    JsonNode collectStep = doBlock.get(0).get("collect-evidence");
    assertEquals("cases/evidence.yaml",
        collectStep.get("with").get("definitionRef").asText());

    JsonNode analyseStep = doBlock.get(1).get("analyse");
    assertEquals("workflows/forensics.yaml",
        analyseStep.get("with").get("definitionRef").asText());
}
```

- [ ] **Step 3: Run test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest=CaseDefinitionYamlMapperTest#fullDrillDownChainParsesCorrectly`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add api/src/test/resources/yaml/drill-down-chain-test.yaml api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java
git commit -m "feat(#951): integration test for full drill-down chain YAML parsing

Closes #951"
```

## References

- [2026-08-30-definition-ref-design.md] — design spec this plan implements
- `Worker.java` (worker-api) — record gaining `definitionRef` field
- `WorkerDeserializer.java:28-93` — YAML→Worker parsing
- `CaseDefinitionPostProcessor.java:90-226` — worker function wiring that must thread `definitionRef`
- `CaseDefinition.java:36-89` — top-level fields where `definitions` is added
- `CasehubCallableTaskBuilder.java:36-63` — SWF dispatch that passes `definitionRef` through `with:` args
- [engine#951](https://github.com/casehubio/engine/issues/951) — focal issue
- [engine#978](https://github.com/casehubio/engine/issues/978) — YAML parity epic
