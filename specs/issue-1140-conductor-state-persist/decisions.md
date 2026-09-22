# Decisions — Conductor State Persistence (#1140)

## D1: Persistence strategy — in-memory now, database-backed later

**Choice:** In-memory implementations as shipped defaults. No EventLog-replay durability. When durable implementations arrive, they follow the new platform direction: JPA-backed entities (database for durability), EventLog for audit only.
**Alternatives:**
- EventLog-replay-backed impls (reconstruct from event history on startup) — the #1132 spec designed this, but #1150 (persistence layer coherence) found two silent data-loss bugs (#1151, #1152) in EventLog-replay-based context recovery. The platform is moving away from this pattern.
- JPA/database-backed impls now — premature; the SPI boundary is the prerequisite. JPA implementations can follow once the SPIs are stable.
**Rationale:** All stores are low-mutation (human-timescale operator configuration or infrequent gate pipeline events). DB writes are trivially cheap. The EventLog writes already in #1132 (GATE_PENDING, DENY_PATTERN_ADDED, etc.) remain correct as audit trail, not durability mechanism. The SPI boundary is what matters now — implementations are swappable. Durable implementations will be JPA entities with their own tables, not CaseContext entries — these stores have typed query semantics (findPending, isBlocked) incompatible with CaseContext's flat key-value model.
**Trade-offs:** State lost on restart. The impact differs by store lifecycle (see D3): inbox entries and improvement blocks are transient coordination state where restart loss is tolerable — the improvement lifecycle restarts. Watch patterns and dynamic deny patterns are operator safety configuration where restart loss is a correctness gap — operators must re-apply configuration after restart. This is a known limitation of the in-memory default, not an acceptable permanent state. `InMemoryResearchCorpus` is precedent for this pattern (in-memory @DefaultBean that loses on restart).
**Sources:** #1150 (persistence layer coherence epic), #1151/#1152 (EventLog-replay data-loss bugs), #1132 spec D16/D26 (EventLog persistence design, now superseded for durability), InMemoryResearchCorpus (runtime-core — precedent for in-memory @DefaultBean that loses on restart)
**Exploration:** quick
**Status:** captured

## D2: SPI module placement — engine-common/spi

**Choice:** SPI interfaces in `common-core/spi` (`io.casehub.engine.common.spi`), alongside CaseInstanceRepository and EventLogRepository.
**Alternatives:**
- `api/spi/improvement` — where SummarizationProvider, EscalationProvider, CapabilityArea live. But those are consumer-facing SPIs that external projects implement. The conductor state repositories are engine-internal coordination stores.
**Rationale:** The contributor guide §SPI Architecture categorises SPIs by type: operational SPIs in `api/spi/`, persistence SPIs in `common-core/spi/`. The conductor stores are persistence SPIs — they manage domain state storage, not pluggable operational behavior. Placement by convention, not by dependency necessity (these SPIs reference only `api/` model types and could technically live in `api/spi/` without circular deps).
**Trade-offs:** `engine-common/spi` grows. Acceptable — it's the correct location by the existing boundary rules.
**Sources:** CaseInstanceRepository.java (common-core/spi), EventLogRepository.java (common-core/spi), SummarizationProvider.java (api/spi/improvement), contributor-guide.md §SPI Architecture
**Exploration:** quick
**Status:** captured

## D3: Separate WatchPatternStore from ConductorInboxRepository

**Choice:** Four SPIs instead of three — split watch patterns out of ConductorInboxRepository into a separate WatchPatternStore.
**Alternatives:**
- Combined ConductorInboxRepository covering both entries and watch patterns (as the issue originally proposed) — simpler, fewer interfaces, but couples different lifecycle concerns.
**Rationale:** Inbox entries are transient case-lifecycle state (created by the gate pipeline, resolved by the conductor, potentially timed out). Watch patterns are persistent operator configuration that survives across improvement cycles. Different lifecycle semantics → different SPIs. ConductorInboxManager injects both and keeps the business logic coordination.
**Trade-offs:** 4 SPIs instead of 3. Each is small and focused. ConductorInboxManager has two injected repositories instead of one.
**Sources:** ConductorInboxManager.java (current entries + watchPatterns maps), #1132 spec §4 (watch pattern persistence described separately from inbox entries)
**Exploration:** quick
**Status:** captured

## D4: In-memory implementations in runtime-core with @DefaultBean

**Choice:** In-memory implementations in `runtime-core` (`io.casehub.engine.internal.improvement`) with `@DefaultBean @ApplicationScoped`. These are shipped defaults, not test infrastructure.
**Alternatives:**
- engine-support-core/persistence-memory — the CaseInstanceRepository pattern. But that's for test-only POJOs without CDI annotations. The conductor impls ARE the production implementation until a persistence backend is added.
**Rationale:** Matches the DefaultSummarizationProvider / DefaultEscalationProvider pattern — shipped engine defaults that consumers can override with `@ApplicationScoped` (non-@DefaultBean) implementations.
**Trade-offs:** The @DefaultBean annotation means these disappear when a single non-default impl is registered. Correct for 1:1 replacement SPIs (PP-20260921-b7c277 confirms @DefaultBean is appropriate here — these are not multi-instance SPIs).
**Sources:** DefaultSummarizationProvider.java (runtime-core, @DefaultBean), DefaultEscalationProvider.java (runtime-core, @DefaultBean), PP-20260921-b7c277 (no-defaultbean-multi-instance-spi protocol)
**Exploration:** quick
**Status:** captured

## D5: Resettable moves to repository implementations

**Choice:** Repository implementations implement Resettable, not domain beans. Domain beans become stateless with respect to persistence concerns.
**Alternatives:**
- Domain beans keep Resettable — but reset() is a storage concern, not business logic.
**Rationale:** After SPI extraction, persistent state lives in the repository implementations. reset()/clear() of persistent state is a storage operation. ImprovementBudgetEnforcer retains ephemeral runtime state (activeImprovements, dailyCounts, lastCompletionTime) that appropriately stays in the domain bean — this is not persistence-worthy state, it's transient tracking that correctly resets to zero on restart.
**Trade-offs:** ImprovementBudgetEnforcer retains Resettable for its remaining in-memory state (activeImprovements, dailyCounts, lastCompletionTime) — only dynamicDenyPatterns is extracted. ImprovementCoordinator loses Resettable entirely (all state extracted). ConductorInboxManager loses Resettable (all state split to ConductorInboxRepository + WatchPatternStore).
**Sources:** ConductorInboxManager.java (reset clears entries + watchPatterns), ImprovementCoordinator.java (reset clears blocks), ImprovementBudgetEnforcer.java (reset clears 4 fields, only dynamicDenyPatterns extracted)
**Exploration:** quick
**Status:** revised (review-R1-12: "stateless logic" → "stateless with respect to persistence concerns"; added ephemeral state rationale)

## D6: Explicit tenancyId parameter on all SPI methods

**Choice:** All SPI methods include `String tenancyId` as the last parameter, matching `EventLogRepository`'s convention.
**Alternatives:**
- Omit tenancyId — the in-memory implementations don't use it. But this would make the SPI contract incorrect for multi-tenant database-backed implementations.
- Resolve tenancyId inside implementations via CDI context — fragile; timer-triggered ticks run outside request scope.
**Rationale:** The SPI contract must support multi-tenant database-backed implementations where tenancyId is part of the WHERE clause. In-memory implementations accept but ignore tenancyId (no tenant isolation in single-map storage). This matches InMemoryCaseInstanceRepository which also accepts tenancyId without enforcing isolation. Domain bean methods gain tenancyId where they interact with SPI methods; callers that already have tenancyId in scope (DefaultEngineEvolutionApi, ResearchPipelineOrchestrator) pass it through.
**Trade-offs:** Every method on 3 domain beans and 4 call sites gains a parameter. Mechanically large but straightforward — no design complexity.
**Depends on:** D1 (SPI boundary for swappable implementations)
**Sources:** EventLogRepository.java (tenancyId convention), CaseInstanceRepository.java (tenancyId on every method), InMemoryCaseInstanceRepository.java (accepts but ignores tenancyId)
**Exploration:** quick
**Status:** captured
