# yaml-core Record Pattern Adoption

Refs: engine#1015 (parent: engine#978)

## Context

The engine's YAML loading pipeline uses 10 hand-coded Jackson deserializers totalling ~2400 lines. The largest — `CaseDefinitionDeserializer` (793 lines) — manually wires ~50 fields with `if (node.has("X")) def.setX(...)` boilerplate. `BindingDeserializer` (670 lines), `CaseDefinitionPostProcessor` (472 lines), and `WorkerDeserializer` (97 lines) follow the same pattern.

The `casehub-platform/yaml-core` module and the `casehub-desiredstate/yaml` module demonstrate a proven alternative: plain Jackson records with null-safe compact constructors, zero custom deserializers for field mapping, and thin converters for domain transforms. Desiredstate's `YamlGraph` is 26 lines. Its `YamlRule` is 22 lines.

This spec migrates the engine to the same pattern.

## Pipeline — Before and After

**Before:**
```
YAML → CaseDefinitionModule (10 registered deserializers)
     → CaseDefinitionDeserializer (793 lines, if/has/get per field)
     → BindingDeserializer (670 lines)
     → WorkerDeserializer, TriggerDeserializer, etc.
     → CaseDefinitionPostProcessor (472 lines, raw JsonNode)
     → CaseDefinition
```

**After:**
```
YAML → VariableResolver (yaml-core, ${env.X} / ${config.X})
     → ForEachExpander (yaml-core, template expansion)
     → Jackson records (automatic field mapping)
       with @JsonDeserialize on polymorphic fields
     → YamlCaseDefinitionConverter (thin domain transforms)
     → CaseDefinition
```

## YAML Record Hierarchy

Plain Jackson records mirroring the YAML structure. Null-safe compact constructors with `Map.of()` / `List.of()` defaults. Package: `io.casehub.api.model.converter.yaml`.

### YamlCaseDefinition (top-level)

```java
public record YamlCaseDefinition(
    String name,
    String namespace,
    String version,
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

Replaces `CaseDefinitionDeserializer.deserialize()` top-level parsing and `CaseDefinitionSpec`.

### YamlCaseSpec (spec block)

All fields that are currently boilerplate in `CaseDefinitionDeserializer` lines 54–344. Jackson auto-maps every field. Polymorphic fields use `@JsonDeserialize`:

```java
public record YamlCaseSpec(
    // Auto-mapped (string, int, boolean, list)
    String agentRouting,
    String implementationRouting,
    String humanTaskRouting,
    String candidateMatching,
    String decompositionStrategy,
    String planningStrategy,
    String contextStoreFactory,
    String expressionLang,
    Integer maxDecompositionDepth,
    Integer maxAdaptations,
    Integer maxEscalations,
    Map<String, Double> routingSignalWeights,
    Map<String, String> workerServiceAccountIds,

    // Nested records (auto-mapped)
    List<YamlCapability> capabilities,
    List<YamlGoal> goals,
    List<YamlMilestone> milestones,
    List<YamlContextConstraint> humanTaskContextConstraints,
    YamlWorkloadConstraint humanTaskWorkloadConstraint,
    YamlMonitoringConfig monitoring,
    YamlReflectionTriggerConfig reflection,
    YamlMemoryRetrievalConfig memoryRetrieval,
    YamlRecoveryPolicy recoveryPolicy,
    YamlPlanningConstraints planningConstraints,
    YamlPortfolioConfig portfolioConfig,
    YamlEpisodicMemoryConfig episodicMemory,

    // Polymorphic (custom deserializer via annotation)
    @JsonDeserialize(using = CaseCompletionDeserializer.class)
    CaseCompletion completion,
    @JsonDeserialize(using = AdaptationConfigDeserializer.class)
    AdaptationConfig adaptationConfig,
    @JsonDeserialize(using = CbrConfigDeserializer.class)
    CbrConfig cbrConfig,

    // Transform-requiring fields (converter handles)
    List<String> types,
    List<String> labels,
    List<YamlSignalType> signals,
    List<YamlChannel> channels,
    List<YamlContextLayer> layers,
    Map<String, YamlCognitiveDemand> cognitiveDemands,
    Map<String, List<String>> authorization,
    List<YamlLabelRule> labelRules,
    List<YamlInboundMapping> inboundMappings,
    List<YamlCompound> compounds,
    List<YamlGoapAction> actions) {

  public YamlCaseSpec {
    if (capabilities == null) capabilities = List.of();
    if (goals == null) goals = List.of();
    if (milestones == null) milestones = List.of();
    if (humanTaskContextConstraints == null) humanTaskContextConstraints = List.of();
    if (types == null) types = List.of();
    if (labels == null) labels = List.of();
    if (signals == null) signals = List.of();
    if (channels == null) channels = List.of();
    if (layers == null) layers = List.of();
    if (cognitiveDemands == null) cognitiveDemands = Map.of();
    if (authorization == null) authorization = Map.of();
    if (labelRules == null) labelRules = List.of();
    if (inboundMappings == null) inboundMappings = List.of();
    if (compounds == null) compounds = List.of();
    if (actions == null) actions = List.of();
    if (routingSignalWeights == null) routingSignalWeights = Map.of();
    if (workerServiceAccountIds == null) workerServiceAccountIds = Map.of();
  }
}
```

### YamlBinding

```java
public record YamlBinding(
    String name,
    String capability,
    @JsonDeserialize(using = TriggerDeserializer.class)
    Trigger on,
    @JsonDeserialize(using = ExpressionEvaluatorDeserializer.class)
    ExpressionEvaluator when,
    String inputProjectionOverride,
    String outcomePolicy,
    String conflictResolverStrategy,
    String lifecycleScope,
    String participation,
    String executionMode,
    String replanHint,
    String replanAfter,
    String exchangeProjection,
    String produces,
    String consumes,
    List<String> producedKeys,
    List<String> contingency,
    Map<String, Object> contextWrite,
    Map<String, Object> signal,

    // Target blocks (polymorphic — converter dispatches)
    YamlHumanTaskTarget humanTask,
    YamlJudgmentTarget judgment,
    YamlSubCaseTarget subCase,
    YamlRecoveryOverride recoveryOverride,
    YamlExchangeProjectionConfig exchangeProjectionConfig) {

  public YamlBinding {
    if (producedKeys == null) producedKeys = List.of();
    if (contingency == null) contingency = List.of();
    if (contextWrite == null) contextWrite = Map.of();
  }
}
```

### YamlWorker

```java
public record YamlWorker(
    String name,
    String description,
    String definitionRef,
    List<String> capabilities,
    String contextType,
    String outputType,
    List<String> sequence,
    YamlExecutionPolicy executionPolicy,
    String forEach,

    // Worker function blocks (converter dispatches to providers)
    YamlAgent agent,
    YamlReact react,
    YamlA2A a2a,
    YamlMcp mcp,
    JsonNode doBlock,

    // GOAP shorthand fields (converter maps to GoapAction)
    Double cost,
    Map<String, Boolean> effect,
    List<String> softDependency,

    // Agent descriptor block (converter maps to AgentDescriptor)
    YamlAgentDescriptor agentDescriptor) {

  public YamlWorker {
    if (capabilities == null) capabilities = List.of();
    if (sequence == null) sequence = List.of();
    if (softDependency == null) softDependency = List.of();
  }
}
```

### Supporting Records

Small records for nested YAML blocks. Each mirrors its YAML shape exactly:

- `YamlCapability(String name, String inputProjection, String outputProjection, YamlCognitiveDemand cognitiveDemand)`
- `YamlGoal(String name, String description, String kind, ExpressionEvaluator when)`
- `YamlMilestone(String name, ExpressionEvaluator when, YamlSla sla)`
- `YamlContextConstraint(ExpressionEvaluator when, YamlConstraintEffect effect, Double weight)`
- `YamlWorkloadConstraint(Integer maxActiveTaskCount, Double loadBalanceWeight)`
- `YamlMonitoringConfig(Boolean enabled, Double perCompletionThreshold, Integer windowSize)`
- `YamlReflectionTriggerConfig(Boolean enabled, Double importanceThreshold, ...)`
- `YamlRecoveryPolicy(Integer maxRetries, Integer maxRerouteAttempts, String classifierId, Boolean enabled)`
- `YamlPlanningConstraints(String timeBudget, Integer resourceLimit, Map<String, Double> weights, Map<String, Integer> costBudgets)`
- `YamlGoapAction(String name, String capability, Map<String, Boolean> preconditions, Map<String, Boolean> effects, Double cost, List<String> softDependency)`
- `YamlCompound(String name, String completionSemantics, String dispatchMode, List<String> children, Map<String, String> scopedBindings, ...)`
- `YamlHumanTaskTarget(...)`, `YamlJudgmentTarget(...)`, `YamlSubCaseTarget(...)`
- `YamlAgent(String model, String modelName, String systemPrompt, ...)`, `YamlReact(Integer maxCycles)`, `YamlA2A(String endpoint, String skill, Boolean streaming, YamlAuth auth)`, `YamlMcp(...)`
- `YamlAgentDescriptor(...)`, `YamlSignalType(String name, String contextType)`, `YamlChannel(...)`
- `YamlLabelRule(String name, ExpressionEvaluator when, List<YamlLabelAction> actions)`
- `YamlInboundMapping(String signal, String connectorType, ExpressionEvaluator correlation, ExpressionEvaluator payload, String correlationResolver)`
- `YamlIterationGroup(List<String> in, String as)` — reuses yaml-core's `IterationGroup` concept
- `YamlConstraintEffect(List<String> preferGroups, List<String> preferUsers, List<String> excludeGroups, List<String> excludeUsers)`

## Converter

`YamlCaseDefinitionConverter` replaces both the boilerplate field mapping in `CaseDefinitionDeserializer` and all logic in `CaseDefinitionPostProcessor`. Single class, single responsibility: YAML records → `CaseDefinition`.

```java
public final class YamlCaseDefinitionConverter {

  public static CaseDefinition convert(
      YamlCaseDefinition yaml,
      ExpressionEngineRegistry registry,
      WorkerFunctionProviderRegistry providers) {

    CaseDefinition def = new CaseDefinition();
    def.setName(yaml.name());
    def.setNamespace(yaml.namespace());
    def.setVersion(yaml.version());

    convertSpec(yaml.spec(), def, registry);
    convertWorkers(yaml.workers(), def, providers);
    convertBindings(yaml.bindings(), def, registry);
    applyDefinitions(yaml.definitions(), def);

    return def;
  }
}
```

**Domain transforms handled by the converter (~10 categories):**

| Transform | What it does |
|-----------|-------------|
| `types` / `labels` | `String` → `Path.parse()` |
| `signals` | `contextType` string → `Class.forName()` → `SignalType<?>` |
| `channels` | `recordType` string → `Class.forName()` |
| `capabilities` | `YamlCapability` → `CapabilityTarget` with JQ input/output projection baking |
| `worker functions` | `YamlAgent` / `YamlReact` / `YamlA2A` / `YamlMcp` / `doBlock` → `WorkerFunction` via provider discovery |
| `GOAP shorthand` | Per-worker `cost` / `effect` / `softDependency` → `GoapAction` list |
| `agent descriptors` | `YamlAgentDescriptor` → `AgentDescriptor` with personality, goals |
| `compounds` | `YamlCompound` → `CompoundDeclaration` with scoped bindings |
| `label rules` | `YamlLabelRule` → `LabelRule` with `CompiledExpression` from JQ |
| `context constraints` | `YamlContextConstraint` → `ContextConstraint` with `ExpressionEvaluator` baking |
| `authorization` | `Map<String, List<String>>` → `Map<AclAction, List<String>>` |
| `binding targets` | `YamlHumanTaskTarget` / `YamlJudgmentTarget` / `YamlSubCaseTarget` → domain target types |

## yaml-core Integration

### VariableResolver

Pre-processing pass in `CaseDefinitionYamlMapper.load()`, before Jackson deserialization:

```java
public static CaseDefinition load(InputStream yamlStream, ...) {
    JsonNode rawNode = objectMapper.readTree(yamlStream.readAllBytes());

    // yaml-core pre-processing
    VariableResolver resolver = new VariableResolver(
        Map.of(
            "env", name -> System.getenv(name),
            "config", configSource),
        Set.of("runtime"));  // deferred: resolved at case start

    Object resolved = resolver.resolve(
        objectMapper.convertValue(rawNode, Map.class));
    JsonNode resolvedNode = objectMapper.valueToTree(resolved);

    // Jackson record deserialization
    YamlCaseDefinition yaml = moduleMapper.convertValue(
        resolvedNode, YamlCaseDefinition.class);

    return YamlCaseDefinitionConverter.convert(yaml, registry, providers);
}
```

Replaces the `use` block config/secret placeholder handling in `CaseDefinitionPostProcessor`.

### ForEachExpander

Template expansion for workers and bindings. Runs after variable resolution, before Jackson deserialization:

```yaml
iterations:
  regions:
    in: [eu, us, ap]
    as: region

workers:
  - name: processor-${each.region}
    forEach: regions
    capabilities: [process-${each.region}]
    agent:
      model: anthropic
      modelName: claude-sonnet-4-20250514
```

Engine defines a `ForEachAdapter<JsonNode>` that handles worker and binding nodes:

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
        // resolve ${each.region} in all string fields
    }
}
```

## Deletions

| File | Lines | Replaced by |
|------|-------|-------------|
| `CaseDefinitionDeserializer` | 793 | `YamlCaseDefinition` + `YamlCaseSpec` records + `YamlCaseDefinitionConverter` |
| `BindingDeserializer` | 670 | `YamlBinding` record + converter |
| `WorkerDeserializer` | 97 | `YamlWorker` record + converter |
| `CbrConfigDeserializer` | 77 | `@JsonDeserialize` on `YamlCaseSpec.cbrConfig` field |
| `CaseDefinitionSpec` | ~250 | YAML records |
| `CaseDefinitionPostProcessor` | 472 | `YamlCaseDefinitionConverter` |
| `CaseDefinitionSpecMixin` | ~50 | Not needed |
| **Total deleted** | **~2400** | |

## Retained (polymorphic deserializers)

| Deserializer | Lines | Why it stays |
|-------------|-------|-------------|
| `TriggerDeserializer` | 147 | Discriminated union (contextChange/schedule/scopeActivated/cloudEvent) |
| `ExpressionEvaluatorDeserializer` | 84 | Registry dispatch with injected `ExpressionEngineRegistry` |
| `GoalExpressionDeserializer` | 92 | Recursive tree (allOf/anyOf/single) |
| `CaseCompletionDeserializer` | 131 | Polymorphic (doneWhen vs goal-kind entries) |
| `AdaptationConfigDeserializer` | 80 | String preset OR object form |
| `SubCaseMappingDeserializer` | 66 | Polymorphic expression dispatch |

These move from `CaseDefinitionModule` registration to `@JsonDeserialize` annotations on record fields, except `ExpressionEvaluatorDeserializer` which stays module-registered (needs CDI-injected registry).

## CaseDefinitionModule After

```java
public CaseDefinitionModule(ExpressionEngineRegistry registry) {
    super("CaseDefinitionModule");
    addDeserializer(ExpressionEvaluator.class,
        new ExpressionEvaluatorDeserializer(registry));
}
```

One registration. All other deserializers discovered via `@JsonDeserialize` annotations.

## CaseDefinitionYamlMapper After

```java
public static CaseDefinition load(
    InputStream yamlStream,
    ObjectMapper objectMapper,
    ExpressionEngineRegistry registry,
    WorkerFunctionProviderRegistry providerRegistry) {

    byte[] bytes = yamlStream.readAllBytes();
    JsonNode rawNode = objectMapper.readTree(bytes);

    // Step 1: yaml-core pre-processing
    VariableResolver resolver = buildResolver(/* config sources */);
    Object resolved = resolver.resolve(
        objectMapper.convertValue(rawNode, Map.class));
    JsonNode resolvedNode = objectMapper.valueToTree(resolved);

    // Step 2: ForEach expansion (workers and bindings)
    resolvedNode = expandForEach(resolvedNode, resolver, objectMapper);

    // Step 3: Jackson record deserialization
    ObjectMapper moduleMapper = objectMapper.copy()
        .registerModule(new CaseDefinitionModule(
            registry != null ? registry : JQ_ONLY))
        .disable(FAIL_ON_UNKNOWN_PROPERTIES);
    moduleMapper.addHandler(UnknownPropertyWarningHandler.INSTANCE);

    YamlCaseDefinition yaml = moduleMapper.convertValue(
        resolvedNode, YamlCaseDefinition.class);

    // Step 4: Domain conversion
    return YamlCaseDefinitionConverter.convert(
        yaml,
        registry != null ? registry : JQ_ONLY,
        providerRegistry != null ? providerRegistry : EMPTY_PROVIDERS);
}
```

## Test Strategy

- **Unit tests for records**: construct `YamlCaseSpec`, `YamlBinding`, `YamlWorker` directly, assert converter output
- **Unit tests for converter**: each transform category has focused tests
- **Integration tests**: existing YAML-string tests become end-to-end tests for `CaseDefinitionYamlMapper.load()`
- **yaml-core tests**: VariableResolver integration, ForEachExpander with worker/binding templates

## Implementation Phases

### Phase 1 — Records and Converter

1. Define YAML record hierarchy (`YamlCaseDefinition`, `YamlCaseSpec`, `YamlBinding`, `YamlWorker`, supporting records)
2. Write `YamlCaseDefinitionConverter` covering all domain transforms
3. Update `CaseDefinitionYamlMapper.load()` to deserialize to records then convert
4. Move polymorphic deserializers to `@JsonDeserialize` annotations
5. Delete `CaseDefinitionDeserializer`, `BindingDeserializer`, `WorkerDeserializer`, `CbrConfigDeserializer`
6. Delete `CaseDefinitionSpec`, `CaseDefinitionSpecMixin`
7. Delete `CaseDefinitionPostProcessor`
8. Rewrite tests against records; keep YAML-string tests as integration tests
9. Verify all 1345 api tests pass

### Phase 2 — yaml-core Integration

1. Add `casehub-platform-yaml-core` dependency to `engine-api`
2. Wire `VariableResolver` into `CaseDefinitionYamlMapper.load()` pre-processing
3. Implement `YamlWorkerForEachAdapter` and `YamlBindingForEachAdapter`
4. Wire `ForEachExpander` into the loading pipeline
5. Add `iterations:` to the YAML schema
6. Add `forEach:` and `when:` to Worker and Binding schemas
7. Add tests for variable resolution and template expansion
8. Update consumer guide

## References

- `casehub-platform/yaml-core/src/main/java/io/casehub/yaml/core/` — VariableResolver, ForEachExpander, Truthiness
- `casehub-desiredstate/yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/model/` — YamlGraph, YamlRule, YamlNode (reference pattern)
- `casehub-desiredstate/yaml/runtime/src/main/java/io/casehub/desiredstate/yaml/YamlRuleConverter.java` — converter pattern
- `api/src/main/java/io/casehub/api/model/converter/deser/CaseDefinitionDeserializer.java` — 793 lines being replaced
- `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionPostProcessor.java` — 472 lines being replaced
- `api/src/main/java/io/casehub/api/model/CaseDefinitionSpec.java` — being deleted
- `schema/src/main/resources/schema/CaseDefinition.yaml` — YAML schema (gains iterations:, forEach:, when:)
