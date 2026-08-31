# HANDOFF — casehub-engine (#1015)

## Session Summary (2026-08-31)

yaml-core record pattern adoption (#1015). Batch 2 (Converter + Wiring) now complete. The `YamlCaseDefinitionConverter` converts YAML records to `CaseDefinition` domain objects, replacing both `CaseDefinitionDeserializer` (793 lines) and `CaseDefinitionPostProcessor` (472 lines). `CaseDefinitionYamlMapper` now uses the records+converter path. All 239 mapper tests and 1390/1391 api tests pass (1 pre-existing failure in deprecated `HumanTaskTargetTest`).

## Progress

- **Batch 1 (Records):** DONE — 32+ record files in `io.casehub.api.model.converter.yaml`
- **Batch 2 (Converter + Wiring):** DONE
  - `YamlCaseDefinitionConverter` — all domain transforms ported
  - `CaseDefinitionYamlMapper` — wired to records+converter path
  - Record fixes: field name aliases, structural corrections, @JsonDeserialize cleanup
  - ExpressionLang context propagation via ObjectReader attributes
  - HumanTask validation ported to converter
- **Batch 3 (Cleanup):** NOT STARTED — delete old deserializers, inline CaseDefinitionSpec
- **Batch 4 (yaml-core):** NOT STARTED — VariableResolver + ForEachExpander integration

## Key Decisions

1. Polymorphic `@JsonDeserialize` annotations REMOVED from records for `ExpressionEvaluatorDeserializer` (no no-arg constructor). Module registration handles `ExpressionEvaluator` type automatically.
2. Other polymorphic deserializers (Trigger, CaseCompletion, etc.) still registered in `CaseDefinitionModule` for the records path.
3. Worker function wiring converts `YamlWorker` to `JsonNode` via `ObjectMapper.valueToTree()` for provider dispatch compatibility.
4. `YamlCaseSpec` gained `workers` and `bindings` fields to support existing YAML structure where these are under `spec:`.
5. `ExpressionEvaluatorDeserializer.EXPRESSION_LANG_KEY` made public for mapper access.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-1015-yaml-core-adoption/2026-08-31-yaml-core-adoption-design.md` |
| Decisions | `specs/issue-1015-yaml-core-adoption/decisions.md` |
| Implementation plan | `plans/2026-08-31-yaml-core-adoption.md` |
| Journal | `JOURNAL.md` |
| Records package | `proj/api/src/main/java/io/casehub/api/model/converter/yaml/` |
| Converter | `proj/api/src/main/java/io/casehub/api/model/converter/YamlCaseDefinitionConverter.java` |
