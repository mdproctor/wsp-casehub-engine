# Decisions — agency#383: Per-agent degradation + cascade policy

## D1: Degradation trigger architecture

**Choice:** Composable `DegradationTrigger` SPI with independent signal tracking
**Alternatives:**
- Inline in BudgetAwareSummariser — couples budget enforcement with degradation logic, misleading class name for latency triggers
- Hard-coded budget + latency check — every new signal source requires changing degradation code
**Rationale:** Budget and latency are independent subsystems. A `DegradationTrigger` functional interface (`boolean shouldDegrade()`) with `CompositeDegradationTrigger` (OR semantics) decouples signal sources from degradation logic. `BudgetDegradationTrigger` adapts `AgentBudgetView.isBudgetExhausted()` — a clean read of existing atomic state. `LatencyDegradationTrigger` tracks LLM call durations independently within the `DegradationAwareSummariser` decorator (wrapping the delegate call, measuring duration, accumulating breach count against `LlmInvocationPolicy` thresholds). `InvocationPolicyMonitor` is a stateless fire-and-forget logger — it cannot be adapted. The latency trigger reuses `LlmInvocationPolicy` threshold configuration but maintains its own breach state. New signals (error rate, cost, external circuit breaker) are just new `DegradationTrigger` implementations — zero changes to existing code. When a trigger transitions from non-degraded to degraded, `DegradationAwareSummariser` emits a `SwarmDegradedEvent` via the existing `ExecutionEventListener` infrastructure (see D6, D10).
**Trade-offs:** One more interface in the SPI surface. Composite pattern adds a thin indirection layer. LatencyDegradationTrigger must track its own state rather than adapting an existing monitor.
**Sources:** `AgentBudgetView.java`, `InvocationPolicyMonitor.java`, `LlmInvocationPolicy.java`, `BudgetAwareSummariser.java`, `SwarmDegradedEvent.java`
**Exploration:** quick
**Status:** revised — removed incorrect claim of adapting InvocationPolicyMonitor breach state; specified LatencyDegradationTrigger as independent tracker reusing LlmInvocationPolicy thresholds; added SwarmDegradedEvent emission on state transition

## D2: Degradation switching location

**Choice:** `DegradationAwareSummariser` decorator wrapping `BudgetAwareSummariser` in the existing summariser chain
**Alternatives:**
- Extend BudgetAwareSummariser — fewer classes but conflates budget enforcement with degradation; latency trigger in a budget-named class is misleading
- Decision loop level (`AgentDecisionLoop.tick()`) — more coarse-grained, misses per-call granularity since a tick may or may not trigger an LLM call
- Inside BudgetAwareSummariser (between sections and budget) — degraded calls bypass budget tracking, creating circular dependency when budget exhaustion is a trigger
**Rationale:** Follows existing decorator pattern. The chain becomes: `reasoner` → `SectionAwareContentSummariser` → `BudgetAwareSummariser` → `DegradationAwareSummariser` → `InFlightGate`. `DegradationAwareSummariser` checks `DegradationTrigger.shouldDegrade()` before each `summarise()` call. If degraded, delegates to a heuristic `ContentSummariser` — bypassing budget acquisition, section enrichment, and LLM call entirely. The heuristic path needs none of these: no LLM budget to acquire, no prompt sections to enrich, no LLM to call. If not degraded, delegates to the inner chain (sections → budget → LLM). `BudgetDegradationTrigger` reads `AgentBudgetView.isBudgetExhausted()` (a state query) independently of `BudgetAwareSummariser.tryAcquire()` (a budget acquisition) — no circular dependency. `DecisionLoopFactory` wires the full chain.
**Trade-offs:** Adds one more wrapper in a chain that is already 4 deep. Acceptable — each wrapper has a single concern. Degraded calls lose prompt section enrichment, which is correct since heuristic paths don't use prompts.
**Sources:** `BudgetAwareSummariser.java`, `InFlightGate.java`, `DecisionLoopFactory.java`, `AgentBudgetView.java`
**Exploration:** quick
**Status:** revised — specified explicit chain position (outside BudgetAwareSummariser); explained why degraded calls correctly bypass budget and sections; ruled out inside-budget position due to circular dependency

## D3: Degradation unit

**Choice:** Per-loop degradation (not per-agent)
**Alternatives:**
- Per-agent — all loops for an agent degrade together. Simpler mental model but requires a new agent-level degradation state that overrides individual loop budget views.
- Hybrid — per-loop internally, cascade counts agents. Added complexity without clear benefit.
**Rationale:** Each decision loop has its own `AgentBudgetView` and `DegradationTrigger`. In CentralisedTopology, routing could degrade while reasoning is still LLM-backed. Matches the existing per-loop budget architecture.
**Trade-offs:** Cascade counting is by loop, not by agent. In Distributed (1 loop per agent) the distinction is moot. In Centralised (3 loops per agent) degradation is more granular.
**Sources:** `AgentBudgetView.java`, `CentralisedTopology.java`, `DistributedTopology.java`
**Exploration:** quick
**Status:** captured

## D4: Degradation lifecycle — stateless per-call

**Choice:** Degradation is stateless — trigger checked per-call, recovery is implicit
**Alternatives:**
- No recovery — once degraded, stays degraded for swarm execution. Requires explicit degradation state tracking at loop level.
- Manual recovery only — topology explicitly un-degrades. Adds recovery API and state management.
**Rationale:** `DegradationTrigger.shouldDegrade()` is a pure function of current conditions. Budget is monotonic (never recovers — correct). Latency is transient (recovers when LLM normalizes — correct). No oscillation risk because each call independently does the right thing. Simpler than "no recovery" — less state to track.
**Trade-offs:** Cascade must be one-way ratchet to avoid topology-level oscillation from transient latency recovery.
**Sources:** First-principles analysis of budget monotonicity and latency transience
**Exploration:** deep-analysis
**Status:** captured

## D5: Cascade threshold strategy

**Choice:** `CascadeThreshold` interface with `SimpleFractionThreshold` and `PriorityWeightedThreshold` implementations, selected via configuration
**Alternatives:**
- Simple fraction only — doesn't account for RESERVED vs NORMAL priority differences
- Priority-weighted only — overkill for simple topologies like Distributed
**Rationale:** Both strategies are valid for different use cases. Configuration-driven selection avoids hard-coding a single approach. Simple fraction is the sensible default.
**Trade-offs:** Two implementations to maintain. Configuration surface is slightly larger.
**Sources:** `BudgetPriority.java`, `DecisionLoopDescriptor.java`
**Exploration:** quick
**Status:** captured

## D6: Cascade tracking — push model via SwarmDegradedEvent

**Choice:** `DegradationAwareSummariser` emits `SwarmDegradedEvent` on degradation state change; topology receives via `CoordinationTopology.onSwarmEvent()` with a new `SwarmCoordinationEvent.LoopDegraded` variant; cascade evaluation triggered by these events
**Alternatives:**
- Pull/polling — topology polls all triggers at feedback time. Fatal flaw: budget-degraded loops stop producing feedback (`AgentDecisionLoop.tick()` short-circuits when `budget.isBudgetExhausted()`), so the most common degradation trigger never causes cascade evaluation.
- Separate heartbeat/polling thread — adds complexity for a problem the existing event infrastructure already solves.
**Rationale:** The codebase already has `SwarmDegradedEvent(String capability, String reason, boolean fallbackActive)` and `ExecutionEventListener.onSwarmDegraded()` — scaffolded but never emitted. The pull model has a dead path: when budget exhaustion causes `AgentDecisionLoop.tick()` to return immediately (line 37), no decision outcome is produced, no feedback fires, and `onFeedback()` is never called. Additionally, 4 of 6 topology `onFeedback()` implementations are empty no-ops (Centralised, Distributed, Hybrid, Hierarchical). The push model resolves this: `DegradationAwareSummariser` detects degradation state transition (first call where `shouldDegrade()` returns true), emits `SwarmDegradedEvent`, and the topology's `onSwarmEvent()` handler evaluates `CascadeThreshold` against received degradation events. A new `SwarmCoordinationEvent.LoopDegraded(String loopName, String reason)` variant carries the event to the topology. Cascade evaluation counts received degradation events, not polled trigger states.
**Trade-offs:** Requires adding a `LoopDegraded` variant to the sealed `SwarmCoordinationEvent` interface. The `DegradationAwareSummariser` needs an event sink (same pattern as `BudgetAwareSummariser`'s `Consumer<BudgetTelemetryEvent>` sink).
**Depends on:** D1 (DegradationTrigger emits SwarmDegradedEvent on state change), D10 (observability integration)
**Sources:** `SwarmDegradedEvent.java`, `ExecutionEventListener.java`, `SwarmCoordinationEvent.java`, `AgentDecisionLoop.java`, `CoordinationTopology.java`
**Exploration:** quick
**Status:** revised — changed from pull model to push model; pull model has a dead path where budget-degraded loops never trigger cascade evaluation; integrates with existing SwarmDegradedEvent infrastructure

## D7: Cascade is one-way ratchet

**Choice:** Once cascade threshold is crossed and topology responds, the response persists for the execution
**Alternatives:**
- Reversible cascade — topology de-escalates when loops recover. Adds hysteresis complexity and risks instability from transient latency recovery.
**Rationale:** Cascade responses (centralise authority, simplify pattern, collapse hierarchy) are structural changes that are expensive to reverse mid-execution. One-way ratchet is predictable and safe. Individual loops can still recover (stateless degradation), but the topology-level response sticks.
**Trade-offs:** If all loops recover from a transient latency spike, the topology response stays escalated unnecessarily. Acceptable — the cascade response is conservative, not harmful.
**Depends on:** D4 (stateless degradation enables transient recovery, which makes one-way cascade necessary)
**Sources:** First-principles analysis
**Exploration:** quick
**Status:** captured

## D8: Per-topology cascade responses

**Choice:** All 6 topology types get specific cascade responses with in-flight drain semantics
**Alternatives:**
- Only the 3 named in the issue (Distributed, Graduated, Hybrid) — matches issue scope but leaves gaps
**Rationale:** Complete coverage avoids undefined behavior when cascade threshold is crossed in any topology. Cascade transitions must complete any in-flight operations before switching structural configuration — e.g., Hive must drain in-flight pattern execution before changing pattern type to avoid feedback from the old pattern arriving at the new pattern's handler.

| Topology | Cascade Response | Structural Changes |
|----------|------------------|--------------------|
| Distributed | Independent — no cascade, each loop degrades independently | None |
| Graduated | Centralise — concentrate authority by forcing all agents to `STIMULUS_DRIVEN` via `GraduationState` | `GraduationState` needs a `forceScope(String agentId, DecisionScope scope)` method to jump directly to a scope rather than stepping one level per `downgrade()` |
| Hybrid | Tighten local budgets — reduce per-agent budget allocation for local loops | Budget reallocation via `AutonomyBudgetPool` adjustments |
| Centralised | All-heuristic — remaining loops also switch to heuristic via SwarmFallbackStrategy | Loop-level degradation via existing DegradationAwareSummariser — no structural mutation |
| Hierarchical | Collapse upward — degraded leaf responsibilities shift to parent | Leaf loops effectively stopped; parent loops expand scope. Observation events remain wired — parent loop handles decisions for collapsed children |
| Hive | Simplify pattern — DEBATE → VOTING → SUPERVISOR | `HiveTopology.patternType` must become mutable (or add a `cascadePattern` field). `HivePatternAdapter.execute()` already dispatches dynamically on pattern type parameter, so the dispatch path works once the topology passes the new type |

**Trade-offs:** Increases scope from M to upper-M. Each topology needs its own cascade response implementation. In-flight drain adds latency to cascade transitions — acceptable since cascade is a topology-level event that prioritises safety over speed.
**Sources:** `CentralisedTopology.java`, `DistributedTopology.java`, `HybridTopology.java`, `HierarchicalTopology.java`, `GraduatedTopology.java`, `HiveTopology.java`, `GraduationState.java`, `HivePatternAdapter.java`, `OrchestrationPatternType.java`
**Exploration:** quick
**Status:** revised — added in-flight drain requirement; acknowledged structural changes per topology (Hive mutable pattern, Graduated scope-jump, Hierarchical collapse without re-wiring); added Structural Changes column

## D9: Heuristic summariser creation pattern

**Choice:** Per-loop-type heuristic `ContentSummariser` wrapping `SwarmFallbackStrategy` methods
**Alternatives:**
- Single generic heuristic summariser — can't handle topology-specific heuristic logic (routing vs aggregation vs provisioning)
- Extend SwarmFallbackStrategy to implement ContentSummariser — changes SwarmFallbackStrategy's responsibility
**Rationale:** `SwarmFallbackStrategy` already has three heuristic methods (routing, aggregation, provisioning). Each Centralised loop type gets a heuristic summariser wrapping the appropriate method. Non-Centralised topologies (where loops do general reasoning) get a no-op/pass-through heuristic — the agent stops making novel decisions but doesn't block the swarm.
**Trade-offs:** Multiple small heuristic summariser classes. Acceptable — they're adapters with minimal logic.
**Sources:** `SwarmFallbackStrategy.java`, `ContentSummariser.java`, `RoutingReasoner.java`, `AggregationReasoner.java`, `ProvisioningReasoner.java`
**Exploration:** quick
**Status:** captured

## D10: Observability integration

**Choice:** Degradation and cascade events surfaced via existing `SwarmDegradedEvent` and `ExecutionEventListener.onSwarmDegraded()` infrastructure
**Alternatives:**
- New telemetry event type — adds parallel infrastructure when scaffolded events already exist
- Logging only — insufficient for programmatic monitoring and operator dashboards
**Rationale:** The codebase already scaffolds `SwarmDegradedEvent(String capability, String reason, boolean fallbackActive)` and `ExecutionEventListener.onSwarmDegraded(SwarmDegradedEvent)` — declared but never emitted. This spec activates them. `DegradationAwareSummariser` emits a `SwarmDegradedEvent` on degradation state transition (non-degraded → degraded) with `capability` set to the loop name and `fallbackActive=true`. Cascade transitions emit a separate event with `capability` set to the topology ID and `reason` describing the cascade response. `BudgetAwareSummariser` already emits `BudgetTelemetryEvent` via a `Consumer<BudgetTelemetryEvent>` sink — `DegradationAwareSummariser` follows the same pattern with a `Consumer<SwarmDegradedEvent>` sink. This enables operators to distinguish "system making good heuristic decisions" from "system degraded and producing lower-quality output" — critical for D7's one-way ratchet where the topology may run in cascade mode with 0 currently-degraded loops.
**Trade-offs:** None — this activates existing scaffolded infrastructure rather than adding new types.
**Sources:** `SwarmDegradedEvent.java`, `ExecutionEventListener.java`, `BudgetAwareSummariser.java`, `BudgetTelemetryEvent.java`
**Exploration:** quick (surfaced by reviewer as implicit decision)
**Status:** captured

## D11: Cascade policy configuration delivery

**Choice:** New `DegradationPolicy` record alongside `AutonomyBudgetPolicy`, delivered via `TopologyContext`
**Alternatives:**
- Extend AutonomyBudgetPolicy with degradation fields — conflates budget parameters with degradation parameters; budget policy is about resource allocation, degradation policy is about quality trade-offs
- TopologyDescriptor — this is an API-level descriptor, not an operational policy; degradation policy is an engine concern
**Rationale:** `TopologyContext` already carries `AutonomyBudgetPolicy` as an operational policy record. A `DegradationPolicy` record specifies cascade threshold selection (`SimpleFractionThreshold` vs `PriorityWeightedThreshold`), threshold parameters (fraction value, weight map), and latency trigger configuration (breach count window, `LlmInvocationPolicy` reference). The topology's `configure()` method receives both policies and wires the `CascadeThreshold` and `DegradationTrigger` accordingly.
**Trade-offs:** Adds a new record to `TopologyContext` — a breaking change to the record signature. Acceptable — all callers construct `TopologyContext` and the migration is mechanical.
**Sources:** `TopologyContext.java`, `AutonomyBudgetPolicy.java`
**Exploration:** quick (surfaced by reviewer as implicit decision)
**Status:** captured
