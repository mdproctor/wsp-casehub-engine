## D1: Ledger linkage approach

**Choice:** Separate `REASONING_RECORDED` `CaseLedgerEntry` with `causedByEntryId` pointing to the `WorkerDecisionEntry`
**Alternatives:**
- LedgerSupplement on WorkerDecisionEntry — avoids second entry but needs new supplement type in ledger repo (cross-repo), and supplements are for cross-cutting concerns not domain data
- Cross-reference pointer (memoryStoreEntryId on metadata) — lighter but reasoning stays outside the Merkle chain, defeating Art.12 purpose
**Rationale:** Follows existing `CaseLedgerEntry` pattern. `causedByEntryId` is already queryable via `LedgerEntryRepository.findCausedBy()`. Two entries per worker completion is consistent with existing patterns (CaseLifecycleEvent + WorkerDecisionEvent are already separate). No cross-repo changes needed.
**Trade-offs:** Additional ledger entry per worker completion when reasoning is present. Marginal storage cost.
**Sources:** `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java` (causedByEntryId field, canonicalBytes includes it), `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java` (existing observer pattern), `ledger/src/main/java/io/casehub/ledger/model/CaseLedgerEntry.java` (entry type to extend)
**Exploration:** quick
**Status:** captured

## D2: Observer architecture for writing the reasoning entry

**Choice:** Single observer — extend `WorkerDecisionEventCapture` to write both entries in the same `@Transactional` method
**Alternatives:**
- Separate observer with CDI event chaining — more decoupled but adds ordering complexity between async observers and requires a new intermediate event type
**Rationale:** The reasoning entry is an extension of the same decision audit. Co-locating avoids ordering concerns. Same transaction guarantees atomicity — both entries are committed together or neither is. The `WorkerDecisionEntry.id` is immediately available after `ledgerRepo.save()`.
**Trade-offs:** Slightly larger single observer. If reasoning storage fails, it could roll back the WorkerDecisionEntry too — but this is desirable (atomic audit).
**Sources:** `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java:62-115` (existing @Transactional observer)
**Exploration:** quick
**Status:** captured

## D3: Passing reasoning to the ledger observer

**Choice:** Add `reasoning` field (7th component) to `WorkerDecisionEvent` record
**Depends on:** D2 (single observer needs reasoning in the same event)
**Alternatives:**
- Separate `ReasoningTraceEvent` CDI event — more events, more wiring, no benefit over extending the existing record
**Rationale:** `WorkerDecisionEvent` is a record in `engine-common`. Adding a 7th component is backward-compatible — existing 5-arg and 6-arg constructors still work via additional backward-compat constructors. The handler already has `event.reasoning()` at the fire site (slot 140 threaded it through `WorkflowExecutionCompleted`).
**Trade-offs:** Marginally wider event payload when reasoning is null (most cases). Negligible.
**Sources:** `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java` (existing record), `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java:291-308` (fire site)
**Exploration:** quick
**Status:** captured
