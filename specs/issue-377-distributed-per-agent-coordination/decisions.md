# Decisions — engine#1284 SwarmTopologyProvider

## D1: authorityFor placement

**Choice:** On TopologyDescriptor (the data type), not on SwarmTopologyProvider (the SPI)
**Alternatives:**
- SPI method — creates consistency hazard: descriptor and authority can disagree
- Both (SPI delegates to descriptor) — unnecessary indirection, same hazard
**Rationale:** Authority is a mechanical derivation from the descriptor's own data (TopologyType × StructuralRole). Putting it on the descriptor makes the descriptor the single source of truth. The SPI stays minimal: one method (`describeTopology`). Pre-release platform means we can add an SPI override later if per-decision-type authority models emerge — no backward-compat cost.
**Trade-offs:** If a future topology needs authority logic beyond what (TopologyType × StructuralRole) determines, we'd need to revisit — but that's a hypothetical not in scope for the three existing topologies.
**Sources:** TopologyType.deriveAuthority() (api/model/coordination/TopologyType.java), #1282 spec §Relationships Between Types, #1282 spec §Deferral decomposition
**Exploration:** deep-analysis
**Status:** captured

## D2: TopologyDescriptor shape

**Choice:** Immutable record with two fields (topologyType, agentRoles map), derived convenience methods (authorityFor, coordinatorAgentId), fail-fast constructor validation
**Alternatives:**
- coordinatorAgentId as a field — consistency hazard with role map, redundant data
- version/epoch field — YAGNI, no consumer needs staleness detection yet
- DecisionType parameter on authorityFor — doesn't participate in computation for existing topologies
**Rationale:** Minimal record with maximal safety. Compact constructor validates role/topology coherence at creation time (calls deriveAuthority, throws on invalid combos). Map.copyOf ensures immutability. Symbolic roles (null structuralRole) skip validation — they exist for rendering, not authority derivation. Agent identity is String, consistent with AgentCapabilityGuard established in #1283.
**Trade-offs:** No version/epoch means no built-in staleness detection — consumers must call describeTopology each time. Acceptable for pre-release.
**Depends on:** D1 (authorityFor lives on descriptor)
**Sources:** AgentCapabilityGuard.check(UUID, String, AgentAction), TopologyType.deriveAuthority(), #1282 spec §Deferral decomposition
**Exploration:** quick
**Status:** captured

## D3: Package placement

**Choice:** TopologyDescriptor in `api/model/coordination/`, SwarmTopologyProvider in `api/spi/coordination/` (new), CentralisedTopologyProvider in `runtime-core/runtime/coordination/`
**Alternatives:**
- SwarmTopologyProvider in `api/spi/stigmergy/` — wrong: topology is broader than stigmergy
- SwarmTopologyProvider flat in `api/spi/` — too crowded, no grouping
**Rationale:** Follows established patterns: model types → `api/model/`, SPI interfaces → `api/spi/<domain>/`, default impls → `runtime-core/runtime/<domain>/`. The `spi/coordination/` subpackage parallels `spi/stigmergy/`, `spi/observation/`, `spi/mesh/`.
**Trade-offs:** New subpackage — acceptable; the coordination domain is growing.
**Sources:** Existing package structure: spi/stigmergy/, spi/observation/, spi/mesh/
**Exploration:** quick
**Status:** captured

## D4: SPI surface — single method

**Choice:** `TopologyProvider` has one method: `TopologyDescriptor describeTopology(UUID caseId)`
**Alternatives:**
- Additional lifecycle methods (onTransition, onGraduation) — topology transitions are internal provider state, not SPI surface
- Query methods (isStable, epoch) — YAGNI, derivable from descriptor
**Rationale:** Minimal SPI. The descriptor carries all state; consumers derive what they need from it. Lifecycle events for topology changes belong on the event/listener layer (SwarmCoordinationListener), not the provider SPI.
**Depends on:** D1 (authorityFor on descriptor, not SPI)
**Sources:** SwarmProvisioningAdvisor (existing minimal SPI pattern), SwarmCoordinationListener (event pattern)
**Exploration:** quick
**Status:** captured

## D5: CentralisedTopologyProvider default

**Choice:** Returns `TopologyDescriptor(CENTRALISED, Map.of())` — empty role map, centralised topology. Wired via SpiDefaultBeans `@Produces @DefaultBean`.
**Alternatives:**
- NoOp returning null — fails consumers, no backward compat
- Return HYBRID — changes default behavior, not backward compatible
**Rationale:** Backward compatible with current SwarmProvisioner behavior (all provisioning is centrally decided). Empty role map means no agent has a declared role — `authorityFor` throws for unknown agents, which is correct: if no topology is configured, the engine shouldn't be making authority-based decisions.
**Trade-offs:** Empty role map means authorityFor is unusable until agents are registered — acceptable, authority queries only make sense when a topology is actively configured.
**Depends on:** D2 (descriptor shape), D3 (package placement)
**Sources:** SpiDefaultBeans.java, NoOpAgentCapabilityGuard pattern
**Exploration:** quick
**Status:** captured

## D6: SwarmProvisioner integration depth

**Choice:** Topology-type gate only — SwarmProvisioner checks topologyType and skips central provisioning for DISTRIBUTED. No per-agent authority checks in provisioner.
**Alternatives:**
- Deep integration: per-agent authorityFor checks before each provision — premature, SwarmProvisioner doesn't track which agent is requesting provision
- No integration: just make the SPI injectable — misses the issue's core requirement
**Rationale:** SwarmProvisioner is inherently a central provisioning mechanism. The topology tells it whether central provisioning is appropriate at all. DISTRIBUTED topologies self-organize — SwarmProvisioner has no role. CENTRALISED/HYBRID proceed with current logic. StigmergyCoordinator/RoleTracker/TeamDetector adaptation belongs in #380/#381, not here.
**Trade-offs:** HYBRID topology gets the same provisioning behavior as CENTRALISED — richer HYBRID-specific provisioning is deferred.
**Sources:** SwarmProvisioner.evaluateAndProvision(), engine#1284 issue body §Integration
**Exploration:** quick
**Status:** captured
