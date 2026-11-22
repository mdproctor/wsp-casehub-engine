## D1: Dispatch model for AgentTaskTarget bindings

**Choice:** Synthetic Worker — create a synthetic Worker with AgentWorkerFunction at CaseDefinition load time for each agent-task binding. BindingExecutorResolver resolves to it; rest of pipeline unchanged.
**Alternatives:**
- Binding-level intercept — extend WorkerExecutionOrchestrator to recognise AgentTaskTarget and dispatch directly via AgentRegistry, bypassing Worker lookup. Changes the core execution pipeline, higher blast radius.
- Dedicated handler class — new AgentTaskDispatchHandler SPI that operates on BindingTarget instead of WorkerFunction. Adds a new SPI layer where the existing one suffices.
**Rationale:** Follows the established slot-based deferred resolution pattern in AgentWorkerFunctionHandler. The execution pipeline stays unchanged — no modifications to WorkerExecutionOrchestrator or DefaultWorkerExecutor. The synthetic Worker is indistinguishable from a slot-based agent Worker at dispatch time.
**Trade-offs:** Synthetic Workers are an abstraction — the agent-task binding doesn't correspond to a user-declared worker. The CaseDefinition's worker list gains entries the user didn't author. This is acceptable because the execution pipeline already treats Workers as an implementation detail.
**Sources:** AgentWorkerFunctionHandler.java:102-146 (deferred slot path), BindingExecutorResolver.java:50 (current stub), WorkerExecutionOrchestrator.java:215-219 (worker lookup by name)
**Exploration:** quick
**Status:** captured

## D2: Config type for capability-based resolution

**Choice:** Sealed `DeferredAgentConfig` interface with `SlotConfig` and `CapabilityConfig` record subtypes. `AgentWorkerFunction.slotConfig()` becomes `deferredConfig()` returning `DeferredAgentConfig`. Handler pattern-matches on the config type.
**Alternatives:**
- Extend AgentSlotConfig with optional capability/domain fields — creates a 10-field record with mutual exclusion invariants, breaks existing `requireNonNull(slot)` contract, naming lie ("SlotConfig" with no slot).
- New AgentCapabilityConfig record (no sealed hierarchy) — cleaner separation but no compile-time exhaustiveness, duplicates fields.
**Rationale:** Sealed interfaces with records are the platform's idiomatic approach to discriminated dispatch (see `BindingTarget` with 7 permits). The compiler's exhaustiveness check in the handler's branch prevents missed variants. Each subtype is self-documenting with only its relevant fields.
**Trade-offs:** Requires refactoring `AgentWorkerFunction` and `AgentWorkerFunctionHandler` — `slotConfig()` → `deferredConfig()`, handler gets a pattern match. This is trivially scoped and improves type safety.
**Sources:** AgentSlotConfig.java:20-29, BindingTarget.java:27-34 (sealed interface precedent), AgentWorkerFunctionHandler.java:102-106
**Exploration:** quick → revised after decision review R1-02
**Depends on:** D1 (synthetic Worker uses this config)
**Status:** revised

## D3: Synthetic Worker creation point

**Choice:** CaseDefinition converter — when a binding has AgentTaskTarget, generate the synthetic Worker in the YAML-to-CaseDefinition conversion step. Synthetic Worker creation utility is a public static method accessible to programmatic CaseDefinition construction.
**Alternatives:**
- WorkerFunctionProvider — providers work from raw YAML worker nodes, not from binding targets. Would need a different trigger mechanism that breaks the provider contract.
- Post-registration hook — scan bindings after CaseDefinition is registered and add synthetic Workers. Requires mutable worker list and late-binding lifecycle complexity.
**Rationale:** The converter already creates Workers from YAML (convertWorkers method) and resolves binding targets (resolveTarget method). Adding synthetic Worker creation alongside target resolution keeps the lifecycle simple: by the time CaseDefinition is registered, all Workers exist.
**Trade-offs:** Couples synthetic Worker creation to the YAML conversion path. Programmatic CaseDefinition construction calls the public utility method directly. Naming convention `__agent-task__<bindingName>` uses a reserved prefix — the converter validates no user-authored Worker uses `__agent-task__` prefix, failing fast on collision.
**Sources:** YamlCaseDefinitionConverter.java:857-875 (resolveTarget), YamlCaseDefinitionConverter.java:497-612 (convertWorkers)
**Exploration:** quick → updated after decision review R1-05, R1-06
**Depends on:** D1 (synthetic Worker approach), D2 (config type)
**Status:** revised

## D4: Input/output mapping source

**Choice:** Use AgentTaskTarget's ExpressionEvaluator fields, carried through `CapabilityConfig`. Evaluation uses `ExpressionEngineRegistry.transform(evaluator, input)`, not direct evaluator method calls. The handler injects `ExpressionEngineRegistry`.
**Alternatives:**
- Convert to JQ strings in config — extract JQ expressions from AgentTaskTarget's evaluators and store as strings. Reuses existing path but couples to JQ and loses evaluator type information.
- Direct evaluator.evaluate(node) — `ExpressionEvaluator` has no such method; evaluation requires the registry. This was an error in the original design.
**Rationale:** AgentTaskTarget carries typed ExpressionEvaluator instances. `ExpressionEngineRegistry` is the platform's standard evaluation path (used by `DefaultWorkerExecutor.applyOutputSchema()` and `DefaultWorkOrchestrator.transformAsMap()`). Injecting it into the handler is consistent.
**Trade-offs:** Adds `ExpressionEngineRegistry` as a new dependency to `AgentWorkerFunctionHandler`. Acceptable — it's already a runtime-tier bean used by the executor.
**Sources:** AgentTaskTarget.java:44-50, ExpressionEngineRegistry (platform API), DefaultWorkerExecutor.java:92 (existing usage pattern)
**Exploration:** quick → updated after decision review R1-03
**Depends on:** D2 (CapabilityConfig carries evaluators)
**Status:** revised

## D5: Match selection strategy

**Choice:** Delegate to existing `AgentSelector` SPI (specifically `EngineAwareAgentSelector` in production). No `matchStrategy` field on `AgentTaskTarget`. The handler queries `AgentRegistry.find()`, then delegates to `AgentSelector.select(candidates, SelectionContext)` for health probing, trust scoring, and routing strategy composition.
**Alternatives:**
- String matchStrategy field on AgentTaskTarget — reinvents the platform's selection pipeline. Bypasses health probing (`CapabilityHealth.probe()`), trust scoring (`TrustScoreSource`), CDI-extensible routing strategies (`LeastLoaded`, `TrustWeighted`, `Semantic`), and org-aware escalation.
- SPI interface per binding — over-engineered for a problem the platform already solves.
**Rationale:** The platform has a layered selection architecture: `AgentRegistry.find()` → `EngineAwareAgentSelector` (health probing + `AgentRoutingStrategy` + trust + org-aware dispatch). This pipeline handles `Selected`, `NoneQualified`, and `Escalated` outcomes. Composing with it gives the capability path the same selection quality as every other dispatch path.
**Trade-offs:** The handler needs to inject `AgentSelector` and handle the `AgentSelection` sealed hierarchy (Selected/NoneQualified/Escalated). This is more code than `matches.get(0)` but correct — capability matching is inherently ambiguous and needs selection intelligence.
**Sources:** AgentSelector.java:5-7, EngineAwareAgentSelector.java:42-78, AgentSelection.java:3-22, SelectionContext.java:5-21, AgentRoutingStrategy.java:46-61
**Exploration:** quick → revised after decision review R1-01
**Depends on:** D1 (dispatch model determines where selection runs)
**Status:** revised

## D6: Model override for capability-based path

**Choice:** `AgentTaskTarget` gains an optional `model` field. When present, it overrides the agent descriptor's model in the `WorkerAgentInvoker`. Parity with slot-based mode where `AgentSlotConfig.model()` can override the descriptor.
**Alternatives:**
- No model override — defer to agent descriptor. Creates a capability gap vs slot-based mode that may surprise users.
**Rationale:** Binding authors may need to specify a model for cost or latency reasons regardless of which agent is selected. This is a one-field addition to the builder.
**Trade-offs:** None significant — the invoker already supports model overrides.
**Sources:** AgentWorkerFunctionHandler.java:122 (slot path model override), AgentTaskTarget.java builder
**Exploration:** quick (surfaced by decision review R1-07)
**Status:** captured
