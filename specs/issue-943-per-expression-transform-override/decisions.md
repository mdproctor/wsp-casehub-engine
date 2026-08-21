## D1: Full scope — all projection call sites

**Choice:** All evalJqAsJsonNode/evalJqAsMap call sites are converted to use the expression registry. No hardcoded JQ anywhere in the projection path.
**Alternatives:**
- Narrow scope (Capability/Binding/Agent only) — leaves SubCase and Orchestrator as JQ-only; creates inconsistency
**Rationale:** Unified expression dispatch eliminates the JQ assumption from every data transform path. Any future expression language gets projections for free.
**Trade-offs:** Larger change surface (5 call sites vs 3), but all follow the same mechanical pattern.
**Sources:** WorkerScheduleEventHandler:114, CaseContextChangedEventHandler:877+976, DefaultWorkOrchestrator:219, AgentBuilder:157-162
**Exploration:** quick
**Status:** captured

## D2: Store resolved evaluators on CapabilityTarget

**Choice:** Expand CapabilityTarget record with `inputProjection` and `outputProjection` ExpressionEvaluator fields. Binding.inputProjectionOverride also becomes ExpressionEvaluator. Foundation tier (worker-api) untouched.
**Alternatives:**
- Resolve at runtime from Strings — creates evaluators repeatedly, adds *Lang fields to foundation tier
- Shadow fields alongside Strings — dual state, divergence risk
**Rationale:** CapabilityTarget is the engine-api bridge between foundation-tier Capability (raw declaration) and engine-tier evaluation (registry dispatch). The evaluator IS the resolved projection. Same pattern as how Binding already stores resolved ExpressionEvaluator for `when` conditions. No foundation-tier change.
**Trade-offs:** CapabilityTarget grows from 1 to 3 record fields (additive, minor).
**Sources:** CapabilityTarget.java:21, ExpressionEngine.java:154, ExpressionEngineRegistry.java:110
**Exploration:** deep-analysis
**Status:** captured

## D3: Agent projections resolved in converter, not builder

**Choice:** AgentConverter.toApiAgent() resolves projection YAML via resolveExpression(), wraps the evaluator + registry into a UnaryOperator<JsonNode> lambda, passes to AgentBuilder.inputTransformer(). Agent stays a pure execution unit.
**Alternatives:**
- Add ExpressionEvaluator overload to AgentBuilder — builder would need registry access, mixing concerns
**Rationale:** Agent is an execution boundary. Language-awareness belongs in the converter/mapper layer where the registry is available. AgentBuilder.inputProjection(String) remains for Java DSL backward compat (JQ-only).
**Trade-offs:** Agent cannot introspect what language its projection uses (not needed — it just applies a function).
**Depends on:** D2 (CapabilityTarget carries evaluators — same resolveExpression utility used)
**Sources:** AgentConverter.java:45-46, AgentBuilder.java:147-163
**Exploration:** quick
**Status:** captured
