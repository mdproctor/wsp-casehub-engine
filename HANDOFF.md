# HANDOFF — casehub-engine (#1015)

## Session Summary (2026-08-31)

yaml-core record pattern adoption (#1015). All 4 batches complete. Net result: ~3200 lines of hand-coded deserializers deleted, replaced by 32+ YAML records + `YamlCaseDefinitionConverter`. `CaseDefinitionSpec` inlined into `CaseDefinition`. yaml-core `VariableResolver` and `ForEachExpander` integrated — YAML definitions support `${env.X}` variable resolution and `forEach` template expansion. 1372/1373 api tests pass (1 pre-existing failure in deprecated `HumanTaskTargetTest`). Also fixed yaml-core `VariableResolver` to support deferred `each` prefix (committed to platform repo).

## Progress

- **Batch 1 (Records):** DONE — 32+ record files in `io.casehub.api.model.converter.yaml`
- **Batch 2 (Converter + Wiring):** DONE
  - `YamlCaseDefinitionConverter` — all domain transforms ported
  - `CaseDefinitionYamlMapper` — wired to records+converter path
  - Record fixes: field name aliases, structural corrections, @JsonDeserialize cleanup
  - ExpressionLang context propagation via ObjectReader attributes
  - HumanTask validation ported to converter
- **Batch 3 (Cleanup):** DONE — 12 files deleted, 3186 lines removed, CaseDefinitionSpec inlined
- **Batch 4 (yaml-core):** DONE
  - `JsonNodeForEachAdapter` — adapts raw JsonNode for ForEachExpander
  - `expandForEach()` in mapper — runs when `iterations:` block present
  - `resolveVariables()` + new `load()` overload — `${env.X}` / `${config.X}` resolution
  - yaml-core fix: deferredPrefixes checked before hardcoded `each` prefix
  - 12 new tests (7 forEach + 5 variable resolution)

## Key Decisions

1. Polymorphic `@JsonDeserialize` annotations REMOVED from records for `ExpressionEvaluatorDeserializer` (no no-arg constructor). Module registration handles `ExpressionEvaluator` type automatically.
2. Other polymorphic deserializers (Trigger, CaseCompletion, etc.) still registered in `CaseDefinitionModule` for the records path.
3. Worker function wiring converts `YamlWorker` to `JsonNode` via `ObjectMapper.valueToTree()` for provider dispatch compatibility.
4. `YamlCaseSpec` gained `workers` and `bindings` fields to support existing YAML structure where these are under `spec:`.
5. `ExpressionEvaluatorDeserializer.EXPRESSION_LANG_KEY` made public for mapper access.
6. ForEach expansion operates at JsonNode level (before deserialization) — stamps `${each.*}` into all string fields via `VariableResolver.resolveMap()`.
7. Variable resolution defers `each` prefix (resolved later during forEach expansion). Fixed yaml-core to support this ordering.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-1015-yaml-core-adoption/2026-08-31-yaml-core-adoption-design.md` |
| Decisions | `specs/issue-1015-yaml-core-adoption/decisions.md` |
| Implementation plan | `plans/2026-08-31-yaml-core-adoption.md` |
| Journal | `JOURNAL.md` |
| Records package | `proj/api/src/main/java/io/casehub/api/model/converter/yaml/` |
| Converter | `proj/api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` |
