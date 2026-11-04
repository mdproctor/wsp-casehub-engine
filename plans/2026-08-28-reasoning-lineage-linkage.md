# Reasoning Lineage Linkage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1007 — feat: link reasoning traces to lineage DAG via causedByEntryId
**Issue group:** #1007

**Goal:** Make worker reasoning traces part of the tamper-evident Merkle-chained ledger by populating `domainData` on the existing `WorkerDecisionEntry`.

**Architecture:** `WorkerDecisionEvent` gains a `reasoning` field. The fire site in `WorkflowExecutionCompletedHandler` threads `event.reasoning()` through. `WorkerDecisionEventCapture` sets `entry.domainData = Map.of("reasoning", truncatedText)` before save. No new entity types, tables, or migrations.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, CDI async events

## Global Constraints

- No new Flyway migrations
- No new JPA entity types
- `domainData` truncation at 4096 chars (matching `AgentExperienceRecorder`)
- Backward-compatible constructors on `WorkerDecisionEvent`

---

## Batch 1: Reasoning in the Merkle chain

### Task 1: Add `reasoning` to `WorkerDecisionEvent` and populate `domainData` in the observer

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java:304-311`
- Modify: `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java:74-110`
- Test: `ledger/src/test/java/io/casehub/ledger/WorkerDecisionEventCaptureTest.java`

**Interfaces:**
- Consumes: `WorkflowExecutionCompleted.reasoning()` (String, nullable — from slot 140)
- Produces: `WorkerDecisionEvent.reasoning()` (String, nullable — new 7th component), `WorkerDecisionEntry.domainData` populated with `{"reasoning": "..."}` when reasoning present

- [ ] **Step 1: Write failing test — reasoning populates domainData**

Add to `WorkerDecisionEventCaptureTest.java`:

```java
@Test
void reasoning_populatesDomainData_whenPresent() {
    final UUID caseId = UUID.randomUUID();

    workerDecisionEvents.fireAsync(
        new WorkerDecisionEvent(
            caseId, "test-tenant", "analyst-v1", "analysis", "trace-r1",
            null, "I chose approach A because the risk indicators were above threshold"));

    Awaitility.await()
        .atMost(5, TimeUnit.SECONDS)
        .untilAsserted(
            () -> {
                final List<WorkerDecisionEntry> entries =
                    repository.findWorkerDecisionsByCaseId(caseId);
                assertThat(entries).hasSize(1);
                final WorkerDecisionEntry entry = entries.get(0);
                assertThat(entry.domainData).isNotNull();
                assertThat(entry.domainData).containsKey("reasoning");
                assertThat(entry.domainData.get("reasoning"))
                    .isEqualTo("I chose approach A because the risk indicators were above threshold");
            });
}

@Test
void reasoning_absent_domainDataNull() {
    final UUID caseId = UUID.randomUUID();

    workerDecisionEvents.fireAsync(
        new WorkerDecisionEvent(caseId, "test-tenant", "worker-no-reasoning", "cap-x", "trace-nr"));

    Awaitility.await()
        .atMost(5, TimeUnit.SECONDS)
        .untilAsserted(
            () -> {
                final List<WorkerDecisionEntry> entries =
                    repository.findWorkerDecisionsByCaseId(caseId);
                assertThat(entries).hasSize(1);
                assertThat(entries.get(0).domainData).isNull();
            });
}

@Test
void reasoning_truncated_whenExceedsLimit() {
    final UUID caseId = UUID.randomUUID();
    final String longReasoning = "x".repeat(5000);

    workerDecisionEvents.fireAsync(
        new WorkerDecisionEvent(
            caseId, "test-tenant", "verbose-worker", "cap-v", "trace-trunc",
            null, longReasoning));

    Awaitility.await()
        .atMost(5, TimeUnit.SECONDS)
        .untilAsserted(
            () -> {
                final List<WorkerDecisionEntry> entries =
                    repository.findWorkerDecisionsByCaseId(caseId);
                assertThat(entries).hasSize(1);
                final String stored = (String) entries.get(0).domainData.get("reasoning");
                assertThat(stored).isNotNull();
                assertThat(stored.length()).isLessThanOrEqualTo(4096);
                assertThat(stored).contains("[...truncated...]");
            });
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl ledger -Dtest="WorkerDecisionEventCaptureTest#reasoning_populatesDomainData_whenPresent+reasoning_absent_domainDataNull+reasoning_truncated_whenExceedsLimit" -DfailIfNoTests=false
```
Expected: compilation failure — `WorkerDecisionEvent` has no 7-arg constructor yet.

- [ ] **Step 3: Add `reasoning` to `WorkerDecisionEvent`**

In `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java`, replace the record declaration:

```java
public record WorkerDecisionEvent(
    UUID caseId,
    String tenancyId,
    String workerId,
    String capabilityTag,
    String traceId,
    SelectionContext selectionContext,
    String reasoning) {

  public WorkerDecisionEvent(
      UUID caseId, String tenancyId, String workerId, String capabilityTag, String traceId) {
    this(caseId, tenancyId, workerId, capabilityTag, traceId, null, null);
  }

  public WorkerDecisionEvent(
      UUID caseId,
      String tenancyId,
      String workerId,
      String capabilityTag,
      String traceId,
      SelectionContext selectionContext) {
    this(caseId, tenancyId, workerId, capabilityTag, traceId, selectionContext, null);
  }
}
```

- [ ] **Step 4: Populate `domainData` in `WorkerDecisionEventCapture`**

In `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java`, add truncation constants and a private method after the LOG field:

```java
private static final int MAX_REASONING_LENGTH = 4096;
private static final String TRUNCATION_MARKER = "\n[...truncated...]\n";
```

Add the truncation method after `onWorkerDecisionEvent()`:

```java
private static String truncateReasoning(String reasoning) {
    if (reasoning.length() <= MAX_REASONING_LENGTH) {
        return reasoning;
    }
    int headLen = MAX_REASONING_LENGTH / 3;
    if (headLen > 0 && Character.isHighSurrogate(reasoning.charAt(headLen - 1))) {
        headLen--;
    }
    int tailLen = MAX_REASONING_LENGTH - headLen - TRUNCATION_MARKER.length();
    int tailStart = reasoning.length() - tailLen;
    if (tailStart < reasoning.length() && Character.isLowSurrogate(reasoning.charAt(tailStart))) {
        tailStart++;
    }
    return reasoning.substring(0, headLen) + TRUNCATION_MARKER + reasoning.substring(tailStart);
}
```

In `onWorkerDecisionEvent()`, add before `ledgerRepo.save(entry, event.tenancyId())` (before line 110):

```java
if (event.reasoning() != null && !event.reasoning().isBlank()) {
    entry.domainData = java.util.Map.of("reasoning", truncateReasoning(event.reasoning()));
}
```

- [ ] **Step 5: Thread reasoning through the fire site**

In `WorkflowExecutionCompletedHandler.java`, update the `workerDecisionEvents.fireAsync()` call (lines 306-311) to use the 7-arg constructor:

```java
workerDecisionEvents
    .fireAsync(
        new WorkerDecisionEvent(
            caseInstance.getUuid(),
            caseInstance.tenancyId,
            worker.name(),
            extractCapabilityTag(caseInstance, worker, bindingName),
            traceId,
            null,
            event.reasoning()))
```

- [ ] **Step 6: Install dependencies and run tests**

```bash
/opt/homebrew/bin/mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl ledger -Dtest="WorkerDecisionEventCaptureTest" -DfailIfNoTests=false
```

Expected: all tests pass (existing 4 + new 3).

- [ ] **Step 7: Verify Merkle chain inclusion**

Add a unit test confirming `canonicalBytes()` includes reasoning via `domainData`:

```java
@Test
void reasoning_inDomainData_includedInCanonicalBytes() {
    final WorkerDecisionEntry entry = new WorkerDecisionEntry();
    entry.caseId = UUID.randomUUID();
    entry.subjectId = entry.caseId;
    entry.workerId = "test-worker";
    entry.capabilityTag = "test-cap";
    entry.tenancyId = "test-tenant";
    entry.sequenceNumber = 1;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = "test-worker";
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "WORKER";
    entry.occurredAt = java.time.Instant.now();

    byte[] withoutReasoning = entry.canonicalBytes();

    entry.domainData = java.util.Map.of("reasoning", "I chose A because threshold exceeded");

    byte[] withReasoning = entry.canonicalBytes();

    assertThat(withReasoning).isNotEqualTo(withoutReasoning);
    assertThat(new String(withReasoning, java.nio.charset.StandardCharsets.UTF_8))
        .contains("reasoning");
}
```

Run:
```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl ledger -Dtest="WorkerDecisionEventCaptureTest" -DfailIfNoTests=false
```

Expected: all 8 tests pass.

- [ ] **Step 8: Run full module test suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl ledger -DfailIfNoTests=false
```

Expected: all ledger tests pass. Also verify no regressions in common and runtime:

```bash
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl common -DfailIfNoTests=false
TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -DfailIfNoTests=false
```

- [ ] **Step 9: Commit**

```bash
git add common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java \
       runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java \
       ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java \
       ledger/src/test/java/io/casehub/ledger/WorkerDecisionEventCaptureTest.java
git commit -m "feat(#1007): link reasoning traces to Merkle chain via domainData on WorkerDecisionEntry

WorkerDecisionEvent gains reasoning (7th component). The fire site in
WorkflowExecutionCompletedHandler threads event.reasoning() through.
WorkerDecisionEventCapture populates entry.domainData with truncated
reasoning text (4096 chars). domainData is included in canonicalBytes()
— reasoning is now tamper-evident in the RFC 9162 leaf hash.

Closes #1007"
```

## References

- [2026-08-28-reasoning-lineage-linkage-design.md] — design spec
- `ledger/api/src/main/java/io/casehub/ledger/api/model/LedgerEntry.java:199,339-342` — domainData field and canonicalBytes inclusion
- `common/src/main/java/io/casehub/engine/common/spi/event/WorkerDecisionEvent.java` — existing record
- `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java:304-311` — fire site
- `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java` — observer
- `runtime/src/main/java/io/casehub/engine/internal/memory/AgentExperienceRecorder.java:178-191` — truncation strategy
- GitHub #1007
