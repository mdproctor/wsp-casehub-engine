## D1: Scope — all three LLM invocations become loops

**Choice:** All three LLM strategies (routing, aggregation, provisioning) become AgentDecisionLoop instances within CentralisedTopology
**Alternatives:**
- Two loops only (routing + aggregation) — provisioning stays standalone. Simpler but leaves the topology incomplete; provisioning is a coordination concern.
- Two loops + provisioning stub — incremental but defers the provisioning wiring to a later issue, risking a half-migrated state.
**Rationale:** The issue explicitly calls for three loops. Provisioning is a coordination decision and belongs in the topology alongside routing and aggregation.
**Trade-offs:** Requires a provisioning ContentSummariser adapter and mapping ProvisioningAdvice to the DecisionOutcome hierarchy.
**Sources:** agency#381 issue body, SwarmProvisioningAdvisorImpl.java, CentralisedTopology.configure() (already defines three descriptors)
**Exploration:** quick
**Status:** captured

## D2: Provisioning outcome mapping — WriteBack

**Choice:** Map ProvisioningAdvice (shouldProvision + reasoning) to DecisionOutcome.WriteBack using signal deposits and context writes
**Alternatives:**
- New DecisionOutcome.Provision variant — type-safe but requires changing the sealed interface in engine-api (cross-repo change to a foundation type).
- Map to DecisionOutcome.Route — provisioning is arguably resource allocation, but the semantic fit is poor and Route wraps RoutingDecision which doesn't carry provisioning data.
**Rationale:** WriteBack carries signal deposits (provision/don't) and context writes (reasoning). Natural fit without new types. The executor can interpret WriteBack from the provisioning loop by checking the loop name.
**Trade-offs:** Less type-safe than a dedicated variant — the executor dispatches on loop identity rather than outcome type. Acceptable given provisioning is the only loop using WriteBack this way.
**Sources:** DecisionOutcome.java (sealed hierarchy), SwarmProvisioningAdvisorImpl.advise(), SignalDeposit record, ContextWrite record
**Exploration:** quick
**Status:** captured

## D3: LlmSwarmCoordinator — eliminate entirely

**Choice:** Delete LlmSwarmCoordinator class, remove build() from SwarmCoordinatorBuilder, migrate tests to CentralisedTopology
**Alternatives:**
- Thin delegation shell — keep LlmSwarmCoordinator wrapping CentralisedTopology. Preserves backward compatibility but the class has no unique behavior after refactoring.
- Deprecate + delegate — mark as @Deprecated, remove later. Defers cleanup but LlmSwarmCoordinator has only 3 references (builder, 4 tests) so the deferral has no practical benefit.
**Rationale:** After the refactoring, assembleCoordinatorPrompt() is redundant (SectionAwareContentSummariser handles prompt assembly in the loop pipeline). All component holding is covered by CentralisedTopology or TopologyContext. The class has 3 references — elimination is low-risk.
**Trade-offs:** Removing build() from SwarmCoordinatorBuilder is a breaking API change. Acceptable because buildTopology() is the replacement and the only external consumer (PatternCompiler) creates SwarmRoutingStrategy directly, not through the coordinator.
**Sources:** LlmSwarmCoordinator.java, SwarmCoordinatorBuilder.java (build() vs buildTopology()), LlmSwarmCoordinatorTest.java
**Exploration:** deep-analysis
**Status:** captured

## D4: Strategy adaptation — adapter classes

**Choice:** Create three ContentSummariser adapter classes (RoutingReasoner, AggregationReasoner, ProvisioningReasoner) that wrap the existing LLM strategies
**Alternatives:**
- Modify strategies directly to implement ContentSummariser — breaks PatternCompiler and other consumers of SwarmRoutingStrategy as RoutingStrategy. Forces domain-specific classes to know about the generic observation pipeline.
- Lambdas in configure() — compact but untestable in isolation. The adaptation logic (extracting domain data from ObservationEvents, building contexts, mapping results) is meaningful enough to deserve isolation.
**Rationale:** Each adapter translates between the generic observation pipeline (ObservationEvent → DecisionOutcome) and domain-specific strategy interfaces (RoutingContext → RoutingDecision, etc.). Existing strategies and their consumers remain untouched.
**Trade-offs:** Three new classes. Each is thin (~30-50 lines) but adds to the class count. Justified by testability and separation of concerns.
**Sources:** ContentSummariser.java, SwarmRoutingStrategy.route(), CollectiveReasoningAggregation.aggregate(), SwarmProvisioningAdvisorImpl.advise(), DecisionLoopFactory.create() signature
**Exploration:** deep-analysis
**Status:** captured

## D5: Loop ownership — topology holds loops, CoordinationPlan unchanged

**Choice:** CentralisedTopology creates AgentDecisionLoops during configure() using TopologyContext.loopFactory(), stores them internally, exposes via loops() accessor. CoordinationPlan stays as-is (descriptors + executor + pool).
**Alternatives:**
- Evolve CoordinationPlan to hold loops — replace List<DecisionLoopDescriptor> with List<AgentDecisionLoop>. Consistent with plan already holding concrete objects, but wrong because the topology needs to hold the loops anyway (onSwarmEvent feeds observations into them).
- Keep descriptors, add reasoner map to plan — more indirection, no clear benefit. Runtime still needs to call factory.create() with the reasoners.
**Rationale:** CentralisedTopology.onSwarmEvent() must feed observations into the loops, so the topology must hold them. The loops are created from descriptors + adapters + the factory provided by TopologyContext. CoordinationPlan's descriptors remain for test inspection. The factory's shared EventStreamBus routes outcomes to the executor.
**Trade-offs:** CentralisedTopology becomes stateful beyond just SwarmContext — it now holds AgentDecisionLoop instances created during configure(). This means configure() must be called before onSwarmEvent() feeds observations. Acceptable since configure() is the initialisation step.
**Depends on:** D4 (adapter classes provide the ContentSummarisers for factory.create())
**Sources:** DecisionLoopFactory.java (constructor takes EventStreamBus, create() takes reasoner), TopologyContext.java (provides loopFactory), CentralisedTopology.onSwarmEvent(), CentralisedTopologyTest.java (asserts on descriptor properties)
**Exploration:** deep-analysis
**Status:** captured
