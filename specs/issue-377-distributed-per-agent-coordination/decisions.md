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
