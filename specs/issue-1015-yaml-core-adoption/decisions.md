## D1: Adoption pattern

**Choice:** Full desiredstate pattern — plain Jackson records for field mapping, `@JsonDeserialize` annotations for polymorphic fields, thin converters for domain transforms
**Alternatives:**
- yaml-core integration only — smaller scope but doesn't reduce deserializer boilerplate
- Both phased — two migration passes through the pipeline
**Rationale:** Eliminates ~40 boilerplate fields across 4 deserializers. Records make the YAML shape self-documenting. The 6 polymorphic deserializers stay as annotation-driven custom logic.
**Trade-offs:** Larger scope. All existing tests need updating.
**Sources:** desiredstate/yaml/runtime (zero custom deserializers), CaseDefinitionDeserializer.java (793 lines)
**Exploration:** quick
**Status:** captured

## D2: PostProcessor conversion

**Choice:** Convert PostProcessor to operate on deserialized records instead of raw JsonNode
**Alternatives:**
- Keep as JsonNode PostProcessor — two-pass pipeline, maintains raw YAML access
- Hybrid — move some fields to records, keep worker function probing on JsonNode
**Rationale:** Full alignment with desiredstate pattern. Worker function blocks (agent:, react:, mcp:, a2a:, do:) become record fields with @JsonDeserialize. GOAP shorthand fields (cost, effect, softDependency) are record fields too. Converter maps them to domain types.
**Trade-offs:** Worker function provider discovery currently probes raw YAML — needs redesign as record-level dispatch.
**Depends on:** D1 (adoption pattern)
**Sources:** CaseDefinitionPostProcessor.java (472 lines)
**Exploration:** quick
**Status:** captured

## D3: Delete CaseDefinitionSpec

**Choice:** Delete CaseDefinitionSpec — YAML records replace it entirely
**Alternatives:**
- Keep CaseDefinitionSpec as adapter — preserves backward compat for direct users
**Rationale:** One less layer. Records are the YAML shape, converter maps directly to CaseDefinition. CaseDefinition stays as-is (the domain model consumers use).
**Trade-offs:** Anything using CaseDefinitionSpec directly needs updating.
**Depends on:** D1 (adoption pattern)
**Sources:** CaseDefinitionSpec.java (33 fields)
**Exploration:** quick
**Status:** captured

## D4: yaml-core integration scope

**Choice:** Include VariableResolver and ForEachExpander in this issue
**Alternatives:**
- Defer to follow-on — tighter scope but two passes through the pipeline
- VariableResolver only — defer ForEachExpander as new user-facing capability
**Rationale:** Full alignment with desiredstate pattern in a single pass. VariableResolver replaces use block handling. ForEachExpander enables template expansion for bindings and workers. Schema additions for iterations: and forEach:.
**Trade-offs:** Larger issue scope. ForEachExpander adds new YAML surface.
**Depends on:** D1 (adoption pattern)
**Sources:** casehub-platform/yaml-core (VariableResolver, ForEachExpander, Truthiness)
**Exploration:** quick
**Status:** captured

## D5: Polymorphic deserializer placement

**Choice:** Keep as standalone deserializer classes, annotated on record fields via @JsonDeserialize
**Alternatives:**
- Inline as static factories on target types — mixes domain model with deserialization
**Rationale:** Deserializers stay in deser/ package. Minimal change to working polymorphic logic. CaseDefinitionModule no longer registers them — Jackson discovers via annotations.
**Trade-offs:** More classes than the inline approach.
**Depends on:** D1 (adoption pattern)
**Sources:** TriggerDeserializer, ExpressionEvaluatorDeserializer, GoalExpressionDeserializer, CaseCompletionDeserializer, AdaptationConfigDeserializer, SubCaseMappingDeserializer
**Exploration:** quick
**Status:** captured

## D6: Test migration strategy

**Choice:** Rewrite tests against records — existing YAML-string tests become integration tests
**Alternatives:**
- Keep existing YAML-string tests — safest but most test code
- Replace with contract tests — generated from schema examples
**Rationale:** Tests construct YAML records directly for fast unit tests of converter logic. Existing YAML-string tests verify end-to-end loading as integration tests. Better coverage of the converter layer.
**Trade-offs:** Significant test rewrite effort alongside production code changes.
**Depends on:** D1 (adoption pattern)
**Sources:** 1345 existing api tests
**Exploration:** quick
**Status:** captured
