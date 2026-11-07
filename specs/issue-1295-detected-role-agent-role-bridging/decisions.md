# Decisions — engine#1295

## D1: User-configurable bridging mode

**Choice:** All three bridging modes (ADVISORY, AUTO_ABOVE_THRESHOLD, AUTOMATIC) are user-configurable via a RoleBridgingPolicy enum. The SPI always produces recommendations; the policy controls what happens with them.
**Alternatives:**
- Hard-code advisory only — limits user control, prevents autonomous swarms from self-organising
- Hard-code automatic — no safety mechanism, risk of oscillation
**Rationale:** The mode is a policy choice, not an architectural one. Different deployments need different behavior — a research swarm might want AUTOMATIC, a clinical workflow needs ADVISORY.
**Trade-offs:** More configuration surface. All three code paths must be tested and maintained.
**Sources:** Issue #1295 design questions, user feedback during brainstorming
**Exploration:** quick
**Status:** captured

## D2: Policy lives on TopologyDescriptor

**Choice:** RoleBridgingPolicy is a field on TopologyDescriptor, alongside topologyType and agentRoles.
**Alternatives:**
- On SwarmConfig — conceptually close (stigmergy-detected), but SwarmConfig is tuning parameters, not structural decisions
- Standalone SPI — maximum flexibility but another config surface
**Rationale:** TopologyDescriptor already holds topology configuration. The bridging policy determines how the topology evolves, so it belongs with other topology decisions. Per-case configuration via case-level TopologyDescriptor.
**Trade-offs:** TopologyDescriptor grows. Non-stigmergy topologies carry a field they don't use (nullable).
**Sources:** TopologyDescriptor.java:22-68, TopologyProvider.java:21-27
**Exploration:** quick
**Depends on:** D1 (user-configurable modes)
**Status:** captured

## D3: Separate RoleBridgingStrategy SPI

**Choice:** New RoleBridgingStrategy interface in api/spi/coordination, separate from TopologyProvider.
**Alternatives:**
- Method on TopologyProvider — simpler but couples structure with pattern matching, forces every topology provider to implement bridging
**Rationale:** Different concerns: TopologyProvider owns structure (who has what role); RoleBridgingStrategy owns pattern-to-role mapping (what behavioral fingerprint means). Different implementations should be pluggable independently.
**Trade-offs:** Two SPIs to implement for full coordination. Implementations need to agree on AgentRole semantics.
**Sources:** TopologyProvider.java:21-27, #1282 spec §AgentRole §Relationship to DetectedRole
**Exploration:** quick
**Status:** captured

## D4: Dual threshold on policy

**Choice:** RoleBridgingPolicy carries minStabilityCount and maxCentroidDrift for AUTO_ABOVE_THRESHOLD mode. Both must be met.
**Alternatives:**
- Confidence-based from strategy — pushes threshold semantics into SPI, less transparent
- Both stability + confidence — most robust but more configuration surface
**Rationale:** Uses existing DetectedRole fields directly (stabilityCount, centroidDrift). Simple, transparent, no new scoring concepts. The stability gate ensures the pattern is real before the strategy maps it.
**Trade-offs:** Threshold semantics are coarse — no per-role or per-agent customisation. A future enhancement could add per-role thresholds without changing the SPI.
**Sources:** DetectedRole.java:21-33 (stabilityCount, centroidDrift fields)
**Depends on:** D1 (user-configurable modes), D2 (policy on TopologyDescriptor)
**Status:** captured

## D5: Strategy returns full AgentRole

**Choice:** RoleBridgingStrategy maps DetectedRole to AgentRole directly — platform roles (COORDINATOR, SUPERVISOR) or domain/task roles (AgentRole.of("reviewer/code", CONTRACTOR)).
**Alternatives:**
- Platform roles only — limits what detection can discover, misses domain-specific patterns
- Structural role only — too coarse, can't distinguish coordinator from supervisor
**Rationale:** Maximum flexibility for consumer implementations. A clinical deployment can map detected patterns to domain-specific roles; a generic swarm can map to platform roles. The strategy decides based on fingerprint content.
**Trade-offs:** Strategy implementations are more complex — they must produce valid AgentRole instances (with correct structural/platform role pairings). Invalid pairings are rejected by AgentRole's compact constructor.
**Sources:** AgentRole.java:22-73 (record with invariants), DetectedRole.java:21-33
**Depends on:** D3 (separate SPI)
**Status:** captured
