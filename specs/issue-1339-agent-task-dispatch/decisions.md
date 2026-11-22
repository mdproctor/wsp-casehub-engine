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

**Choice:** Extend AgentSlotConfig with optional capability and domain fields. Handler branches: slot present → query by slot; capability present → query by capability+domain.
**Alternatives:**
- New AgentCapabilityConfig record — cleaner separation but introduces a parallel type hierarchy with duplicated fields (model, userMessageTemplate, projections).
- Sealed DeferredAgentConfig interface — most type-safe but requires refactoring AgentWorkerFunction and AgentWorkerFunctionHandler for a sealed hierarchy. Heavy for two variants.
**Rationale:** AgentSlotConfig already carries all the execution config fields. Adding two optional fields (capability, domain) is minimal change. The handler already has a deferred path that checks `agent.isDeferred()` — adding a branch within that path is natural.
**Trade-offs:** AgentSlotConfig becomes overloaded — slot-based and capability-based configs share one type. A validation invariant is needed (exactly one of slot/capability must be non-null). This is an acceptable trade-off: fewer types to maintain, single deferred execution path with a branch.
**Sources:** AgentSlotConfig.java:20-29, AgentWorkerFunctionHandler.java:102-106 (isDeferred check)
**Exploration:** quick
**Depends on:** D1 (synthetic Worker uses this config)
**Status:** captured

## D3: Synthetic Worker creation point

**Choice:** CaseDefinition converter — when a binding has AgentTaskTarget, generate the synthetic Worker in the YAML-to-CaseDefinition conversion step. Worker is in the definition from the start.
**Alternatives:**
- WorkerFunctionProvider — providers work from raw YAML worker nodes, not from binding targets. Would need a different trigger mechanism that breaks the provider contract.
- Post-registration hook — scan bindings after CaseDefinition is registered and add synthetic Workers. Requires mutable worker list and late-binding lifecycle complexity.
**Rationale:** The converter already creates Workers from YAML (convertWorkers method) and resolves binding targets (resolveTarget method). Adding synthetic Worker creation alongside target resolution keeps the lifecycle simple: by the time CaseDefinition is registered, all Workers exist.
**Trade-offs:** Couples synthetic Worker creation to the YAML conversion path. Programmatic CaseDefinition construction would need to create synthetic Workers separately. Acceptable because YAML is the primary authoring path.
**Sources:** YamlCaseDefinitionConverter.java:857-875 (resolveTarget), YamlCaseDefinitionConverter.java:497-612 (convertWorkers)
**Exploration:** quick
**Depends on:** D1 (synthetic Worker approach), D2 (config type)
**Status:** captured

## D4: Input/output mapping source

**Choice:** Use AgentTaskTarget's ExpressionEvaluator fields directly. The handler reads mappings from the BindingTarget itself (passed through the synthetic Worker's metadata or as evaluator references), not from AgentSlotConfig's JQ string fields.
**Alternatives:**
- Convert to JQ strings in config — extract JQ expressions from AgentTaskTarget's evaluators and store as strings in AgentSlotConfig's existing inputProjection/outputProjection fields. Reuses existing path but couples to JQ and loses evaluator type information.
- Support both paths — AgentSlotConfig carries optional evaluators alongside optional JQ strings. Handler checks evaluators first, falls back to JQ strings. Most flexible but most complex, violates YAGNI.
**Rationale:** AgentTaskTarget already carries typed ExpressionEvaluator instances (inputMapping/outputMapping). These are richer than raw JQ strings — they support the ExpressionEngineRegistry's type system. Converting to strings would lose type information and couple to JQ.
**Trade-offs:** The handler needs access to the original AgentTaskTarget or its evaluators at execution time. This means either the synthetic Worker carries the evaluators, or the handler looks up the binding target from the execution context. Slight increase in handler complexity.
**Sources:** AgentTaskTarget.java:44-50 (evaluator fields), AgentSlotConfig.java:24-25 (JQ string fields)
**Exploration:** quick
**Depends on:** D2 (config carries capability+domain, not mappings)
**Status:** captured

## D5: Match selection strategy

**Choice:** String enum field on AgentTaskTarget (matchStrategy, nullable, default "first"). Values: "first" (default), "best-domain", "round-robin". Interpreted by the handler at dispatch time.
**Alternatives:**
- SPI interface (AgentMatchSelector) — CDI-discovered named implementations. Most extensible but heavy for three known strategies. Adds a new SPI where a switch statement suffices.
- Functional on AgentSlotConfig — Function<List<AgentMatch>, AgentMatch> selector. Flexible but not serialisable from YAML, breaks the config's value-object semantics.
**Rationale:** A string field is serialisable from YAML, doesn't require new types, and is extensible by adding new string values without changing the type hierarchy. The handler interprets the string with a switch — straightforward and testable.
**Trade-offs:** No compile-time exhaustiveness check on strategy values. Invalid strategy strings are runtime errors. Acceptable because strategies are a small, stable set and validation can be added at parse time.
**Sources:** AgentTaskTarget.java:22-97 (builder pattern), AgentRegistry.java:22 (find returns ordered list)
**Exploration:** quick
**Depends on:** D1 (dispatch model determines where strategy is applied)
**Status:** captured
