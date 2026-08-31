# Design Journal — issue-1015-yaml-core-adoption

## 2026-08-31 — Session 1: Design + Record Foundation

### Context Discovery

Started by investigating #985 (YAML humanTask constraints) — found it already implemented. Closed it. This led to examining the YAML pipeline holistically, discovering `yaml-core` in `casehub-platform`, and the desiredstate pattern of zero custom deserializers.

### Key Insight — desiredstate pattern

Desiredstate's YAML module uses plain Jackson records (e.g. `YamlGraph` = 26 lines, `YamlRule` = 22 lines) with zero custom deserializers. Domain logic lives in thin converters. Engine has 2587 lines of hand-coded `if (node.has("X"))` boilerplate across 4 deserializers + a 472-line PostProcessor.

### Survey Results

Surveyed all 10 custom deserializers. Found ~40 fields are pure boilerplate (Jackson can handle automatically), 6 deserializers handle genuine polymorphism (discriminated unions, recursive trees, preset-or-object) that needs custom logic regardless.

### Design Decisions (6, all quick picks)

1. Full desiredstate pattern — records + @JsonDeserialize annotations + converters
2. PostProcessor converts to record-based (no more raw JsonNode)
3. Delete CaseDefinitionSpec — records replace it
4. Include yaml-core VariableResolver + ForEachExpander in scope
5. Keep polymorphic deserializers as standalone classes on annotations
6. Rewrite tests against records

### Implementation Progress

Batch 1 complete (2/2 tasks):
- Task 1: 28 supporting YAML records created and compiling
- Task 2: 4 top-level records (YamlCaseDefinition, YamlCaseSpec, YamlBinding, YamlWorker) created and compiling

32 record files total in `io.casehub.api.model.converter.yaml`. All compile clean.

### What's Next (3 batches remaining)

- **Batch 2**: YamlCaseDefinitionConverter (all 13 domain transform categories) + wire into CaseDefinitionYamlMapper
- **Batch 3**: Delete old code (~2587 lines), inline CaseDefinitionSpec into CaseDefinition
- **Batch 4**: yaml-core integration (VariableResolver + ForEachExpander + schema additions)
