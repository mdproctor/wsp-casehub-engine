# SwarmTopologyProvider — Topology-Aware Provisioning and Lifecycle

**Issue:** engine#1284
**Parent epic:** engine#1281 (Swarm Mechanical Enforcement Layer)
**Cross-repo parent:** agency#377 (Distributed Per-Agent LLM Coordination)
**Depends on:** engine#1282 (TopologyType/TopologyDescriptor vocabulary types) — done

## Summary

Add `TopologyProvider` SPI to engine-api so the engine becomes topology-aware.
SwarmProvisioner consults the topology before making central provisioning decisions —
in a DISTRIBUTED topology, central provisioning is inappropriate and is skipped.

Three new types:
- `TopologyDescriptor` — immutable record describing the current topology state
- `TopologyProvider` — SPI for querying the current topology
- `CentralisedTopologyProvider` — backward-compatible default implementation

## TopologyDescriptor

**Package:** `io.casehub.api.model.coordination`

```java
public record TopologyDescriptor(
    TopologyType topologyType,
    Map<String, AgentRole> agentRoles
) {
    public TopologyDescriptor {
        Objects.requireNonNull(topologyType, "topologyType must not be null");
        agentRoles = agentRoles != null ? Map.copyOf(agentRoles) : Map.of();
        for (var entry : agentRoles.entrySet()) {
            Objects.requireNonNull(entry.getKey(), "agent ID must not be null");
            Objects.requireNonNull(entry.getValue(), "agent role must not be null");
            AgentRole role = entry.getValue();
            if (role.structuralRole() != null) {
                // fail-fast: validates role/topology coherence
                topologyType.deriveAuthority(role.structuralRole());
            }
        }
    }

    public DecisionAuthority authorityFor(String agentId) {
        AgentRole role = agentRoles.get(agentId);
        if (role == null) {
            throw new IllegalArgumentException(
                "Unknown agent: " + agentId + " — not present in topology descriptor");
        }
        if (role.structuralRole() == null) {
            throw new IllegalArgumentException(
                "Agent " + agentId + " has symbolic role '" + role.name()
                + "' — symbolic roles have no structural position and cannot derive authority");
        }
        return topologyType.deriveAuthority(role.structuralRole());
    }

    public Optional<String> coordinatorAgentId() {
        return agentRoles.entrySet().stream()
            .filter(e -> e.getValue().structuralRole() == StructuralRole.INITIATOR)
            .map(Map.Entry::getKey)
            .findFirst();
    }
}
```

**Design properties:**
- Immutable — `Map.copyOf` in compact constructor
- Fail-fast — validates every (topologyType, structuralRole) pairing at construction
- `authorityFor` is a derivation from the descriptor's own data — single source of truth (D1)
- `coordinatorAgentId` is derived, not stored — no consistency hazard
- Symbolic roles (null structuralRole) are allowed in the map but cannot derive authority
- Agent identity is `String` — consistent with `AgentCapabilityGuard.check(UUID, String, AgentAction)`
- No version/epoch — YAGNI for pre-release

## SwarmTopologyProvider

**Package:** `io.casehub.api.spi.coordination`

```java
public interface SwarmTopologyProvider {
    TopologyDescriptor describeTopology(UUID caseId);
}
```

One method. Authority derivation lives on TopologyDescriptor (D1). Lifecycle events
(topology transitions, graduation) are internal provider state — consumers observe
changes by calling `describeTopology` again or via SwarmCoordinationListener events.

## CentralisedTopologyProvider

**Package:** `io.casehub.engine.runtime.coordination`

```java
public class CentralisedTopologyProvider implements SwarmTopologyProvider {
    @Override
    public TopologyDescriptor describeTopology(UUID caseId) {
        return new TopologyDescriptor(TopologyType.CENTRALISED, Map.of());
    }
}
```

Returns `CENTRALISED` with an empty role map. Backward compatible — existing
SwarmProvisioner behavior is all-central. Wired via `SpiDefaultBeans` producer.

## Integration — SwarmProvisioner

SwarmProvisioner receives `TopologyProvider` as a constructor parameter.
At the top of `evaluateAndProvision`, before any budget checks:

```java
var descriptor = topologyProvider.describeTopology(caseId);
if (descriptor.topologyType() == TopologyType.DISTRIBUTED) {
    return List.of(); // central provisioning not applicable
}
```

CENTRALISED and HYBRID proceed with existing logic unchanged. The topology-type
gate is the only integration point for #1284 — richer HYBRID-specific provisioning
behavior is deferred to #380 (CoordinationTopology SPI).

Same gate in `evaluateDeprovisioning` — DISTRIBUTED topologies manage their own
agent lifecycle.

## Integration — SpiDefaultBeans

Add to the `// --- Coordination SPIs ---` section:

```java
@Produces
@DefaultBean
@ApplicationScoped
SwarmTopologyProvider centralisedTopologyProvider() {
    return new CentralisedTopologyProvider();
}
```

## What This Issue Does NOT Include

- **Per-decision-type authority models** — authority is (TopologyType x StructuralRole) only. DecisionType as an authority dimension is deferred until a topology requires it.
- **Topology transitions / graduation** — transition policies belong to specific topology providers (#380).
- **StigmergyCoordinator adaptation** — behavioral changes based on topology belong in #381.
- **RoleTracker / TeamDetector adaptation** — informational only; behavioral adaptation is #380+.
- **YAML surface** — DSL parity for TopologyDescriptor is agency#385.

## Downstream Composability — eidos#216 (Self-Organising Agent Society)

Groups will use TopologyType as a mutable property. This design is compatible:
- TopologyDescriptor is a snapshot — the provider returns current state each call.
  TopologyType mutation on a group is invisible to consumers of the descriptor.
- Agent identity is String — a group coordinator's AgentRef has a string representation.
  The `Map<String, AgentRole>` naturally accommodates group coordinators as topology
  participants.
- The SPI describes topology, not provisioning mechanics. Group-level
  provision/deprovision is SwarmProvisioner's concern, not the topology provider's.

## Testing Strategy

1. **TopologyDescriptor construction tests:**
   - Valid combinations for all three topologies
   - Invalid combinations throw at construction (CENTRALISED + PEER, DISTRIBUTED + INITIATOR, etc.)
   - Null/empty role map handling
   - Symbolic roles allowed, skip validation
   - Immutability — modifying the source map doesn't affect the descriptor

2. **TopologyDescriptor.authorityFor tests:**
   - Correct authority derivation for all valid (topology, role) combinations
   - Unknown agent throws IllegalArgumentException
   - Symbolic role throws IllegalArgumentException

3. **TopologyDescriptor.coordinatorAgentId tests:**
   - Returns INITIATOR agent for CENTRALISED/HYBRID
   - Returns empty for DISTRIBUTED (no INITIATOR)
   - Returns empty for empty role map

4. **CentralisedTopologyProvider tests:**
   - Returns CENTRALISED descriptor with empty role map
   - Same descriptor for different case IDs (stateless)

5. **SwarmProvisioner integration tests:**
   - DISTRIBUTED topology → evaluateAndProvision returns empty list
   - CENTRALISED topology → existing behavior unchanged
   - HYBRID topology → existing behavior unchanged

## References

- `api/model/coordination/TopologyType.java` — deriveAuthority method, topology semantics
- `api/model/coordination/AgentRole.java` — role record with structural/platform facets
- `api/spi/AgentCapabilityGuard.java` — established agent identity as String
- `runtime-core/runtime/coordination/NoOpAgentCapabilityGuard.java` — default impl pattern
- `runtime/quarkus/SpiDefaultBeans.java` — DefaultBean producer pattern
- `runtime-core/runtime/stigmergy/SwarmProvisioner.java` — integration target
- `specs/issue-1282-coordination-vocabulary-types/SPEC.md` — vocabulary spec, deferral table
- engine#1284 issue — original requirements
