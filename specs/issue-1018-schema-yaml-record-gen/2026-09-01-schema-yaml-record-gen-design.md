# Schema-Driven YAML Record Generation

Refs: engine#1018 (parent: engine#1017)

## Context

The 44 YAML record types in `io.casehub.api.model.converter.yaml` were hand-written in #1015. They mirror the JSON Schema at `schema/src/main/resources/schema/CaseDefinition.yaml` but there is no enforcement — when a field is added to the schema, the corresponding record must be manually updated. This drift risk grows as the YAML surface expands.

The `codegen` module already extends jsonschema2pojo to generate POJOs from the same schema into the `schema` module (`io.casehub.model`). This spec adds a second generation target: Java records for YAML deserialization, generated into the `api` module.

## Architecture

```
CaseDefinition.yaml ─┬─ CasehubCodegen ──────→ io.casehub.model (POJOs, schema module)
                      │
                      └─ CasehubRecordCodegen ─→ io.casehub.api.model.converter.yaml (records, api module)
                         ↑
            yaml-record-mappings.yaml
```

`CasehubRecordCodegen` is a new main class in the `codegen` module. It reads the JSON Schema + a YAML mapping file and emits Java record source files. Run via `exec-maven-plugin` in `api/pom.xml` at `generate-sources` phase — the same pattern used by the schema module for POJO generation.

## Generator

### Inputs

1. **`CaseDefinition.yaml`** — the JSON Schema (existing, unchanged)
2. **`yaml-record-mappings.yaml`** — declares which schema types become records and how fields are mapped (new file, lives alongside the schema in `schema/src/main/resources/schema/`)

### Output

Java record `.java` files in `api/target/generated-sources/yaml-records/io/casehub/api/model/converter/yaml/`.

### Processing

For each type declared in the mapping file:

1. Locate the schema type in `$defs` (or the root object for `CaseDefinition`)
2. Walk schema `properties` — each property becomes a record component
3. Apply field overrides from the mapping:
   - **Type override** — replace schema-derived Java type with a specified type (e.g., `ExpressionEvaluator` instead of `String`)
   - **Deserializer** — emit `@JsonDeserialize(using = <class>.class)` on the component
   - **Alias** — emit `@JsonAlias("<name>")` on the component
   - **Property** — emit `@JsonProperty("<name>")` on the component (for reserved words like `do`)
   - **Skip** — omit the field entirely (for `_codegen*` directives)
4. Append extra fields from the mapping (fields present in the record but not in the schema)
5. Generate a null-safe compact constructor: every `List` component defaults to `List.of()`, every `Map` component defaults to `Map.of()`
6. Emit `@JsonIgnoreProperties(ignoreUnknown = true)` on the record class
7. Emit the Apache 2.0 license header

### Schema Type → Java Type Defaults

When no override is specified in the mapping file:

| Schema type | Java type |
|-------------|-----------|
| `string` | `String` |
| `integer` | `Integer` |
| `number` | `Double` |
| `boolean` | `Boolean` |
| `array` of `string` | `List<String>` |
| `array` of `$ref: Foo` | `List<YamlFoo>` (look up record name) |
| `object` with `additionalProperties: { type: X }` | `Map<String, X>` |
| `$ref: Foo` | `YamlFoo` (look up record name, fall back to schema type name) |
| `object` (no additional info) | `JsonNode` |

The `$ref` lookup chain: check mapping for a `record:` name → use it. If unmapped, use the schema type name directly (e.g., `Trigger` stays `Trigger` — it's a domain type with a custom deserializer).

## Mapping File

Lives at `schema/src/main/resources/schema/yaml-record-mappings.yaml`. Structure:

```yaml
package: io.casehub.api.model.converter.yaml
skipPatterns:
  - "_codegen*"

imports:
  Trigger: io.casehub.api.model.Trigger
  ExpressionEvaluator: io.casehub.platform.api.expression.ExpressionEvaluator
  CaseCompletion: io.casehub.api.model.CaseCompletion
  CbrConfig: io.casehub.api.model.cbr.CbrConfig
  AdaptationConfig: io.casehub.api.model.AdaptationConfig
  JsonNode: com.fasterxml.jackson.databind.JsonNode

deserializers:
  TriggerDeserializer: io.casehub.api.model.converter.deser.TriggerDeserializer
  CaseCompletionDeserializer: io.casehub.api.model.converter.deser.CaseCompletionDeserializer
  CbrConfigDeserializer: io.casehub.api.model.converter.deser.CbrConfigDeserializer
  AdaptationConfigDeserializer: io.casehub.api.model.converter.deser.AdaptationConfigDeserializer

types:
  CaseDefinition:
    record: YamlCaseDefinition
    source: root
    fields:
      spec: { type: YamlCaseSpec }
      iterations: { type: "Map<String, YamlIterationGroup>" }
      context: { type: JsonNode }
      semanticData: { type: "Map<String, Object>" }
      episodic: { type: JsonNode }
      use: { type: JsonNode }
    extra:
      - { name: dsl, type: String }
      - { name: title, type: String }
      - { name: summary, type: String }
      - { name: expressionLang, type: String }
      - { name: contextType, type: String }

  CaseDefinitionSpec:
    record: YamlCaseSpec
    fields:
      completion: { type: CaseCompletion, deserializer: CaseCompletionDeserializer }
      cbrConfig: { type: CbrConfig, deserializer: CbrConfigDeserializer, alias: cbr }
      adaptationConfig: { type: AdaptationConfig, deserializer: AdaptationConfigDeserializer, alias: adaptation }
      reflectionTrigger: { type: YamlReflectionTriggerConfig, alias: reflection }
      quorum: { type: YamlQuorumConfig }
      actions: { type: "List<YamlGoapAction>", alias: goapActions }
    extra:
      - { name: workers, type: "List<YamlWorker>" }
      - { name: bindings, type: "List<YamlBinding>" }
      - { name: decomposition, type: YamlDecomposition }

  Binding:
    record: YamlBinding
    fields:
      on: { type: Trigger, deserializer: TriggerDeserializer }
      when: { type: ExpressionEvaluator }
      inputProjectionOverride: { type: ExpressionEvaluator }
      outcomePolicy: { type: JsonNode }
      exchangeProjection: { type: JsonNode }
    extra:
      - { name: judgment, type: YamlJudgmentTarget }
      - { name: subCase, type: YamlSubCaseTarget }
      - { name: humanTask, type: YamlHumanTaskTarget }
      - { name: recoveryOverride, type: YamlRecoveryOverride }

  Worker:
    record: YamlWorker
    fields:
      doBlock: { property: do, type: JsonNode }
      effect: { type: "Map<String, Boolean>" }
    extra:
      - { name: agent, type: YamlAgent }
      - { name: react, type: YamlReact }
      - { name: a2a, type: YamlA2A }
      - { name: mcp, type: YamlMcp }
      - { name: agentDescriptor, type: YamlAgentDescriptor }

  # Simple 1:1 types — record name derived, all fields from schema, no overrides
  Capability: { record: YamlCapability }
  Goal:
    record: YamlGoal
    fields:
      when: { type: ExpressionEvaluator }
      condition: { type: ExpressionEvaluator }
  Milestone:
    record: YamlMilestone
    fields:
      entryCriteria: { type: ExpressionEvaluator }
      condition: { type: ExpressionEvaluator }
  # ... remaining types follow the same pattern
```

The full mapping file covers all 44 record types. Most are simple 1:1 mappings with no overrides — only the top-level types (CaseDefinition, CaseDefinitionSpec, Binding, Worker) and types with polymorphic fields need explicit field declarations.

## Build Integration

### api/pom.xml additions

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>generate-yaml-records</id>
            <phase>generate-sources</phase>
            <goals><goal>java</goal></goals>
            <configuration>
                <mainClass>io.casehub.codegen.CasehubRecordCodegen</mainClass>
                <includePluginDependencies>true</includePluginDependencies>
                <arguments>
                    <argument>${project.basedir}/../schema/src/main/resources/schema/CaseDefinition.yaml</argument>
                    <argument>${project.basedir}/../schema/src/main/resources/schema/yaml-record-mappings.yaml</argument>
                    <argument>${project.build.directory}/generated-sources/yaml-records</argument>
                    <argument>io.casehub.api.model.converter.yaml</argument>
                </arguments>
            </configuration>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-codegen</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</plugin>
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>add-yaml-record-sources</id>
            <phase>generate-sources</phase>
            <goals><goal>add-source</goal></goals>
            <configuration>
                <sources>
                    <source>${project.build.directory}/generated-sources/yaml-records</source>
                </sources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### codegen/pom.xml

No changes — `codegen` has no new dependencies. `CasehubRecordCodegen` uses Jackson for YAML parsing (already a transitive dependency via jsonschema2pojo-core) and string templates for code generation.

## Deletions

| Files | Count | Replaced by |
|-------|-------|-------------|
| `api/src/main/java/io/casehub/api/model/converter/yaml/Yaml*.java` | 44 | Generated records in `api/target/generated-sources/yaml-records/` |

**Retained (not generated):**
- `JsonNodeForEachAdapter.java` — utility class, not a record
- `YamlCaseDefinitionConverter.java` — domain logic
- All polymorphic deserializers in `deser/`
- `CaseDefinitionModule`, `CaseDefinitionYamlMapper`

## Validation

Generated records replace hand-written ones at compile time. The type system is the drift detector:

- **Field missing from record:** `YamlCaseDefinitionConverter` fails to compile (references absent component)
- **Field type changed:** Converter type-check fails
- **Field added to schema but not mapped:** No record component generated — converter must opt in via mapping file, then tests verify the domain transform

All 1372 existing api tests must pass without modification. If they do, the generated records are field-compatible with the hand-written ones.

## Implementation Phases

### Phase 1 — Generator core

1. `CasehubRecordCodegen.main()` — reads schema + mapping, emits records
2. Schema parser — walks `$defs` and root, extracts property names/types
3. Mapping parser — loads YAML mapping file
4. Record emitter — string template generation for `.java` files
5. Unit tests for the generator in the codegen module

### Phase 2 — Mapping file

1. Write `yaml-record-mappings.yaml` covering all 44 record types
2. Verify generated output matches hand-written records (structural comparison, not character-exact)

### Phase 3 — Build wiring + switchover

1. Add exec-maven-plugin + build-helper-maven-plugin to `api/pom.xml`
2. Delete hand-written record files
3. Verify all 1372 api tests pass
4. Update CLAUDE.md with generated-sources documentation

## References

- `codegen/src/main/java/io/casehub/codegen/CasehubCodegen.java` — existing POJO generation pattern
- `codegen/src/main/java/io/casehub/codegen/CasehubRuleFactory.java` — existing codegen customization
- `schema/pom.xml` — existing exec-maven-plugin + build-helper wiring
- `api/src/main/java/io/casehub/api/model/converter/yaml/` — 44 hand-written records (baseline)
- `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` — consumer of records
- `specs/issue-1015-yaml-core-adoption/2026-08-31-yaml-core-adoption-design.md` — #1015 spec (hand-written records)
- engine#977 — TypeScript writer (parallel effort, same schema, different output language)
