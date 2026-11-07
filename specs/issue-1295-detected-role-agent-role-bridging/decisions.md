# Decisions — engine#1295

## D1: User-configurable bridging mode

**Choice:** All three bridging modes (ADVISORY, AUTO_ABOVE_THRESHOLD, AUTOMATIC) are user-configurable via a RoleBridgingPolicy enum. The SPI always produces recommendations; the policy controls what happens with them.
**Alternatives:**
- Hard-code advisory only — limits user control, prevents autonomous swarms from self-organising
- Hard-code automatic — no safety mechanism, risk of oscillation
- Progressive/adaptive mode (auto-graduates based on acceptance rate) — conflates strategy improvement with policy configuration; the strategy gets better over time (more accurate confidence), the policy stays fixed (user's risk tolerance); merging them creates an abstraction that serves neither concern well
**Rationale:** The mode is a policy choice, not an architectural one. Different deployments need different behavior — a research swarm might want AUTOMATIC, a clinical workflow needs ADVISORY. The three modes have near-zero marginal implementation cost over ADVISORY-only — the SPI always produces recommendations; the mode is a single branch controlling the response. AUTOMATIC mode requires oscillation protection (see D7).
**Trade-offs:** More configuration surface. All three code paths must be tested and maintained. AUTOMATIC mode is safe only with the oscillation protection specified in D7.
**Sources:** Issue #1295 design questions, user feedback during brainstorming
**Exploration:** quick
**Status:** captured

## D2: Policy lives on TopologyContext

**Choice:** RoleBridgingPolicy type and instance both live in agency, on TopologyContext alongside AutonomyBudgetPolicy and DegradationPolicy.
**Alternatives:**
- Type in engine-api, instance in agency — inconsistent with peer policies; AutonomyBudgetPolicy (`agency.orchestration.budget`) and DegradationPolicy (`agency.orchestration.degradation`) are both agency types, not engine-api types; engine never evaluates the bridging policy; the SPI signature (`DetectedRole → RoleSuggestion`) does not reference the policy type
- On TopologyDescriptor — conflates immutable topology state with evolution policy; TopologyDescriptor is the OUTPUT of TopologyProvider, not an input controlling provider behavior; non-stigmergy topologies carry a permanently-null field
- On SwarmConfig — wrong module; SwarmConfig is in `model.stigmergy` (stigmergy-specific), bridging is a coordination concern; SwarmConfig does carry behavioral policies (IntegrationPolicy, ProvisionBudget) but these are stigmergy-specific, not coordination-layer concerns
- Standalone SPI — maximum flexibility but another config surface
**Rationale:** TopologyDescriptor is an immutable state snapshot — what the topology IS. RoleBridgingPolicy is a behavioral policy controlling how the topology EVOLVES. TopologyContext already bundles parallel evolution policies (AutonomyBudgetPolicy, DegradationPolicy) alongside the descriptor. All three policies are consumed only by agency's topology layer — engine never evaluates them. The bridging policy follows the same convention: agency type on TopologyContext.
**Trade-offs:** The policy type is not in the shared engine-api vocabulary — it's agency-specific. This is correct: the policy controls agency-side orchestration behavior, not cross-layer vocabulary.
**Sources:** TopologyContext.java:13 (record with budgetPolicy, degradationPolicy), AutonomyBudgetPolicy.java:3 (agency.orchestration.budget), DegradationPolicy.java:9 (agency.orchestration.degradation), TopologyDescriptor.java:22 (immutable state record)
**Exploration:** quick
**Depends on:** D1 (user-configurable modes)
**Status:** revised — R1-05: moved from TopologyDescriptor to TopologyContext. R2-01: type moved from engine-api to agency, consistent with peer policies (AutonomyBudgetPolicy, DegradationPolicy).

## D3: Separate RoleBridgingStrategy SPI

**Choice:** New RoleBridgingStrategy interface in api/spi/coordination, separate from TopologyProvider.
**Alternatives:**
- Method on TopologyProvider — simpler but couples structure with pattern matching, forces every topology provider to implement bridging
- Extend SwarmCoordinationListener — conflates cross-layer event transport with intra-agency mapping; the listener is multi-instance (CDI Instance / Spring ObjectProvider) while the strategy is a single SPI selected per configuration; different lifecycle, different cardinality
**Rationale:** Different concerns: TopologyProvider owns structure (who has what role); RoleBridgingStrategy owns pattern-to-role mapping (what behavioral fingerprint means). Different implementations should be pluggable independently. The SPI lives in engine-api because its input (DetectedRole) and output types are engine-api vocabulary — agency provides implementations.
**Trade-offs:** Two SPIs to implement for full coordination. The orchestration flow (D6) connects them.
**Sources:** TopologyProvider.java:21-27, #1282 spec §AgentRole §Relationship to DetectedRole
**Exploration:** quick
**Status:** captured

## D4: Confidence-based threshold on policy

**Choice:** RoleBridgingPolicy carries three threshold parameters for AUTO_ABOVE_THRESHOLD and AUTOMATIC modes. The RoleBridgingStrategy returns a confidence score [0, 1] as part of each RoleSuggestion, synthesizing detection quality signals (stabilityCount, centroidDrift, cluster quality) into a single dimension.

Policy parameters:
- `minConfidence` — promotion threshold; a suggestion must meet or exceed this to trigger a role assignment (used by AUTO_ABOVE_THRESHOLD and AUTOMATIC modes)
- `demotionConfidence` — demotion threshold; a bridged role is revoked when confidence drops below this value; must be lower than minConfidence to provide hysteresis (see D7)
- `minHoldCycles` — minimum detection cycles a bridged role is maintained before demotion is eligible, even if confidence drops; prevents demotion on transient fluctuations (see D7)

**Alternatives:**
- Raw detection thresholds (minStabilityCount + maxCentroidDrift with AND-semantics) — leaks detection implementation into policy; users must understand RoleTracker internals to set meaningful values; rigid AND-semantics doesn't capture the compensating relationship between stability and drift
- Named profiles only (CONSERVATIVE/MODERATE/AGGRESSIVE) — insufficient for advanced users who need fine-grained control; can be layered as convenience constants for common parameter combinations
- Single threshold (minConfidence only) — insufficient for demotion; symmetric promote/demote at the same threshold causes oscillation at the boundary (see D7)
**Rationale:** The strategy is the expert on detection quality — it's the right place to synthesize multiple quality signals into a single confidence score. The policy thresholds on a single confidence dimension, completely decoupled from detection internals. If DetectedRole gains new quality indicators (member count, cluster silhouette score), only the strategy changes — the policy API is stable. The three-parameter set (minConfidence, demotionConfidence, minHoldCycles) is the minimum needed for safe automatic operation: promotion gating, demotion with hysteresis, and temporal damping.
**Trade-offs:** Confidence is opaque — harder to debug why a specific role scored 0.72. Strategy implementations bear the complexity of synthesizing multiple signals. Mitigation: the strategy can log the component scores. Three parameters is more configuration surface than one, but each serves a distinct purpose and the relationship between them must be documented (demotionConfidence < minConfidence).
**Sources:** DetectedRole.java:21-33 (stabilityCount, centroidDrift fields)
**Depends on:** D1 (user-configurable modes), D5 (RoleSuggestion carries confidence)
**Status:** revised — R1-11: changed from raw detection thresholds to strategy-returned confidence. R2-03: enumerated complete threshold inventory (minConfidence, demotionConfidence, minHoldCycles) — D7 references these parameters.

## D5: Strategy returns RoleSuggestion

**Choice:** RoleBridgingStrategy returns RoleSuggestion (suggestedName, suggestedStructuralRole, suggestedPlatformRole, confidence, source DetectedRole) instead of a fully-constructed AgentRole. The topology layer creates the AgentRole, applying topology-appropriate structural constraints. Both suggestedStructuralRole and suggestedPlatformRole are nullable: null means the strategy has no assessment for that facet, and the topology assigns from its own logic.
**Alternatives:**
- Return full AgentRole — strategy must know topology constraints to produce valid roles; topology-invalid roles are caught late at TopologyDescriptor construction; every strategy implementation must duplicate topology constraint knowledge
- Return role name only — insufficient; the strategy's assessment of platform role semantics (coordinator vs supervisor) and structural position (contractor vs peer) is lost
- Omit suggestedStructuralRole (derive from platformRole only) — PlatformRole maps to INITIATOR and PEER only; cannot express CONTRACTOR structural position; domain roles like `AgentRole.of("reviewer/code", CONTRACTOR)` become unrepresentable
**Rationale:** Separates detection insight from topology constraint enforcement. The strategy is the expert on what a behavioral pattern MEANS — both the semantic role (this looks like a coordinator) and the structural position (this agent acts as a subordinate worker, not a peer). The topology is the expert on what structural positions are VALID (CENTRALISED allows INITIATOR, DISTRIBUTED does not). For platform roles, suggestedStructuralRole is redundant with `platformRole.requiredStructuralRole()` — the topology can validate consistency. For domain roles without a platform role, suggestedStructuralRole is the only way the strategy can express its structural position assessment. The consumer creates the AgentRole using both: the strategy's full assessment and its own structural constraints. Invalid suggestions are filtered, not thrown as exceptions.
**Trade-offs:** New intermediate type (RoleSuggestion) with five fields. Topology implementations must translate suggestions into AgentRoles — but this is the correct responsibility boundary. Redundancy between suggestedStructuralRole and suggestedPlatformRole for platform roles is acceptable: it allows the topology to detect inconsistent suggestions (e.g., suggestedStructuralRole=PEER but suggestedPlatformRole=COORDINATOR which requires INITIATOR).
**Sources:** AgentRole.java:22-73 (record with invariants), DetectedRole.java:21-33, TopologyType.java:18-84 (structural role constraints per topology), PlatformRole.java:18-34 (maps to INITIATOR and PEER only, no CONTRACTOR)
**Depends on:** D3 (separate SPI)
**Status:** revised — R1-14: changed from full AgentRole to RoleSuggestion. R2-02: added suggestedStructuralRole to express CONTRACTOR and other non-platform structural positions.

## D6: Agency owns bridging orchestration

**Choice:** The bridging orchestration lives in agency's topology layer. The flow is: engine's RoleTracker detects roles → SwarmProvisioner.notifyRolesChanged fires SwarmCoordinationListener.onRoleChanged → SwarmCoordinationListenerBridge forwards as SwarmCoordinationEvent.RoleChanged → CoordinationTopology.onSwarmEvent receives it → topology invokes RoleBridgingStrategy → applies RoleBridgingPolicy → updates internal role state if warranted.
**Alternatives:**
- Engine orchestrates — engine acquires topology management responsibility that belongs to agency; engine's runtime layer would need agency-specific types
- New coordinator layer between engine and agency — over-engineering; the existing listener bridge already handles cross-layer events
**Rationale:** Agency owns the topology (CoordinationTopology, HiveTopology). The existing engine→agency event flow (SwarmCoordinationListener → SwarmCoordinationListenerBridge → SwarmCoordinationEvent) already delivers detected roles to the topology layer. The only missing piece is the no-op HiveTopology.onSwarmEvent — bridging makes it substantive. The SPI types live in engine-api (shared vocabulary); the orchestration lives in agency (where the topology lives).
**Trade-offs:** HiveTopology (and other CoordinationTopology implementations) gain bridging responsibility. Non-stigmergic topologies that don't receive swarm events need a different integration path (or the default no-op onSwarmEvent). Relationship to #1290 TransitionPolicy: if the topology receives suggestions it can't honor given its current type (e.g., PEER suggestions in a CENTRALISED topology), one response is to trigger a topology transition via TransitionPolicy — the interaction point is through the topology's decision logic, not direct coupling between bridging and transition concerns.
**Sources:** SwarmCoordinationListenerBridge.java:15, HiveTopology.java:38 (onSwarmEvent no-op), SwarmProvisioner.java:546 (notifyRolesChanged), SwarmCoordinationEvent.java:10 (RoleChanged record)
**Depends on:** D1, D2, D3, D5
**Status:** captured

## D7: Demotion semantics and oscillation protection

**Choice:** Bridged role assignments are reversible. Oscillation is structurally prevented by D4's three policy parameters: asymmetric confidence thresholds (minConfidence > demotionConfidence) provide hysteresis at the promotion/demotion boundary; minHoldCycles prevents demotion during transient confidence drops.
**Alternatives:**
- Permanent promotion only — simple but accumulates stale roles; an agent that stops exhibiting coordinator behavior stays COORDINATOR indefinitely
- Symmetric thresholds (promote and demote at the same confidence level) — causes oscillation at the boundary when confidence fluctuates around the threshold
- No demotion in ADVISORY mode — correct; the operator controls role assignments manually in ADVISORY mode
**Rationale:** Asymmetric thresholds are the standard hysteresis approach for any promote/demote system. The minimum hold duration adds a temporal dimension — even if confidence drops briefly during a transient, the role is maintained. Together, these make AUTOMATIC mode safe without requiring operator intervention. In AUTO_ABOVE_THRESHOLD mode, the same mechanisms apply but the operator retains override capability.
**Trade-offs:** The relationship between minConfidence and demotionConfidence needs careful documentation — setting them too close reintroduces oscillation risk. See D4 for the complete parameter inventory.
**Sources:** Issue #1295 design question 1 (direction: one-way or two-way), #1290 (TransitionPolicy — role demotion may trigger topology transition)
**Depends on:** D1 (AUTOMATIC mode), D4 (confidence-based threshold — defines minConfidence, demotionConfidence, minHoldCycles)
**Status:** captured

## D8: Multi-agent conflict resolution

**Choice:** The topology resolves conflicts when multiple agents are simultaneously suggested for structurally-exclusive positions. For single-occupancy positions (e.g., one INITIATOR in CENTRALISED topology), the highest-confidence suggestion wins. Non-exclusive positions (PEER, CONTRACTOR) allow multiple simultaneous bridged roles.
**Alternatives:**
- First-detected wins — simple but ignores quality; a low-confidence early detection blocks a high-confidence later one
- Strategy resolves conflicts — wrong layer; the strategy maps individual DetectedRoles without cross-agent visibility; the topology knows structural constraints and all current role assignments
- Reject all conflicting suggestions — safe but overly conservative; perfectly valid high-confidence detections are discarded
**Rationale:** The topology knows its own structural constraints (how many INITIATORs allowed, whether PEERs are permitted). Conflict resolution is a topology concern, not a strategy concern. Highest-confidence tiebreaking is natural and consistent with the confidence-based threshold design (D4). Equal-confidence ties use a stable tiebreaker (lexicographic agent ID) to prevent flip-flopping.
**Trade-offs:** The topology must track suggestion confidence across agents to compare. The stable tiebreaker is arbitrary — but determinism matters more than fairness for structural positions.
**Sources:** TopologyType.java:18-84 (structural constraints per topology), #1282 spec §TopologyType
**Depends on:** D5 (RoleSuggestion with confidence), D6 (topology orchestrates)
**Status:** captured
