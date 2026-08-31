## D1: Nesting depth

**Choice:** Arbitrary nesting — compound tasks can contain other compound tasks
**Alternatives:**
- One level only (root → leaves) — simpler YAML but limits expressiveness
- Two levels max — covers most cases but artificial constraint
**Rationale:** Matches the Java type system where CompoundTask holds DecompositionMethods which produce TaskNodes (leaf or compound). Complex playbooks need multi-level branching.
**Trade-offs:** Deeper nesting makes YAML harder to read; validation needed to prevent unreasonable depth
**Sources:** TaskNode.java sealed interface, DecompositionMethod.java
**Exploration:** quick
**Status:** captured

## D2: YAML placement

**Choice:** Under `spec:` as `spec.decomposition:`
**Alternatives:**
- Top level `decomposition:` — breaks convention that behavioral config lives under spec
- Under `spec.compounds:` — merges distinct concepts (runtime grouping vs planning hierarchy)
**Rationale:** The spec block already owns behavioral configuration (capabilities, goals, compounds, strategies). Consistent placement.
**Trade-offs:** None significant — all alternatives are viable, this follows existing convention
**Sources:** YamlCaseSpec.java, CaseDefinition.java
**Exploration:** quick
**Status:** captured

## D3: Leaf task binding model

**Choice:** Capability reference only (`capability: triage-assessment`)
**Alternatives:**
- Capability + optional overrides (inputProjection, timeout per step) — duplicates binding concerns
- Capability or inline binding — blurs HTN/binding boundary
**Rationale:** Clean separation: HTN declares WHAT to do (task hierarchy), bindings declare HOW (worker dispatch, triggers). Consistent with how the existing binding model works.
**Trade-offs:** Per-step configuration requires a separate binding with the right capability target
**Sources:** Issue #987 proposed shape, Binding.java, CapabilityTarget.java
**Exploration:** quick
**Status:** captured

## D4: Coexistence with decompositionStrategy

**Choice:** HTN coexists as the identity strategy — registered automatically when `spec.decomposition` is present
**Alternatives:**
- HTN replaces decompositionStrategy entirely — can't combine with runtime planners
- HTN as fallback only — hard to reason about priority ordering
**Rationale:** When decompositionStrategy is omitted or "identity", the explicit HTN tree IS the plan. When set to goap/llm/portfolio, the runtime strategy takes precedence. Makes explicit HTN a named strategy ("explicit") that the engine discovers from the CaseDefinition.
**Trade-offs:** Need clear documentation on which takes priority when both are present
**Sources:** DecompositionStrategy.java, CaseDefinition.decompositionStrategy field
**Exploration:** quick
**Status:** captured
