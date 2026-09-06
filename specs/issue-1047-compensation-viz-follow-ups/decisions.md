# Decisions — #1047 Binding Graph Projection

## D1: Graph scope — generalize from compensation to full binding graph

**Choice:** Replace `CompensationGraphProjection` with a generalized `BindingGraphProjection` that supports typed edges (COMPENSATION, DATA_FLOW, TRIGGER_DEPENDENCY). `CaseDefinitionType` gets a `bindingGraph` field replacing `compensationGraph`.
**Alternatives:**
- Extend CompensationGraph — add data flow edges to existing types. Quick but naming becomes misleading.
- Both as separate fields — keep compensationGraph as-is, add separate bindingGraph. Redundant nodes.
**Rationale:** Pre-release platform, breaking changes cost nothing. A design-time viewer needs all edge types in one graph. The compensation-specific types were narrowly scoped for the initial saga work (#390); the follow-up naturally generalizes.
**Trade-offs:** Breaks any consumer of the `compensationGraph` GraphQL field. Runtime compensation queries (timeline, chain) are unaffected — they're separate concerns.
**Sources:** `CompensationGraphProjection.java`, `CaseDefinitionType.java`, issue #1047 body ("Alternatively: a broader bindingGraph field")
**Exploration:** quick
**Status:** captured

## D2: Trigger dependency edges via requiredKeys

**Choice:** Add `requiredKeys: Set<String>` to `Binding` as the symmetric counterpart to `producedKeys`. Trigger dependency edges computed from `A.producedKeys ∩ B.requiredKeys ≠ ∅`.
**Alternatives:**
- Defer trigger edges entirely — ship compensation + data flow edges only. Leaves the producedKeys/requiredKeys asymmetry unfixed.
- Parse JQ expressions — extract referenced paths from trigger conditions. Fragile because JQ is a full language (pipes, functions, conditionals make static path extraction a compiler problem).
- Coarse producedKeys-only inference — connect every producer to every ContextChangeTrigger binding. Not fragile but very noisy.
**Rationale:** Fixes the producedKeys/requiredKeys declaration asymmetry. Parallels the existing GOAP effects/preconditions model. Explicit, precise, non-fragile. Useful beyond visualization — engine could use for validation and scheduling.
**Trade-offs:** New field on Binding — YAML schema, builder, accessors. Only useful when both sides declare keys; bindings without requiredKeys produce no trigger edges.
**Sources:** `Binding.java` (producedKeys field), GOAP `GoapAction` (effects/preconditions pattern), `CaseDefinitionYamlMapper`
**Exploration:** deep-analysis
**Status:** captured
