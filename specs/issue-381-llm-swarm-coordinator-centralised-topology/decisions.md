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

## D2: Provisioning outcome mapping — DecisionOutcome.Provision variant

**Choice:** Add a `record Provision(ProvisioningAdvice advice) implements DecisionOutcome` variant to the sealed interface
**Alternatives:**
- WriteBack mapping using signal deposits and context writes — loses type safety for all four ProvisioningAdvice fields, requires loop-identity dispatching which is architecturally impossible (executor dispatches on type only, EventStreamBus carries no source metadata, no loop name in executor signature).
- Map to DecisionOutcome.Route — provisioning is arguably resource allocation, but Route wraps RoutingDecision which doesn't carry provisioning data.
**Rationale:** DecisionOutcome is in agency-core (`io.casehub.agency.orchestration.loop.DecisionOutcome`), not engine-api — adding a variant is a same-repo change with the same blast radius as any other modification to this file. A dedicated Provision variant gives exhaustive pattern matching in the executor (`case Provision p -> executeProvisioning(p)`), carries the full typed ProvisioningAdvice record directly (shouldProvision, adjustedCapabilities, adjustedPolicy, reasoning), and eliminates the need for loop-identity dispatching.
**Trade-offs:** Adds a new variant to the sealed interface. All existing switch expressions on DecisionOutcome must add a Provision case. This is mechanical and the compile-time enforcement is the point — it forces every caller to be explicit about provisioning.
**Sources:** DecisionOutcome.java (sealed hierarchy in agency-core), SwarmProvisioningAdvisor.ProvisioningAdvice (engine-api — 4 typed fields), DecisionOutcomeExecutor.execute() signature (no loop name param), EventStreamBus.subscribe() (Predicate<E> filter on payload only)
**Exploration:** quick → revised after review
**Status:** revised — original WriteBack mapping rejected: false premise about DecisionOutcome location (agency-core not engine-api), loop-identity dispatching architecturally impossible, typed fields silently dropped

## D3: LlmSwarmCoordinator — eliminate entirely

**Choice:** Delete LlmSwarmCoordinator class, remove build() from SwarmCoordinatorBuilder
**Alternatives:**
- Thin delegation shell — keep LlmSwarmCoordinator wrapping CentralisedTopology. Preserves backward compatibility but the class has no unique behavior after refactoring.
- Deprecate + delegate — mark as @Deprecated, remove later. Defers cleanup but LlmSwarmCoordinator has only 2 references so the deferral has no practical benefit.
**Rationale:** After the refactoring, assembleCoordinatorPrompt() is redundant (SectionAwareContentSummariser is designed to handle prompt assembly in the loop pipeline — the current implementation has a bug where promptContext is built but not passed to the delegate, which must be fixed during implementation). All component holding is covered by CentralisedTopology or TopologyContext. LlmSwarmCoordinator has exactly 2 references — both in SwarmCoordinatorBuilder.build() (return type at line 84, constructor at line 114). LlmSwarmCoordinatorTest does not exist. Elimination is trivially safe.
**Trade-offs:** Removing build() from SwarmCoordinatorBuilder is a breaking API change. Acceptable because build() has 0 callers (dead code) and buildTopology() is the replacement.
**Sources:** LlmSwarmCoordinator.java (2 refs in SwarmCoordinatorBuilder.build()), SwarmCoordinatorBuilder.java (build() has 0 callers — dead code), SectionAwareContentSummariser.java (summarise() bug — promptContext not forwarded to delegate)
**Exploration:** deep-analysis
**Status:** revised — corrected evidence (2 references not 3, PatternCompiler does not exist, LlmSwarmCoordinatorTest does not exist, build() has 0 callers)

## D4: Strategy adaptation — adapter classes

**Choice:** Create three ContentSummariser adapter classes (RoutingReasoner, AggregationReasoner, ProvisioningReasoner) that wrap the existing LLM strategies
**Alternatives:**
- Modify strategies directly to implement ContentSummariser — breaks consumers of SwarmRoutingStrategy as RoutingStrategy (e.g. SwarmCoordinatorBuilder.buildTopology()). Forces domain-specific classes to know about the generic observation pipeline.
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
**Depends on:** D4 (adapter classes provide the ContentSummarisers for factory.create()), D2 (Provision variant ensures outcome type distinguishes source loop)
**Sources:** DecisionLoopFactory.java (constructor takes EventStreamBus, create() takes reasoner), TopologyContext.java (provides loopFactory), CentralisedTopology.onSwarmEvent(), CentralisedTopologyTest.java (asserts on descriptor properties)
**Exploration:** deep-analysis
**Status:** captured

## D6: ObservationEvent as untyped container — type safety at adapter boundary

**Choice:** Keep ObservationEvent as `record ObservationEvent(String key, Object value, Instant timestamp)` with type safety enforced at the adapter boundary (D4 adapters)
**Alternatives:**
- Typed observation variants — sealed interface with RoutingObservation, AggregationObservation, ProvisioningObservation. Gives compile-time safety but requires parameterizing AgentDecisionLoop, DecisionLoopFactory, SummarisationRunner, and EventStreamBus on observation type.
**Rationale:** The adapter pattern (D4) creates a type boundary — each adapter receives typed domain inputs and produces ObservationEvents. Since each loop only receives observations from its own adapter, there is no cross-loop type confusion risk. The key field provides a runtime discriminator sufficient for filtering. Parameterizing the entire loop framework on observation type would propagate generics through at least 5 classes (AgentDecisionLoop, DecisionLoopFactory, SummarisationRunner, ContentSummariser, EventStreamBus) for minimal benefit — the observation type is consumed only by the adapter's own summarise() method.
**Trade-offs:** Runtime casts in adapter summarise() methods instead of compile-time type checking on ObservationEvent. Acceptable because each adapter controls both the production and consumption of its own events within a single class.
**Sources:** ObservationEvent.java (String key, Object value), ContentSummariser.java, AgentDecisionLoop.java, DecisionLoopFactory.java
**Exploration:** surfaced-by-review
**Status:** captured

## D7: EventStreamBus — no source metadata required

**Choice:** Use the shared EventStreamBus<DecisionOutcome> without source metadata on LevelEvent
**Alternatives:**
- Add source metadata to LevelEvent — e.g., a source field identifying the producing loop. Requires changing the LevelEvent record in casehub-agency-summarisation-api (cross-repo change to a foundation type).
- Per-loop EventStreamBus instances — each loop gets its own bus with separate subscriptions.
**Rationale:** With D2 revised to use a Provision variant, each loop produces a distinct DecisionOutcome variant (Route, Aggregate, Provision). EventStreamBus.subscribe() takes a Predicate<E> filter on the payload — outcome type IS the source discriminator. No source metadata is needed.
**Trade-offs:** If two loops ever produce the same outcome variant, source ambiguity returns. Currently not a concern — Route is routing-only, Aggregate is aggregation-only, Provision is provisioning-only. WriteBack remains available but is no longer overloaded for provisioning.
**Depends on:** D2 (Provision variant eliminates the only source of WriteBack ambiguity)
**Sources:** EventStreamBus.java (subscribe takes Predicate<E> filter), LevelEvent.java (payload, timestamp, level, tenancyId — no source field), DecisionOutcome.java (5 sealed variants after adding Provision)
**Exploration:** surfaced-by-review
**Status:** captured
