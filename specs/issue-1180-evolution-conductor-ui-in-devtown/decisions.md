## D1: Capability area architecture

**Choice:** Devtown areas alongside engine areas
**Alternatives:**
- Devtown areas only, disable engine defaults — cleaner health scores but requires suppression mechanism
- Devtown areas as specialisations of engine areas — reuses existing infrastructure but forces awkward conceptual mapping
**Rationale:** Full integration with the conductor's health scoring, regression detection, and compliance progression. Devtown's ImprovementConfig configures which areas are active for its case instances — engine defaults don't fire unless they have data sources.
**Trade-offs:** Must handle the case where engine default areas have no data in devtown context (they return inert/neutral assessments rather than dragging the score)
**Sources:** engine CapabilityAreaRegistry, EvolutionTicker health_refresh gate, #1148 design spec §1.1
**Exploration:** quick
**Status:** captured
