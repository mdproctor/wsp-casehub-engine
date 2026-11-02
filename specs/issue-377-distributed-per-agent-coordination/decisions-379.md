# Decisions — agency#379 AgentDecisionLoop

## D1: LLM invocation model

**Choice:** ContentSummariser — the loop's reasoning phase IS summarisation (observe events, synthesise a decision)
**Alternatives:**
- StructuredAgentInvoker — keeps current synchronous JSON-parsed pattern from LlmSwarmCoordinator, but duplicates invocation tracking and doesn't compose with emission policies
- New ReasoningInvoker abstraction — bridges structured output with async composability, but unnecessary given ContentSummariser already provides the right semantics
**Rationale:** ContentSummariser is async, batch-oriented, and composes naturally with InFlightGate and ChangeGatedEmissionPolicy. The centralised coordinator's direct LLM calls would be refactored to this pattern in #381.
**Trade-offs:** ContentSummariser returns `CompletionStage<R>` from `List<T>` — routing and aggregation currently expect structured JSON responses. The summariser implementation will need to handle JSON parsing internally.
**Sources:** ContentSummariser.java (summarisation-api), InFlightGate.java (stigmergy/llm), LlmSwarmCoordinator.java (swarm), agency#379 issue body
**Exploration:** quick
**Status:** captured

## D2: Action output model — sealed DecisionOutcome

**Choice:** Sealed interface DecisionOutcome with variants: WriteBack (observations, signals, context writes), Route (wraps RoutingDecision), Aggregate (wraps AggregationResult), Composite (list of the above)
**Alternatives:**
- Flat List<Action> — more granular but loses semantic grouping of "one decision produced these effects"
- Keep separate (generic R on summariser output) — no new type, but loop is less self-describing and harder to observe uniformly
**Rationale:** Clean type safety via pattern matching. Each variant carries only what it needs. Observable for events/logging. Bridges the existing duality between return-value semantics (RoutingDecision, AggregationResult) and write-back semantics (EnvironmentWriter).
**Trade-offs:** New sealed hierarchy to maintain. Consumers must pattern-match.
**Sources:** RoutingDecision.java, AggregationResult.java, EnvironmentWriter.java, agency#379 issue body §unified action type
**Exploration:** quick
**Status:** captured

## D3: Trigger mechanism — compose with SummarisationRunner

**Choice:** AgentDecisionLoop composes with SummarisationRunner rather than reinventing the pipeline. ObservationStrategy maps to emission policies: EVENT_DRIVEN → ChangeGatedEmissionPolicy, TICK_DRIVEN → WindowPolicy, DEMAND_DRIVEN → explicit flush()
**Alternatives:**
- Own trigger loop — manages its own buffer and trigger logic, but duplicates accumulation/emission/tick logic
- ObservationStrategy as SPI — extensible but adds abstraction for three known variants
**Rationale:** SummarisationRunner is proven, tick-driven, handles accumulation, compaction, emission policies, and failure recovery. The loop doesn't reinvent this — it delegates. Each ObservationStrategy maps naturally to an existing emission policy type.
**Trade-offs:** Coupled to SummarisationRunner's tick model. DEMAND_DRIVEN uses flush() which bypasses emission policy — but that's intentional for demand-driven observation.
**Sources:** SummarisationRunner.java, EmissionPolicy.java, ChangeGatedEmissionPolicy.java, WindowPolicy.java, ObservationStrategy enum
**Exploration:** quick
**Status:** captured

## D4: Scope enforcement — inside the loop

**Choice:** The loop checks `scope.isPermitted(decisionType)` before including each action in the DecisionOutcome. Rejected actions are logged as "scope-gated".
**Alternatives:**
- Delegate to AgentCapabilityGuard — simpler loop but no agency-side visibility into blocked actions
- Both layers — defense in depth but potentially redundant checks
**Rationale:** Agency-side enforcement gives the loop visibility into what it tried and what was blocked. AgentCapabilityGuard in the engine is a separate defense-in-depth layer, not the primary gate. The loop knows its own scope and can make local decisions about rejected actions (e.g., fallback behavior).
**Trade-offs:** Scope enforcement logic in two layers (loop + guard). But they serve different purposes — loop enforces playbook-level scope, guard enforces platform-level capability.
**Sources:** DecisionScope.java, AgentCapabilityGuard.java, DecisionType.java
**Exploration:** quick
**Status:** captured

## D5: Per-agent ObservationContext

**Choice:** A new record capturing the agent's view: agentId, caseId, DecisionScope, CommunicationMode, TopologyDescriptor, plus an observation source. The loop doesn't know about SwarmContext directly.
**Alternatives:**
- Share SwarmContext — couples per-agent loops to swarm-level state; local agents in DISTRIBUTED topology don't need swarm progress/teams
- Generic context `<C>` — maximum flexibility but generics complexity everywhere
**Rationale:** Clean separation. The centralised coordinator parameterisation constructs ObservationContext from SwarmContext. Distributed agents get a lighter view. Testable in isolation without swarm infrastructure.
**Trade-offs:** New type to maintain. Centralised coordinator needs a mapping layer from SwarmContext → ObservationContext.
**Sources:** SwarmContext.java, TopologyDescriptor.java, DecisionScope.java, CommunicationMode.java
**Exploration:** quick
**Status:** captured

## D6: Budget tracking — per-agent with swarm cap

**Choice:** Each loop has its own budget tracker. A shared swarm-level budget exists as a composite — the per-agent tracker delegates to a shared cap for global governance.
**Alternatives:**
- Per-agent only — no global governance; N agents × M calls = N×M total regardless of policy
- Swarm-level only — coarse; one agent can exhaust budget for all others
**Rationale:** Matches the issue's description ("bound to a single agent with budget") while allowing global budget governance. Per-agent tracking enables fine-grained observability (which agent used how much budget).
**Trade-offs:** Composite budget adds a layer of indirection. Contention on the shared cap under high concurrency — but SwarmInvocationTracker already uses atomic counters, which handle this well.
**Depends on:** D5 (per-agent context carries budget reference)
**Sources:** SwarmInvocationTracker.java, SwarmInvocationPolicy.java, agency#379 issue body §budget
**Exploration:** quick
**Status:** captured

## D7: Playbook model — descriptor + runtime split

**Choice:** Two types: DecisionLoopDescriptor (data record, YAML-serialisable playbook) and AgentDecisionLoop (runtime that executes a descriptor for a specific agent). Factory assembles runtime from descriptor.
**Alternatives:**
- Monolithic builder (SwarmCoordinatorBuilder pattern) — fewer types but not YAML-friendly; builders with lambdas don't serialise
- Generic pipeline `<C, O>` — maximally flexible but generics infection throughout coordination layer
**Rationale:** Natural shape for a playbook-oriented system. Descriptor maps cleanly to YAML (#385). Runtime is assembled by factory that resolves prompt section names from registry, constructs emission policy from ObservationStrategy, and wires budget. Clean separation of configuration and execution.
**Trade-offs:** More types (descriptor, factory, runtime) than a monolithic builder. But each has a clear single responsibility.
**Sources:** SwarmCoordinatorBuilder.java (existing builder pattern), DSL parity principle (CLAUDE.md), agency#385 (YAML parity issue)
**Exploration:** quick
**Status:** captured

## D8: YAML-aware design, Java-first delivery

**Choice:** Design Java types so they CAN be populated from YAML (record-like, no callbacks in constructors, builder fields correspond to YAML keys). Don't implement YAML parsing in this issue — that's #385.
**Alternatives:**
- YAML-first delivery — build YAML descriptor AND Java runtime in this issue; pulls #385 scope forward
- Java-only — don't constrain design by YAML serializability; risk: Java API may not map cleanly to YAML later
**Rationale:** Ensures #385 is "add YAML descriptor" not "redesign the runtime." The YAML surface is a separate concern but the data model must accommodate it.
**Trade-offs:** Constrains some Java API patterns (no lambda fields on descriptors, no callback-heavy builders). Worth it for clean DSL parity.
**Depends on:** D7 (descriptor/runtime split enables YAML-awareness)
**Sources:** DSL parity principle, yaml-record-mappings.yaml, CaseDefinition.yaml
**Exploration:** quick
**Status:** captured

## D9: Prompt section references by name

**Choice:** The descriptor lists prompt section names (strings). A PromptSectionRegistry resolves them to SwarmPromptSection instances at runtime.
**Alternatives:**
- Inline declaration — prompt section config inline in descriptor; self-contained but bloats descriptor and duplicates
- Both named + inline — maximum flexibility but more complex descriptor schema
**Rationale:** Matches YAML ergonomics (`sections: [experience-replay, role-context]`). Enables sharing sections across playbooks. Registry is a natural CDI/Spring bean.
**Trade-offs:** Requires a registry. Sections must be registered by name before the loop can be assembled. Runtime error if name not found (fail-fast at factory time, not at first tick).
**Depends on:** D7 (descriptor carries names), D8 (YAML-friendly design)
**Sources:** SwarmPromptSection.java (existing functional interface), SwarmExperienceSection (existing concrete section)
**Exploration:** quick
**Status:** captured

## D10: Loop produces, doesn't execute

**Choice:** The loop outputs DecisionOutcome. Execution (applying actions via EnvironmentWriter, routing agents, etc.) is the caller's responsibility.
**Alternatives:**
- Loop executes actions — receives ActionExecutor and applies DecisionOutcomes; self-contained but couples loop to execution infrastructure
- Configurable via descriptor — some playbooks self-execute, others produce; more flexible but more complex
**Rationale:** Keeps the loop a pure observe→reason→decide primitive. Execution varies by topology: centralised applies routing centrally, distributed writes through EnvironmentWriter locally. The topology implementation (#380, #381) owns execution strategy.
**Trade-offs:** Caller must handle DecisionOutcome pattern matching and execution. But this is the topology layer's natural responsibility.
**Depends on:** D2 (DecisionOutcome is the output type)
**Sources:** EnvironmentWriter.java, RoutingDecision.java, AggregationResult.java, agency#380 (CoordinationTopology), agency#381 (CentralisedTopology refactor)
**Exploration:** quick
**Status:** captured
