# Reasoning Lineage Linkage — Design Spec

**Issue:** casehubio/engine#1007
**Date:** 2026-08-28
**Depends on:** engine#950 (landed via slot 140)

## Problem

Worker reasoning traces are stored in `CaseMemoryStore` (queryable, semantic) and EventLog metadata (durable). Neither participates in the tamper-evident Merkle chain. For EU AI Act Art.12 compliance, reasoning behind a decision must be part of the causal audit graph — not a side-channel correlated after the fact.

## Solution

Populate `domainData` on the existing `WorkerDecisionEntry` with the reasoning text. `LedgerEntry.domainData` is a `Map<String, Object>` already included in `canonicalBytes()` — reasoning becomes part of the Merkle-chained digest with no schema changes.

## Design

### Change 1: Add `reasoning` to `WorkerDecisionEvent`

`WorkerDecisionEvent` (record in `engine-common`) gains a 7th component:

```java
public record WorkerDecisionEvent(
    UUID caseId,
    String tenancyId,
    String workerId,
    String capabilityTag,
    String traceId,
    SelectionContext selectionContext,
    String reasoning) {           // NEW — nullable

  // Existing backward-compat constructors preserved
  public WorkerDecisionEvent(
      UUID caseId, String tenancyId, String workerId,
      String capabilityTag, String traceId) {
    this(caseId, tenancyId, workerId, capabilityTag, traceId, null, null);
  }

  public WorkerDecisionEvent(
      UUID caseId, String tenancyId, String workerId,
      String capabilityTag, String traceId,
      SelectionContext selectionContext) {
    this(caseId, tenancyId, workerId, capabilityTag, traceId,
         selectionContext, null);
  }
}
```

### Change 2: Thread reasoning through the fire site

`WorkflowExecutionCompletedHandler` already has `event.reasoning()` (from slot 140's `WorkflowExecutionCompleted`). Update the `workerDecisionEvents.fireAsync()` call to pass reasoning:

```java
workerDecisionEvents.fireAsync(
    new WorkerDecisionEvent(
        caseInstance.getUuid(),
        caseInstance.tenancyId,
        worker.name(),
        extractCapabilityTag(caseInstance, worker, bindingName),
        traceId,
        null,              // selectionContext
        event.reasoning())) // reasoning — NEW
```

### Change 3: Populate `domainData` in `WorkerDecisionEventCapture`

In `onWorkerDecisionEvent()`, after constructing the `WorkerDecisionEntry` and before `ledgerRepo.save()`:

```java
if (event.reasoning() != null && !event.reasoning().isBlank()) {
    entry.domainData = Map.of("reasoning", event.reasoning());
}
```

The reasoning text on `WorkflowExecutionCompleted` is the raw text from `WorkerResult.reasoning()`. `AgentExperienceRecorder.truncateReasoning()` truncates at 4096 chars for CaseMemoryStore writes. The ledger observer must apply the same truncation before setting `domainData`:

```java
if (event.reasoning() != null && !event.reasoning().isBlank()) {
    String truncated = truncateForLedger(event.reasoning());
    entry.domainData = Map.of("reasoning", truncated);
}
```

`truncateForLedger()` reuses the same 4096-char head/tail strategy from `AgentExperienceRecorder`. Keeping both stores at the same truncation length ensures the Merkle-chained text matches the queryable text — no divergence in what "the reasoning" is.

`domainData` is included in `canonicalBytes()` via:
```java
if (domainData != null && !domainData.isEmpty()) {
    canonical.append("|").append(canonicalDomainData(domainData));
}
```

This makes reasoning part of the RFC 9162 leaf hash. The reasoning is now tamper-evident and verifiable through the Merkle Mountain Range.

### No change to `WorkerDecisionEntry`

`domainData` is inherited from `LedgerEntry` — no new columns, no new table, no Flyway migration. The existing JSONB column stores the reasoning map alongside any other domain data. `domainContentBytes()` (which covers typed fields like `workerId`, `capabilityTag`) is unaffected — `domainData` has its own path in `canonicalBytes()`.

### Dual-write semantics

Reasoning is written to two stores with complementary roles:

| Store | Purpose | Consistency | Truncation |
|-------|---------|-------------|------------|
| `CaseMemoryStore` | Queryable semantic store for worker retrieval | Fire-and-forget (virtual thread) | 4096 chars (in `storeReasoning()`) |
| `WorkerDecisionEntry.domainData` | Tamper-evident audit for Art.12 compliance | Transactional (`@Transactional` observer) | 4096 chars (in `truncateForLedger()`) |

If CaseMemoryStore write fails, workers lose context but the audit chain is intact. If the ledger observer fails (which logs and swallows), reasoning is still in EventLog metadata and CaseMemoryStore. The ledger is authoritative for compliance.

### Querying reasoning from the ledger

Existing query methods work without changes:
- `findBySubjectId(caseId, tenancyId)` — returns all entries for a case, including those with `domainData.reasoning`
- `findCausedBy(entryId, tenancyId)` — traverses the causal chain (reasoning entries don't need this since they're on the decision entry itself)
- REST: `DecisionLineage` (parent#363) already assembles from ledger entries — reasoning appears automatically as `domainData` on decision entries

## Files Changed

| File | Change |
|------|--------|
| `common/.../event/WorkerDecisionEvent.java` | Add `reasoning` 7th component + backward-compat constructors |
| `runtime/.../handler/WorkflowExecutionCompletedHandler.java` | Pass `event.reasoning()` to `WorkerDecisionEvent` fire site |
| `ledger/.../service/WorkerDecisionEventCapture.java` | Set `entry.domainData` when reasoning present |

Three files, ~15 lines of production code.

## Testing

1. **Unit test:** `WorkerDecisionEventCaptureTest` — verify `domainData` is populated when reasoning is non-null, absent when null
2. **Unit test:** Verify `canonicalBytes()` includes reasoning via `domainData` path — tamper-evidence assertion
3. **Unit test:** Backward compatibility — existing 5-arg and 6-arg `WorkerDecisionEvent` constructors still work, reasoning defaults to null

## Not in Scope

- New ledger entry types or Flyway migrations
- Changes to `CaseMemoryStore` reasoning storage (#950 handles this)
- Changes to `WorkerContextProvider` enrichment (#950 handles this)
- Full-text reasoning (untruncated) in the ledger — deferred, 4096 chars matches memory store
- `routingRationale` Merkle inclusion — deliberately excluded per existing design (informational, not compliance-required)

## References

- `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:199` — `domainData` field
- `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:339-342` — `canonicalBytes()` includes `domainData`
- `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java` — existing observer
- `ledger/src/main/java/io/casehub/ledger/model/WorkerDecisionEntry.java:86-88` — `routingRationale` excluded from Merkle (precedent)
- `runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java` — truncation at 4096
- `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java` — existing 6-component record
- Decision review R1-02 — surfaced `domainData` as the better approach
- Decision review R1-13 — surfaced dual-write semantics question
