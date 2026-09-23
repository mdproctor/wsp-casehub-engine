# Decisions — CaseContextRecoveryStrategy SPI (#1166)

## D1: SPI has two methods — recover + onContextChanged

**Choice:** Two-method SPI: `recover(UUID caseId, String tenancyId)` for cache-miss recovery, `onContextChanged(UUID caseId, String tenancyId, CaseContext context)` as a blackboard propagation participant.
**Alternatives:**
- Single-method SPI (recover only), always write snapshot in repository — simpler but narrows the SPI to "load context" and loses the extension point for participating in the blackboard propagation cycle
**Rationale:** `onContextChanged` is architecturally part of the blackboard propagation, not just "persist the snapshot." Recovery strategies participate in the context-change cycle — the snapshot strategy persists, the event-log strategy is a no-op, and future strategies could stream, replicate, or maintain materialized views. The callback is called synchronously before the async `CaseContextChangedEvent` dispatch, within the caller's thread (same transaction boundary as the event log append).
**Trade-offs:** Every `CaseContextChangedEvent` dispatch site must also call `strategy.onContextChanged()` — ~15 sites. This is mitigated by the existing single-event-type pattern (all dispatch through `eventDispatcher.dispatch(new CaseContextChangedEvent(...))`).
**Sources:** `CaseContextChangedEvent.java:32`, `VertxEventDispatcher.java`, `WorkflowExecutionCompletedHandler.java:388`, issue #1166
**Exploration:** deep-analysis
**Status:** captured

## D2: Snapshot stored as JSONB column on CaseInstanceEntity

**Choice:** Add `context_snapshot` (JSONB, nullable) column to `CaseInstanceEntity`. Serialized from `CaseContext.asJsonNode()`, deserialized via `CaseContextImpl.fromLayerDocument()`.
**Alternatives:**
- Separate `CaseContextSnapshot` entity/table — clean separation but adds a new table, new SPI, new InMemory/JPA/Spring implementations for no meaningful gain
- Separate `CaseContextSnapshotStore` persistence SPI — extensible but the snapshot is keyed by caseId and always written alongside the CaseInstance, so a separate store adds indirection without value
**Rationale:** The CaseInstance entity is already updated in every `updateStateAndAppendEvent` transaction. Adding a JSONB column to that same entity means the snapshot is persisted with zero additional transactions. `findByUuid` returns the snapshot alongside the instance — no second query. InMemory implementation is trivial (CaseInstance already has the field in memory).
**Trade-offs:** Slightly larger row size on `case_instance` table. Mitigated by the context being bounded in size and JSONB being efficiently stored in PostgreSQL.
**Sources:** `CaseInstanceEntity.java:47`, `CaseInstance.java:27`, `InMemoryCaseInstanceRepository.java`
**Exploration:** deep-analysis
**Status:** captured

## D3: Delta-based replay as future optimization (not this issue)

**Choice:** Design the SPI to not preclude delta-based replay, but defer implementation. The Drools pattern (capture state changes as function + input parameters, store input arrays, replay by re-applying to functions) could be a third strategy in the future.
**Alternatives:**
- Implement delta-based replay now — premature; the infrastructure for structured function+params capture doesn't exist yet
**Rationale:** The current `onContextChanged` signature can evolve to carry deltas when needed (pre-release, breaking changes cost nothing). The `CaseContextChangedEvent` already carries `changedLayer` and could be extended with structured diffs. The two-method SPI is the right foundation — delta replay is an optimization on top.
**Trade-offs:** None for now. Future work item.
**Sources:** Drools persistence architecture (Drools command-based persistence), user design direction
**Exploration:** quick
**Status:** captured
