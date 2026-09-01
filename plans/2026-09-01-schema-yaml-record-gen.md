# Schema-Driven YAML Record Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1018 — schema-driven YAML record generation
**Issue group:** #1018

**Goal:** Auto-generate the 44 YAML deserialization record types from the JSON Schema at build time, replacing hand-written records.

**Architecture:** `CasehubRecordCodegen` (new main class in `codegen` module) reads `CaseDefinition.yaml` + `yaml-record-mappings.yaml`, emits Java record source files via string templates. Run by `exec-maven-plugin` in `api/pom.xml` at `generate-sources` phase. Generated records replace the hand-written files in `api/src/main/java/io/casehub/api/model/converter/yaml/`.

**Tech Stack:** Jackson (YAML parsing of schema + mapping file), Java string templates for code generation, Maven exec-maven-plugin + build-helper-maven-plugin.

## Global Constraints

- Records generate into `api/target/generated-sources/yaml-records/io/casehub/api/model/converter/yaml/`
- Every generated record gets `@JsonIgnoreProperties(ignoreUnknown = true)` and the Apache 2.0 license header
- List components default to `List.of()`, Map components to `Map.of()`, Set components to `Set.of()` in compact constructors
- `JsonNodeForEachAdapter.java` is NOT a record — it stays hand-written
- All 1372 existing api tests must pass after switchover without modification

---

## Batch 1: Generator Core

### Task 1: Schema parser and mapping parser

**Files:**
- Create: `codegen/src/main/java/io/casehub/codegen/record/SchemaParser.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/MappingParser.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/SchemaType.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/SchemaField.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/RecordMapping.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/TypeMapping.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/FieldOverride.java`
- Create: `codegen/src/main/java/io/casehub/codegen/record/ExtraField.java`
- Test: `codegen/src/test/java/io/casehub/codegen/record/SchemaParserTest.java`
- Test: `codegen/src/test/java/io/casehub/codegen/record/MappingParserTest.java`

**Interfaces:**
- Produces: `SchemaParser.parse(JsonNode schemaRoot) → Map<String, SchemaType>` — returns all `$defs` types + the root type keyed by type name
- Produces: `SchemaType(String name, List<SchemaField> fields)` — parsed schema type
- Produces: `SchemaField(String name, String schemaType, boolean isArray, boolean isMap, String refTarget, String mapValueType)` — parsed field info
- Produces: `MappingParser.parse(JsonNode mappingRoot) → RecordMapping` — returns the full mapping config
- Produces: `RecordMapping(String packageName, List<String> skipPatterns, Map<String, String> imports, Map<String, String> deserializers, Map<String, TypeMapping> types)` — top-level mapping
- Produces: `TypeMapping(String recordName, String source, Map<String, FieldOverride> fields, List<ExtraField> extra)` — per-type mapping
- Produces: `FieldOverride(String type, String deserializer, String alias, String property)` — per-field overrides
- Produces: `ExtraField(String name, String type)` — fields not in schema

- [ ] **Step 1: Write SchemaParser test**

```java
package io.casehub.codegen.record;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SchemaParserTest {

  private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

  @Test
  void parsesDefTypes() throws Exception {
    var schema = YAML.readTree("""
        $defs:
          Foo:
            type: object
            properties:
              name:
                type: string
              count:
                type: integer
              tags:
                type: array
                items:
                  type: string
              metadata:
                type: object
                additionalProperties:
                  type: string
        type: object
        properties:
          title:
            type: string
        """);

    Map<String, SchemaType> types = SchemaParser.parse(schema);

    assertTrue(types.containsKey("Foo"));
    SchemaType foo = types.get("Foo");
    assertEquals(4, foo.fields().size());

    SchemaField name = foo.fieldByName("name");
    assertEquals("string", name.schemaType());
    assertFalse(name.isArray());
    assertFalse(name.isMap());

    SchemaField count = foo.fieldByName("count");
    assertEquals("integer", count.schemaType());

    SchemaField tags = foo.fieldByName("tags");
    assertTrue(tags.isArray());
    assertEquals("string", tags.schemaType());

    SchemaField metadata = foo.fieldByName("metadata");
    assertTrue(metadata.isMap());
    assertEquals("string", metadata.mapValueType());

    assertTrue(types.containsKey("CaseDefinition"));
    SchemaType root = types.get("CaseDefinition");
    assertEquals(1, root.fields().size());
  }

  @Test
  void parsesRefFields() throws Exception {
    var schema = YAML.readTree("""
        $defs:
          Bar:
            type: object
            properties:
              baz:
                $ref: "#/$defs/Baz"
              items:
                type: array
                items:
                  $ref: "#/$defs/Baz"
          Baz:
            type: object
            properties:
              value:
                type: string
        type: object
        properties: {}
        """);

    Map<String, SchemaType> types = SchemaParser.parse(schema);
    SchemaType bar = types.get("Bar");

    SchemaField baz = bar.fieldByName("baz");
    assertEquals("ref", baz.schemaType());
    assertEquals("Baz", baz.refTarget());

    SchemaField items = bar.fieldByName("items");
    assertTrue(items.isArray());
    assertEquals("ref", items.schemaType());
    assertEquals("Baz", items.refTarget());
  }

  @Test
  void skipsFieldsMatchingPattern() throws Exception {
    var schema = YAML.readTree("""
        $defs:
          Spec:
            type: object
            properties:
              realField:
                type: string
              _codegenFoo:
                type: string
              _codegenBar:
                $ref: "#/$defs/Spec"
        type: object
        properties: {}
        """);

    Map<String, SchemaType> types = SchemaParser.parse(schema);
    SchemaType spec = types.get("Spec");
    assertEquals(3, spec.fields().size());
    assertNotNull(spec.fieldByName("_codegenFoo"));
  }

  @Test
  void parsesMapWithRefValue() throws Exception {
    var schema = YAML.readTree("""
        $defs:
          Config:
            type: object
            properties:
              weights:
                type: object
                additionalProperties:
                  type: number
              boolMap:
                type: object
                additionalProperties:
                  type: boolean
        type: object
        properties: {}
        """);

    Map<String, SchemaType> types = SchemaParser.parse(schema);
    SchemaType config = types.get("Config");

    SchemaField weights = config.fieldByName("weights");
    assertTrue(weights.isMap());
    assertEquals("number", weights.mapValueType());

    SchemaField boolMap = config.fieldByName("boolMap");
    assertTrue(boolMap.isMap());
    assertEquals("boolean", boolMap.mapValueType());
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl codegen -Dtest=SchemaParserTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — classes not yet created

- [ ] **Step 3: Implement SchemaParser and model records**

Create `SchemaField.java`:
```java
package io.casehub.codegen.record;

public record SchemaField(
    String name,
    String schemaType,
    boolean isArray,
    boolean isMap,
    String refTarget,
    String mapValueType) {

  public SchemaField(String name, String schemaType) {
    this(name, schemaType, false, false, null, null);
  }
}
```

Create `SchemaType.java`:
```java
package io.casehub.codegen.record;

import java.util.List;

public record SchemaType(String name, List<SchemaField> fields) {

  public SchemaField fieldByName(String name) {
    return fields.stream()
        .filter(f -> f.name().equals(name))
        .findFirst()
        .orElse(null);
  }
}
```

Create `SchemaParser.java`:
```java
package io.casehub.codegen.record;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class SchemaParser {

  private SchemaParser() {}

  public static Map<String, SchemaType> parse(JsonNode schemaRoot) {
    Map<String, SchemaType> types = new LinkedHashMap<>();

    JsonNode defs = schemaRoot.path("$defs");
    if (!defs.isMissingNode()) {
      Iterator<Map.Entry<String, JsonNode>> it = defs.fields();
      while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        String typeName = entry.getKey();
        JsonNode typeNode = entry.getValue();
        if (isObjectType(typeNode)) {
          types.put(typeName, parseType(typeName, typeNode));
        }
      }
    }

    if (isObjectType(schemaRoot)) {
      types.put("CaseDefinition", parseType("CaseDefinition", schemaRoot));
    }

    return types;
  }

  private static SchemaType parseType(String name, JsonNode typeNode) {
    List<SchemaField> fields = new ArrayList<>();
    JsonNode props = typeNode.path("properties");
    if (!props.isMissingNode()) {
      Iterator<Map.Entry<String, JsonNode>> it = props.fields();
      while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        fields.add(parseField(entry.getKey(), entry.getValue()));
      }
    }
    return new SchemaType(name, fields);
  }

  private static SchemaField parseField(String name, JsonNode fieldNode) {
    if (fieldNode.has("$ref")) {
      String ref = fieldNode.get("$ref").asText();
      String target = ref.substring(ref.lastIndexOf('/') + 1);
      return new SchemaField(name, "ref", false, false, target, null);
    }

    String type = fieldNode.path("type").asText("");

    if ("array".equals(type)) {
      JsonNode items = fieldNode.path("items");
      if (items.has("$ref")) {
        String ref = items.get("$ref").asText();
        String target = ref.substring(ref.lastIndexOf('/') + 1);
        return new SchemaField(name, "ref", true, false, target, null);
      }
      String itemType = items.path("type").asText("object");
      return new SchemaField(name, itemType, true, false, null, null);
    }

    if ("object".equals(type) && fieldNode.has("additionalProperties")) {
      JsonNode addProps = fieldNode.get("additionalProperties");
      if (addProps.isObject()) {
        String valueType = addProps.path("type").asText("object");
        return new SchemaField(name, "object", false, true, null, valueType);
      }
      return new SchemaField(name, "object", false, true, null, "object");
    }

    return new SchemaField(name, type.isEmpty() ? "object" : type);
  }

  private static boolean isObjectType(JsonNode node) {
    return "object".equals(node.path("type").asText(""));
  }
}
```

- [ ] **Step 4: Write MappingParser test**

```java
package io.casehub.codegen.record;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;

class MappingParserTest {

  private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

  @Test
  void parsesTypeMapping() throws Exception {
    var mapping = YAML.readTree("""
        package: io.casehub.api.model.converter.yaml
        skipPatterns:
          - "_codegen*"
        imports:
          Trigger: io.casehub.api.model.Trigger
        deserializers:
          TriggerDeserializer: io.casehub.api.model.converter.deser.TriggerDeserializer
        types:
          Binding:
            record: YamlBinding
            fields:
              on:
                type: Trigger
                deserializer: TriggerDeserializer
              replanHint:
                alias: replanAfter
              doBlock:
                property: do
                type: JsonNode
            extra:
              - name: judgment
                type: YamlJudgmentTarget
        """);

    RecordMapping result = MappingParser.parse(mapping);

    assertEquals("io.casehub.api.model.converter.yaml", result.packageName());
    assertEquals(1, result.skipPatterns().size());
    assertEquals("_codegen*", result.skipPatterns().get(0));
    assertEquals("io.casehub.api.model.Trigger", result.imports().get("Trigger"));

    TypeMapping binding = result.types().get("Binding");
    assertNotNull(binding);
    assertEquals("YamlBinding", binding.recordName());

    FieldOverride on = binding.fields().get("on");
    assertEquals("Trigger", on.type());
    assertEquals("TriggerDeserializer", on.deserializer());

    FieldOverride replan = binding.fields().get("replanHint");
    assertEquals("replanAfter", replan.alias());

    FieldOverride doBlock = binding.fields().get("doBlock");
    assertEquals("do", doBlock.property());
    assertEquals("JsonNode", doBlock.type());

    assertEquals(1, binding.extra().size());
    assertEquals("judgment", binding.extra().get(0).name());
    assertEquals("YamlJudgmentTarget", binding.extra().get(0).type());
  }

  @Test
  void parsesSimpleTypeWithNoOverrides() throws Exception {
    var mapping = YAML.readTree("""
        package: test.pkg
        types:
          Simple:
            record: YamlSimple
        """);

    RecordMapping result = MappingParser.parse(mapping);
    TypeMapping simple = result.types().get("Simple");
    assertNotNull(simple);
    assertEquals("YamlSimple", simple.recordName());
    assertTrue(simple.fields().isEmpty());
    assertTrue(simple.extra().isEmpty());
  }
}
```

- [ ] **Step 5: Implement MappingParser and model records**

Create `ExtraField.java`:
```java
package io.casehub.codegen.record;

public record ExtraField(String name, String type) {}
```

Create `FieldOverride.java`:
```java
package io.casehub.codegen.record;

public record FieldOverride(String type, String deserializer, String alias, String property) {}
```

Create `TypeMapping.java`:
```java
package io.casehub.codegen.record;

import java.util.List;
import java.util.Map;

public record TypeMapping(
    String recordName,
    String source,
    Map<String, FieldOverride> fields,
    List<ExtraField> extra) {

  public TypeMapping {
    if (fields == null) fields = Map.of();
    if (extra == null) extra = List.of();
  }
}
```

Create `RecordMapping.java`:
```java
package io.casehub.codegen.record;

import java.util.List;
import java.util.Map;

public record RecordMapping(
    String packageName,
    List<String> skipPatterns,
    Map<String, String> imports,
    Map<String, String> deserializers,
    Map<String, TypeMapping> types) {

  public RecordMapping {
    if (skipPatterns == null) skipPatterns = List.of();
    if (imports == null) imports = Map.of();
    if (deserializers == null) deserializers = Map.of();
    if (types == null) types = Map.of();
  }
}
```

Create `MappingParser.java`:
```java
package io.casehub.codegen.record;

import com.fasterxml.jackson.databind.JsonNode;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class MappingParser {

  private MappingParser() {}

  public static RecordMapping parse(JsonNode root) {
    String pkg = root.path("package").asText("");
    List<String> skipPatterns = readStringList(root.path("skipPatterns"));
    Map<String, String> imports = readStringMap(root.path("imports"));
    Map<String, String> deserializers = readStringMap(root.path("deserializers"));
    Map<String, TypeMapping> types = new LinkedHashMap<>();

    JsonNode typesNode = root.path("types");
    if (!typesNode.isMissingNode()) {
      Iterator<Map.Entry<String, JsonNode>> it = typesNode.fields();
      while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        types.put(entry.getKey(), parseTypeMapping(entry.getValue()));
      }
    }

    return new RecordMapping(pkg, skipPatterns, imports, deserializers, types);
  }

  private static TypeMapping parseTypeMapping(JsonNode node) {
    String recordName = node.path("record").asText(null);
    String source = node.path("source").asText(null);
    Map<String, FieldOverride> fields = new LinkedHashMap<>();
    List<ExtraField> extra = new ArrayList<>();

    JsonNode fieldsNode = node.path("fields");
    if (!fieldsNode.isMissingNode()) {
      Iterator<Map.Entry<String, JsonNode>> it = fieldsNode.fields();
      while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        fields.put(entry.getKey(), parseFieldOverride(entry.getValue()));
      }
    }

    JsonNode extraNode = node.path("extra");
    if (extraNode.isArray()) {
      for (JsonNode item : extraNode) {
        extra.add(new ExtraField(
            item.path("name").asText(),
            item.path("type").asText()));
      }
    }

    return new TypeMapping(recordName, source, fields, extra);
  }

  private static FieldOverride parseFieldOverride(JsonNode node) {
    return new FieldOverride(
        node.path("type").asText(null),
        node.path("deserializer").asText(null),
        node.path("alias").asText(null),
        node.path("property").asText(null));
  }

  private static List<String> readStringList(JsonNode node) {
    List<String> list = new ArrayList<>();
    if (node.isArray()) {
      for (JsonNode item : node) {
        list.add(item.asText());
      }
    }
    return list;
  }

  private static Map<String, String> readStringMap(JsonNode node) {
    Map<String, String> map = new LinkedHashMap<>();
    if (node.isObject()) {
      Iterator<Map.Entry<String, JsonNode>> it = node.fields();
      while (it.hasNext()) {
        Map.Entry<String, JsonNode> entry = it.next();
        map.put(entry.getKey(), entry.getValue().asText());
      }
    }
    return map;
  }
}
```

- [ ] **Step 6: Run both tests to verify they pass**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl codegen -Dtest="SchemaParserTest,MappingParserTest" -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add codegen/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#1018): SchemaParser + MappingParser — parse JSON Schema and mapping file

Refs #1018"
```

### Task 2: Record emitter and CasehubRecordCodegen main class

**Files:**
- Create: `codegen/src/main/java/io/casehub/codegen/record/RecordEmitter.java`
- Create: `codegen/src/main/java/io/casehub/codegen/CasehubRecordCodegen.java`
- Test: `codegen/src/test/java/io/casehub/codegen/record/RecordEmitterTest.java`

**Interfaces:**
- Consumes: `SchemaParser.parse(JsonNode) → Map<String, SchemaType>`, `MappingParser.parse(JsonNode) → RecordMapping`
- Produces: `RecordEmitter.emit(SchemaType, TypeMapping, RecordMapping) → String` — returns the full Java source file content for one record
- Produces: `CasehubRecordCodegen.main(String[])` — CLI entry point: `<schemaFile> <mappingFile> <outputDir> <package>`

- [ ] **Step 1: Write RecordEmitter test**

```java
package io.casehub.codegen.record;

import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class RecordEmitterTest {

  private static final RecordMapping MAPPING = new RecordMapping(
      "io.casehub.api.model.converter.yaml",
      List.of("_codegen*"),
      Map.of(
          "Trigger", "io.casehub.api.model.Trigger",
          "JsonNode", "com.fasterxml.jackson.databind.JsonNode"),
      Map.of("TriggerDeserializer", "io.casehub.api.model.converter.deser.TriggerDeserializer"),
      Map.of());

  @Test
  void emitsSimpleRecord() {
    var schemaType = new SchemaType("Foo", List.of(
        new SchemaField("name", "string"),
        new SchemaField("count", "integer")));
    var typeMapping = new TypeMapping("YamlFoo", null, Map.of(), List.of());

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("package io.casehub.api.model.converter.yaml;"));
    assertTrue(source.contains("@JsonIgnoreProperties(ignoreUnknown = true)"));
    assertTrue(source.contains("public record YamlFoo("));
    assertTrue(source.contains("String name"));
    assertTrue(source.contains("Integer count"));
  }

  @Test
  void emitsListDefaultsInCompactConstructor() {
    var schemaType = new SchemaType("Bar", List.of(
        new SchemaField("tags", "string", true, false, null, null),
        new SchemaField("meta", "object", false, true, null, "string")));
    var typeMapping = new TypeMapping("YamlBar", null, Map.of(), List.of());

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("List<String> tags"));
    assertTrue(source.contains("Map<String, String> meta"));
    assertTrue(source.contains("public YamlBar {"));
    assertTrue(source.contains("if (tags == null)"));
    assertTrue(source.contains("tags = List.of()"));
    assertTrue(source.contains("if (meta == null)"));
    assertTrue(source.contains("meta = Map.of()"));
  }

  @Test
  void emitsFieldOverrides() {
    var schemaType = new SchemaType("Baz", List.of(
        new SchemaField("on", "ref", false, false, "Trigger", null),
        new SchemaField("hint", "string")));
    var typeMapping = new TypeMapping("YamlBaz", null, Map.of(
        "on", new FieldOverride("Trigger", "TriggerDeserializer", null, null),
        "hint", new FieldOverride(null, null, "replanAfter", null)),
        List.of());

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("@JsonDeserialize(using = TriggerDeserializer.class) Trigger on"));
    assertTrue(source.contains("@JsonAlias(\"replanAfter\") String hint"));
    assertTrue(source.contains("import io.casehub.api.model.Trigger;"));
    assertTrue(source.contains("import io.casehub.api.model.converter.deser.TriggerDeserializer;"));
  }

  @Test
  void emitsJsonPropertyAnnotation() {
    var schemaType = new SchemaType("Qux", List.of());
    var typeMapping = new TypeMapping("YamlQux", null,
        Map.of("doBlock", new FieldOverride("JsonNode", null, null, "do")),
        List.of(new ExtraField("doBlock", "JsonNode")));

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("@JsonProperty(\"do\") JsonNode doBlock"));
  }

  @Test
  void emitsExtraFields() {
    var schemaType = new SchemaType("Parent", List.of(
        new SchemaField("name", "string")));
    var typeMapping = new TypeMapping("YamlParent", null, Map.of(),
        List.of(new ExtraField("child", "YamlChild"),
                new ExtraField("items", "List<String>")));

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("String name"));
    assertTrue(source.contains("YamlChild child"));
    assertTrue(source.contains("List<String> items"));
  }

  @Test
  void skipsFieldsMatchingSkipPattern() {
    var schemaType = new SchemaType("Spec", List.of(
        new SchemaField("realField", "string"),
        new SchemaField("_codegenFoo", "string"),
        new SchemaField("_codegenBar", "ref", false, false, "Baz", null)));
    var typeMapping = new TypeMapping("YamlSpec", null, Map.of(), List.of());

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.contains("String realField"));
    assertFalse(source.contains("_codegenFoo"));
    assertFalse(source.contains("_codegenBar"));
  }

  @Test
  void emitsLicenseHeader() {
    var schemaType = new SchemaType("Lic", List.of(new SchemaField("x", "string")));
    var typeMapping = new TypeMapping("YamlLic", null, Map.of(), List.of());

    String source = RecordEmitter.emit(schemaType, typeMapping, MAPPING);

    assertTrue(source.startsWith("/*\n * Copyright 2026-Present The Case Hub Authors"));
    assertTrue(source.contains("Apache License, Version 2.0"));
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl codegen -Dtest=RecordEmitterTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — `RecordEmitter` not yet created

- [ ] **Step 3: Implement RecordEmitter**

Create `RecordEmitter.java`:
```java
package io.casehub.codegen.record;

import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public final class RecordEmitter {

  private static final String LICENSE = """
      /*
       * Copyright 2026-Present The Case Hub Authors
       *
       * Licensed under the Apache License, Version 2.0 (the "License");
       * you may not use this file except in compliance with the License.
       * You may obtain a copy of the License at
       *
       * http://www.apache.org/licenses/LICENSE-2.0
       *
       * Unless required by applicable law or agreed to in writing, software
       * distributed under the License is distributed on an "AS IS" BASIS,
       * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
       * See the License for the specific language governing permissions and
       * limitations under the License.
       */""";

  private RecordEmitter() {}

  public static String emit(SchemaType schemaType, TypeMapping typeMapping, RecordMapping mapping) {
    String recordName = typeMapping.recordName();
    List<ComponentInfo> components = resolveComponents(schemaType, typeMapping, mapping);
    Set<String> imports = resolveImports(components, mapping);

    StringBuilder sb = new StringBuilder();
    sb.append(LICENSE).append("\n");
    sb.append("package ").append(mapping.packageName()).append(";\n\n");

    for (String imp : imports) {
      sb.append("import ").append(imp).append(";\n");
    }
    if (!imports.isEmpty()) sb.append("\n");

    sb.append("@JsonIgnoreProperties(ignoreUnknown = true)\n");
    sb.append("public record ").append(recordName).append("(\n");

    for (int i = 0; i < components.size(); i++) {
      ComponentInfo c = components.get(i);
      sb.append("    ");
      if (c.annotations() != null && !c.annotations().isEmpty()) {
        sb.append(c.annotations()).append(" ");
      }
      sb.append(c.javaType()).append(" ").append(c.name());
      if (i < components.size() - 1) sb.append(",");
      sb.append("\n");
    }
    sb.append(")");

    List<ComponentInfo> defaultable = components.stream()
        .filter(ComponentInfo::needsDefault)
        .toList();

    if (defaultable.isEmpty()) {
      sb.append(" {}\n");
    } else {
      sb.append(" {\n\n");
      sb.append("  public ").append(recordName).append(" {\n");
      for (ComponentInfo c : defaultable) {
        sb.append("    if (").append(c.name()).append(" == null) {\n");
        sb.append("      ").append(c.name()).append(" = ").append(c.defaultValue()).append(";\n");
        sb.append("    }\n");
      }
      sb.append("  }\n");
      sb.append("}\n");
    }

    return sb.toString();
  }

  private static List<ComponentInfo> resolveComponents(
      SchemaType schemaType, TypeMapping typeMapping, RecordMapping mapping) {
    List<ComponentInfo> components = new ArrayList<>();

    for (SchemaField field : schemaType.fields()) {
      if (shouldSkip(field.name(), mapping.skipPatterns())) continue;

      FieldOverride override = typeMapping.fields().get(field.name());
      String javaType;
      if (override != null && override.type() != null) {
        javaType = override.type();
      } else {
        javaType = resolveJavaType(field, mapping);
      }

      String fieldName = field.name();
      if (override != null && override.property() != null) {
        fieldName = override.property().equals(field.name()) ? field.name() :
            typeMapping.fields().entrySet().stream()
                .filter(e -> e.getValue().equals(override))
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(field.name());
      }
      // If the override is on a different key name, use that key
      for (var entry : typeMapping.fields().entrySet()) {
        if (entry.getValue() == override && !entry.getKey().equals(field.name())) {
          fieldName = entry.getKey();
          break;
        }
      }

      String annotations = buildAnnotations(field.name(), fieldName, override, mapping);
      components.add(new ComponentInfo(fieldName, javaType, annotations,
          isCollectionType(javaType), defaultForType(javaType)));
    }

    for (ExtraField extra : typeMapping.extra()) {
      FieldOverride override = typeMapping.fields().get(extra.name());
      String annotations = override != null ?
          buildAnnotations(extra.name(), extra.name(), override, mapping) : null;
      components.add(new ComponentInfo(extra.name(), extra.type(), annotations,
          isCollectionType(extra.type()), defaultForType(extra.type())));
    }

    return components;
  }

  private static boolean shouldSkip(String fieldName, List<String> skipPatterns) {
    for (String pattern : skipPatterns) {
      if (pattern.endsWith("*")) {
        String prefix = pattern.substring(0, pattern.length() - 1);
        if (fieldName.startsWith(prefix)) return true;
      } else if (pattern.equals(fieldName)) {
        return true;
      }
    }
    return false;
  }

  private static String resolveJavaType(SchemaField field, RecordMapping mapping) {
    if (field.isArray()) {
      if (field.refTarget() != null) {
        String resolved = resolveRefType(field.refTarget(), mapping);
        return "List<" + resolved + ">";
      }
      return "List<" + mapPrimitive(field.schemaType()) + ">";
    }
    if (field.isMap()) {
      return "Map<String, " + mapPrimitive(field.mapValueType()) + ">";
    }
    if ("ref".equals(field.schemaType()) && field.refTarget() != null) {
      return resolveRefType(field.refTarget(), mapping);
    }
    return mapPrimitive(field.schemaType());
  }

  private static String resolveRefType(String refTarget, RecordMapping mapping) {
    TypeMapping targetMapping = mapping.types().get(refTarget);
    if (targetMapping != null && targetMapping.recordName() != null) {
      return targetMapping.recordName();
    }
    return refTarget;
  }

  private static String mapPrimitive(String schemaType) {
    return switch (schemaType) {
      case "string" -> "String";
      case "integer" -> "Integer";
      case "number" -> "Double";
      case "boolean" -> "Boolean";
      default -> "JsonNode";
    };
  }

  private static String buildAnnotations(
      String schemaName, String fieldName, FieldOverride override, RecordMapping mapping) {
    if (override == null) return null;
    List<String> annotations = new ArrayList<>();

    if (override.deserializer() != null) {
      annotations.add("@JsonDeserialize(using = " + override.deserializer() + ".class)");
    }
    if (override.alias() != null) {
      annotations.add("@JsonAlias(\"" + override.alias() + "\")");
    }
    if (override.property() != null) {
      annotations.add("@JsonProperty(\"" + override.property() + "\")");
    }

    return annotations.isEmpty() ? null : String.join(" ", annotations);
  }

  private static Set<String> resolveImports(List<ComponentInfo> components, RecordMapping mapping) {
    Set<String> imports = new LinkedHashSet<>();
    imports.add("com.fasterxml.jackson.annotation.JsonIgnoreProperties");

    boolean hasList = false, hasMap = false, hasSet = false;

    for (ComponentInfo c : components) {
      if (c.javaType().startsWith("List<")) hasList = true;
      if (c.javaType().startsWith("Map<")) hasMap = true;
      if (c.javaType().startsWith("Set<")) hasSet = true;

      String baseType = extractBaseType(c.javaType());
      if (mapping.imports().containsKey(baseType)) {
        imports.add(mapping.imports().get(baseType));
      }

      if (c.annotations() != null) {
        if (c.annotations().contains("@JsonDeserialize")) {
          imports.add("com.fasterxml.jackson.databind.annotation.JsonDeserialize");
          String deser = extractDeserializer(c.annotations());
          if (deser != null && mapping.deserializers().containsKey(deser)) {
            imports.add(mapping.deserializers().get(deser));
          }
        }
        if (c.annotations().contains("@JsonAlias")) {
          imports.add("com.fasterxml.jackson.annotation.JsonAlias");
        }
        if (c.annotations().contains("@JsonProperty")) {
          imports.add("com.fasterxml.jackson.annotation.JsonProperty");
        }
      }
    }

    if (hasList) imports.add("java.util.List");
    if (hasMap) imports.add("java.util.Map");
    if (hasSet) imports.add("java.util.Set");

    return imports;
  }

  private static String extractBaseType(String javaType) {
    if (javaType.contains("<")) {
      int start = javaType.indexOf('<') + 1;
      int end = javaType.lastIndexOf('>');
      String inner = javaType.substring(start, end);
      if (inner.contains(",")) {
        return inner.substring(inner.lastIndexOf(' ') + 1).trim();
      }
      return inner;
    }
    return javaType;
  }

  private static String extractDeserializer(String annotations) {
    int start = annotations.indexOf("using = ");
    if (start < 0) return null;
    start += 8;
    int end = annotations.indexOf(".class", start);
    if (end < 0) return null;
    return annotations.substring(start, end);
  }

  private static boolean isCollectionType(String javaType) {
    return javaType.startsWith("List<") || javaType.startsWith("Map<") || javaType.startsWith("Set<");
  }

  private static String defaultForType(String javaType) {
    if (javaType.startsWith("List<")) return "List.of()";
    if (javaType.startsWith("Map<")) return "Map.of()";
    if (javaType.startsWith("Set<")) return "Set.of()";
    return null;
  }

  record ComponentInfo(String name, String javaType, String annotations,
                        boolean needsDefault, String defaultValue) {}
}
```

- [ ] **Step 4: Implement CasehubRecordCodegen main class**

Create `CasehubRecordCodegen.java`:
```java
package io.casehub.codegen;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.codegen.record.MappingParser;
import io.casehub.codegen.record.RecordEmitter;
import io.casehub.codegen.record.RecordMapping;
import io.casehub.codegen.record.SchemaParser;
import io.casehub.codegen.record.SchemaType;
import io.casehub.codegen.record.TypeMapping;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class CasehubRecordCodegen {

  public static void main(String[] args) throws IOException {
    if (args.length < 4) {
      System.err.println(
          "Usage: CasehubRecordCodegen <schemaFile> <mappingFile> <outputDir> <targetPackage>");
      System.exit(1);
    }

    File schemaFile = new File(args[0]);
    File mappingFile = new File(args[1]);
    File outputDir = new File(args[2]);
    String targetPackage = args[3];

    ObjectMapper yaml = new ObjectMapper(new YAMLFactory());
    JsonNode schemaRoot = yaml.readTree(schemaFile);
    JsonNode mappingRoot = yaml.readTree(mappingFile);

    Map<String, SchemaType> schemaTypes = SchemaParser.parse(schemaRoot);
    RecordMapping mapping = MappingParser.parse(mappingRoot);

    Path packageDir = outputDir.toPath().resolve(targetPackage.replace('.', '/'));
    Files.createDirectories(packageDir);

    int generated = 0;
    for (Map.Entry<String, TypeMapping> entry : mapping.types().entrySet()) {
      String schemaTypeName = entry.getKey();
      TypeMapping typeMapping = entry.getValue();

      SchemaType schemaType = schemaTypes.get(schemaTypeName);
      if (schemaType == null) {
        schemaType = new SchemaType(schemaTypeName, java.util.List.of());
      }

      String source = RecordEmitter.emit(schemaType, typeMapping, mapping);
      Path outputFile = packageDir.resolve(typeMapping.recordName() + ".java");
      Files.writeString(outputFile, source);
      generated++;
    }

    System.out.printf("Generated %d record(s) to %s%n", generated, packageDir);
  }
}
```

- [ ] **Step 5: Run all codegen tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl codegen -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add codegen/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#1018): RecordEmitter + CasehubRecordCodegen — emit Java records from schema+mapping

Refs #1018"
```

## Batch 2: Mapping File + Build Wiring

### Task 3: Write the complete mapping file

**Files:**
- Create: `schema/src/main/resources/schema/yaml-record-mappings.yaml`
- Test: Run generator against real schema, compare output to hand-written records

**Interfaces:**
- Consumes: `CasehubRecordCodegen.main(String[])` from Task 2
- Produces: `yaml-record-mappings.yaml` — complete mapping for all 44 record types

- [ ] **Step 1: Write the mapping file**

Create `schema/src/main/resources/schema/yaml-record-mappings.yaml` covering all 44 record types. Audit each hand-written record file to extract the correct field list, type overrides, aliases, and extra fields.

The mapping file must declare every record type currently in `api/src/main/java/io/casehub/api/model/converter/yaml/Yaml*.java`. Use the hand-written records as the source of truth for field names, types, and annotations.

- [ ] **Step 2: Run the generator against the real schema**

```bash
mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml
java -cp codegen/target/classes:$(mvn dependency:build-classpath -pl codegen -q -DincludeScope=runtime -Dmdep.outputFile=/dev/stdout -f /Users/mdproctor/claude/casehub/engine/pom.xml) io.casehub.codegen.CasehubRecordCodegen schema/src/main/resources/schema/CaseDefinition.yaml schema/src/main/resources/schema/yaml-record-mappings.yaml /tmp/yaml-record-gen io.casehub.api.model.converter.yaml
```

- [ ] **Step 3: Compare generated output to hand-written records**

For each generated file, compare the record component list (names, types, annotations) against the hand-written file. Fix mapping file entries until all 44 records match structurally.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add schema/src/main/resources/schema/yaml-record-mappings.yaml
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#1018): yaml-record-mappings.yaml — complete mapping for 44 record types

Refs #1018"
```

### Task 4: Build wiring and switchover

**Files:**
- Modify: `api/pom.xml` — add exec-maven-plugin + build-helper-maven-plugin
- Delete: `api/src/main/java/io/casehub/api/model/converter/yaml/Yaml*.java` (44 files, NOT `JsonNodeForEachAdapter.java`)

**Interfaces:**
- Consumes: `CasehubRecordCodegen.main(String[])` from Task 2, `yaml-record-mappings.yaml` from Task 3
- Produces: Generated records at `api/target/generated-sources/yaml-records/`

- [ ] **Step 1: Add exec-maven-plugin to api/pom.xml**

Add the `generate-yaml-records` execution in the `<build><plugins>` section, and the `add-yaml-record-sources` execution for build-helper-maven-plugin.

- [ ] **Step 2: Run the generation to verify it works**

Run: `mvn generate-sources -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: "Generated 44 record(s)" message, files in `api/target/generated-sources/yaml-records/`

- [ ] **Step 3: Delete hand-written record files**

Delete all 44 `Yaml*.java` files from `api/src/main/java/io/casehub/api/model/converter/yaml/`, keeping `JsonNodeForEachAdapter.java`.

Use `ide_refactor_safe_delete` for each file — this confirms no other files reference the source location (the generated files at the same package will satisfy all imports).

- [ ] **Step 4: Run full api test suite**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: All 1372 tests PASS

- [ ] **Step 5: Run full reactor build**

Run: `mvn install -DskipTests -q -f /Users/mdproctor/claude/casehub/engine/pom.xml && TESTCONTAINERS_RYUK_DISABLED=true mvn test -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: Full build passes — no downstream modules broken

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/pom.xml
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/model/converter/yaml/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#1018): wire record generation into api build, delete hand-written records

44 Yaml*.java files replaced by generated records from CaseDefinition.yaml + yaml-record-mappings.yaml.
JsonNodeForEachAdapter.java retained (utility, not a record).

Closes #1018"
```

## References

- `specs/issue-1018-schema-yaml-record-gen/2026-09-01-schema-yaml-record-gen-design.md` — design spec
- `specs/issue-1018-schema-yaml-record-gen/decisions.md` — D1-D4 decisions
- `codegen/src/main/java/io/casehub/codegen/CasehubCodegen.java` — existing POJO codegen pattern
- `schema/pom.xml:42-91` — existing exec-maven-plugin wiring
- `schema/src/main/resources/schema/CaseDefinition.yaml` — JSON Schema source
- `api/src/main/java/io/casehub/api/model/converter/yaml/` — 44 hand-written records (baseline)
- `api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` — consumer
- engine#1015 — yaml-core record adoption (created the hand-written records)
- engine#1017 — parent epic
