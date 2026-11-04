# yaml-core Record Pattern Adoption — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1015 — feat: adopt yaml-core record pattern — eliminate hand-coded deserializers
**Issue group:** #1015

**Goal:** Replace 2587 lines of hand-coded Jackson deserializers with plain records + thin converter, following the desiredstate pattern. Wire yaml-core VariableResolver and ForEachExpander.

**Architecture:** YAML records mirror the YAML shape exactly (auto-deserialized by Jackson). A single `YamlCaseDefinitionConverter` handles all domain transforms (Path.parse, Class.forName, expression baking, worker function construction, GOAP shorthand). Polymorphic deserializers (Trigger, ExpressionEvaluator, GoalExpression, CaseCompletion, AdaptationConfig, SubCaseMapping) stay as `@JsonDeserialize` annotations on record fields.

**Tech Stack:** Java 21 records, Jackson (existing), casehub-platform-yaml-core (new dependency)

## Global Constraints

- All 1345 existing api tests must continue to pass after every batch
- `CaseDefinition` (the domain model) does not change its public API — only the YAML→model path changes
- Polymorphic deserializers (6 files, 600 lines) are retained as-is
- `ExpressionEvaluatorDeserializer` stays module-registered (needs CDI-injected registry)
- Use `ide_insert_member` / `ide_replace_member` for structural edits, `ide_refactor_safe_delete` for deletions
- Build verification: `mvn install -DskipTests -q` then `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dcheckstyle.skip=true`

---

## Batch 1: YAML Record Hierarchy [foundation — records exist, nothing changes yet]

### Task 1: Supporting YAML Records

Create the small records that represent nested YAML blocks. Each record mirrors the YAML shape with null-safe compact constructors. Package: `io.casehub.api.model.converter.yaml`.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCapability.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlGoal.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlMilestone.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlSla.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlContextConstraint.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlWorkloadConstraint.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlConstraintEffect.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlMonitoringConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlReflectionTriggerConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlMemoryRetrievalConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlRecoveryPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlRecoveryOverride.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlPlanningConstraints.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlPortfolioConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlEpisodicMemoryConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlGoapAction.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCompound.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlSignalType.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlChannel.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlContextLayer.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCognitiveDemand.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlLabelRule.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlLabelAction.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlInboundMapping.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlExecutionPolicy.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlAgent.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlReact.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlA2A.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlMcp.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlAuth.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlAgentDescriptor.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlHumanTaskTarget.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlJudgmentTarget.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlSubCaseTarget.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlQuorumConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlIterationGroup.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/yaml/YamlRecordDeserializationTest.java`

**Interfaces:**
- Consumes: nothing (additive)
- Produces: record types consumed by `YamlCaseSpec`, `YamlBinding`, `YamlWorker` (Task 2)

- [ ] **Step 1: Derive record fields from existing deserializer code**

Read `CaseDefinitionDeserializer.java` lines 54–344 to identify every field parsed from `specNode`. Read `BindingDeserializer.java` for binding fields. Read `WorkerDeserializer.java` for worker fields. Each `if (node.has("fieldName"))` maps to a record component.

- [ ] **Step 2: Create supporting records**

Each record follows the desiredstate pattern — compact constructor with null-safe defaults:
```java
public record YamlCapability(
    String name,
    String inputProjection,
    String outputProjection,
    YamlCognitiveDemand cognitiveDemand) {
  public YamlCapability {
    // no null guards needed — Jackson passes null for absent fields
  }
}
```

For list/map fields, use null-safe defaults:
```java
public record YamlGoapAction(
    String name, String capability,
    Map<String, Boolean> preconditions,
    Map<String, Boolean> effects,
    Double cost, List<String> softDependency) {
  public YamlGoapAction {
    if (preconditions == null) preconditions = Map.of();
    if (effects == null) effects = Map.of();
    if (softDependency == null) softDependency = List.of();
  }
}
```

Polymorphic fields use `@JsonDeserialize` annotations:
```java
public record YamlGoal(
    String name, String description, String kind,
    @JsonDeserialize(using = ExpressionEvaluatorDeserializer.class)
    ExpressionEvaluator when) {}
```

- [ ] **Step 3: Write deserialization test**

Test that Jackson round-trips a YAML string to the record types:
```java
@Test
void deserializesCapability() throws Exception {
    var yaml = """
        name: analysis
        inputProjection: ".transaction"
        """;
    var cap = mapper.readValue(yaml, YamlCapability.class);
    assertThat(cap.name()).isEqualTo("analysis");
    assertThat(cap.inputProjection()).isEqualTo(".transaction");
}
```

One test per record group (capabilities, goals, milestones, constraints, worker blocks, binding targets).

- [ ] **Step 4: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="YamlRecordDeserializationTest" -Dcheckstyle.skip=true`

- [ ] **Step 5: Commit**

```
wip: add supporting YAML records (36 files) Refs #1015
```

### Task 2: Top-Level YAML Records

Create `YamlWorker`, `YamlBinding`, `YamlCaseSpec`, `YamlCaseDefinition` — the four records that replace the four hand-coded deserializers.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlWorker.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlBinding.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCaseSpec.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlCaseDefinition.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/yaml/YamlTopLevelDeserializationTest.java`

**Interfaces:**
- Consumes: all supporting records from Task 1
- Produces: `YamlCaseDefinition` — the entry point for converter (Task 3)

- [ ] **Step 1: Create YamlWorker record**

Fields from `WorkerDeserializer.java`: `name`, `description`, `definitionRef`, `capabilities` (List<String>), `contextType`, `outputType`, `sequence` (List<String>), `executionPolicy` (YamlExecutionPolicy), `forEach` (String — yaml-core expansion key).

Worker function blocks as nested records: `agent` (YamlAgent), `react` (YamlReact), `a2a` (YamlA2A), `mcp` (YamlMcp). The `do:` block stays as `@JsonProperty("do") JsonNode doBlock` (Serverless Workflow needs raw JSON).

GOAP shorthand fields: `cost` (Double), `effect` (Map<String, Boolean>), `softDependency` (List<String>).

Agent descriptor: `agentDescriptor` (YamlAgentDescriptor).

- [ ] **Step 2: Create YamlBinding record**

Fields from `BindingDeserializer.java`: `name`, `capability`, `on` (@JsonDeserialize Trigger), `when` (@JsonDeserialize ExpressionEvaluator), `inputProjectionOverride`, `outcomePolicy`, `conflictResolverStrategy`, `lifecycleScope`, `participation`, `executionMode`, `replanHint`, `replanAfter` (alias for replanHint), `exchangeProjection`, `produces`, `consumes`, `producedKeys` (List<String>), `contingency` (List<String>), `contextWrite` (Map), `signal` (Map), `permissionIntent`.

Binding targets as nullable nested records: `humanTask` (YamlHumanTaskTarget), `judgment` (YamlJudgmentTarget), `subCase` (YamlSubCaseTarget), `recoveryOverride` (YamlRecoveryOverride).

Handle `replanAfter` → `replanHint` alias via `@JsonAlias("replanAfter")` on the record component.

- [ ] **Step 3: Create YamlCaseSpec record**

All fields from `CaseDefinitionDeserializer.java` lines 185–340 that target `spec.setX()`. Reference the field survey (40+ fields). Use `@JsonDeserialize` on `completion`, `adaptationConfig`, `cbrConfig`.

- [ ] **Step 4: Create YamlCaseDefinition record**

```java
public record YamlCaseDefinition(
    String name, String namespace, String version,
    YamlCaseSpec spec,
    List<YamlWorker> workers,
    List<YamlBinding> bindings,
    Map<String, JsonNode> definitions,
    Map<String, YamlIterationGroup> iterations) {
  public YamlCaseDefinition {
    if (workers == null) workers = List.of();
    if (bindings == null) bindings = List.of();
    if (definitions == null) definitions = Map.of();
    if (iterations == null) iterations = Map.of();
  }
}
```

- [ ] **Step 5: Write end-to-end deserialization test**

Load a full YAML case definition (use an existing test YAML from `CaseDefinitionYamlMapperTest`) and deserialize to `YamlCaseDefinition`. Assert all fields populated correctly. This validates the record hierarchy works with Jackson without custom deserializers.

```java
@Test
void fullCaseDefinitionRoundTrip() throws Exception {
    var yaml = """
        name: test
        namespace: io.casehub.test
        version: "1.0"
        spec:
          capabilities:
            - name: analysis
          humanTaskRouting: constraint
          humanTaskContextConstraints:
            - when: ".priority == \\"high\\""
              effect:
                preferGroups: [seniors]
              weight: 0.8
        workers:
          - name: analyser
            capabilities: [analysis]
            agent:
              model: anthropic
        bindings:
          - name: trigger
            capability: analysis
            on:
              contextChange: {}
        """;
    ObjectMapper moduleMapper = new ObjectMapper(new YAMLFactory())
        .registerModule(new CaseDefinitionModule(JQ_ONLY))
        .disable(FAIL_ON_UNKNOWN_PROPERTIES);
    var yamlDef = moduleMapper.readValue(yaml, YamlCaseDefinition.class);

    assertThat(yamlDef.name()).isEqualTo("test");
    assertThat(yamlDef.spec().capabilities()).hasSize(1);
    assertThat(yamlDef.workers()).hasSize(1);
    assertThat(yamlDef.workers().get(0).agent()).isNotNull();
    assertThat(yamlDef.bindings()).hasSize(1);
    assertThat(yamlDef.bindings().get(0).on()).isNotNull();
}
```

- [ ] **Step 6: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="YamlTopLevelDeserializationTest" -Dcheckstyle.skip=true`

- [ ] **Step 7: Commit**

```
wip: add top-level YAML records (YamlCaseDefinition, YamlCaseSpec, YamlBinding, YamlWorker) Refs #1015
```

## Batch 2: Converter + Mapper Wiring [records produce CaseDefinition, tests pass]

### Task 3: YamlCaseDefinitionConverter

The single converter class that replaces both `CaseDefinitionDeserializer`'s domain transforms and all `CaseDefinitionPostProcessor` logic.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/YamlCaseDefinitionConverterTest.java`

**Interfaces:**
- Consumes: `YamlCaseDefinition` (from Task 2), `ExpressionEngineRegistry`, `WorkerFunctionProviderRegistry`
- Produces: `CaseDefinition` (the existing domain model — unchanged API)

- [ ] **Step 1: Write converter test for basic field mapping**

```java
@Test
void convertsBasicFields() {
    var yaml = new YamlCaseDefinition("test", "io.casehub", "1.0",
        new YamlCaseSpec(/* basic fields */), List.of(), List.of(),
        Map.of(), Map.of());
    var def = YamlCaseDefinitionConverter.convert(yaml, JQ_ONLY, EMPTY_PROVIDERS);
    assertThat(def.getName()).isEqualTo("test");
    assertThat(def.getNamespace()).isEqualTo("io.casehub");
}
```

- [ ] **Step 2: Implement converter skeleton**

```java
public final class YamlCaseDefinitionConverter {
  private YamlCaseDefinitionConverter() {}

  public static CaseDefinition convert(
      YamlCaseDefinition yaml,
      ExpressionEngineRegistry registry,
      WorkerFunctionProviderRegistry providers) {
    var def = CaseDefinition.builder()
        .name(yaml.name())
        .namespace(yaml.namespace())
        .version(yaml.version())
        .build();
    if (yaml.spec() != null) convertSpec(yaml.spec(), def, registry);
    convertWorkers(yaml.workers(), def, providers, registry);
    convertBindings(yaml.bindings(), def, registry);
    if (!yaml.definitions().isEmpty()) def.setDefinitions(yaml.definitions());
    return def;
  }
  // private methods for each transform category...
}
```

- [ ] **Step 3: Add domain transform tests and implementations incrementally**

For each transform category (reference the spec's table), write a failing test then implement:

1. **types/labels** → `Path.parse()`
2. **signals** → `Class.forName()` → `SignalType<?>`
3. **capabilities** → `Capability` + `CapabilityTarget` with JQ input/output projection baking
4. **worker functions** → provider discovery (try providers first, then agent block, then contextType/outputType typed sync, then NONE)
5. **GOAP shorthand** → per-worker `cost`/`effect`/`softDependency` → `GoapAction` list
6. **agent descriptors** → `YamlAgentDescriptor` → `AgentDescriptor` with personality, goals
7. **compounds** → `YamlCompound` → `CompoundDeclaration`
8. **label rules** → `YamlLabelRule` → `LabelRule` with `CompiledExpression` from JQ
9. **context constraints** → `YamlContextConstraint` → `ContextConstraint` with expression baking
10. **authorization** → string keys → `AclAction` enum
11. **inbound mappings** → `YamlInboundMapping` → `InboundSignalMapping`
12. **binding targets** → humanTask/judgment/subCase/signal dispatch
13. **contextType bridge** → `contextType` on spec → `contextBridgeType` + `expressionLang` side-effect

Port logic from `CaseDefinitionDeserializer` private methods and `CaseDefinitionPostProcessor` methods. The code exists — it moves from deserializer/postprocessor to converter.

- [ ] **Step 4: Run full test suite to verify converter produces correct output**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="YamlCaseDefinitionConverterTest" -Dcheckstyle.skip=true`

- [ ] **Step 5: Commit**

```
feat: YamlCaseDefinitionConverter — all domain transforms Refs #1015
```

### Task 4: Wire Converter into CaseDefinitionYamlMapper

Switch the mapper's `load()` methods to use the records+converter path. Both old and new paths must produce identical output — verify by running all 1345 tests.

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java`

**Interfaces:**
- Consumes: `YamlCaseDefinition` (Task 2), `YamlCaseDefinitionConverter` (Task 3)
- Produces: `CaseDefinition` via the new path (same public API)

- [ ] **Step 1: Update CaseDefinitionYamlMapper.load() to use records path**

Replace the existing `convertValue(processedNode, CaseDefinition.class)` call with:
```java
YamlCaseDefinition yaml = moduleMapper.convertValue(processedNode, YamlCaseDefinition.class);
CaseDefinition def = YamlCaseDefinitionConverter.convert(yaml, registry, providerRegistry);
```

Remove the `CaseDefinitionPostProcessor` call — the converter now handles everything.

- [ ] **Step 2: Simplify CaseDefinitionModule**

Remove registrations for `CaseDefinition.class`, `Worker.class`, `Binding.class`, `CbrConfig.class` (now handled by records + `@JsonDeserialize`). Keep only `ExpressionEvaluator.class` (needs registry injection).

Remove mixin registrations for `CaseDefinitionSpec` and `CaseDefinition` (no longer deserialized to these types).

Keep `GoapActionMixin` if GOAP actions are still deserialized directly (check if converter handles them instead).

- [ ] **Step 3: Run ALL api tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dcheckstyle.skip=true`

This is the critical gate — all 1345 tests must pass. If any fail, the records or converter have a field mapping gap. Fix before proceeding.

- [ ] **Step 4: Commit**

```
feat: wire records+converter path in CaseDefinitionYamlMapper Refs #1015
```

## Batch 3: Delete Old Code [cleanup — ~2587 lines removed]

### Task 5: Delete Replaced Files

Remove all files replaced by the records+converter.

**Files:**
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionDeserializer.java` (793 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/BindingDeserializer.java` (670 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/WorkerDeserializer.java` (97 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/CbrConfigDeserializer.java` (77 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java` (472 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/CaseDefinitionSpec.java` (408 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionSpecMixin.java` (41 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionMixin.java` (29 lines) — use `ide_refactor_safe_delete`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/GoapActionMixin.java` — use `ide_refactor_safe_delete` (if converter handles GOAP action deserialization)
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — inline `CaseDefinitionSpec` fields, remove `spec` field and `getSpec()` method
- Delete: `api/src/test/java/io/casehub/api/model/converter/deser/PropertyMappingTest.java` — use `ide_refactor_safe_delete` (tests mixin registration that no longer exists)

**Interfaces:**
- Consumes: Task 4 (mapper already using records path — these files are dead code)
- Produces: clean codebase with no dead deserializers

- [ ] **Step 1: Inline CaseDefinitionSpec fields into CaseDefinition**

`CaseDefinition` has `private final CaseDefinitionSpec spec` and delegates getters to it. Inline all spec fields as direct fields on `CaseDefinition`. Update all `getSpec().getX()` callers (found via `ide_find_references` on `getSpec()`).

- [ ] **Step 2: Delete replaced files**

Use `ide_refactor_safe_delete` for each file. If safe delete reports usages, they are in the files being deleted (circular references within the old pipeline) — force delete.

- [ ] **Step 3: Delete PropertyMappingTest**

This test validates mixin registration for `CaseDefinitionSpec` which no longer exists.

- [ ] **Step 4: Run ALL api tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dcheckstyle.skip=true`

All tests must pass. Fix compilation errors from deleted references.

- [ ] **Step 5: Run full project build**

Run: `mvn install -DskipTests -q` to verify no downstream modules break.

Then: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -Dcheckstyle.skip=true` for the full project.

- [ ] **Step 6: Commit**

```
feat: delete 2587 lines of hand-coded deserializers — records+converter replaces them Refs #1015

Deleted: CaseDefinitionDeserializer (793), BindingDeserializer (670),
WorkerDeserializer (97), CbrConfigDeserializer (77),
CaseDefinitionPostProcessor (472), CaseDefinitionSpec (408),
CaseDefinitionSpecMixin (41), CaseDefinitionMixin (29).

Inlined CaseDefinitionSpec fields into CaseDefinition.
```

## Batch 4: yaml-core Integration [new capability — variable resolution + template expansion]

### Task 6: VariableResolver Integration

Add yaml-core dependency and wire `VariableResolver` as a pre-processing pass in the YAML loading pipeline.

**Files:**
- Modify: `api/pom.xml` — add `casehub-platform-yaml-core` dependency
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — add variable resolution step
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperVariableTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.resolver.VariableResolver`, `io.casehub.yaml.core.resolver.VariableSource`
- Produces: resolved YAML with `${env.X}` / `${config.X}` substituted before Jackson deserialization

- [ ] **Step 1: Add yaml-core dependency to api/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-yaml-core</artifactId>
</dependency>
```

Add version property to root `pom.xml` if not already present via platform parent.

- [ ] **Step 2: Write test for variable resolution**

```java
@Test
void resolvesEnvironmentVariables() throws Exception {
    var yaml = """
        name: ${env.APP_NAME}
        namespace: io.casehub.test
        version: "1.0"
        spec:
          capabilities:
            - name: analysis
        """;
    // Set env via VariableSource
    var def = loadWithVariables(yaml, Map.of("APP_NAME", "resolved-app"));
    assertThat(def.getName()).isEqualTo("resolved-app");
}
```

- [ ] **Step 3: Wire VariableResolver into CaseDefinitionYamlMapper**

Add a new `load()` overload that accepts `VariableResolver`, or wire it into the existing load with an optional resolver parameter.

The resolution pass runs BEFORE Jackson deserialization:
```java
// Convert JsonNode to Map for VariableResolver
Object rawMap = objectMapper.convertValue(rawNode, Object.class);
Object resolved = resolver.resolve(rawMap);
JsonNode resolvedNode = objectMapper.valueToTree(resolved);
// Then proceed with Jackson deserialization of resolvedNode
```

- [ ] **Step 4: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CaseDefinitionYamlMapperVariableTest" -Dcheckstyle.skip=true`

- [ ] **Step 5: Commit**

```
feat: wire yaml-core VariableResolver into YAML loading pipeline Refs #1015
```

### Task 7: ForEachExpander Integration

Wire `ForEachExpander` for template expansion of workers and bindings. Add schema support.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlWorkerForEachAdapter.java`
- Create: `api/src/main/java/io/casehub/api/model/converter/yaml/YamlBindingForEachAdapter.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — add forEach expansion step
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` — add `iterations:`, `forEach:`, `when:` to Worker and Binding
- Modify: `schema/src/main/resources/ts/CaseDefinition.ts` — add corresponding TS types
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperForEachTest.java`

**Interfaces:**
- Consumes: `io.casehub.yaml.core.foreach.ForEachExpander`, `io.casehub.yaml.core.foreach.ForEachAdapter`
- Produces: expanded workers/bindings list (template `processor-${each.region}` → `processor-eu`, `processor-us`, `processor-ap`)

- [ ] **Step 1: Write ForEach expansion test**

```java
@Test
void expandsWorkerForEach() throws Exception {
    var yaml = """
        name: test
        namespace: io.casehub.test
        version: "1.0"
        iterations:
          regions:
            in: [eu, us, ap]
            as: region
        spec:
          capabilities:
            - name: process
        workers:
          - name: processor-${each.region}
            forEach: regions
            capabilities: [process]
        bindings:
          - name: trigger
            capability: process
            on:
              contextChange: {}
        """;
    var def = CaseDefinitionYamlMapper.load(new ByteArrayInputStream(yaml.getBytes()));
    assertThat(def.getWorkers()).hasSize(3);
    assertThat(def.getWorkers().stream().map(Worker::name).toList())
        .containsExactly("processor-eu", "processor-us", "processor-ap");
}
```

- [ ] **Step 2: Implement ForEachAdapter for workers**

```java
public class YamlWorkerForEachAdapter implements ForEachAdapter<JsonNode> {
    @Override public Object getForEach(JsonNode element) {
        return element.has("forEach") ? element.get("forEach").asText() : null;
    }
    @Override public String getWhen(JsonNode element) {
        return element.has("when") ? element.get("when").asText() : null;
    }
    @Override public JsonNode stamp(JsonNode element, String stampedId,
                                     VariableResolver resolver) {
        // Deep-resolve ${each.*} in all string values
        Object map = new ObjectMapper().convertValue(element, Object.class);
        Object resolved = resolver.resolve(map);
        ObjectNode result = (ObjectNode) new ObjectMapper().valueToTree(resolved);
        result.remove("forEach");  // consumed — not a worker field
        return result;
    }
}
```

- [ ] **Step 3: Wire ForEach expansion into mapper**

After VariableResolver, before Jackson deserialization:
```java
// Extract iterations block
Map<String, YamlIterationGroup> iterations = parseIterations(resolvedNode);

// Expand workers
JsonNode expandedWorkers = expandForEach(
    resolvedNode.get("workers"), iterations, resolver,
    new YamlWorkerForEachAdapter());

// Expand bindings
JsonNode expandedBindings = expandForEach(
    resolvedNode.get("bindings"), iterations, resolver,
    new YamlBindingForEachAdapter());

// Replace in resolved node before Jackson
```

- [ ] **Step 4: Add schema properties**

Add to Worker in `CaseDefinition.yaml`:
```yaml
forEach:
  type: string
  description: "Iteration group reference for template expansion."
when:
  $ref: "#/$defs/ExpressionOrOverride"
  description: "Conditional inclusion — evaluated during forEach expansion."
```

Add top-level `iterations:` block:
```yaml
iterations:
  type: object
  additionalProperties:
    type: object
    properties:
      in:
        type: array
        items:
          type: string
      as:
        type: string
    required: [in, as]
```

Also add `definitionRef` to Worker properties (fixes the schema gap identified earlier):
```yaml
definitionRef:
  type: string
  description: "Cross-file reference to an external definition."
```

- [ ] **Step 5: Run tests, verify pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -Dtest="CaseDefinitionYamlMapperForEachTest" -Dcheckstyle.skip=true`

Then run full api test suite to ensure no regressions.

- [ ] **Step 6: Commit**

```
feat: wire yaml-core ForEachExpander — template expansion for workers and bindings Refs #1015

Adds iterations: block to YAML schema. Workers and bindings support
forEach: and when: for template expansion. Also adds definitionRef
to Worker schema properties.
```

---

## References

- [2026-08-31-yaml-core-adoption-design.md] — design spec this plan implements
- [CaseDefinitionDeserializer.java:54-793] — primary file being replaced
- [BindingDeserializer.java:53-670] — secondary file being replaced
- [CaseDefinitionPostProcessor.java:49-472] — post-processor being replaced
- [CaseDefinitionSpec.java:42-408] — intermediate model being deleted
- [casehub-platform/yaml-core/] — VariableResolver, ForEachExpander source
- [casehub-desiredstate/yaml/runtime/] — reference pattern (zero deserializers)
- [GitHub #1015] — tracking issue
- [GitHub #978] — parent epic (YAML DSL completeness)
