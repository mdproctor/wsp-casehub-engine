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

---

# Decisions — Command Centre Conductor (#1132)

## D10: Scope and spec boundary

**Choice:** One unified spec covering both command centre (API/observability surface) and bootstrap from zero (readiness-driven progression). The bootstrap IS the HIL's first interaction with the command centre — surfacing readiness gaps and closing them through mutations.
**Alternatives:**
- Two specs — command centre API first, bootstrap second. Cleaner separation but delays the end-to-end story and duplicates the API surface.
- Command centre only — defer bootstrap to Epic 2/3 (LLM research). Loses the progressive disclosure story.
**Rationale:** The command centre surfaces ReadinessValidator gaps. The bootstrap is the HIL using the command centre to close those gaps. Splitting them would mean designing the API surface twice. The bootstrap adds no new infrastructure — it's the readiness progression from D4 exposed through the API from D12.
**Trade-offs:** Larger spec. Mitigated by bootstrap being thin — it's a documentation/usage pattern on top of existing ReadinessValidator, not new components.
**Sources:** issue #1132, D4 (per-area compliance model), memory: command-centre-conductor, memory: evolution-from-zero
**Exploration:** quick
**Status:** captured

## D11: Gate pipeline tracing

**Choice:** TickTrace record — tick() returns a TickTrace with per-gate results (passed/blocked + reason). Recent traces stored in a ring buffer per case (queryable via command centre API). EventLog gets a TICK_EVALUATED entry only when something notable happens (gate blocks, proposal generated, regression detected).
**Alternatives:**
- Full EventLog per tick — every tick writes a detailed EventLog entry. Complete audit trail but high volume (one entry per tick interval + every event-driven tick).
- Observable pattern — tick() emits CDI events for each gate decision. More decoupled but adds CDI event overhead per gate per tick.
**Rationale:** The command centre needs to show what each gate decided, but most ticks are uneventful (evolution disabled, or circuit breaker closed and no consensus). A ring buffer gives the dashboard recent history without EventLog volume. Notable outcomes (blocked by circuit breaker, proposal generated) get EventLog persistence for audit. The TickTrace return type also makes tick() testable — assertions on the trace instead of side-effect inspection.
**Trade-offs:** Ring buffer is in-memory — lost on restart. Acceptable: the EventLog captures notable outcomes (the ones worth persisting), and the ring buffer rebuilds naturally as ticks fire.
**Sources:** EvolutionTicker.java (current void tick()), CaseHubEventType (CIRCUIT_BREAKER_TRIPPED etc. already exist as EventLog types)
**Exploration:** quick
**Status:** captured

## D12: API surface organization

**Choice:** Single @McpDomain("engine/evolution") class with queries (evolution state snapshot, readiness report, tick history, research corpus, category states, deny list) and mutations (pause/unpause category, manual circuit breaker reset, add/remove runtime deny patterns, trigger readiness validation).
**Alternatives:**
- Split domains — separate engine/evolution/health, engine/evolution/research, engine/evolution/controls. Finer-grained but fragments a unified concern across multiple classes.
- Extend existing domains — add to engine/events and engine/control. No new domain but stretches existing responsibilities beyond their original purpose.
**Rationale:** The evolution loop is one system with one conceptual surface. The command centre is the single pane of glass for that system. A single McpDomain keeps all evolution operations discoverable together. The existing domains (engine/control, engine/events) have clear existing responsibilities (case lifecycle control, event log queries) that should not be diluted.
**Trade-offs:** Single class may grow large if many endpoints. Mitigated by the API being query-heavy with few mutations — most of the logic is in existing beans (ReadinessValidator, HealthScoreTracker, etc.).
**Depends on:** D10 (scope)
**Sources:** DefaultEngineCaseControlApi.java (@McpDomain pattern), DefaultEngineEventLogApi.java (query pattern)
**Exploration:** quick
**Status:** captured

## D13: Evolution event streaming

**Choice:** Dedicated EvolutionStreamBroadcaster with its own BroadcastProcessor<EvolutionEvent>. Subscribes to evolution-specific CDI events (@ObservesAsync). Follows the ExecutionStateBroadcaster precedent — separate concern, separate stream.
**Alternatives:**
- Extend CaseStreamBroadcaster — add evolution event handlers to the existing broadcaster. One stream per case, all events mixed. Simpler but every consumer gets everything and must filter.
- No streaming — query-only API, command centre polls. Simpler to build but no real-time updates for gate decisions, circuit breaker trips, or regression detection.
**Rationale:** ExecutionStateBroadcaster already established the pattern of a dedicated broadcaster for a specific concern. Evolution events are high-signal, low-frequency — circuit breaker trips, regression detection, compliance changes. They deserve their own stream. Consumers (command centre UI, monitoring tools) subscribe to evolution events without filtering case lifecycle noise.
**Trade-offs:** Another BroadcastProcessor in memory. Acceptable — evolution events are low-frequency (ticks are at most every 60 minutes, most events are gate blocks or proposals).
**Depends on:** D12 (API surface — the broadcaster is wired into the evolution domain)
**Sources:** CaseStreamBroadcaster.java (BroadcastProcessor pattern), ExecutionStateBroadcaster.java (dedicated broadcaster precedent)
**Exploration:** quick
**Status:** captured

## D14: Bootstrap from zero mechanism

**Choice:** Readiness-driven progression through command centre mutations. No separate "briefing" artifact. The ReadinessValidator (from #1131) surfaces gaps at each compliance level. The HIL uses the command centre API to close gaps: validate readiness → read remediation hints → execute mutations (enable evolution, configure rollback policy, verify area registration). Bootstrap IS the L0→L1→L2→L3 journey from D4.
**Alternatives:**
- Seed configuration file — bootstrap YAML defining initial areas, health policy, improvement config. New artifact format with new parsing.
- Interactive wizard flow — multi-step API: startBootstrap() → assessCurrent() → proposeAreas() → confirmAreas(). Guided but complex server-side state machine with session state.
**Rationale:** The ReadinessReport already contains per-area compliance levels, check results with expected/actual values, and remediation hints. The command centre query surfaces this report. The mutations (enable evolution, set rollback policy, configure consensus threshold) already correspond to ImprovementConfig fields. No new infrastructure is needed — the bootstrap is the ReadinessValidator's output consumed through the command centre's API surface.
**Trade-offs:** Less guided than a wizard — the HIL must interpret the readiness report and decide which mutations to apply. Mitigated by remediation hints being specific and actionable ("set evolutionEnabled=true", "configure rollbackPolicy with autoRevertThreshold").
**Depends on:** D10 (scope), D12 (API surface)
**Sources:** ReadinessValidator.java, ReadinessReport.java (CheckResult with remediation field), issue #1132 ("bootstrap from zero — no pre-programmed improvement scripts"), memory: evolution-from-zero
**Exploration:** quick
**Status:** captured

## D15: Evolution state snapshot model

**Choice:** Single composed EvolutionStateSnapshot record returned by one query. Composed server-side from existing components: health score + per-area breakdown (from HealthScoreTracker), circuit breaker state (from ImprovementCircuitBreaker), category states with pause/suppression info (from ImprovementCategoryTracker), compliance level (from ReadinessValidator), recent tick traces (from TickTrace ring buffer), active improvements count (from ImprovementBudgetEnforcer).
**Alternatives:**
- Separate queries — individual getHealth(), getCircuitBreakerState(), getCategoryStates(), etc. Requires N round-trips to build the dashboard. Better for partial refreshes but worse for initial load.
- Snapshot + targeted queries — initial snapshot for full view, targeted queries for drill-down. Two API patterns.
**Rationale:** The command centre's primary use case is "show me the current state of evolution." A single query provides everything needed to render the dashboard. All data sources are in-memory beans (HealthScoreTracker, ImprovementCircuitBreaker, ImprovementCategoryTracker) — composition is cheap. Additional drill-down queries (tick trace details, research corpus contents, area history) complement the snapshot for deeper investigation.
**Trade-offs:** Snapshot payload may be large if many areas/categories. Acceptable — 10 areas, category states are sparse (only categories with outcome history), tick traces bounded by ring buffer size.
**Depends on:** D11 (tick traces in snapshot), D12 (API surface)
**Sources:** HealthScoreTracker.java (latestSnapshot()), ImprovementCircuitBreaker.java (state()), ImprovementCategoryTracker.java (states map), ReadinessValidator.java (validate())
**Exploration:** quick
**Status:** captured

## D16: Structural deny list model

**Choice:** Two-layer deny list: static safety base (ImprovementBudgetEnforcer.STRUCTURAL_DENIED_PATTERNS, read-only, hardcoded) + dynamic operator additions (runtime mutable via command centre mutations). The command centre query shows both layers. The HIL can add patterns (block a category or component from improvement) but cannot remove the built-in safety patterns. Effective deny list = static union dynamic.
**Alternatives:**
- Fully mutable — entire deny list runtime-configurable. Maximum flexibility but a single API call could remove circuit breaker self-protection.
- Read-only only — deny list visible but not mutable. Changes require code deployment.
**Rationale:** The #1115 spec's foundational safety invariant: "The improvement system must not be able to modify its own safety constraints." Making the static list mutable via API would violate this invariant — an improvement case that gains API access could remove its own deny pattern. The additive layer gives operators control over what the system can improve without weakening the built-in safety base. The two layers are visible together in the snapshot so operators understand both.
**Trade-offs:** Operators cannot remove built-in deny patterns even when they want to (e.g., to allow improving a safety component). This is intentional — modifying safety components requires code change and review, not an API call.
**Depends on:** D12 (API surface)
**Sources:** ImprovementBudgetEnforcer.java (STRUCTURAL_DENIED_PATTERNS), #1115 spec §3 (structural deny list update), #1115 spec safety invariant
**Exploration:** quick
**Status:** captured
