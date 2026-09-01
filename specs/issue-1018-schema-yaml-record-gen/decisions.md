## D1: Target module for generated records

**Choice:** Stay in api module — generate into `api/target/generated-sources/yaml-records`
**Alternatives:**
- New `casehub-engine-yaml-model` module — cleaner separation but adds a reactor module and build dependency
- Stay in codegen module — can't depend on api types without circular deps
**Rationale:** Records reference api-level types (Trigger, CaseCompletion, ExpressionEvaluator). Generating into api avoids cross-module type resolution. Matches how schema module generates `io.casehub.model` POJOs via exec-maven-plugin.
**Trade-offs:** api module's build becomes slightly more complex (exec-maven-plugin execution + build-helper-maven-plugin source addition).
**Sources:** `schema/pom.xml` (existing exec-maven-plugin pattern), `api/src/main/java/io/casehub/api/model/converter/yaml/` (45 hand-written records)
**Exploration:** quick
**Status:** captured

## D2: Generator architecture

**Choice:** Extend codegen module with `CasehubRecordCodegen` main class + `yaml-record-mappings.yaml` mapping file. Run via exec-maven-plugin in api/pom.xml.
**Alternatives:**
- Custom Maven plugin module (`casehub-schema-generator-maven-plugin`) — proper Mojo lifecycle, reusable across repos, but significantly more boilerplate (Mojo annotations, plugin descriptor, integration tests, deployment)
- Schema extensions (`x-yaml-*`) — embeds mapping metadata in JSON Schema, single source of truth, but pollutes schema with Java-specific codegen concerns and conflicts with non-Java consumers (#977 TypeScript)
**Rationale:** Reuses existing build pattern (exec-maven-plugin calling a main class in codegen). codegen module is purpose-built for this. String template generation works well for records (simple structure: component list + compact constructor). No Maven plugin ceremony needed.
**Trade-offs:** String templates are less type-safe than JCodeModel. Mapping file is a second source of truth alongside the schema. But records are structurally simple and the mapping file is the right place for Java-specific concerns.
**Depends on:** D1 (target module)
**Sources:** `codegen/src/main/java/io/casehub/codegen/CasehubCodegen.java` (existing pattern), issue #1018 (recommends Option B — we chose a lighter variant)
**Exploration:** quick
**Status:** captured

## D3: Mapping file format

**Choice:** YAML mapping file (`yaml-record-mappings.yaml`) declaring per-type record name, field overrides, extra fields, and skip patterns
**Alternatives:**
- Java annotations on schema (`x-java-record` extensions) — single file but mixes schema concerns with Java codegen
- Properties file — simple but poor nesting for complex per-field overrides
**Rationale:** YAML is consistent with the schema format. Natural nesting for type → field → override structure. Easy to read, diff, and review. Keeps Java-specific concerns (deserializer class names, import paths) separate from the language-agnostic schema.
**Trade-offs:** Second file to maintain alongside schema. But mapping changes are less frequent than schema changes — they only change when a new polymorphic type is introduced or a field needs a Java-specific annotation.
**Depends on:** D2 (generator architecture)
**Sources:** Hand-written records with `@JsonDeserialize`, `@JsonAlias`, `@JsonProperty` annotations
**Exploration:** quick
**Status:** captured

## D4: Validation strategy

**Choice:** Compile + existing tests — generated records replace hand-written ones; type system detects drift
**Alternatives:**
- Golden-file comparison — diff generated output against committed golden files. Catches formatting changes but adds golden file maintenance.
- Schema-record structural test — programmatic check that every schema property has a matching record component. Catches unmapped new fields but duplicates what the mapping file already declares.
**Rationale:** The converter and all 1372 api tests compile against the record types. If a generated record is missing a field the converter expects, compilation fails. If a field type changes, the converter type-checks fail. The type system IS the drift detector — no separate test infrastructure needed.
**Trade-offs:** Won't catch fields added to the schema but not yet consumed by the converter. But this is the correct behavior — a schema field only matters when the converter uses it, and adding it to the mapping file is the explicit opt-in.
**Depends on:** D1 (target module)
**Sources:** 1372 api module tests, `YamlCaseDefinitionConverter.java`
**Exploration:** quick
**Status:** captured
