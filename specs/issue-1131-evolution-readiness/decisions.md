# Decisions — Evolution Readiness Methodology (#1131)

## D1: Assessment scope

**Choice:** Project-level — caseId is a correlation key to the project, not individual case instances
**Alternatives:**
- Per-case — each case instance has its own health score, sensors pull from case-scoped EventLog
- Both (tiered) — project-level for infrastructure, per-case for operational health
**Rationale:** The issue explicitly describes project-level health (test pass rate, build time, coverage). The evolution loop improves the project, not individual case executions. Per-case health is a different concern (case quality tracking).
**Trade-offs:** Per-case operational health (was this case executed well?) is out of scope. Can be added later as a separate CapabilityArea tier.
**Sources:** issue #1131, CapabilityArea.java, HealthPolicy.java (weights map names project-level concerns)
**Exploration:** quick
**Status:** captured

## D2: Data ingestion model

**Choice:** MetricSource SPI in api/ — projects implement it to feed metrics, CapabilityArea impls consume it
**Alternatives:**
- EventLog-sourced — CapabilityAreas query EventLog for domain events and compute health from patterns
- Configuration-declared — YAML config declares health data inline, static until next config push
- External push API — REST/webhook endpoint receives metrics from CI/CD
**Rationale:** Clean separation: engine defines the contract (what metrics mean), projects provide the telemetry (how metrics are collected). Consistent with existing SPI-first architecture. No-op default means engine compiles without external dependencies.
**Trade-offs:** Projects must implement MetricSource — not zero-config. But L0 (inert) is explicitly the "nothing configured" state, so this is by design.
**Sources:** contributor-guide.md §SPI Architecture, CapabilityArea.java, ImprovementConfig.java
**Exploration:** quick
**Status:** captured

## D3: Capability area scope

**Choice:** All 8 areas get concrete implementations — rule-based for measurable areas, heuristic defaults for abstract ones
**Alternatives:**
- Core 4 only (stability, performance, execution, safety) — stub the rest
- Just stability + performance — minimum viable
**Rationale:** All 8 areas are named in HealthPolicy weights. Providing heuristic defaults for abstract areas (autonomy, cognitive-reasoning, coordination) means the health score is meaningful from day one. The cognitive layer (Epics 2-3) replaces heuristics with real assessment.
**Trade-offs:** Heuristic defaults for abstract areas may encode wrong assumptions. Mitigated by @DefaultBean — consumers override when they have real data.
**Sources:** HealthPolicy.java (effectiveWeights), memory: evolution-from-zero
**Exploration:** quick
**Status:** captured

## D4: Compliance level model

**Choice:** ComplianceLevel enum (L0_INERT, L1_OBSERVE, L2_PROPOSE, L3_AUTONOMOUS) + ComplianceChecklist records
**Alternatives:**
- YAML schema with profiles — more flexible custom levels but adds DSL complexity
- Annotation-based — compile-time safety but rigid, can't vary per deployment
**Rationale:** Enum is discoverable, type-safe, and progressive (L1 is a subset of L2). Checklist records list required MetricSources, CapabilityAreas, and config per level. ReadinessValidator checks a project against a level.
**Trade-offs:** Fixed 4 levels — can't define custom intermediate levels. Acceptable because the L0–L3 progression is the methodology, not arbitrary configuration.
**Sources:** issue #1131, memory: evolution-readiness-methodology
**Exploration:** quick
**Status:** captured

## D5: Module placement

**Choice:** runtime-core — same module as HealthScoreTracker, CapabilityAreaRegistry, EvolutionTicker
**Alternatives:**
- New module: evolution — clean separation but adds module overhead
- common-core — more reusable but pulls improvement concerns into the shared layer
**Rationale:** All improvement infrastructure lives in runtime-core/internal/improvement/. Adding concrete CapabilityArea impls and the ReadinessValidator here is consistent. Uses @DefaultBean for consumer override.
**Trade-offs:** runtime-core grows. Acceptable — the improvement package is already cohesive and self-contained.
**Sources:** contributor-guide.md §Module Structure, improvement package listing
**Exploration:** quick
**Status:** captured

## D6: Validator output model

**Choice:** ReadinessReport record with targetLevel, currentLevel, List<CheckResult>, pass/fail verdict
**Alternatives:**
- Text report — human-readable but not programmatically consumable
- Both record + formatted output — scope creep
**Rationale:** Programmatic output is consumed by the command centre UI (#1132) and by tests. CheckResult includes check name, expected state, actual state, and remediation hint. toString() can be added later if needed.
**Trade-offs:** No human-readable output format initially. Command centre UI provides the rendering.
**Sources:** issue #1132 (command centre), memory: command-centre-conductor
**Exploration:** quick
**Status:** captured

## D7: CapabilityArea registration

**Choice:** CDI auto-discovery via Instance<CapabilityArea> — all @ApplicationScoped implementations auto-registered
**Alternatives:**
- Explicit registration — projects call areaRegistry.register() manually
- Config-driven — YAML declares which areas to activate
**Rationale:** Consistent with NamedStrategy pattern already in the codebase. Projects add areas by providing @ApplicationScoped implementations. @DefaultBean defaults yield to consumer overrides.
**Trade-offs:** No per-deployment area selection without CDI configuration. If needed, areas can check ImprovementConfig to self-disable.
**Depends on:** D5 (module placement — auto-discovery requires same CDI context)
**Sources:** contributor-guide.md §CDI Conventions, EngineStrategyResolver pattern
**Exploration:** quick
**Status:** captured

## D8: MetricSource granularity

**Choice:** Named metric values — MetricSource.query(metricName) returns Optional<MetricValue> (double, Instant, labels)
**Alternatives:**
- Typed metric per area — stronger typing but more interfaces
- Metric registry with push — supports time-series but more infrastructure
**Rationale:** Simple, extensible, low coupling. CapabilityAreas query by well-known metric names (e.g. 'test.pass.rate', 'build.duration.seconds'). New metrics don't require new interfaces. Labels enable filtering (e.g. module-scoped metrics).
**Trade-offs:** No time-series storage — metrics are point-in-time snapshots. Trend analysis would need the metric registry approach. HealthScoreTracker already maintains snapshot history, so this is acceptable for now.
**Sources:** CapabilityAreaAssessment.java (has assessedAt timestamp), HealthScoreTracker.java (maintains history)
**Exploration:** quick
**Status:** captured
