# Decisions — Evolution Readiness Methodology (#1131)

## D1: Assessment scope

**Choice:** Project-level — caseId is a correlation key to the project, not individual case instances. A project must be a CaseHub case instance to participate in health tracking — the entire improvement infrastructure (HealthScoreTracker, CircuitBreaker, CategoryTracker, RegressionDetector) is keyed by case UUID.
**Alternatives:**
- Per-case — each case instance has its own health score, sensors pull from case-scoped EventLog
- Both (tiered) — project-level for infrastructure, per-case for operational health
- Standalone metrics — health tracking without the case lifecycle (rejected: the evolution loop presupposes a case — EventLog, signal registry, and CBR integration all flow through the case lifecycle)
**Rationale:** The issue explicitly describes project-level health (test pass rate, build time, coverage). The evolution loop improves the project, not individual case executions. Per-case health is a different concern (case quality tracking). **Scope boundary:** the 10 bootstrap areas measure *case lifecycle health* (case completion rate, worker success rate, orchestration efficiency, etc.) from EventLog events — not CI pipeline health (test pass rate, build time, coverage). CI health assessment requires custom CapabilityArea implementations that CDI-inject CI data services (e.g. a `CiStabilityCapabilityArea` that queries a CI status API). The `CapabilityAreaRegistry` override mechanism (§3) accommodates this — the bootstrap areas cover what the engine can measure natively.
**Trade-offs:** Projects without a CaseHub case instance cannot use improvement health tracking. Acceptable — evolution readiness presupposes the evolution loop, which presupposes a case.
**Sources:** issue #1131, CapabilityArea.java, HealthPolicy.java (weights map names project-level concerns)
**Exploration:** quick
**Status:** revised (R1-08: made case dependency explicit; review-R1-12: clarified case lifecycle health vs CI health scope boundary)

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
**Rationale:** Each CapabilityArea can be at a different readiness level. A project can realistically be L3 for stability (autonomous improvement with CI data) while remaining L0 for cognitive-reasoning (no cognitive layer). Per-area compliance tracks the per-dimension journey to evolution capability. The global project level (min of participating area levels where level > L0) provides a conservative single indicator for the command centre dashboard — **observational only, not a programmatic gate**. No code path in the evolution pipeline checks ComplianceLevel to gate or modulate behavior. The existing gate pipeline (`evolutionEnabled` → health refresh → circuit breaker → consensus → category suppression → budget → conflict) is complete. ComplianceLevel informs the HIL about the project's readiness state but does not enforce it. See D17 for the explicit interaction semantics between ComplianceLevel and `evolutionEnabled`. The ReadinessValidator reports per-area breakdown, enabling the command centre UI (#1132) to show granular progression. ComplianceChecklist is per-area: each area at each level specifies required data flows, minimum assessment quality, and config requirements. The checklist references CapabilityArea implementations and ImprovementConfig settings (not MetricSources — see D2 revision).
**Trade-offs:** More complex model than a single enum. The per-area checklist requires defining requirements for each area at each level. Fixed 4 levels remain — the progression is the methodology, not arbitrary configuration. ComplianceLevel being observational means a misconfigured project (L0 with `evolutionEnabled: true`) is not blocked — the system handles this gracefully via the circuit breaker (health score 0.0 trips immediately) but the misconfiguration is visible in the dashboard.
**Sources:** issue #1131 ("methodology to become evolution-capable" — a per-area journey), #1115 spec §6 (10 areas with varying maturity), issue #1132 (command centre UI needs granular dashboard)
**Exploration:** quick
**Status:** revised (R1-05: changed from single global level to per-area compliance with global rollup; R1-10: ComplianceChecklist now defined as per-area record; review-R1-04: clarified ComplianceLevel is observational, not a gate)

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

**Choice:** ReadinessReport record with targetLevel, projectLevel, List<AreaCompliance> (per-area level + checks), pass/fail verdict, evaluatedAt timestamp
**Alternatives:**
- Text report — human-readable but not programmatically consumable
- Both record + formatted output — scope creep
**Rationale:** Programmatic output is consumed by the command centre UI (#1132) and by tests. AreaCompliance includes area ID, area compliance level, and List<CheckResult> (check name, expected state, actual state, remediation hint). The report provides both the per-area breakdown (from D4 revision) and the global project level. The `evaluatedAt` timestamp (Instant) records when the assessment was performed — essential for snapshot composition (D15) where consumers need to distinguish a fresh assessment from a stale cached one. toString() can be added later if needed. Temporal tracking (progression history) is handled by D9 — reports are persisted as EventLog entries.
**Trade-offs:** No human-readable output format initially. Command centre UI provides the rendering.
**Sources:** issue #1132 (command centre), D4 (per-area compliance model), D9 (EventLog persistence), #1131 spec §4 (ReadinessReport record includes evaluatedAt)
**Exploration:** quick
**Status:** revised (review-R1-13: added evaluatedAt timestamp — already present in spec §4 but missing from decision summary)

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

**Choice:** EventLog-based persistence for compliance state, readiness reports, AND circuit breaker state — all following the same event-sourced reconstruction pattern
**Alternatives:**
- Ephemeral in-memory only — compliance state lost on restart (rejected: compliance progression history needed for command centre UI #1132)
- Database-backed persistence — more complex than needed for the data volume
- Case context storage — possible but EventLog is the established pattern for improvement lifecycle events
- Compliance persistence only, circuit breaker deferred — rejected: circuit breaker state is safety-critical and uses the same EventLog reconstruction pattern; deferring it creates a window where restart forgets the breaker was OPEN, allowing proposals during a health crisis
**Rationale:** The #1115 spec designed event-sourced restart recovery for the circuit breaker (§4 "Restart recovery" section, `restoreFromEventLog` method in spec code) but this was not yet implemented — `ImprovementCircuitBreaker` state is currently in-memory only via `ConcurrentHashMap`, starting empty on every restart. Circuit breaker persistence is safety-critical: a restart that forgets the breaker was OPEN means the system resumes proposing improvements during a health crisis until the next tick re-evaluates (up to 60 minutes). This implementation covers both concerns with the same pattern: (1) `COMPLIANCE_LEVEL_CHANGED` events persist compliance progression history, with current level reconstructed from the most recent entry on restart; (2) `CIRCUIT_BREAKER_TRIPPED`/`CIRCUIT_BREAKER_RECOVERING`/`CIRCUIT_BREAKER_RESET` events (already defined in CaseHubEventType) persist circuit breaker state transitions, with current state and halfOpenCount reconstructed on restart. The circuit breaker event types already exist — only the EventLog write and startup reconstruction logic need to be added to `ImprovementCircuitBreaker`. `ReadinessReport` evaluations are also EventLog entries (command centre can query for historical reports).
**Trade-offs:** EventLog volume increases. Acceptable — improvement lifecycle events are already EventLog-based and low-frequency (compliance changes are rare events, circuit breaker transitions are even rarer).
**Sources:** #1115 spec §4 (circuit breaker EventLog reconstruction — "Restart recovery" section, designed but not yet implemented), CaseHubEventType.java (CIRCUIT_BREAKER_TRIPPED, CIRCUIT_BREAKER_RECOVERING, CIRCUIT_BREAKER_RESET already defined)
**Exploration:** quick (surfaced by reviewer R1-11 — implicit decision made explicit)
**Status:** revised (review-R1-03: expanded scope to include circuit breaker persistence alongside compliance — safety-critical state should not be deferred)

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
**Rationale:** The command centre needs to show what each gate decided, but most ticks are uneventful (evolution disabled, or circuit breaker closed and no consensus). A ring buffer gives the dashboard recent history without EventLog volume. Notable outcomes (blocked by circuit breaker, proposal generated) get EventLog persistence for audit. The TickTrace return type also makes tick() testable — assertions on the trace instead of side-effect inspection. A periodic `TICK_HEARTBEAT` EventLog entry (every 24 ticks or every 24 hours, whichever comes first) provides restart-survivable evidence that the evolution loop is active. Without it, the command centre can show "last notable event: 3 days ago" with no way to distinguish "healthy and quiet" from "broken and silent."
**Trade-offs:** Ring buffer is in-memory — lost on restart. Acceptable: the EventLog captures notable outcomes, the ring buffer rebuilds as ticks fire, and the periodic heartbeat provides a liveness signal that survives restart. The tick() return type change from `void` to `TickTrace` is a breaking change to the existing method signature — `CaseContextChangedEventHandler` (the current sole caller) must be updated to handle or ignore the return value. This is mechanically simple but must be coordinated.
**Sources:** EvolutionTicker.java (current void tick()), CaseHubEventType (CIRCUIT_BREAKER_TRIPPED etc. already exist as EventLog types)
**Exploration:** quick
**Status:** revised (review-R1-08: added periodic TICK_HEARTBEAT for restart observability; review-R1-09: documented tick() return type as breaking change)

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
- Subscribe to EventLog writes instead of CDI events — avoids new event types but couples the broadcaster to the persistence layer rather than the domain layer.
**Rationale:** ExecutionStateBroadcaster already established the pattern of a dedicated broadcaster for a specific concern. It subscribes to `PlanItemStateChangedEvent` and `CaseContextUpdatedEvent` — existing CDI events fired by the engine's execution layer. The evolution components currently write EventLog entries directly but do NOT fire CDI events. For the broadcaster to subscribe, new CDI event types must be introduced and fired alongside the existing EventLog writes. Required CDI event types: (1) `CircuitBreakerStateChangedEvent(UUID caseId, CircuitBreakerState oldState, CircuitBreakerState newState)` — fired by `ImprovementCircuitBreaker.evaluate()` on state transitions; (2) `ComplianceLevelChangedEvent(UUID caseId, ComplianceLevel oldLevel, ComplianceLevel newLevel)` — fired by `ReadinessValidator.validate()` when the level changes; (3) `RegressionDetectedEvent(UUID caseId, UUID improvementCaseId, double confidence, String category)` — fired by `RegressionDetector.onMetricsDegraded()`; (4) `TickEvaluatedEvent(UUID caseId, TickTrace trace)` — fired by `EvolutionTicker.tick()` when notable (gate blocked, proposal generated). These events live in `engine-common` alongside `PlanItemStateChangedEvent`. The broadcaster subscribes to all four via `@ObservesAsync` and composes them into the evolution event stream.
**Trade-offs:** Another BroadcastProcessor in memory + 4 new CDI event types. Acceptable — evolution events are low-frequency (ticks are at most every 60 minutes, most events are gate blocks or proposals). The CDI events are thin wrappers over data that's already being computed.
**Depends on:** D12 (API surface — the broadcaster is wired into the evolution domain)
**Sources:** CaseStreamBroadcaster.java (BroadcastProcessor pattern), ExecutionStateBroadcaster.java (dedicated broadcaster precedent), PlanItemStateChangedEvent.java (CDI event precedent)
**Exploration:** quick
**Status:** revised (review-R1-10: specified the 4 required CDI event types and where they are fired)

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
**Rationale:** The command centre's primary use case is "show me the current state of evolution." A single query provides everything needed to render the dashboard. Most data sources are in-memory beans — `HealthScoreTracker.latestSnapshot()` (cached), `ImprovementCircuitBreaker.state()` (ConcurrentHashMap lookup), `ImprovementCategoryTracker` (ConcurrentHashMap lookup), tick trace ring buffer (in-memory). The compliance level component uses the **latest cached compliance level** — the result of the most recent `ReadinessValidator.validate()` invocation — NOT a re-validation. `ReadinessValidator.validate()` calls `area.assess(caseId, tenancyId)` for each registered area, and each area queries EventLog (a database call). With 10 areas, re-validation would trigger 10+ database queries per snapshot. Instead, the snapshot reads the compliance level from the most recent `COMPLIANCE_LEVEL_CHANGED` EventLog entry or from a cached in-memory value maintained by the validator. Validation is triggered explicitly (via command centre mutation or periodic scheduled evaluation), not on every snapshot query. Additional drill-down queries (tick trace details, research corpus contents, area history) complement the snapshot for deeper investigation.
**Trade-offs:** Snapshot payload may be large if many areas/categories. Acceptable — 10 areas, category states are sparse (only categories with outcome history), tick traces bounded by ring buffer size. The cached compliance level may be stale if validation hasn't been triggered recently — the `evaluatedAt` timestamp (D6) lets consumers assess staleness.
**Depends on:** D6 (evaluatedAt for staleness), D11 (tick traces in snapshot), D12 (API surface)
**Sources:** HealthScoreTracker.java (latestSnapshot()), ImprovementCircuitBreaker.java (state()), ImprovementCategoryTracker.java (states map), ReadinessValidator.java (validate())
**Exploration:** quick
**Status:** revised (review-R1-07: clarified that snapshot uses cached compliance level, not re-validation; composition is cheap for all components except compliance which is cached)

## D16: Structural deny list model

**Choice:** Two-layer deny list: static safety base (ImprovementBudgetEnforcer.STRUCTURAL_DENIED_PATTERNS, read-only, hardcoded) + dynamic operator additions (runtime mutable via command centre mutations, EventLog-persisted). The command centre query shows both layers. The HIL can add patterns (block a category or component from improvement) but cannot remove the built-in safety patterns. Effective deny list = static union dynamic.
**Alternatives:**
- Fully mutable — entire deny list runtime-configurable. Maximum flexibility but a single API call could remove circuit breaker self-protection.
- Read-only only — deny list visible but not mutable. Changes require code deployment.
- Dynamic deny patterns without persistence — lost on restart (rejected: an operator who blocks a category via the command centre loses that protection on every restart — a safety regression)
**Rationale:** The #1115 spec's foundational safety invariant: "The improvement system must not be able to modify its own safety constraints." Making the static list mutable via API would violate this invariant — an improvement case that gains API access could remove its own deny pattern. The additive layer gives operators control over what the system can improve without weakening the built-in safety base. The two layers are visible together in the snapshot so operators understand both. **Persistence:** Dynamic deny pattern mutations are persisted as EventLog entries — `DENY_PATTERN_ADDED(pattern, addedBy, timestamp)` and `DENY_PATTERN_REMOVED(pattern, removedBy, timestamp)`. On restart, the current dynamic deny set reconstructs from the full EventLog history for these event types: replay all ADDED/REMOVED events in order. This follows the same event-sourced reconstruction pattern as D9 (compliance level and circuit breaker state). New `CaseHubEventType` values required: `DENY_PATTERN_ADDED`, `DENY_PATTERN_REMOVED`.
**Trade-offs:** Operators cannot remove built-in deny patterns even when they want to (e.g., to allow improving a safety component). This is intentional — modifying safety components requires code change and review, not an API call. EventLog volume is negligible — deny pattern changes are rare operator actions.
**Depends on:** D9 (EventLog persistence pattern), D12 (API surface)
**Sources:** ImprovementBudgetEnforcer.java (STRUCTURAL_DENIED_PATTERNS), #1115 spec §3 (structural deny list update), #1115 spec safety invariant
**Exploration:** quick
**Status:** revised (review-R1-06: added EventLog persistence for dynamic deny patterns — operator-added deny patterns must survive restart)

## D17: ComplianceLevel vs evolutionEnabled — separate controls

**Choice:** ComplianceLevel and `evolutionEnabled` are intentionally separate controls with no programmatic coupling. ComplianceLevel is a readiness *assessment* (where IS the project). `evolutionEnabled` is a *control knob* (what SHOULD happen). Neither derives from the other.
**Alternatives:**
- Derive `evolutionEnabled` from ComplianceLevel — ComplianceLevel >= L2 implies `evolutionEnabled: true` (rejected: conflates assessment with control; a project at L3 may need evolution temporarily disabled during a release freeze without changing its readiness level)
- ComplianceLevel gates the ticker — add ComplianceLevel as gate 0 in EvolutionTicker (rejected: adds a redundant gate with unclear semantics; the existing 10-gate pipeline already covers all safety concerns)
- Unify into a single control — single enum replaces both (rejected: loses the ability to express "ready but paused" vs "not ready")
**Rationale:** The two controls serve different lifecycle purposes. A project progresses through compliance levels as it gains capability (registers areas, configures signal sources, sets up rollback policy). `evolutionEnabled` is an operational switch — flip it off during a release freeze, flip it back on after. Making them independent means: (1) ComplianceLevel reflects the project's *readiness*, not its *activity*; (2) `evolutionEnabled` controls *activity* without affecting the readiness assessment; (3) misconfiguration states are handled gracefully — L0 with `evolutionEnabled: true` means the ticker runs but no areas are registered, health score is 0.0, circuit breaker trips immediately, no proposals generated. L2 with `evolutionEnabled: false` means the compliance level correctly describes what the project *could* do, even though evolution is currently paused. The ReadinessReport surfaces both states — compliance level AND `evolutionEnabled` — so the command centre dashboard shows when they're misaligned (e.g., "Project at L2 but evolution disabled"). The HIL decides when to reconcile them.
**Trade-offs:** No automatic enforcement that `evolutionEnabled` matches ComplianceLevel. A misconfigured project can have `evolutionEnabled: true` at L0. This is handled safely (circuit breaker trips, no proposals generated) but may confuse operators. The dashboard surfaces the mismatch.
**Sources:** ImprovementConfig.java (effectiveEvolutionEnabled()), EvolutionTicker.java (gate 1 checks evolutionEnabled), D4 (ComplianceLevel is observational)
**Exploration:** quick (surfaced by review-R1-05 — implicit decision made explicit)
**Status:** captured

## D18: tenancyId threading through the evolution pipeline

**Choice:** Add `tenancyId` parameter to `CapabilityArea.assess(UUID caseId, String tenancyId)` and thread it through the complete evolution pipeline: `EvolutionTicker.tick()` → `HealthScoreTracker.refresh()/computeScore()` → `CapabilityArea.assess()` → `EventLogRepository.findByCaseAndTypes(caseId, types, tenancyId)`. Also through `ImprovementCircuitBreaker.evaluate()` → `tracker.computeScore()`.
**Alternatives:**
- Resolve tenancyId inside each CapabilityArea via CDI context — fragile; relies on tenant context propagation which may not be active during ticker evaluation
- tenancyId in a ThreadLocal — same fragility; timer-triggered ticks run outside request scope
- Keep assess(UUID caseId) without tenancyId — EventLogRepository.findByCaseAndTypes() requires tenancyId for tenant-scoped queries; would need a version without tenant filtering, breaking the multi-tenancy model
**Rationale:** `EventLogRepository.findByCaseAndTypes()` requires `tenancyId` for tenant-scoped queries. The SPI has no external consumers yet — the entire point of #1131 is providing the first concrete implementations. Adding the parameter now is a clean SPI change with no migration burden. The tenancyId originates from the `CaseContextChangedEvent` (event-driven path) or from `ImprovementConfig` storage (timer path) and flows through the complete call chain without needing ThreadLocal or CDI context.
**Trade-offs:** Breaking SPI change — `CapabilityArea.assess(UUID caseId)` becomes `assess(UUID caseId, String tenancyId)`. This ripples through 6+ method signatures across `HealthScoreTracker`, `ImprovementCircuitBreaker`, and `EvolutionTicker`. Acceptable — no production consumers implement this SPI yet, and the breakage is the point (forces every area implementation to be tenant-aware).
**Sources:** CapabilityArea.java (current assess signature), EventLogRepository (findByCaseAndTypes requires tenancyId), #1131 spec §2 (SPI change section), EvolutionTicker.java (already has tenancyId in tick()), HealthScoreTracker.java (already accepts tenancyId in computeScore and refresh)
**Exploration:** quick (surfaced by review-R1-16 — implicit decision made explicit)
**Status:** captured

## D19: ABSENT area exclusion from health scoring

**Choice:** `HealthScoreTracker.computeScore()` and `refresh()` skip areas where `assessment.landscapePosition() == ABSENT` from the weighted average. Only areas with real data (landscapePosition != ABSENT) contribute to the composite health score.
**Alternatives:**
- Include all areas with default weight — 7 neutral areas at 0.5 drag the composite down to ~0.66, barely above the circuit breaker's 0.6 threshold, despite every measured area being healthy. This creates a false degradation signal.
- Exclude areas with zero weight instead of ABSENT — requires HealthPolicy.effectiveWeights() to encode activation state in weights, conflating weight (relative importance) with activation (data availability)
- Exclude areas not registered in CapabilityAreaRegistry — the registry contains ALL 10 bootstrap areas after startup; registration ≠ data availability. An area can be registered but return ABSENT because no relevant events exist yet.
**Rationale:** The ABSENT landscape position means "no data available to assess." Including these in the weighted average would penalise projects that haven't configured all 10 areas — the weighted average would pull toward 0.5 (the neutral default) instead of reflecting the actual health of configured areas. The circuit breaker threshold (default 0.6) would trip for a project with 3 healthy areas (scoring 0.9+) and 7 unconfigured areas (scoring 0.5 = ABSENT), giving a composite of ~0.66 — barely above threshold. Excluding ABSENT areas means the composite reflects only what's actually measured. This is the correct behavior for progressive onboarding (D4) — a project at L1 with 3 areas configured should see health based on those 3, not penalised by the 7 not-yet-configured.
**Trade-offs:** A project with NO areas returning non-ABSENT data has a health score of 0.0 (totalWeight is 0, returns 0.0). This is correct — if no area has data, health cannot be computed, and the circuit breaker should trip. The `computeScore()` and `refresh()` implementations already handle this: `totalWeight > 0 ? weightedSum / totalWeight : 0.0`.
**Sources:** HealthScoreTracker.java (computeScore and refresh both implement this exclusion), AbstractCapabilityArea.java (neutralAssessment returns ABSENT), CapabilityAreaAssessment.java (LandscapePosition.ABSENT), #1131 spec §2 (ABSENT exclusion section)
**Exploration:** quick (surfaced by review-R1-17 — implicit decision made explicit)
**Status:** captured
