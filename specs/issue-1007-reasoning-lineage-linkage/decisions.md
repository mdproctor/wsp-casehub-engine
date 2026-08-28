## D1: Ledger linkage approach (REVISED after decision review R1-02)

**Choice:** Store reasoning in `domainData` on the existing `WorkerDecisionEntry`
**Alternatives:**
- Separate `REASONING_RECORDED` `CaseLedgerEntry` with `causedByEntryId` — achieves Art.12 but adds a second entry, a new entity type, a new table, and a Flyway migration for no additional benefit. `domainData` provides the same Merkle-chain inclusion with zero schema changes.
- LedgerSupplement on WorkerDecisionEntry — avoids second entry but needs new supplement type in ledger repo (cross-repo). `ComplianceSupplement` does carry domain data (reviewer R1-03 corrected the original rationale), so the semantic objection was wrong. However `domainData` is simpler — no new type, no cross-repo change.
- Cross-reference pointer (memoryStoreEntryId on metadata) — reasoning stays outside the Merkle chain, defeating Art.12 purpose.
**Rationale:** `LedgerEntry.domainData` is a `Map<String, Object>` already included in `canonicalBytes()`. Setting `entry.domainData = Map.of("reasoning", truncatedText)` puts reasoning in the Merkle chain with zero schema changes. Causal linkage is implicit — reasoning IS on the decision entry, no indirection via `causedByEntryId`. No new entity, table, discriminator, or Flyway migration. Works with the existing observer via a field assignment.
**Trade-offs:** `domainData` is serialised to JSONB — large reasoning traces affect storage and `canonicalBytes()` computation. Mitigated by using the truncated version (4096 chars, matching CaseMemoryStore).
**Sources:** `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:199` (domainData field), `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:339-342` (canonicalBytes includes domainData), `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java` (existing observer), review finding R1-02
**Exploration:** quick → revised after light decision review
**Status:** revised

## D2: Observer architecture — WITHDRAWN

**Status:** withdrawn
**Reason:** D1 revision eliminates the second entry. The existing `WorkerDecisionEventCapture` observer writes one `WorkerDecisionEntry` with `domainData` populated when reasoning is present. No architectural change to the observer pattern.

## D3: Passing reasoning to the ledger observer

**Choice:** Add `reasoning` field (7th component) to `WorkerDecisionEvent` record
**Alternatives:**
- Separate `ReasoningTraceEvent` CDI event — more events, more wiring, no benefit over extending the existing record
**Rationale:** `WorkerDecisionEvent` is a record in `engine-common`. Adding a 7th component is backward-compatible — existing 5-arg and 6-arg constructors still work via additional backward-compat constructors. The handler already has `event.reasoning()` at the fire site (slot 140 threaded it through `WorkflowExecutionCompleted`).
**Trade-offs:** Marginally wider event payload when reasoning is null (most cases). Negligible.
**Sources:** `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java` (existing record), `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java` (fire site)
**Exploration:** quick
**Status:** captured

## D4: Dual-write semantics (raised by review R1-13)

**Choice:** Both stores serve complementary purposes — neither replaces the other
**Alternatives:**
- Deprecate CaseMemoryStore reasoning — loses queryability and semantic retrieval for subsequent workers
- Deprecate ledger reasoning — loses tamper-evidence and causal graph participation
**Rationale:** CaseMemoryStore (`worker-reasoning` domain) is the queryable semantic store — subsequent workers retrieve prior reasoning via `AgentMemoryRetriever`. The ledger (`domainData` on `WorkerDecisionEntry`) is the tamper-evident audit record for Art.12 compliance. Different consistency guarantees (fire-and-forget vs transactional) match their roles: memory store tolerates loss, ledger does not. Both receive the same truncated text (4096 chars from `AgentExperienceRecorder.truncateReasoning()`).
**Trade-offs:** Reasoning is written twice. Storage cost is marginal. If they diverge (memory store write fails, ledger succeeds), the ledger is authoritative for compliance; memory store absence means workers lose context but the audit chain is intact.
**Sources:** `runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java` (storeReasoning, truncateReasoning), review finding R1-13
**Exploration:** quick (surfaced by review, clear answer)
**Status:** captured
