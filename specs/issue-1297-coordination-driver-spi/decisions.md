# Decisions — CoordinationDriver SPI (#1297)

## D1: All four behavioral facets

**Choice:** CoordinationDriver defines methods for all four COORDINATOR facets: task delegation, result aggregation, agent allocation, lifecycle authority
**Alternatives:**
- Delegate + aggregate only — narrower SPI, leaves allocation/lifecycle to other layers
- Marker interface only — documented contract with no compile-time guarantee
**Rationale:** The epic's goal is moving behavioral contracts to engine-api. A complete contract means engine has the full COORDINATOR surface. Agency provides the intelligence; engine defines the shape.
**Trade-offs:** Larger SPI surface. Worth it for completeness — the four facets are the definitive COORDINATOR behaviors per the issue.
**Sources:** Issue #1297 body (four facets listed), epic #1296 design principle
**Exploration:** quick
**Status:** captured

## D2: Two-layer pattern (marker + SPI)

**Choice:** `CoordinatorBehavior` marker in `model.coordination` + `CoordinationDriver extends CoordinatorBehavior` in `spi.coordination`
**Alternatives:**
- Single SPI only — simpler but inconsistent with SUPERVISOR pattern
**Rationale:** Follows the established `SupervisorGateBehavior` / `OversightGateService` pattern. Future-proofs for multiple COORDINATOR implementations at different granularities (topology-level vs sub-topology coordination).
**Trade-offs:** One extra interface for consistency — minimal overhead
**Sources:** SupervisorGateBehavior.java, OversightGateService.java, PlatformRole.java
**Exploration:** quick
**Status:** captured

## D3: New CoordinationTask record

**Choice:** New `CoordinationTask` record as the delegation unit, carrying topology context and coordination metadata
**Alternatives:**
- Reuse WorkRequest — case-level, lacks topology awareness
- Generic type parameter — CoordinationDriver<T> — flexible but harder to consume
**Rationale:** Coordination-level delegation is distinct from case-level work submission. CoordinationTask carries target agent constraints and aggregation strategy hints that WorkRequest doesn't have.
**Trade-offs:** Another type to maintain. Acceptable — the two concerns (case-level work vs coordination-level delegation) are genuinely different.
**Sources:** WorkOrchestrator.java (WorkRequest is case-level), issue #1297 (scope distinction)
**Exploration:** quick
**Status:** captured

## D4: CoordinationResult sealed interface

**Choice:** `CoordinationResult` sealed interface with `Aggregated`, `Partial`, `Failed` variants
**Alternatives:**
- Map<String, Object> — generic, no type safety
- Single record with status enum — simpler but status may grow
**Rationale:** Pattern-matchable by engine. Each variant carries its own typed payload. Aligns with how TopologyType and TransitionTrigger use sealed interfaces in this codebase.
**Trade-offs:** More types than a single record. Worth it — engine can exhaustively switch on result type.
**Sources:** TransitionTrigger (sealed interface pattern established in #1290), DecisionOutcome (agency parallel)
**Depends on:** D1 (aggregation is one of the four facets)
**Exploration:** quick
**Status:** captured
