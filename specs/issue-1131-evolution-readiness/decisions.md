# Decisions — Evolution Readiness Methodology (#1131)

## D1: Assessment scope

**Choice:** Project-level — caseId is a correlation key to the project, not individual case instances. A project must be a CaseHub case instance to participate in health tracking — the entire improvement infrastructure (HealthScoreTracker, CircuitBreaker, CategoryTracker, RegressionDetector) is keyed by case UUID.
**Alternatives:**
- Per-case — each case instance has its own health score, sensors pull from case-scoped EventLog
- Both (tiered) — project-level for infrastructure, per-case for operational health
- Standalone metrics — health tracking without the case lifecycle (rejected: the evolution loop presupposes a case — EventLog, signal registry, and CBR integration all flow through the case lifecycle)
**Rationale:** The issue explicitly describes project-level health (test pass rate, build time, coverage). The evolution loop improves the project, not individual case executions. Per-case health is a different concern (case quality tracking).
**Trade-offs:** Projects without a CaseHub case instance cannot use improvement health tracking. Acceptable — evolution readiness presupposes the evolution loop, which presupposes a case.
**Sources:** issue #1131, CapabilityArea.java, HealthPolicy.java (weights map names project-level concerns)
**Exploration:** quick
**Status:** revised (R1-08: made case dependency explicit as a scope boundary)

## D2: Data ingestion model

**Choice:** Direct query — each CapabilityArea.assess() implementation directly queries its data sources (EventLog, ActivityTracker, signal registry, CDI-injected external services)
**Alternatives:**
- MetricSource SPI — intermediate abstraction between data sources and CapabilityArea (rejected: contradicts #1115 §4 which explicitly removed MetricsSnapshot and rejected parallel metrics infrastructure)
- Configuration-declared — YAML config declares health data inline, static until next config push
- External push API — REST/webhook endpoint receives metrics from CI/CD
**Rationale:** The #1115 spec (§4) explicitly states "No parallel metrics infrastructure" and removes MetricsSnapshot. Each CapabilityArea implementation already knows what data it needs. A MetricSource SPI creates an abstraction layer between data sources and consumers with no architectural benefit — it indirects the access without adding value. External metrics (CI pass rate, test coverage, build time) are provided by capability area implementations that CDI-inject whatever services they need — not by a separate metrics pipeline.
**Trade-offs:** Each CapabilityArea implementation is responsible for its own data access — more coupled per-area but removes the unnecessary MetricSource indirection. The registry-based override pattern (§3) allows consumers to provide richer implementations.
**Sources:** #1115 spec §4 (explicit rejection of MetricsSnapshot), issue #1131 (describes data flow through EventLog entries, CI outcomes, test results — not through a new SPI), CapabilityArea.java, HealthScoreTracker.java
**Exploration:** quick
**Status:** revised (R1-02: aligned with #1115 spec's explicit rejection of parallel metrics infrastructure)

## D3: Capability area scope

**Choice:** All 10 areas from the #1115 bootstrap taxonomy get concrete implementations — rule-based for measurable areas, heuristic defaults for abstract ones
**Alternatives:**
- Core 4 only (stability, performance, execution, safety) — stub the rest
- 8 areas from HealthPolicy weights — silently drops perception and cognitive-memory (rejected: weights should follow the authoritative taxonomy, not the other way around)
- Just stability + performance — minimum viable
**Rationale:** The #1115 methodology spec (§10) defines 10 bootstrap areas: Stability, Performance, Execution, Coordination, Perception, Autonomy, Cognitive reasoning, Cognitive memory, Safety, Integration. HealthPolicy.effectiveWeights() must be updated to include perception and cognitive-memory (currently 8 of 10). Providing heuristic defaults for abstract areas means the health score is meaningful from day one.
**Trade-offs:** Heuristic defaults for abstract areas (perception, cognitive-memory, cognitive-reasoning) may encode wrong assumptions. Mitigated by registry-based override (§3) — consumers replace default areas when they have real data. Perception and Cognitive memory will have minimal assessment capability until the cognitive layer (Epics 2-3) provides real data.
**Sources:** #1115 spec §6 (10 bootstrap areas), methodology spec §10 (full taxonomy table), HealthPolicy.java (needs perception and cognitive-memory weights added)
**Exploration:** quick
**Status:** revised (R1-04: corrected from 8 to 10 areas to match #1115 taxonomy; HealthPolicy.effectiveWeights() update required)

## D4: Compliance level model

**Choice:** Per-area ComplianceLevel enum (L0_INERT, L1_OBSERVE, L2_PROPOSE, L3_AUTONOMOUS) + per-area ComplianceChecklist + global project level computed as min(area levels where level > L0), falling back to L0
**Alternatives:**
- Single global ComplianceLevel — creates a cliff where one lagging area blocks the entire project (rejected: cognitive-reasoning at L0 would block the entire project from L2 until Epics 2-3 ship)
- YAML schema with profiles — more flexible custom levels but adds DSL complexity
- Annotation-based — compile-time safety but rigid, can't vary per deployment
**Rationale:** Each CapabilityArea can be at a different readiness level. A project can realistically be L3 for stability (autonomous improvement with CI data) while remaining L0 for cognitive-reasoning (no cognitive layer). Per-area compliance tracks the per-dimension journey to evolution capability. The global project level (min of participating area levels where level > L0) provides a conservative single indicator for gating behavior. The ReadinessValidator reports per-area breakdown, enabling the command centre UI (#1132) to show granular progression. ComplianceChecklist is per-area: each area at each level specifies required data flows, minimum assessment quality, and config requirements. The checklist references CapabilityArea implementations and ImprovementConfig settings (not MetricSources — see D2 revision).
**Trade-offs:** More complex model than a single enum. The per-area checklist requires defining requirements for each area at each level. Fixed 4 levels remain — the progression is the methodology, not arbitrary configuration.
**Sources:** issue #1131 ("methodology to become evolution-capable" — a per-area journey), #1115 spec §6 (10 areas with varying maturity), issue #1132 (command centre UI needs granular dashboard)
**Exploration:** quick
**Status:** revised (R1-05: changed from single global level to per-area compliance with global rollup; R1-10: ComplianceChecklist now defined as per-area record)

## D5: Module placement

**Choice:** Follow #1115 placement pattern — model types and enums in api/model/stigmergy, SPI interfaces in api/spi/improvement, implementations in runtime-core/internal/improvement
**Alternatives:**
- Everything in runtime-core — incorrect per contributor guide §SPI Architecture; model types are API surface consumed by other modules and external projects
- New module: evolution — clean separation but adds module overhead
- common-core — more reusable but pulls improvement concerns into the shared layer
**Rationale:** The #1115 spec (§11) established the placement pattern with an explicit module placement table: model types (HealthPolicy, RollbackPolicy, CapabilityAreaAssessment) in api/model/stigmergy, SPI interfaces (CapabilityArea, ResearchScoper) in api/spi/improvement, implementations (HealthScoreTracker, CapabilityAreaRegistry) in runtime-core. The #1131 components follow the same split: ComplianceLevel enum → api, ComplianceChecklist record → api/model/stigmergy, ReadinessReport record → api/model/stigmergy, ReadinessValidator → runtime-core, concrete CapabilityArea impls → runtime-core/internal/improvement/area.
**Trade-offs:** runtime-core grows with implementations. api/ grows with model types. Both are architecturally correct — no new modules needed.
**Sources:** #1115 spec §11 (module placement table), contributor-guide.md §SPI Architecture, CapabilityAreaAssessment.java (already in api/model/stigmergy), CapabilityArea.java (already in api/spi/improvement)
**Exploration:** quick
**Status:** revised (R1-06: corrected to match #1115 placement pattern with api/runtime-core split)

## D6: Validator output model

**Choice:** ReadinessReport record with targetLevel, projectLevel, List<AreaCompliance> (per-area level + checks), pass/fail verdict
**Alternatives:**
- Text report — human-readable but not programmatically consumable
- Both record + formatted output — scope creep
**Rationale:** Programmatic output is consumed by the command centre UI (#1132) and by tests. AreaCompliance includes area ID, area compliance level, and List<CheckResult> (check name, expected state, actual state, remediation hint). The report provides both the per-area breakdown (from D4 revision) and the global project level. toString() can be added later if needed. Temporal tracking (progression history) is handled by D9 — reports are persisted as EventLog entries.
**Trade-offs:** No human-readable output format initially. Command centre UI provides the rendering.
**Sources:** issue #1132 (command centre), D4 (per-area compliance model), D9 (EventLog persistence)
**Exploration:** quick
**Status:** captured

## D7: CapabilityArea registration

**Choice:** CDI-discovered bootstrap that calls register() at startup — preserves the registry's runtime mutability for taxonomy evolution
**Alternatives:**
- CDI auto-discovery via Instance<CapabilityArea> replacing the registry (rejected: breaks runtime register()/deprecate() that #1115 taxonomy evolution depends on — CDI Instance resolves at startup, cannot add areas at runtime)
- Explicit manual registration — no automatic bootstrap, projects must call register() themselves
- Config-driven — YAML declares which areas to activate
**Rationale:** The #1115 spec (§6) explicitly requires runtime register()/deprecate() for taxonomy evolution: merge (register new combined area + deprecate old), split (register new sub-areas + deprecate parent). All operations emit CAPABILITY_AREA_CHANGED EventLog entries. CDI Instance<CapabilityArea> resolves at startup and cannot add new instances at runtime. The solution: a startup observer discovers @ApplicationScoped CapabilityArea implementations via CDI and calls registry.register() for each. This gives CDI-based default discovery AND runtime mutability. CapabilityArea does NOT extend NamedStrategy (verified via ide_type_hierarchy — NamedStrategy is in platform.api.routing, not present in the engine project). NamedStrategies are per-case-selectable routing strategies resolved by ID; CapabilityAreas are assessment providers that all run on every health evaluation — architecturally different patterns.
**Trade-offs:** Two mechanisms (CDI discovery for bootstrap + explicit registry for runtime evolution) instead of one. Acceptable — they serve different purposes. The bootstrap observer is a one-time startup concern; the registry supports the full lifecycle.
**Depends on:** D5 (module placement)
**Sources:** #1115 spec §6 (taxonomy evolution — register/deprecate/merge/split), CapabilityAreaRegistry.java (existing register()/deprecate() API), CapabilityArea.java type hierarchy (no NamedStrategy relationship), EngineStrategyResolver.java (handles NamedStrategy, not CapabilityArea)
**Exploration:** quick
**Status:** revised (R1-03: restored runtime register()/deprecate() via CDI bootstrap observer instead of CDI replacement)

## D8: MetricSource granularity

**Choice:** Withdrawn — MetricSource SPI no longer exists (D2 revised to remove it)
**Alternatives:** N/A
**Rationale:** D2 was revised to remove the MetricSource SPI entirely, aligning with the #1115 spec's explicit rejection of parallel metrics infrastructure. Without MetricSource, D8 has no subject. Each CapabilityArea.assess() implementation defines its own data access pattern with full type safety internal to the area.
**Sources:** D2 revision, #1115 spec §4
**Exploration:** quick
**Status:** withdrawn (R1-07: dependent on D2 which was revised to remove MetricSource)

## D9: Persistence strategy for #1131 components

**Choice:** EventLog-based persistence for compliance state and readiness reports, following the #1115 circuit breaker pattern
**Alternatives:**
- Ephemeral in-memory only — compliance state lost on restart (rejected: compliance progression history needed for command centre UI #1132)
- Database-backed persistence — more complex than needed for the data volume
- Case context storage — possible but EventLog is the established pattern for improvement lifecycle events
**Rationale:** The #1115 spec designed event-sourced restart recovery for the circuit breaker (§4 "Restart recovery" section, `restoreFromEventLog` method in spec code) but this was not yet implemented — `ImprovementCircuitBreaker` state is currently in-memory only via `ConcurrentHashMap`, starting empty on every restart. The #1131 compliance level persistence is the first implementation of this pattern: `COMPLIANCE_LEVEL_CHANGED` events persist progression history, and on restart the current level reconstructs from the most recent entry. This establishes the precedent for completing the circuit breaker event-sourcing (tracked as a follow-up). `ReadinessReport` evaluations are also EventLog entries (command centre can query for historical reports).
**Trade-offs:** EventLog volume increases. Acceptable — improvement lifecycle events are already EventLog-based and low-frequency (compliance changes are rare events, not per-tick).
**Sources:** #1115 spec §4 (circuit breaker EventLog reconstruction — "Restart recovery" section, designed but not yet implemented)
**Exploration:** quick (surfaced by reviewer R1-11 — implicit decision made explicit)
**Status:** captured
