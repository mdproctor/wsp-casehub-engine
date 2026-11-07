# Decisions — engine#1294

## D1: Formalise, not consolidate

**Choice:** Keep both ConductorInboxManager and OversightGateService as separate mechanisms. Make the SUPERVISOR relationship explicit via types and documentation.
**Alternatives:**
- Consolidate — merge conductor gating into a unified SPI. Forces the conductor's rich evolution-specific semantics into a generic shape.
- Replace conductor — retire ConductorInboxManager and migrate to PlatformRole-dispatched behavior. Loses domain-specific context (escalation triggers, watch patterns, confidence).
**Rationale:** The two mechanisms operate at different granularities (pipeline-stage vs worker-action) with different richness. Merging would either lose evolution-specific context or bloat the generic SPI. The overlap is conceptual, not structural — both implement SUPERVISOR gating but in different domains.
**Trade-offs:** Two code paths for approval-gating persist. Future gate implementations need to know which to use.
**Sources:** PlatformRole.java:18-33, OversightGateService.java:20-24, ConductorInboxManager.java:27-90, issue-1282 spec §PlatformRole
**Exploration:** quick
**Status:** captured

## D2: Shared marker interface

**Choice:** Create a `SupervisorGateBehavior` marker interface that both ConductorInboxManager and OversightGateService implement/extend.
**Alternatives:**
- Shared gate lifecycle SPI — common ApprovalGate interface with open/resolve/query methods. More structural but forces different APIs into a common shape.
- PlatformRole reference only — annotations and Javadoc without a shared type. Documents the relationship but doesn't enforce it.
**Rationale:** A marker interface gives type-level discoverability (CDI can find all SUPERVISOR gate implementations) without constraining their APIs. It's the lightest change that makes the relationship machine-readable.
**Trade-offs:** No shared method contract — the marker is a classification, not an API.
**Sources:** PlatformRole.java:18-33, AgentRole.java:29
**Exploration:** quick
**Status:** captured

## D3: Placement in api/model/coordination

**Choice:** Place `SupervisorGateBehavior` in `io.casehub.api.model.coordination` alongside `PlatformRole`.
**Alternatives:**
- api/spi package — treats it as an SPI contract rather than vocabulary. Mixes behavioral vocabulary with implementation contracts.
- api/spi/supervision — new package. Clean separation but excessive structure for a marker interface.
**Rationale:** This is a coordination vocabulary concept — it describes the behavioral contract of the SUPERVISOR role, not an implementation contract. Placing it alongside PlatformRole makes the conceptual relationship clear.
**Trade-offs:** model.coordination grows slightly. The package now has both data types (enums, records) and a behavioral marker.
**Sources:** PlatformRole.java:18, issue-1282 spec §Package
**Exploration:** quick
**Status:** captured

## D4: ActionRiskClassifier is a separate concern

**Choice:** Don't mark ActionRiskClassifier as SupervisorGateBehavior. Document the relationship in Javadoc cross-references.
**Alternatives:**
- Include in marker — mark ActionRiskClassifier too. Conflates classification with gating.
- New marker for classifiers — SupervisorClassification. Adds type-level distinction but introduces a second marker for one class.
**Rationale:** ActionRiskClassifier decides IF gating is needed (triage). It's not a gate implementation — it's the step that precedes the gate. Marking it as SupervisorGateBehavior would blur the classifier/gate boundary.
**Trade-offs:** The full SUPERVISOR pipeline (classify → gate → resolve) isn't captured in a single type hierarchy. The relationship is documented, not enforced.
**Sources:** ActionRiskClassifier.java:34-37, RiskDecision.java:40-53, WorkflowExecutionCompletedHandler.java:98
**Exploration:** quick
**Status:** captured

## D5: Bare marker interface

**Choice:** `SupervisorGateBehavior` is a bare marker — no methods, just Javadoc.
**Alternatives:**
- Marker with metadata — `PlatformRole role()` default method and `String gateDescription()`. Runtime introspection but couples the marker to PlatformRole.
**Rationale:** The marker's purpose is type-level classification, not runtime behavior. Adding methods creates coupling that doesn't serve any current use case.
**Trade-offs:** No runtime introspection. If future tooling needs to enumerate SUPERVISOR gates, it must use CDI Instance scanning, not method calls.
**Sources:** n/a
**Exploration:** quick
**Status:** captured
