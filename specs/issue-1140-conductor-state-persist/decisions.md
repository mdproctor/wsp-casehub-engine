# Decisions — Conductor State Persistence (#1140)

## D1: Persistence strategy — in-memory now, database-backed later

**Choice:** In-memory implementations as shipped defaults. No EventLog-replay durability. When durable implementations arrive, they follow the new platform direction: database-backed (likely CaseContext snapshot via #1166), EventLog for audit only.
**Alternatives:**
- EventLog-replay-backed impls (reconstruct from event history on startup) — the #1132 spec designed this, but #1150 (persistence layer coherence) found two silent data-loss bugs (#1151, #1152) in EventLog-replay-based context recovery. The platform is moving away from this pattern.
- JPA/database-backed impls now — premature; #1166 CaseContextRecoveryStrategy SPI will provide the right abstraction. All three stores are case-scoped and naturally fit inside CaseContext.
**Rationale:** All stores are low-mutation (human-timescale operator configuration or infrequent gate pipeline events). DB writes are trivially cheap. The EventLog writes already in #1132 (GATE_PENDING, DENY_PATTERN_ADDED, etc.) remain correct as audit trail, not durability mechanism. The SPI boundary is what matters now — implementations are swappable.
**Trade-offs:** State lost on restart until #1166 ships. Acceptable — the SPI boundary means durable impls slot in without domain bean changes.
**Sources:** #1150 (persistence layer coherence epic), #1166 (CaseContextRecoveryStrategy SPI), #1151/#1152 (EventLog-replay data-loss bugs), #1132 spec D16/D26 (EventLog persistence design, now superseded for durability)
**Exploration:** quick
**Status:** captured

## D2: SPI module placement — engine-common/spi

**Choice:** SPI interfaces in `common-core/spi` (`io.casehub.engine.common.spi`), alongside CaseInstanceRepository and EventLogRepository.
**Alternatives:**
- `api/spi/improvement` — where SummarizationProvider, EscalationProvider, CapabilityArea live. But those are consumer-facing SPIs that external projects implement. The conductor state repositories are engine-internal coordination stores.
**Rationale:** The `api/spi/improvement` package is for SPIs external consumers implement (pluggable summarization, escalation, research). The conductor repositories are internal engine concerns — no external consumer will provide a custom ConductorInboxRepository. `engine-common/spi` is where internal engine SPIs live.
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

**Choice:** Repository implementations implement Resettable, not domain beans. Domain beans become stateless logic.
**Alternatives:**
- Domain beans keep Resettable — but reset() is a storage concern, not business logic.
**Rationale:** After SPI extraction, domain beans inject repositories and contain only logic. The state lives in the repository implementations. reset()/clear() is a storage operation.
**Trade-offs:** ImprovementBudgetEnforcer retains Resettable for its remaining in-memory state (activeImprovements, dailyCounts, lastCompletionTime) — only dynamicDenyPatterns is extracted. ImprovementCoordinator loses Resettable entirely (all state extracted). ConductorInboxManager loses Resettable (all state split to ConductorInboxRepository + WatchPatternStore).
**Sources:** ConductorInboxManager.java (reset clears entries + watchPatterns), ImprovementCoordinator.java (reset clears blocks), ImprovementBudgetEnforcer.java (reset clears 4 fields, only dynamicDenyPatterns extracted)
**Exploration:** quick
**Status:** captured
