# YAML Deserialization Simplification — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1006 — audit: simplify CaseDefinition YAML deserialization — retire manual mapper, streamline CaseDefinitionModule

**Goal:** Delete the 1800-line bridge layer (`convertToApiModel` + generated POJO dependency), wire the existing `CaseDefinitionModule` into production, and reduce the mapper to a thin facade over Jackson + a post-processor for runtime concerns.

**Architecture:** The current load path is YAML → generated POJOs (67 classes) → manual `convertToApiModel()` (1100 lines) → API domain types. The `CaseDefinitionModule` already deserializes directly to API types via 8 custom deserializers + 3 mixins, but is only used in tests. We complete the module path (add missing spec-level fields, post-processor for runtime wiring), rewire `load()` to use it, and delete the bridge.

**Tech Stack:** Jackson 2.x (ObjectMapper, SimpleModule, StdDeserializer, mixins), Quarkus (CDI)

## Global Constraints

- Pre-release platform — no backward compatibility constraints
- Jackson annotations stay externalized (mixins + custom module, never on domain types)
- D5 alignment: `Capability.inputProjection`/`outputProjection` (renamed upstream in worker-api, engine must catch up)
- All existing `CaseDefinitionYamlMapper*Test` and `*DeserializerTest` classes must pass after rewire

---

## Batch 1: Foundation — D5 alignment + complete module path

### Task 1: Fix D5 alignment — Capability field rename

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CapabilityTarget.java:32-37`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (lines 446, 460, 470 — `cap.inputSchema()` → `cap.inputProjection()`)
- Modify: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionDeserializer.java:148-149`
- Rename: any other `Capability.inputSchema()`/`outputSchema()` call sites across the engine (use `ide_find_references`)
- Test: existing `CaseDefinitionModuleIntegrationTest` line 93 (`ct.capability().inputSchema()`)

**Interfaces:**
- Consumes: `io.casehub.worker.api.Capability` (record: `inputProjection`, `outputProjection` — D5 rename already applied in worker-api source)
- Produces: all engine code compiles against the renamed Capability accessors

- [ ] **Step 1: Install updated worker-api**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/worker/pom.xml
```

- [ ] **Step 2: Find all `inputSchema()` / `outputSchema()` references on Capability**

Use `ide_find_references` on `io.casehub.worker.api.Capability#inputSchema` and `outputSchema` to get the full list of call sites.

- [ ] **Step 3: Fix CapabilityTarget.java**

Change `capability.inputSchema()` → `capability.inputProjection()` and `capability.outputSchema()` → `capability.outputProjection()` (4 references on lines 32-37).

- [ ] **Step 4: Fix all other call sites**

Update every reference found in Step 2. Use `ide_edit_member` or `ide_replace_member` for each file.

- [ ] **Step 5: Fix CaseDefinitionModuleIntegrationTest**

Line 93: `ct.capability().inputSchema()` → `ct.capability().inputProjection()`

- [ ] **Step 6: Verify compilation**

```bash
ide_build_project
```

Expected: `CapabilityTarget.java` errors resolved. Remaining errors are all in `convertToApiModel` (generated POJO references) — those die in Batch 2.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "fix: D5 alignment — Capability.inputProjection/outputProjection rename Refs #1006"
```

### Task 2: Add missing spec-level field handling to CaseDefinitionDeserializer

The current `CaseDefinitionDeserializer` handles ~12 field groups. The manual mapper handles ~20 more. This task adds the missing fields so the module path covers everything the bridge does.

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionDeserializer.java` — add field handlers
- Modify: `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionSpecMixin.java` — add `defaultQuorum` → `quorum` mapping
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/AdaptationConfigDeserializer.java` — handles string preset + object dual form
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — register AdaptationConfigDeserializer
- Test: `api/src/test/java/io/casehub/api/model/converter/deser/CaseDefinitionModuleIntegrationTest.java` — add tests for every new field

**Interfaces:**
- Consumes: existing `CaseDefinitionSpec` setters (all exist, verified in audit)
- Produces: `CaseDefinitionModule` can deserialize ALL spec-level fields that the bridge handles

- [ ] **Step 1: Write failing test — CBR config deserialization**

Add to `CaseDefinitionModuleIntegrationTest`:
```java
@Test
void cbrConfig_deserializes() throws Exception {
    String yaml = """
        namespace: test
        name: cbr
        version: "1.0.0"
        spec:
          capabilities:
            - name: cap
          workers:
            - name: w
              capabilities: [cap]
          bindings:
            - name: b
              capability: cap
              on:
                contextChange: {}
          cbr:
            domain: test-domain
            topK: 5
            minSimilarity: 0.7
        """;
    CaseDefinition result = yamlMapper.readValue(yaml, CaseDefinition.class);
    assertNotNull(result.getCbrConfig());
    assertEquals("test-domain", result.getCbrConfig().domain());
    assertEquals(5, result.getCbrConfig().topK());
    assertEquals(0.7, result.getCbrConfig().minSimilarity());
}
```

- [ ] **Step 2: Write failing test — adaptation config object form**

```java
@Test
void adaptationConfig_objectForm_deserializes() throws Exception {
    String yaml = """
        namespace: test
        name: adapt
        version: "1.0.0"
        spec:
          capabilities:
            - name: cap
          workers:
            - name: w
              capabilities: [cap]
          bindings:
            - name: b
              capability: cap
              on:
                contextChange: {}
          adaptation:
            trigger: every-step
            optimization: forward-replan
            threshold: 0.5
        """;
    CaseDefinition result = yamlMapper.readValue(yaml, CaseDefinition.class);
    assertNotNull(result.getAdaptationConfig());
    assertEquals("every-step", result.getAdaptationConfig().trigger());
    assertEquals("forward-replan", result.getAdaptationConfig().optimization());
    assertEquals(0.5, result.getAdaptationConfig().threshold());
}
```

- [ ] **Step 3: Write failing test — adaptation config preset form**

```java
@Test
void adaptationConfig_presetForm_deserializes() throws Exception {
    String yaml = """
        namespace: test
        name: adapt-preset
        version: "1.0.0"
        spec:
          capabilities:
            - name: cap
          workers:
            - name: w
              capabilities: [cap]
          bindings:
            - name: b
              capability: cap
              on:
                contextChange: {}
          adaptation: adaptive
        """;
    CaseDefinition result = yamlMapper.readValue(yaml, CaseDefinition.class);
    assertNotNull(result.getAdaptationConfig());
    assertEquals("every-step", result.getAdaptationConfig().trigger());
    assertEquals("forward-replan", result.getAdaptationConfig().optimization());
}
```

- [ ] **Step 4: Write failing test — recovery, monitoring, quorum, reflection, memoryRetrieval**

```java
@Test
void configRecords_deserialize() throws Exception {
    String yaml = """
        namespace: test
        name: configs
        version: "1.0.0"
        spec:
          capabilities:
            - name: cap
          workers:
            - name: w
              capabilities: [cap]
          bindings:
            - name: b
              capability: cap
              on:
                contextChange: {}
          recoveryPolicy:
            maxRetries: 5
            maxRerouteAttempts: 2
            classifierId: heuristic
            revisionStrategyId: forward-replan
            replanStrategyId: llm
            enabled: true
          monitoring:
            enabled: true
            perCompletionThreshold: 0.4
            windowSize: 3
          quorum:
            instances: 3
            required: 2
          reflection:
            enabled: true
            importanceThreshold: 4.0
            maxUnreflectedOutcomes: 8
            maxSourceMemories: 40
          memoryRetrieval:
            enabled: true
            maxMemories: 15
        """;
    CaseDefinition result = yamlMapper.readValue(yaml, CaseDefinition.class);
    assertNotNull(result.getRecoveryPolicy());
    assertEquals(5, result.getRecoveryPolicy().maxRetries());
    assertNotNull(result.getMonitoringConfig());
    assertEquals(0.4, result.getMonitoringConfig().perCompletionThreshold());
    assertNotNull(result.getDefaultQuorum());
    assertEquals(3, result.getDefaultQuorum().instances());
    assertNotNull(result.getReflectionTrigger());
    assertEquals(4.0, result.getReflectionTrigger().importanceThreshold());
    assertNotNull(result.getMemoryRetrieval());
    assertEquals(15, result.getMemoryRetrieval().maxMemories());
}
```

- [ ] **Step 5: Run all four tests to confirm they fail**

```bash
mvn test -pl api -Dtest="CaseDefinitionModuleIntegrationTest" -DfailIfNoTests=false -q 2>&1 | tail -20
```

- [ ] **Step 6: Add `defaultQuorum` → `quorum` mapping to CaseDefinitionSpecMixin**

```java
@JsonProperty("quorum")
abstract QuorumConfig getDefaultQuorum();
```

- [ ] **Step 7: Create AdaptationConfigDeserializer**

Handles string preset form → `AdaptationConfig.of(...)` and object form → standard record deserialization.

```java
public class AdaptationConfigDeserializer extends StdDeserializer<AdaptationConfig> {
    public AdaptationConfigDeserializer() { super(AdaptationConfig.class); }

    @Override
    public AdaptationConfig deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        JsonNode node = p.readValueAsTree();
        if (node.isTextual()) {
            return switch (node.asText()) {
                case "adaptive" -> AdaptationConfig.of("every-step", "forward-replan");
                case "conservative" -> AdaptationConfig.of("on-failure", "forward-replan");
                case "progress" -> new AdaptationConfig("progress", "forward-replan",
                    AdaptationConfig.DEFAULT_PROGRESS_THRESHOLD, null, null);
                case "off" -> null;
                default -> throw ctxt.weirdStringException(node.asText(),
                    AdaptationConfig.class, "Unknown adaptation preset: " + node.asText());
            };
        }
        if (node.isObject()) {
            String trigger = node.has("trigger") ? node.get("trigger").asText() : "every-step";
            String optimization = node.has("optimization") ? node.get("optimization").asText()
                : node.has("revision") ? node.get("revision").asText() : "forward-replan";
            Double threshold = node.has("threshold") ? node.get("threshold").asDouble() : null;
            String metaReasoner = node.has("metaReasoner") ? node.get("metaReasoner").asText() : null;
            String repair = node.has("repair") ? node.get("repair").asText() : null;
            Double contingencyThreshold = node.has("contingencyThreshold")
                ? node.get("contingencyThreshold").asDouble() : null;
            return new AdaptationConfig(trigger, optimization, threshold, metaReasoner, repair, contingencyThreshold);
        }
        throw ctxt.weirdStringException(node.toString(), AdaptationConfig.class,
            "adaptation must be a string preset or object");
    }
}
```

- [ ] **Step 8: Register AdaptationConfigDeserializer in CaseDefinitionModule**

```java
addDeserializer(AdaptationConfig.class, new AdaptationConfigDeserializer());
```

- [ ] **Step 9: Add remaining spec-level field handling to CaseDefinitionDeserializer**

For each field group, add reading logic in the `deserialize()` method after the existing field handlers. Most are simple delegation to `readValue` or direct setter calls:

```java
// In CaseDefinitionDeserializer.deserialize(), after existing handling:

// Config records — Jackson handles these via registered deserializers + mixins
CaseDefinitionSpec spec = def.getSpec();
if (specNode.has("recoveryPolicy")) {
    spec.setRecoveryPolicy(readValue(specNode.get("recoveryPolicy"), RecoveryPolicy.class, codec, ctxt));
}
if (specNode.has("monitoring")) {
    spec.setMonitoringConfig(readValue(specNode.get("monitoring"), MonitoringConfig.class, codec, ctxt));
}
if (specNode.has("cbr")) {
    spec.setCbrConfig(readValue(specNode.get("cbr"), CbrConfig.class, codec, ctxt));
}
if (specNode.has("quorum")) {
    spec.setDefaultQuorum(readValue(specNode.get("quorum"), QuorumConfig.class, codec, ctxt));
}
if (specNode.has("reflection")) {
    spec.setReflectionTrigger(readValue(specNode.get("reflection"), ReflectionTriggerConfig.class, codec, ctxt));
}
if (specNode.has("memoryRetrieval")) {
    spec.setMemoryRetrieval(readValue(specNode.get("memoryRetrieval"), MemoryRetrievalConfig.class, codec, ctxt));
}
if (specNode.has("adaptation")) {
    spec.setAdaptationConfig(readValue(specNode.get("adaptation"), AdaptationConfig.class, codec, ctxt));
}
if (specNode.has("planningConstraints")) {
    spec.setPlanningConstraints(readValue(specNode.get("planningConstraints"), PlanningConstraints.class, codec, ctxt));
}
if (specNode.has("portfolioConfig")) {
    spec.setPortfolioConfig(readValue(specNode.get("portfolioConfig"), PortfolioConfig.class, codec, ctxt));
}

// Simple maps and scalars
if (specNode.has("routingSignalWeights")) {
    Map<String, Double> weights = new LinkedHashMap<>();
    specNode.get("routingSignalWeights").fields()
        .forEachRemaining(e -> weights.put(e.getKey(), e.getValue().asDouble()));
    spec.setRoutingSignalWeights(weights);
}
if (specNode.has("workerServiceAccountIds")) {
    Map<String, String> ids = new LinkedHashMap<>();
    specNode.get("workerServiceAccountIds").fields()
        .forEachRemaining(e -> ids.put(e.getKey(), e.getValue().asText()));
    spec.setWorkerServiceAccountIds(Map.copyOf(ids));
}
if (specNode.has("goalToEffectKeys")) {
    Map<String, Set<String>> gtek = new LinkedHashMap<>();
    specNode.get("goalToEffectKeys").fields().forEachRemaining(e -> {
        Set<String> keys = new LinkedHashSet<>();
        e.getValue().forEach(v -> keys.add(v.asText()));
        gtek.put(e.getKey(), Set.copyOf(keys));
    });
    spec.setGoalToEffectKeys(Map.copyOf(gtek));
}
if (specNode.has("authorization")) {
    Map<AclAction, List<String>> auth = new EnumMap<>(AclAction.class);
    specNode.get("authorization").fields().forEachRemaining(e -> {
        AclAction action = AclAction.valueOf(e.getKey().toUpperCase(Locale.ROOT));
        List<String> groups = new ArrayList<>();
        e.getValue().forEach(g -> groups.add(g.asText()));
        auth.put(action, List.copyOf(groups));
    });
    spec.setAuthorization(Map.copyOf(auth));
}

// Compound declarations
if (specNode.has("compounds") && specNode.get("compounds").isArray()) {
    for (JsonNode cn : specNode.get("compounds")) {
        // ... parse CompoundDeclaration (entry/exit conditions, scoped bindings, etc.)
    }
}

// GOAP actions (spec-level)
if (specNode.has("goapActions") && specNode.get("goapActions").isArray()) {
    // ... parse GoapAction list
}
```

See the full method body in `convertToApiModel` for reference — each block is a direct port, removing the `io.casehub.model.*` intermediary.

- [ ] **Step 10: Handle CaseDefinition-level fields (not under spec)**

In `CaseDefinitionDeserializer.deserialize()`, after identity fields:

```java
// Use (secrets, configMaps)
if (root.has("use")) {
    def.setUse(readValue(root.get("use"), Use.class, codec, ctxt));
}

// Types and labels
if (root.has("types") && root.get("types").isArray()) {
    Set<Path> types = new LinkedHashSet<>();
    root.get("types").forEach(n -> types.add(Path.parse(n.asText())));
    def.setTypes(types);
}
if (root.has("labels") && root.get("labels").isArray()) {
    Set<Path> labels = new LinkedHashSet<>();
    root.get("labels").forEach(n -> labels.add(Path.parse(n.asText())));
    def.setLabels(labels);
}

// Episodic memory — mixin maps "episodic" → episodicMemoryConfig
if (root.has("episodic")) {
    def.setEpisodicMemoryConfig(readValue(root.get("episodic"), EpisodicMemoryConfig.class, codec, ctxt));
}

// Layers — mixin maps "layers" → layerNames
if (root.has("layers") && root.get("layers").isArray()) {
    List<String> names = new ArrayList<>();
    root.get("layers").forEach(n -> names.add(n.asText()));
    def.setLayerNames(names);
}
```

- [ ] **Step 11: Run all integration tests to confirm they pass**

```bash
mvn test -pl api -Dtest="CaseDefinitionModuleIntegrationTest" -DfailIfNoTests=false -q
```

- [ ] **Step 12: Commit**

```bash
git add -A && git commit -m "feat: complete CaseDefinitionDeserializer for all spec-level fields Refs #1006"
```

### Task 3: Create CaseDefinitionPostProcessor for runtime concerns

Post-processing handles concerns that can't be expressed as Jackson deserialization: worker function wiring, Class.forName resolution, GOAP shorthand, adaptation presets (already handled by deserializer), and agent descriptors.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java`
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionPostProcessorTest.java`

**Interfaces:**
- Consumes: `CaseDefinition` (deserialized via module), `JsonNode rawNode` (for raw access to agent blocks, GOAP shorthand), `ExpressionEngineRegistry`, `WorkerFunctionProviderRegistry`
- Produces: `CaseDefinition` with runtime fields populated (worker functions, agent descriptors, typed signals, channels, label rules, inbound mappings, cognitive demands, GOAP shorthand)

- [ ] **Step 1: Write failing test — worker function wiring (agent)**

```java
@Test
void postProcessor_wiresAgentFunction() throws Exception {
    String yaml = """
        namespace: test
        name: agent-test
        version: "1.0.0"
        spec:
          capabilities:
            - name: analyse
          workers:
            - name: analyst
              capabilities: [analyse]
              agent:
                model: ollama
                modelName: test-model
                systemPrompt: "You are a test agent"
          bindings:
            - name: b
              capability: analyse
              on:
                contextChange: {}
        """;
    ObjectMapper mapper = new ObjectMapper(new YAMLFactory())
        .registerModule(new CaseDefinitionModule(null));
    JsonNode rawNode = mapper.readTree(yaml);
    CaseDefinition def = mapper.readValue(yaml, CaseDefinition.class);
    CaseDefinitionPostProcessor.process(def, rawNode, null, null, node -> null);
    assertNotNull(def.getWorkers().get(0).function());
    assertNotEquals(WorkerFunction.NONE, def.getWorkers().get(0).function());
}
```

- [ ] **Step 2: Write failing test — GOAP shorthand on workers**

- [ ] **Step 3: Write failing test — typed signals (Class.forName)**

- [ ] **Step 4: Write failing test — label rules (CompiledExpression)**

- [ ] **Step 5: Write failing test — agent descriptors**

- [ ] **Step 6: Run tests to confirm they fail**

- [ ] **Step 7: Implement CaseDefinitionPostProcessor**

Extract from `convertToApiModel`:
- Worker function construction (agent, contextType, sequence resolution, discovery)
- GOAP shorthand (cost/effect/softDependency on workers)
- Agent descriptor parsing
- Typed signals (`Class.forName`)
- Label rules (JQ compilation to `CompiledExpression`)
- Inbound mappings
- Channel declarations (`Class.forName` for recordType)
- Cognitive demands (per-capability extraction)
- contextType / defaultWorkerBridge / expressionLang resolution

Static method: `CaseDefinitionPostProcessor.process(CaseDefinition def, JsonNode rawNode, ObjectMapper objectMapper, ExpressionEngineRegistry registry, WorkerFunctionProviderRegistry providerRegistry)`

- [ ] **Step 8: Run tests to confirm they pass**

- [ ] **Step 9: Commit**

```bash
git add -A && git commit -m "feat: create CaseDefinitionPostProcessor for runtime concerns Refs #1006"
```

---

## Batch 2: Rewire and clean up

### Task 4: Rewire load() to module path, delete bridge code

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — rewrite `load()` to use `CaseDefinitionModule` + `PostProcessor`, delete `convertToApiModel()` and all `convert*()` helper methods
- Modify: `api/src/main/java/io/casehub/api/engine/YamlCaseHub.java` — update `getDefinition()` to use new load path
- Delete: all `io.casehub.model.*` imports from the mapper
- Test: all existing `CaseDefinitionYamlMapper*Test` classes must pass

**Interfaces:**
- Consumes: `CaseDefinitionModule`, `CaseDefinitionPostProcessor`
- Produces: `CaseDefinitionYamlMapper.load()` that goes YAML → Jackson (with module) → PostProcessor → CaseDefinition. No generated POJOs.

- [ ] **Step 1: Rewrite `load(InputStream, ...)` to use module path**

```java
public static CaseDefinition load(InputStream yamlStream, ObjectMapper objectMapper,
    ExpressionEngineRegistry registry, WorkerFunctionProviderRegistry providerRegistry) throws IOException {
    byte[] bytes = yamlStream.readAllBytes();
    ObjectMapper moduleMapper = objectMapper.copy()
        .registerModule(new CaseDefinitionModule(registry))
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    JsonNode rawNode = moduleMapper.readTree(bytes);
    CaseDefinition def = moduleMapper.readValue(bytes, CaseDefinition.class);
    CaseDefinitionPostProcessor.process(def, rawNode, objectMapper, registry, providerRegistry);
    return def;
}
```

- [ ] **Step 2: Rewrite `load(JsonNode, ...)` similarly**

- [ ] **Step 3: Rewrite `load(InputStream)` (simple overload)**

- [ ] **Step 4: Delete `convertToApiModel()` and all helper methods**

Delete: `convertToApiModel`, `convertBinding`, `convertSubCase`, `convertTrigger`, `convertGoalExpression`, `resolveGoalKind`, `parseGoalExpressionFromNode`, `parseGoalElement`, `convertHumanTask`, `parseCandidateSet`, `convertExecutionPolicy`, `convertAgentDescriptor`, `applyExchangeFields`, `convertCompletionStrategy`, `flattenExpressionOverrides`, `createLenientMapper`, `TypedMvelRegistry` inner class.

- [ ] **Step 5: Delete `resolveExpression` if no longer needed, or keep if PostProcessor uses it**

Check if `CaseDefinitionPostProcessor` references `resolveExpression`. If so, keep it (or move it to PostProcessor). If not, delete.

- [ ] **Step 6: Remove all `io.casehub.model.*` imports**

- [ ] **Step 7: Remove `JQ_ONLY` and `EMPTY_PROVIDERS` static fields if no longer needed**

- [ ] **Step 8: Run ALL mapper tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q 2>&1 | tail -30
```

Expected: all tests pass. If any fail, the CaseDefinitionDeserializer or PostProcessor is missing a field — fix and re-run.

- [ ] **Step 9: Run full build**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q
```

- [ ] **Step 10: Commit**

```bash
git add -A && git commit -m "feat: rewire load() to CaseDefinitionModule, delete 1800-line bridge Closes #1006"
```

### Task 5: Eliminate TriggerDeserializer via WRAPPER_OBJECT mixin

The current trigger YAML pattern IS Jackson's WRAPPER_OBJECT — the property name (`contextChange`, `schedule`, `scopeActivated`) identifies the type. Jackson handles this natively with `@JsonTypeInfo(use=NAME, include=WRAPPER_OBJECT)`.

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/converter/deser/TriggerMixin.java`
- Delete: `api/src/main/java/io/casehub/api/model/converter/deser/TriggerDeserializer.java` (use `ide_refactor_safe_delete`)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java` — remove TriggerDeserializer registration, add TriggerMixin
- Test: existing `TriggerDeserializerTest` renamed to `TriggerMixinTest`, all tests still pass

**Interfaces:**
- Produces: Trigger deserialization via Jackson's built-in WRAPPER_OBJECT support instead of custom deserializer

- [ ] **Step 1: Create TriggerMixin**

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.WRAPPER_OBJECT)
@JsonSubTypes({
    @JsonSubTypes.Type(value = ContextChangeTrigger.class, name = "contextChange"),
    @JsonSubTypes.Type(value = ScheduleTrigger.class, name = "schedule"),
    @JsonSubTypes.Type(value = ScopeActivatedTrigger.class, name = "scopeActivated")
})
public abstract class TriggerMixin {}
```

- [ ] **Step 2: Register mixin, remove deserializer from CaseDefinitionModule**

Replace `addDeserializer(Trigger.class, new TriggerDeserializer())` with mixin registration in `setupModule()`:
```java
context.setMixInAnnotations(Trigger.class, TriggerMixin.class);
```

- [ ] **Step 3: Verify ContextChangeTrigger/ScheduleTrigger/ScopeActivatedTrigger are Jackson-deserializable**

Check constructors and fields. If ScheduleTrigger uses factory methods (`cron()`, `delay()`), a mixin `@JsonCreator` may be needed. If ContextChangeTrigger needs ExpressionEvaluator for its `filter` field, the registered ExpressionEvaluatorDeserializer handles it.

- [ ] **Step 4: Run trigger tests**

```bash
mvn test -pl api -Dtest="TriggerDeserializerTest" -DfailIfNoTests=false -q
```

If tests pass: delete `TriggerDeserializer.java` via `ide_refactor_safe_delete`. If tests fail: the mixin approach may not handle all edge cases (filter expression resolution, schedule cron/every dispatch). In that case, keep the deserializer — it's only 120 lines and genuinely handles complexity.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "refactor: replace TriggerDeserializer with WRAPPER_OBJECT mixin Refs #1006"
```

## References

- [docs/specs/issue-422-ts-programming-model/2026-08-25-case-definition-module-design.md] — CaseDefinitionModule design spec
- [docs/specs/issue-422-ts-programming-model/2026-08-24-schema-generator-design.md] — model-canonical schema generator design
- [docs/specs/issue-422-ts-programming-model/decisions.md] — D1-D5 decisions
- [api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java] — bridge code to delete
- [api/src/main/java/io/casehub/api/model/converter/CaseDefinitionModule.java] — module to wire into production
- [api/src/main/java/io/casehub/api/model/converter/deser/] — 8 custom deserializers + 3 mixins
- [api/src/main/java/io/casehub/api/model/CapabilityTarget.java:32-37] — D5 alignment fix
- [GitHub #1006] — focal issue
- [GitHub casehubio/parent#422] — epic
