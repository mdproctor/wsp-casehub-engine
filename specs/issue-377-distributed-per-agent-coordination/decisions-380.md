# Decisions — agency#380 CoordinationTopology SPI

## D1: Extend TopologyType, skip CoordinationPattern/AuthorityModel

**Choice:** Add HIERARCHICAL and GRADUATED to the existing TopologyType enum in engine-api. Do not create CoordinationPattern or AuthorityModel as separate types — the issue's mapping table becomes test assertions, not new enums.
**Alternatives:**
- New CoordinationPattern + AuthorityModel enums in engine-api — maximally explicit but introduces two new cross-repo types for what are documentation-level concepts
- Agency-only enums — keeps engine minimal but breaks the "shared topology model" intent
- Documentation-only — no types at all, mapping lives in spec only
**Rationale:** TopologyType already captures topology identity. HIERARCHICAL and GRADUATED are natural extensions of the same axis (how agents are structurally arranged). CoordinationPattern (HIVE/SWARM) and AuthorityModel are classification concepts, not runtime discriminators — no code branches on them.
**Trade-offs:** TopologyType grows from 3 to 5 values. deriveAuthority must be implemented for the new values. Minor — the enum is small and well-tested.
**Sources:** TopologyType.java, engine#1282 issue body §Relationship table, TopologyTypeTest.java
**Exploration:** quick
**Status:** captured

## D2: Rename SwarmTopologyProvider → TopologyProvider

**Choice:** Rename SwarmTopologyProvider to TopologyProvider in engine-api as part of #380 using IntelliJ refactoring. The rename is mechanical — one interface, one impl (CentralisedTopologyProvider), one SpiDefaultBeans reference.
**Alternatives:**
- Extend SwarmTopologyProvider as-is — CoordinationTopology extends the old name; names diverge from issue spec
**Rationale:** engine#1284 (which specified this rename) is closed. The rename is trivial with IntelliJ and the issue spec depends on the new name. Doing it now avoids a naming inconsistency that would need cleanup later.
**Trade-offs:** Cross-repo change in engine, but the rename is mechanical and the only consumer in this slot is CentralisedTopologyProvider.
**Sources:** SwarmTopologyProvider.java, CentralisedTopologyProvider.java, SpiDefaultBeans.java, engine#1284 issue
**Exploration:** quick
**Status:** captured

## D3: Full LlmSwarmCoordinator → CentralisedTopology refactor (absorbs #381)

**Choice:** Fully refactor LlmSwarmCoordinator into CentralisedTopology as part of #380, closing #381. IntelliJ refactoring handles the mechanical rename/move/update-references.
**Alternatives:**
- Skeleton in #380, fill in #381 — defers the real SPI validation to a separate issue; risk of skeleton diverging from actual needs
- Skip for #381 entirely — no concrete reference implementation to validate the SPI shape
**Rationale:** CentralisedTopology is the only topology with existing production code. Refactoring it validates the SPI shape against real code — if the SPI doesn't fit the existing coordinator cleanly, that's a signal the SPI is wrong. Better to discover that during SPI design than in a follow-up issue. IntelliJ refactoring makes the mechanical work trivial.
**Trade-offs:** Larger scope for #380. Accepted — the refactoring IS the SPI validation.
**Sources:** LlmSwarmCoordinator.java, SwarmCoordinatorBuilder.java, SwarmContext.java, all SwarmCoordinator*Test files
**Exploration:** quick
**Status:** captured

## D4: Full configure + events for all 6 topologies

**Choice:** All 6 concrete topologies have working configure() returning real CoordinationPlans AND meaningful onSwarmEvent() handlers that update loop state. Full R&D implementations, not stubs.
**Alternatives:**
- Descriptors + stubs — validates SPI contract but minimal R&D value
- Interface + factory only — validates class hierarchy only, no behavior
**Rationale:** The topologies are R&D starting points. Full implementations give real value for experimentation and validate that the SPI supports diverse coordination strategies. Each topology's event handling exercises different aspects of the SPI.
**Trade-offs:** Significantly more code. More tests needed. Higher risk of implementation details leaking into the SPI design. Mitigated by the CentralisedTopology refactoring (D3) validating the SPI first.
**Depends on:** D3 (CentralisedTopology validates the SPI shape before other implementations)
**Sources:** agency#380 issue body §Implementations, DecisionLoopDescriptor.java, AgentDecisionLoop.java
**Exploration:** quick
**Status:** captured

## D5: HiveTopology bridges via adapter layer

**Choice:** HivePatternAdapter wraps SupervisorBuilder/DebateBuilder/VotingBuilder output and translates their coordination into DecisionOutcome events. HiveTopology's configure() returns a CoordinationPlan whose loop descriptors map to pattern roles. The orchestration pattern (supervisor/debate/voting) is a HiveTopology configuration parameter.
**Alternatives:**
- Dual coordination — runs both a decision loop AND the existing pattern; more complex, preserves pattern semantics exactly but redundant coordination
- Pattern as ContentSummariser — wraps patterns as reasoning strategies inside decision loops; tightest integration but requires refactoring pattern builders
**Rationale:** Adapter layer composes existing code without modifying it. The pattern builders are stable, tested infrastructure — wrapping their output is safer than refactoring their internals. The adapter translates between pattern-specific types and DecisionOutcome, keeping both layers clean.
**Trade-offs:** Adapter adds a translation layer. Pattern-specific nuances may be lost in translation to the generic DecisionOutcome model. Acceptable for R&D — if a pattern's semantics can't be expressed as DecisionOutcome variants, that's valuable feedback on the SPI.
**Sources:** SupervisorBuilder.java, DebateBuilder.java, VotingBuilder.java, DecisionOutcome.java
**Exploration:** quick
**Status:** captured

## D6: GraduatedTopology — mutable wrapper over immutable DecisionScope

**Choice:** GraduatedTopology holds a mutable GraduationState that tracks the current DecisionScope per agent. onFeedback() adjusts scope up/down between STIMULUS_DRIVEN → TASK_ADAPTATION → FULL_AUTONOMY. The loop's DecisionContext is rebuilt with the new scope on each tick. DecisionScope enum stays immutable.
**Alternatives:**
- DecisionScope as sealed interface — refactors enum to sealed interface with GraduatedScope variant; changes engine-api model type for one topology's needs
- Loop-level interception — scope stays fixed, topology filters outcomes post-hoc; simpler but semantically wrong (the loop thinks it has full scope but outcomes get blocked)
**Rationale:** Graduation moves between existing scope values — the scope space is already well-defined. Mutable GraduationState is per-topology instance state, not model mutation. DecisionScope enum stays stable for all other consumers.
**Trade-offs:** DecisionContext is rebuilt on scope change — slightly more allocation. Acceptable; scope changes are infrequent relative to ticks.
**Sources:** DecisionScope.java, DecisionContext.java, AgentDecisionLoop.java
**Exploration:** quick
**Status:** captured

## D7: CoordinationPlan — descriptors + executor

**Choice:** CoordinationPlan is a record holding List<DecisionLoopDescriptor> (the playbooks to create) plus a DecisionOutcomeExecutor functional interface. Each topology provides its own executor. The plan is a complete recipe: "create these loops, execute outcomes this way."
**Alternatives:**
- Descriptors + execution strategy enum — less flexible, better for YAML serialisation but limits execution strategies to a fixed set
- Full runtime config — includes descriptors, executor, event routing, feedback policy; risks becoming a god-object
**Rationale:** The descriptor list defines WHAT loops to create. The executor defines HOW to handle their output. Together they're a complete coordination recipe. Functional interface for executor keeps the plan flexible — centralised topology routes centrally, distributed topology writes locally, hive topology delegates to patterns.
**Trade-offs:** DecisionOutcomeExecutor is a functional interface, not YAML-serialisable. Acceptable — the executor is topology-specific behavior, not configuration. YAML parity (#385) serialises the descriptors; the executor is wired by the topology implementation.
**Depends on:** D4 (full implementations need real executors)
**Sources:** DecisionLoopDescriptor.java, DecisionOutcome.java, LlmSwarmCoordinator.java (existing execution logic)
**Exploration:** quick
**Status:** captured

## D8: TopologyContext — universal record + constructor injection

**Choice:** TopologyContext is a 5-field immutable record: caseId (UUID), descriptor (TopologyDescriptor), agentProvider (AgentProvider), loopFactory (DecisionLoopFactory), invocationPolicy (SwarmInvocationPolicy). Topology-specific dependencies (SwarmContext, GraduationState, GroupHierarchy) are constructor-injected into each topology implementation.
**Alternatives:**
- Rich context + typed lookup — TopologyContext carries universal fields plus get(Class<T>) accessor; single object but loses compile-time guarantees
- Rich context + explicit optionals — Optional<SwarmContext>, Optional<GroupHierarchy> etc. as fields; full type safety but context grows with every new topology
**Rationale:** Clean separation between per-invocation context (TopologyContext, passed to configure()) and per-instance services (constructor-injected, held by the topology). Universal fields maximize directly available information. Topology-specific deps are retrievable via the topology's own constructor — full compile-time type safety, no stringly-typed lookups. Supports topologies beyond swarm without swarm-specific fields on the context.
**Trade-offs:** Each topology's constructor signature is different — factory/CDI must know what each topology needs. Acceptable; topology creation is a configuration-time activity, not hot-path.
**Sources:** First-principles analysis of universal vs topology-specific needs. AgentProvider.java, DecisionLoopFactory.java, SwarmInvocationPolicy.java, SwarmContext.java
**Exploration:** deep-analysis
**Status:** captured

## D9: CoordinationFeedback — outcome-based

**Choice:** CoordinationFeedback is a record carrying agentId (String), the DecisionOutcome that was executed, and a FeedbackSignal enum (SUCCESS, PARTIAL, FAILURE, SCOPE_VIOLATION). Generated by the outcome executor after applying a decision. GraduatedTopology uses signal patterns to adjust scope. Other topologies can ignore feedback or use it for metrics.
**Alternatives:**
- External evaluator feedback — quality score + reason string from external hook; decoupled but requires external infrastructure
- Composite sealed interface — both outcome-based and external variants; most flexible but adds type complexity for what primarily serves one topology
**Rationale:** Outcome-based feedback is naturally generated by the execution path — no external infrastructure needed. FeedbackSignal captures the essential graduation signal: did the agent's decision succeed, partially succeed, fail, or violate its scope? These four signals map cleanly to scope adjustment policy (success → consider upgrading, failure → consider downgrading, scope violation → hard downgrade).
**Trade-offs:** Feedback is binary/categorical, not continuous. A quality score (0.0-1.0) would give finer-grained graduation signal. Acceptable for R&D — categorical signals are easier to reason about and test.
**Sources:** DecisionOutcome.java, DecisionScope.java, GraduatedTopology concept from issue body
**Exploration:** quick
**Status:** captured

## D10: Package placement — all in agency-core/orchestration/topology/

**Choice:** CoordinationTopology interface and all 6 concrete implementations in agency-core under `io.casehub.agency.orchestration.topology`. Supporting types (TopologyContext, CoordinationPlan, CoordinationFeedback, etc.) in the same package.
**Alternatives:**
- SPI in agency-core, impls in new agency-topology module — isolates R&D implementations but adds a module
- Per-implementation subpackages — centralised/, distributed/, etc.; more granular but 6 packages with 1-2 classes each is excessive
**Rationale:** All implementations share the same SPI, use the same supporting types, and are tested together. A single package keeps the code cohesive and discoverable. The package parallels the existing `orchestration/loop/` and `orchestration/swarm/` packages.
**Trade-offs:** Package grows to ~15-20 classes. Acceptable for a cohesive domain. If individual topologies grow complex enough to warrant their own packages, that's a future refactoring signal.
**Sources:** agency-core/orchestration/loop/ package structure, agency-core/orchestration/swarm/ package structure
**Exploration:** quick
**Status:** captured
