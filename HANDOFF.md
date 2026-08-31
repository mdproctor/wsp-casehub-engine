# HANDOFF — casehub-engine (#1015)

## Session Summary (2026-08-31)

yaml-core record pattern adoption (#1015). Designed and started implementing the migration from hand-coded Jackson deserializers (~2587 lines) to plain records + thin converter, following the desiredstate pattern. Batch 1 complete — 32 YAML record files created and compiling. Also closed #985 (already implemented), updated governed yield tracking (#1009–#1013 + blocks#220–#221 all landed).

## Progress

- **Batch 1 (Records):** DONE — 32 record files in `io.casehub.api.model.converter.yaml`
- **Batch 2 (Converter + Wiring):** NOT STARTED — `YamlCaseDefinitionConverter` + wire into mapper
- **Batch 3 (Cleanup):** NOT STARTED — delete ~2587 lines, inline CaseDefinitionSpec
- **Batch 4 (yaml-core):** NOT STARTED — VariableResolver + ForEachExpander integration

## Key Decision

The 6 polymorphic deserializers (Trigger, ExpressionEvaluator, GoalExpression, CaseCompletion, AdaptationConfig, SubCaseMapping) stay as `@JsonDeserialize` annotations on record fields — they handle genuine polymorphism that records can't replace.

## References

| Artifact | Path |
|----------|------|
| Design spec | `specs/issue-1015-yaml-core-adoption/2026-08-31-yaml-core-adoption-design.md` |
| Decisions | `specs/issue-1015-yaml-core-adoption/decisions.md` |
| Implementation plan | `plans/2026-08-31-yaml-core-adoption.md` |
| Journal | `JOURNAL.md` |
| Records package | `proj/api/src/main/java/io/casehub/api/model/converter/yaml/` |
