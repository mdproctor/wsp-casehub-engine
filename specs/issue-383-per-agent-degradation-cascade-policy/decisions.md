# Decisions — agency#383: Per-agent degradation + cascade policy

## D1: Degradation trigger architecture

**Choice:** Composable `DegradationTrigger` SPI with adapter pattern
**Alternatives:**
- Inline in BudgetAwareSummariser — couples budget enforcement with degradation logic, misleading class name for latency triggers
- Hard-coded budget + latency check — every new signal source requires changing degradation code
**Rationale:** Budget and latency are independent subsystems. A `DegradationTrigger` functional interface (`boolean shouldDegrade()`) with `CompositeDegradationTrigger` (OR semantics) decouples signal sources from degradation logic. `BudgetDegradationTrigger` adapts `AgentBudgetView.isBudgetExhausted()`. `LatencyDegradationTrigger` adapts `InvocationPolicyMonitor` breach state. New signals (error rate, cost, external circuit breaker) are just new implementations — zero changes to existing code.
**Trade-offs:** One more interface in the SPI surface. Composite pattern adds a thin indirection layer.
**Sources:** `AgentBudgetView.java`, `InvocationPolicyMonitor.java`, `BudgetAwareSummariser.java`
**Exploration:** quick
**Status:** captured

## D2: Degradation switching location

**Choice:** `DegradationAwareSummariser` decorator in the existing summariser chain
**Alternatives:**
- Extend BudgetAwareSummariser — fewer classes but conflates budget enforcement with degradation; latency trigger in a budget-named class is misleading
- Decision loop level (`AgentDecisionLoop.tick()`) — more coarse-grained, misses per-call granularity since a tick may or may not trigger an LLM call
**Rationale:** Follows existing decorator pattern (`InFlightGate` → `BudgetAwareSummariser` → `SectionAwareContentSummariser`). Checks `DegradationTrigger.shouldDegrade()` before each `summarise()` call. If degraded, delegates to a heuristic `ContentSummariser`. If not, delegates to the LLM summariser. `DecisionLoopFactory` wires it.
**Trade-offs:** Adds one more wrapper in a chain that is already 4 deep. Acceptable — each wrapper has a single concern.
**Sources:** `BudgetAwareSummariser.java`, `InFlightGate.java`, `DecisionLoopFactory.java`
**Exploration:** quick
**Status:** captured

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

## D6: Cascade tracking — pull model

**Choice:** Topology polls all `DegradationTrigger` instances at feedback time
**Alternatives:**
- Push/event-based — DegradationAwareSummariser emits events, topology subscribes. More reactive but requires event bus infrastructure that doesn't exist.
**Rationale:** Topology already creates loops via `DecisionLoopFactory` and can hold trigger references. `onFeedback()` is the natural evaluation point — after any loop tick produces feedback, the topology polls all triggers and evaluates `CascadeThreshold`. No new event infrastructure needed.
**Trade-offs:** Polling is slightly less reactive than push — cascade evaluation happens at feedback time, not at the exact moment of degradation. Acceptable because cascade is a topology-level concern that doesn't need sub-tick precision.
**Sources:** `CoordinationTopology.onFeedback()`, `DecisionLoopFactory.java`
**Exploration:** quick
**Status:** captured

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

**Choice:** All 6 topology types get specific cascade responses
**Alternatives:**
- Only the 3 named in the issue (Distributed, Graduated, Hybrid) — matches issue scope but leaves gaps
**Rationale:** Complete coverage avoids undefined behavior when cascade threshold is crossed in any topology.

| Topology | Cascade Response |
|----------|-----------------|
| Distributed | Independent — no cascade, each loop degrades independently |
| Graduated | Centralise — concentrate authority via GraduationState scope adjustments |
| Hybrid | Tighten local budgets — reduce per-agent budget allocation for local loops |
| Centralised | All-heuristic — remaining loops also switch to heuristic via SwarmFallbackStrategy |
| Hierarchical | Collapse upward — degraded leaf responsibilities shift to parent |
| Hive | Simplify pattern — DEBATE → VOTING → SUPERVISOR |

**Trade-offs:** Increases scope from M to upper-M. Each topology needs its own cascade response implementation.
**Sources:** `CentralisedTopology.java`, `DistributedTopology.java`, `HybridTopology.java`, `HierarchicalTopology.java`, `GraduatedTopology.java`, `HiveTopology.java`
**Exploration:** quick
**Status:** captured

## D9: Heuristic summariser creation pattern

**Choice:** Per-loop-type heuristic `ContentSummariser` wrapping `SwarmFallbackStrategy` methods
**Alternatives:**
- Single generic heuristic summariser — can't handle topology-specific heuristic logic (routing vs aggregation vs provisioning)
- Extend SwarmFallbackStrategy to implement ContentSummariser — changes SwarmFallbackStrategy's responsibility
**Rationale:** `SwarmFallbackStrategy` already has three heuristic methods (routing, aggregation, provisioning). Each Centralised loop type gets a heuristic summariser wrapping the appropriate method. Non-Centralised topologies (where loops do general reasoning) get a no-op/pass-through heuristic — the agent stops making novel decisions but doesn't block the swarm.
**Trade-offs:** Multiple small heuristic summariser classes. Acceptable — they're adapters with minimal logic.
**Sources:** `SwarmFallbackStrategy.java`, `ContentSummariser.java`
**Exploration:** quick
**Status:** captured
