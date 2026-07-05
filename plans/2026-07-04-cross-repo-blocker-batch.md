# Cross-Repo Blocker Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Resolve six cross-repo blockers (#651, #650, #583, #644, #640, #626) — enriching engine-api boundary types, adding blocking repository SPIs, and propagating changes to 8 consumer repos.

**Architecture:** Changes flow from engine-api outward. Foundation types first (AgentRoutingContext, AgentAssignment, CaseHubEventType), then repository rename via IntelliJ across 26 repos, then CaseEventRecorder implementation, then consumer migration, then CI.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven multi-module, IntelliJ MCP for semantic refactoring

## Global Constraints

- All engine modules build with `mvn install -DskipTests -q` before running module-specific tests
- Always include `TESTCONTAINERS_RYUK_DISABLED=true` for test runs
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all renames, moves, and find-references — never bash grep/find for semantic code operations
- Consumer repos must be verified on `main` before any change; stop if not
- **quarkmind is currently on `issue-191-milestone-trust-scoring`** — skip quarkmind migration (Task 9C) until it returns to main
- Every commit references an issue: `Refs #N` or `Closes #N`

---

### Task 1: AgentRoutingContext — add tenancyId (#651)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/AgentRoutingContext.java`
- Modify: `api/src/test/java/io/casehub/api/spi/AgentRoutingStrategyContractTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/LeastLoadedAgentStrategyTest.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java`
- Modify: `engine-ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java`

**Interfaces:**
- Produces: `AgentRoutingContext(UUID caseId, String capabilityName, JsonNode caseContext, String tenancyId)` — all downstream tasks use this 4-field constructor

- [ ] **Step 1: Update the record** — add `String tenancyId` as 4th field in `AgentRoutingContext.java`. Update the Javadoc to document the new field.

```java
public record AgentRoutingContext(UUID caseId, String capabilityName, JsonNode caseContext, String tenancyId) {}
```

- [ ] **Step 2: Update contract test** — `AgentRoutingStrategyContractTest`. All `new AgentRoutingContext(caseId, "capability", context)` become `new AgentRoutingContext(caseId, "capability", context, "test-tenant")`. Add a test that verifies `tenancyId()` accessor.

- [ ] **Step 3: Update production construction sites** — two sites:

In `CaseContextChangedEventHandler.publishWorkerSchedule()` (~line 370):
```java
final AgentRoutingContext ctx =
    new AgentRoutingContext(
        caseInstance.getUuid(),
        capability.name(),
        caseInstance.getCaseContext().panel(ContextPanel.WORKING).asJsonNode(),
        caseInstance.tenancyId);
```

In `DefaultWorkOrchestrator.doSubmit()` (~line 153):
```java
final AgentRoutingContext ctx =
    new AgentRoutingContext(
        instance.getUuid(),
        capability.name(),
        instance.getCaseContext().panel(ContextPanel.WORKING).asJsonNode(),
        instance.tenancyId);
```

- [ ] **Step 4: Update all test construction sites** — `LeastLoadedAgentStrategyTest`, `TrustWeightedAgentStrategyTest`, `SemanticAgentRoutingStrategyTest`. Every `new AgentRoutingContext(...)` gets a 4th arg `"test-tenant"`.

- [ ] **Step 5: Build and test**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl ledger -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-ai -q
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat(#651): add tenancyId to AgentRoutingContext — required by CBR routing

Refs #651"
```

---

### Task 2: AgentAssignment — mandatory rationale (#650)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/routing/AgentAssignment.java`
- Modify: `api/src/test/java/io/casehub/api/spi/AgentAssignmentTest.java`
- Modify: `api/src/test/java/io/casehub/api/spi/AgentRoutingStrategyContractTest.java`
- Modify: `ledger/src/main/java/io/casehub/ledger/routing/TrustCandidateClassifier.java`
- Modify: `ledger/src/test/java/io/casehub/ledger/routing/TrustWeightedAgentStrategyTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/LeastLoadedAgentStrategy.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/LeastLoadedAgentStrategyTest.java`
- Modify: `engine-ai/src/test/java/io/casehub/engine/ai/routing/SemanticAgentRoutingStrategyTest.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/orchestration/DefaultWorkOrchestrator.java`

**Interfaces:**
- Consumes: `AgentRoutingContext` (4-field, from Task 1)
- Produces: `AgentAssignment.Assigned(String workerId, String rationale)`, `AgentAssignment.Unresolvable(String rationale)`, `AgentAssignment.EscalateToOversight(String capabilityName, EscalationReason reason, String rationale)` — 3-arg factory methods `assign(workerId, rationale)`, `unresolvable(rationale)`, `escalate(cap, reason, rationale)`

- [ ] **Step 1: Update AgentAssignment sealed interface** — add `String rationale` to all 3 records, replace factory methods (remove old, add new with rationale):

```java
public sealed interface AgentAssignment
    permits AgentAssignment.Assigned, AgentAssignment.Unresolvable, AgentAssignment.EscalateToOversight {

  record Assigned(String workerId, String rationale) implements AgentAssignment {}
  record Unresolvable(String rationale) implements AgentAssignment {}
  record EscalateToOversight(String capabilityName, EscalationReason reason, String rationale) implements AgentAssignment {}

  static AgentAssignment assign(final String workerId, final String rationale) {
    return new Assigned(workerId, rationale);
  }

  static AgentAssignment unresolvable(final String rationale) {
    return new Unresolvable(rationale);
  }

  static AgentAssignment escalate(final String capabilityName, final EscalationReason reason, final String rationale) {
    return new EscalateToOversight(capabilityName, reason, rationale);
  }
}
```

- [ ] **Step 2: Update AgentAssignmentTest** — all factory method calls and record constructors need rationale. Verify `rationale()` accessor on each variant.

- [ ] **Step 3: Update TrustCandidateClassifier.ScoredCandidate** — add `String rationale` field:

```java
public record ScoredCandidate(ClassifiedCandidate classified, double finalScore, String rationale) {}
```

- [ ] **Step 4: Update TrustCandidateClassifier.decide()** — use `best.rationale()` for Assigned, generate rationale for Unresolvable and EscalateToOversight:

```java
if (best != null && best.finalScore() > 0.0) {
  return AgentAssignment.assign(best.classified().candidate().workerId(), best.rationale());
}

final boolean anyBorderline = classified.stream().anyMatch(c -> c.phase() == Phase.BORDERLINE);

return anyBorderline
    ? AgentAssignment.escalate(capabilityName, EscalationReason.BORDERLINE_STALEMATE,
        "all candidates borderline for capability '%s' — oversight required".formatted(capabilityName))
    : AgentAssignment.unresolvable(
        "all candidates excluded for capability '%s'".formatted(capabilityName));
```

- [ ] **Step 5: Update LeastLoadedAgentStrategy** — generate rationale when constructing the assignment. This strategy returns `AgentAssignment` directly (does not use `TrustCandidateClassifier`). Find the `assign()` call and add rationale. For empty candidates, add rationale to `unresolvable()`.

Rationale format for 2+ candidates: `"selected %s: load %d (vs next: %s load %d)".formatted(best.workerId(), bestLoad, second.workerId(), secondLoad)`

Rationale format for 1 candidate: `"selected %s: load %d (sole candidate)".formatted(best.workerId(), bestLoad)`

Rationale for unresolvable (empty candidates): `"no candidates available"`

- [ ] **Step 6: Update TrustWeightedAgentStrategy** — update where `ScoredCandidate` is constructed. Find all `new ScoredCandidate(cc, score)` and add rationale as 3rd arg.

For QUALIFIED phase: `"selected %s: trust %.2f, blended %.2f (threshold %.2f)".formatted(workerId, trustScore, blendedScore, threshold)`

For BOOTSTRAP phase: `"selected %s: availability %.2f (bootstrap)".formatted(workerId, workloadScore)`

Also handle the bootstrap-only guard: `AgentAssignment.escalate(...)` call needs rationale `"bootstrap only — no qualified agents for capability '%s'".formatted(capabilityName)`.

And the empty-candidates guard: `AgentAssignment.unresolvable(...)` needs rationale `"no candidates available"`.

- [ ] **Step 7: Update SemanticAgentRoutingStrategy** — same pattern as TrustWeightedAgentStrategy but with semantic scores.

For QUALIFIED phase: `"selected %s: semantic %.2f, trust %.2f, blended %.2f".formatted(workerId, semanticScore, trustScore, blendedScore)`

For BOOTSTRAP phase: `"selected %s: availability %.2f (bootstrap)".formatted(workerId, workloadScore)`

Same escalation and unresolvable rationale strings as TrustWeightedAgentStrategy.

- [ ] **Step 8: Update callers** — `CaseContextChangedEventHandler` and `DefaultWorkOrchestrator` pattern-match on `AgentAssignment` variants. Update:

In `CaseContextChangedEventHandler.publishWorkerSchedule()`: the `case AgentAssignment.Assigned a ->` branch logs `a.rationale()`:
```java
case AgentAssignment.Assigned a -> {
  LOG.infof("Agent selected: worker='%s' capability='%s' binding='%s' rationale='%s'",
      a.workerId(), capability.name(), binding.getName(), a.rationale());
  yield scheduleWorker(..., a.workerId(), signalId);
}
```

In `DefaultWorkOrchestrator.doSubmit()`: the `case AgentAssignment.Unresolvable()` pattern needs updating to `case AgentAssignment.Unresolvable u` (to accept the rationale field). Same for `EscalateToOversight`.

- [ ] **Step 9: Update AgentRoutingStrategyContractTest** — all `AgentAssignment.assign()`, `.unresolvable()`, `.escalate()` calls need rationale args. Update inner class strategy implementations.

- [ ] **Step 10: Update strategy test classes** — `TrustWeightedAgentStrategyTest`, `LeastLoadedAgentStrategyTest`, `SemanticAgentRoutingStrategyTest`. All assertions on `AgentAssignment` need to verify rationale. All `ScoredCandidate` constructors need 3rd arg.

- [ ] **Step 11: Build and test**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl ledger -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl engine-ai -q
```

- [ ] **Step 12: Commit**

```bash
git add -A && git commit -m "feat(#650): mandatory rationale on AgentAssignment — routing transparency

Every routing strategy must now explain its decision. Rationale is mandatory
on all three AgentAssignment variants (Assigned, Unresolvable, EscalateToOversight).
TrustCandidateClassifier.ScoredCandidate carries rationale through the scoring pipeline.

Refs #650"
```

---

### Task 3: CaseHubEventType + EventStreamType additions (#626 types)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java`
- Modify: `api/src/main/java/io/casehub/api/model/event/EventStreamType.java`

**Interfaces:**
- Produces: 7 new `CaseHubEventType` enum values, 1 new `EventStreamType` value — used by Task 5 (CaseEventRecorder)

- [ ] **Step 1: Add orchestration event types** to `CaseHubEventType.java` — add after the ACTION_GATE block:

```java
  ORCHESTRATION_STARTED,
  ORCHESTRATION_COMPLETED,
  AGENT_ROUTED,
  AGENT_DISPATCHED,
  AGENT_COMPLETED,
  AGENT_FAILED,
  ORCHESTRATION_ESCALATED,
```

- [ ] **Step 2: Add ORCHESTRATION stream type** to `EventStreamType.java`:

```java
public enum EventStreamType {
  CASE,
  WORKER,
  TIMER,
  SYSTEM,
  ORCHESTRATION
}
```

- [ ] **Step 3: Build**

```bash
mvn install -DskipTests -q
```

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat(#626): add orchestration event types and ORCHESTRATION stream type

7 new CaseHubEventType values (ORCHESTRATION_STARTED/COMPLETED, AGENT_ROUTED/
DISPATCHED/COMPLETED/FAILED, ORCHESTRATION_ESCALATED) and ORCHESTRATION stream
type. These operate at the routing/orchestration layer, distinct from existing
WORKER_* events at the execution layer.

Refs #626"
```

---

### Task 4: CaseEventRecorder SPI (#626 interfaces + no-ops)

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/CaseEventRequest.java`
- Create: `api/src/main/java/io/casehub/api/spi/CaseEventRecorder.java`
- Create: `api/src/main/java/io/casehub/api/spi/ReactiveCaseEventRecorder.java`
- Create: `api/src/main/java/io/casehub/api/spi/NoOpCaseEventRecorder.java`
- Create: `api/src/main/java/io/casehub/api/spi/NoOpReactiveCaseEventRecorder.java`
- Create: `api/src/test/java/io/casehub/api/spi/CaseEventRecorderContractTest.java`

**Interfaces:**
- Consumes: `CaseHubEventType`, `EventStreamType` (from Task 3)
- Produces: `CaseEventRecorder`, `ReactiveCaseEventRecorder`, `CaseEventRequest` — used by Task 8 (DefaultCaseEventRecorder) and blocks#12

- [ ] **Step 1: Create CaseEventRequest record:**

```java
package io.casehub.api.spi;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import java.util.UUID;

public record CaseEventRequest(
    UUID caseId,
    CaseHubEventType type,
    EventStreamType stream,
    String workerId,
    String tenancyId,
    JsonNode payload,
    JsonNode metadata) {}
```

- [ ] **Step 2: Create blocking CaseEventRecorder:**

```java
package io.casehub.api.spi;

public interface CaseEventRecorder {
  void record(CaseEventRequest request);
  Long recordAndReturnId(CaseEventRequest request);
}
```

- [ ] **Step 3: Create ReactiveCaseEventRecorder:**

```java
package io.casehub.api.spi;

import io.smallrye.mutiny.Uni;

public interface ReactiveCaseEventRecorder {
  Uni<Void> record(CaseEventRequest request);
  Uni<Long> recordAndReturnId(CaseEventRequest request);
}
```

- [ ] **Step 4: Create NoOpCaseEventRecorder** (`@DefaultBean @ApplicationScoped`):

```java
package io.casehub.api.spi;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpCaseEventRecorder implements CaseEventRecorder {
  @Override
  public void record(final CaseEventRequest request) {}

  @Override
  public Long recordAndReturnId(final CaseEventRequest request) {
    return 0L;
  }
}
```

- [ ] **Step 5: Create NoOpReactiveCaseEventRecorder** (`@DefaultBean @ApplicationScoped`):

```java
package io.casehub.api.spi;

import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpReactiveCaseEventRecorder implements ReactiveCaseEventRecorder {
  @Override
  public Uni<Void> record(final CaseEventRequest request) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Long> recordAndReturnId(final CaseEventRequest request) {
    return Uni.createFrom().item(0L);
  }
}
```

- [ ] **Step 6: Create contract test** — verifies no-ops return without error and that the request record carries all fields:

```java
package io.casehub.api.spi;

import static org.junit.jupiter.api.Assertions.*;
import com.fasterxml.jackson.databind.node.NullNode;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class CaseEventRecorderContractTest {

  @Test
  void caseEventRequest_exposesAllFields() {
    final UUID caseId = UUID.randomUUID();
    final var request = new CaseEventRequest(
        caseId, CaseHubEventType.AGENT_ROUTED, EventStreamType.ORCHESTRATION,
        "worker-1", "tenant-1", NullNode.instance, NullNode.instance);

    assertEquals(caseId, request.caseId());
    assertEquals(CaseHubEventType.AGENT_ROUTED, request.type());
    assertEquals(EventStreamType.ORCHESTRATION, request.stream());
    assertEquals("worker-1", request.workerId());
    assertEquals("tenant-1", request.tenancyId());
  }

  @Test
  void noOpRecorder_recordDoesNotThrow() {
    final var recorder = new NoOpCaseEventRecorder();
    final var request = new CaseEventRequest(
        UUID.randomUUID(), CaseHubEventType.AGENT_ROUTED, EventStreamType.ORCHESTRATION,
        "w", "t", NullNode.instance, NullNode.instance);
    assertDoesNotThrow(() -> recorder.record(request));
    assertEquals(0L, recorder.recordAndReturnId(request));
  }

  @Test
  void noOpReactiveRecorder_recordDoesNotThrow() {
    final var recorder = new NoOpReactiveCaseEventRecorder();
    final var request = new CaseEventRequest(
        UUID.randomUUID(), CaseHubEventType.AGENT_ROUTED, EventStreamType.ORCHESTRATION,
        "w", "t", NullNode.instance, NullNode.instance);
    assertDoesNotThrow(() -> recorder.record(request).await().indefinitely());
    assertEquals(0L, recorder.recordAndReturnId(request).await().indefinitely());
  }
}
```

- [ ] **Step 7: Build and test**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl api -q
```

- [ ] **Step 8: Commit**

```bash
git add -A && git commit -m "feat(#626): CaseEventRecorder SPI + no-op defaults

CaseEventRequest record, blocking CaseEventRecorder, reactive ReactiveCaseEventRecorder,
and @DefaultBean no-ops in api/spi/. Follows PlanItemStore dual-stack convention.

Refs #626"
```

---

### Task 5: Repository rename — reactive prefix (#640 Part 1)

**Precondition:** IntelliJ workspace with all 26 repos must be open and indexed.

**Files:** 4 interface renames across engine + all consumer repos (IntelliJ handles all sites)

**Interfaces:**
- Produces: `ReactiveCaseInstanceRepository`, `ReactiveCrossTenantCaseInstanceRepository`, `ReactiveEventLogRepository`, `ReactiveCrossTenantEventLogRepository` — renamed from unqualified names

This task uses IntelliJ semantic refactoring exclusively. **Do not use bash find/replace.**

- [ ] **Step 1: Verify IntelliJ workspace is ready**

```
Call ide_index_status with project_path=/Users/mdproctor/claude/casehub/engine
```

Verify `isDumbMode: false`.

- [ ] **Step 2: Rename CaseInstanceRepository → ReactiveCaseInstanceRepository**

Use `mcp__intellij-index__ide_refactor_rename` on `common/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java` with `newName=ReactiveCaseInstanceRepository`. This updates all imports, injection sites, `quarkus.arc.selected-alternatives`, and `quarkus.arc.exclude-types` across all 26 repos.

- [ ] **Step 3: Rename CrossTenantCaseInstanceRepository → ReactiveCrossTenantCaseInstanceRepository**

Same refactoring on `common/src/main/java/io/casehub/engine/common/spi/CrossTenantCaseInstanceRepository.java` with `newName=ReactiveCrossTenantCaseInstanceRepository`.

- [ ] **Step 4: Rename EventLogRepository → ReactiveEventLogRepository**

Same refactoring on `common/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java` with `newName=ReactiveEventLogRepository`.

- [ ] **Step 5: Rename CrossTenantEventLogRepository → ReactiveCrossTenantEventLogRepository**

Same refactoring on `common/src/main/java/io/casehub/engine/common/spi/CrossTenantEventLogRepository.java` with `newName=ReactiveCrossTenantEventLogRepository`.

- [ ] **Step 6: Sync files** — IntelliJ refactoring modifies files on disk. Run `ide_sync_files` to ensure consistency.

- [ ] **Step 7: Build engine**

```bash
mvn install -DskipTests -q
```

Fix any compilation errors from config strings not caught by the refactoring (e.g. `quarkus.arc.selected-alternatives` values that may use simple class names).

- [ ] **Step 8: Commit all repos**

Commit each changed repo separately. Check `git status` in each repo under `/Users/mdproctor/claude/casehub/` and commit those with changes:

```bash
git add -A && git commit -m "refactor(#640): rename CaseInstanceRepository/EventLogRepository to Reactive* prefix

Rename 4 repository interfaces to Reactive* prefix following PlanItemStore/
ReactivePlanItemStore convention. Blocking interfaces (unqualified names) will
be created in the next step.

Refs #640"
```

For consumer repos, use the same message but with `Refs casehubio/engine#640`.

---

### Task 6: Repository rename — create blocking interfaces (#640 Part 2)

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/CaseInstanceRepository.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/CrossTenantCaseInstanceRepository.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/EventLogRepository.java`
- Create: `common/src/main/java/io/casehub/engine/common/spi/CrossTenantEventLogRepository.java`

**Interfaces:**
- Produces: 4 blocking interfaces mirroring the reactive versions with direct return types instead of `Uni<>`

- [ ] **Step 1: Create blocking CaseInstanceRepository** — mirror of `ReactiveCaseInstanceRepository` with direct return types. Use the exact method signatures from the spec §3.

- [ ] **Step 2: Create blocking CrossTenantCaseInstanceRepository** — mirror of `ReactiveCrossTenantCaseInstanceRepository`:

```java
package io.casehub.engine.common.spi;

import io.casehub.engine.common.internal.model.CaseInstance;
import java.util.UUID;

public interface CrossTenantCaseInstanceRepository {
  CaseInstance findByUuid(UUID caseId);
}
```

- [ ] **Step 3: Create blocking EventLogRepository** — mirror of `ReactiveEventLogRepository` with direct return types. Use the exact method signatures from the spec §3.

- [ ] **Step 4: Create blocking CrossTenantEventLogRepository** — mirror of `ReactiveCrossTenantEventLogRepository`. Use the exact method signatures from the spec §3.

- [ ] **Step 5: Build**

```bash
mvn install -DskipTests -q
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat(#640): create blocking CaseInstanceRepository + EventLogRepository interfaces

4 new blocking interfaces matching the PlanItemStore convention (unqualified =
blocking, Reactive prefix = reactive). Default methods return List.of() where
applicable.

Refs #640"
```

---

### Task 7: Repository rename — implementation split (#640 Part 3)

**Files (per persistence layer — 2 new classes each, 1 modified):**

**persistence-memory (4 new classes):**
- Rename existing: `InMemoryCaseInstanceRepository` → `InMemoryReactiveCaseInstanceRepository`
- Create: `InMemoryCaseInstanceRepository` (blocking, canonical)
- Rename existing: `InMemoryEventLogRepository` → `InMemoryReactiveEventLogRepository`
- Create: `InMemoryEventLogRepository` (blocking, canonical)
- Same for CrossTenant variants

**persistence-hibernate (4 new classes):**
- Rename existing: `JpaCaseInstanceRepository` → `JpaReactiveCaseInstanceRepository`
- Create: `JpaCaseInstanceRepository` (blocking, delegates to reactive)
- Rename existing: `JpaEventLogRepository` → `JpaReactiveEventLogRepository`
- Create: `JpaEventLogRepository` (blocking, delegates to reactive)
- Same for CrossTenant variants

**testing (4 new classes):**
- Rename existing: `TestCaseInstanceRepository` → `TestReactiveCaseInstanceRepository`
- Create: `TestCaseInstanceRepository` (blocking, canonical)
- Rename existing: `TestEventLogRepository` → `TestReactiveEventLogRepository`
- Create: `TestEventLogRepository` (blocking, canonical)

**Interfaces:**
- Consumes: Blocking `CaseInstanceRepository`, `EventLogRepository` (from Task 6); Reactive `ReactiveCaseInstanceRepository`, `ReactiveEventLogRepository` (from Task 5)
- Produces: Full dual-stack implementations in all 3 persistence layers

- [ ] **Step 1: persistence-memory — rename existing implementations to Reactive*** using IntelliJ `ide_refactor_rename`. Rename `InMemoryCaseInstanceRepository` → `InMemoryReactiveCaseInstanceRepository`, etc.

- [ ] **Step 2: persistence-memory — create blocking implementations.** Each blocking class is the canonical implementation (ConcurrentHashMap-backed). The renamed reactive class injects the blocking delegate and wraps with `Uni.createFrom().item(...)`.

- [ ] **Step 3: persistence-hibernate — rename existing implementations to Reactive*** using IntelliJ `ide_refactor_rename`.

- [ ] **Step 4: persistence-hibernate — create blocking implementations.** Each blocking class injects the reactive delegate and awaits via `.await().indefinitely()`. Reactive is canonical (Panache reactive).

- [ ] **Step 5: testing — rename and create** using the same pattern as persistence-memory (blocking canonical).

- [ ] **Step 6: Update all `quarkus.arc.selected-alternatives`** and `quarkus.arc.exclude-types` in test `application.properties` across all modules — add both blocking and reactive class names where currently only one appears.

- [ ] **Step 7: Build and test all persistence modules**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl persistence-memory -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl persistence-hibernate -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl testing -q
```

- [ ] **Step 8: Full build and test**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -q
```

- [ ] **Step 9: Commit all repos**

```bash
git add -A && git commit -m "feat(#640): dual-stack blocking/reactive repository implementations

Split InMemory*, Jpa*, and Test* implementations into separate blocking and
reactive classes. Memory: blocking canonical, reactive wraps. JPA: reactive
canonical (Panache), blocking awaits. Testing: blocking canonical.

Closes #640"
```

---

### Task 8: DefaultCaseEventRecorder implementation (#626)

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultReactiveCaseEventRecorder.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseEventRecorder.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseEventRecorderTest.java`

**Interfaces:**
- Consumes: `CaseEventRecorder`, `ReactiveCaseEventRecorder`, `CaseEventRequest` (from Task 4); `ReactiveEventLogRepository` (from Task 5)
- Produces: `@ApplicationScoped` implementations that delegate to `ReactiveEventLogRepository`

- [ ] **Step 1: Write test** — unit test with a recording `ReactiveEventLogRepository` that captures appended `EventLog` objects:

```java
@Test
void record_delegatesToEventLogRepository() {
    // Arrange: recording reactive event log repo
    // Act: recorder.record(request).await().indefinitely()
    // Assert: captured EventLog has correct caseId, type, stream, workerId, tenancyId, payload, metadata
}

@Test
void recordAndReturnId_returnsId() {
    // Arrange: recording reactive event log repo that returns 42L
    // Act: Long id = recorder.recordAndReturnId(request).await().indefinitely()
    // Assert: id == 42L
}
```

- [ ] **Step 2: Implement DefaultReactiveCaseEventRecorder** — `@ApplicationScoped`, injects `ReactiveEventLogRepository`. Constructs `EventLog` from `CaseEventRequest` fields, delegates to `repo.append()` / `repo.appendAndReturnId()`.

- [ ] **Step 3: Implement DefaultCaseEventRecorder** — `@ApplicationScoped`, injects `DefaultReactiveCaseEventRecorder`. Blocking delegates via `.await().indefinitely()`.

- [ ] **Step 4: Build and test**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -q
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat(#626): DefaultCaseEventRecorder — event write SPI implementation

@ApplicationScoped implementations delegating to ReactiveEventLogRepository.
Reactive is canonical; blocking awaits. Constructs EventLog domain objects
internally — consumers never import EventLog.

Closes #626"
```

---

### Task 9: Consumer repo migration (#644)

**Precondition:** Verify each repo is on `main` before changing. **quarkmind is currently on `issue-191-milestone-trust-scoring` — skip it.**

**Files (per repo):** Each consumer repo's strategy implementation files.

#### Task 9A: TrustRoutingPolicyProvider — add id() (6 repos)

For each of: devtown, aml, clinical, life, ops — read the class, add:

```java
@Override
public String id() {
    return "<repo-name>";
}
```

| Repo | File | id() |
|------|------|------|
| devtown | `app/src/main/java/io/casehub/devtown/app/routing/DevtownTrustRoutingPolicyProvider.java` | `"devtown"` |
| aml | `app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java` | `"aml"` |
| clinical | `runtime/src/main/java/io/casehub/clinical/routing/ClinicalTrustRoutingPolicyProvider.java` | `"clinical"` |
| life | `app/src/main/java/io/casehub/life/app/routing/LifeTrustRoutingPolicyProvider.java` | `"life"` |
| ops | `deployment/src/main/java/io/casehub/ops/deployment/DeploymentTrustRoutingPolicyProvider.java` | `"ops-deployment"` |

- [ ] **Step 1:** For each repo, verify on `main`: `git -C /Users/mdproctor/claude/casehub/<repo> branch --show-current`
- [ ] **Step 2:** Add `id()` method to each class
- [ ] **Step 3:** Commit each repo: `git -C /Users/mdproctor/claude/casehub/<repo> add -A && git -C /Users/mdproctor/claude/casehub/<repo> commit -m "chore(casehubio/engine#644): add id() to TrustRoutingPolicyProvider — NamedStrategy compliance"`

#### Task 9B: ActionRiskClassifier — StaticSetStrategy.of() (6 repos)

For each of: devtown, clinical, life, soc, iot — replace `List.of(...)` with `StaticSetStrategy.of(...)` in `RiskDecision.GateRequired` constructors. Add `import io.casehub.api.spi.routing.StaticSetStrategy`.

For aml — the migration is in the `AmlActionType` enum (field type `List<String>` → `CandidateSetStrategy`, constants use `StaticSetStrategy.of(...)`, return type of `candidateGroups()` → `CandidateSetStrategy`).

| Repo | File(s) |
|------|---------|
| devtown | `review/src/main/java/io/casehub/devtown/review/DevtownActionRiskClassifier.java` |
| aml | `app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java` + `AmlActionType` enum |
| clinical | `runtime/src/main/java/io/casehub/clinical/routing/ClinicalActionRiskClassifier.java` |
| life | `app/src/main/java/io/casehub/life/app/routing/LifeActionRiskClassifier.java` |
| soc | Find `SocActionRiskClassifier` in soc repo |
| iot | `webapp-api/src/main/java/io/casehub/iot/webapp/risk/IoTActionRiskClassifier.java` |

- [ ] **Step 1:** For each repo, read the classifier file to find all `List.of(...)` → `StaticSetStrategy.of(...)` sites
- [ ] **Step 2:** Make the changes, add the import
- [ ] **Step 3:** For aml, also update the `AmlActionType` enum
- [ ] **Step 4:** Commit each repo: `git -C /Users/mdproctor/claude/casehub/<repo> add -A && git -C /Users/mdproctor/claude/casehub/<repo> commit -m "chore(casehubio/engine#644): migrate ActionRiskClassifier to StaticSetStrategy.of()"`

#### Task 9C: quarkmind full migration (BLOCKED — not on main)

**Skip until quarkmind returns to main.** When ready, the migration includes:
1. `DispositionAwareRoutingStrategy` — add `id()` returning `"quarkmind-disposition"`
2. `DispositionAwareRoutingStrategy` — update `AgentAssignment.unresolvable()`, `.escalate()` calls with rationale strings
3. `DispositionAwareRoutingStrategy` — update `ScoredCandidate` constructors with rationale
4. `DispositionAwareRoutingStrategyTest` — update `AgentRoutingContext` constructors with 4th arg
5. `QuarkMindTrustRoutingPolicyProvider` — add `id()` returning `"quarkmind"`

- [ ] **Step 1:** File a GitHub issue on quarkmind for the migration, referencing engine#644

---

### Task 10: CI dispatch list (#583)

**Files:**
- Modify: `.github/workflows/publish.yml`

- [ ] **Step 1: Update the dispatch loop** — add `blocks soc iot clinical quarkmind ops`:

```yaml
      - name: Trigger downstream CI
        if: github.event_name != 'pull_request' && success()
        run: |
          for repo in scaffold openclaw claudony workers aml devtown life blocks soc iot clinical quarkmind ops; do
            gh api repos/casehubio/$repo/dispatches \
              -f event_type=upstream-published \
              -f client_payload[source]="${GITHUB_REPOSITORY}" \
              2>/dev/null && echo "  ✅ $repo triggered" || echo "  ⚠️  $repo trigger failed"
          done
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/publish.yml && git commit -m "ci(#583): add blocks, soc, iot, clinical, quarkmind, ops to downstream dispatch

All repos depending on engine packages now receive upstream-published events.

Closes #583"
```

---

## Verification

After all tasks complete:

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q
```

All engine modules must compile and pass. Consumer repos should be verified individually where changes were made.

## Issues to file

- [ ] `casehubio/engine` — naming consistency cleanup for `CaseMetaModelRepository` and `SubCaseGroupRepository` (also reactive-only)
- [ ] `casehubio/quarkmind` — full engine#644 migration (blocked on current branch)
