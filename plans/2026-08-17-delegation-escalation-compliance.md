# Delegation and Escalation Compliance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #645 — Observe delegation and escalation compliance dimensions from eidos framework
**Issue group:** #645

**Goal:** Add DELEGATION and ESCALATION compliance dimensions to `BehavioralComplianceRecorder`, following the existing two-gate observation pattern.

**Architecture:** Extend the existing recorder with two new private methods (`recordDelegation`, `recordEscalation`) and two new constructor dependencies (`Instance<PlanItemStore>`, `VocabularyRegistry`). No new classes. No changes to eidos-api or WorkflowExecutionCompletedHandler call sites.

**Tech Stack:** Java 21, Quarkus CDI, eidos-api (BehavioralExpectations, ComplianceDimension, BehavioralSignalStore), Mockito for tests

## Global Constraints

- No changes to eidos-api — all constants and expectation methods already exist
- No changes to WorkflowExecutionCompletedHandler call sites — CaseInstance already passed to recorder
- Follow existing `recordLatency`/`recordAttestation` pattern exactly
- `Instance<>` guard pattern for optional dependencies (`PlanItemStore`)
- Direct injection for always-available dependencies (`VocabularyRegistry`)

---

## Batch 1: Delegation and Escalation Compliance Observation

### Task 1: Add delegation observation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorder.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorderTest.java`

**Interfaces:**
- Consumes: `BehavioralExpectations.delegationExpected(AgentDisposition)` from eidos-api, `PlanItemStore.findByCaseId(UUID, String)` from common SPI, `ComplianceDimension.DELEGATION` from eidos-api
- Produces: `recordDelegation(String, String, String, AgentDescriptor, CaseDefinition, UUID)` — private method, no external consumers

- [ ] **Step 1: Write failing test — delegation COMPLIANT when compound children exist**

Add to `BehavioralComplianceRecorderTest.java`:

```java
@Test
void delegationCompliant_recordsCompliantSignal() {
  AgentDescriptor delegating = AgentDescriptor.builder()
      .agentId("agent-1")
      .name("Test Agent")
      .slot("test")
      .tenancyId("tenant-1")
      .capabilities(java.util.List.of(
          AgentCapability.builder().name("analysis").latencyHintP50Ms(1000L).build()))
      .disposition(AgentDisposition.builder().delegation(true).build())
      .build();
  when(definition.agentDescriptorFor("worker-1")).thenReturn(Optional.of(delegating));
  when(definition.getDecompositionStrategy()).thenReturn("llm");

  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  PlanItemRecord compoundChild = PlanItemRecord.primitive(
      caseUuid, "pi-1", "child-binding", TaskStatus.COMPLETED,
      java.time.Instant.now(), TargetType.CAPABILITY, null, "tenant-1",
      "child task", "worker-2", "Worker 2");
  // Simulate parentCompoundId by using the full constructor
  PlanItemRecord withParent = new PlanItemRecord(
      caseUuid, "pi-1", "child-binding", TaskStatus.COMPLETED,
      java.time.Instant.now(), null, TargetType.CAPABILITY, null, "tenant-1",
      "child task", "worker-2", "Worker 2",
      PlanItemType.PRIMITIVE, null, null, null, false,
      "compound-1", null, null, null);
  when(planItemStore.findByCaseId(caseUuid, "tenant-1"))
      .thenReturn(java.util.List.of(withParent));

  recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.DELEGATION), eq(BehavioralSignal.COMPLIANT));
}
```

This requires updating setUp to add `planItemStore` mock and updating the recorder constructor. Add these fields and setUp changes:

```java
private PlanItemStore planItemStore;
private VocabularyRegistry vocabularyRegistry;
```

In setUp:
```java
planItemStore = mock(PlanItemStore.class);
Instance<PlanItemStore> planItemStoreInstance = mock(Instance.class);
when(planItemStoreInstance.isResolvable()).thenReturn(true);
when(planItemStoreInstance.get()).thenReturn(planItemStore);

vocabularyRegistry = mock(VocabularyRegistry.class);

recorder = new BehavioralComplianceRecorder(storeInstance, registry, planItemStoreInstance, vocabularyRegistry);
```

Also add imports:
```java
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.internal.model.PlanItemType;
import io.casehub.engine.common.internal.model.TargetType;
import io.casehub.api.model.TaskStatus;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.VocabularyRegistry;
import java.util.UUID;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#delegationCompliant_recordsCompliantSignal -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: Compilation failure — constructor doesn't accept 4 args yet

- [ ] **Step 3: Update constructor and add recordDelegation method**

Update `BehavioralComplianceRecorder.java`:

Add fields:
```java
private final Instance<PlanItemStore> planItemStore;
private final VocabularyRegistry vocabularyRegistry;
```

Update constructor:
```java
@Inject
public BehavioralComplianceRecorder(
    Instance<BehavioralSignalStore> signalStore,
    CaseDefinitionRegistry registry,
    Instance<PlanItemStore> planItemStore,
    VocabularyRegistry vocabularyRegistry) {
  this.signalStore = signalStore;
  this.registry = registry;
  this.planItemStore = planItemStore;
  this.vocabularyRegistry = vocabularyRegistry;
}
```

Add two new calls in the `record()` method after `recordAttestation`:
```java
recordDelegation(agentId, tenancyId, capabilityName, descriptor, definition, caseInstance.getUuid());
recordEscalation(agentId, tenancyId, capabilityName, descriptor, outcome);
```

Add `recordDelegation` method:
```java
private void recordDelegation(
    String agentId,
    String tenancyId,
    String capabilityName,
    AgentDescriptor descriptor,
    CaseDefinition definition,
    UUID caseId) {
  if (!BehavioralExpectations.delegationExpected(descriptor.disposition())) {
    return;
  }
  boolean hasDecomposition = definition.getDecompositionStrategy() != null
      || definition.getBindings().stream()
          .anyMatch(b -> b.target() instanceof SubCaseTarget);
  if (!hasDecomposition) {
    return;
  }
  if (!planItemStore.isResolvable()) {
    return;
  }
  List<PlanItemRecord> items = planItemStore.get().findByCaseId(caseId, tenancyId);
  boolean delegated = items.stream().anyMatch(r -> r.parentCompoundId() != null);
  BehavioralSignal signal = delegated ? BehavioralSignal.COMPLIANT : BehavioralSignal.VIOLATED;
  signalStore
      .get()
      .record(agentId, tenancyId, capabilityName, ComplianceDimension.DELEGATION, signal);
  LOG.debugf(
      "Delegation %s: agent=%s capability=%s delegated=%s",
      signal, agentId, capabilityName, delegated);
}
```

Add stub `recordEscalation` (empty, implemented in Task 2):
```java
private void recordEscalation(
    String agentId,
    String tenancyId,
    String capabilityName,
    AgentDescriptor descriptor,
    WorkerOutcome<?> outcome) {
  // Implemented in Task 2
}
```

Add imports:
```java
import io.casehub.api.model.SubCaseTarget;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.eidos.api.VocabularyRegistry;
import java.util.List;
import java.util.UUID;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#delegationCompliant_recordsCompliantSignal -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write failing test — delegation VIOLATED when no compound children**

```java
@Test
void delegationViolated_recordsViolatedSignal() {
  AgentDescriptor delegating = AgentDescriptor.builder()
      .agentId("agent-1")
      .name("Test Agent")
      .slot("test")
      .tenancyId("tenant-1")
      .capabilities(java.util.List.of(
          AgentCapability.builder().name("analysis").latencyHintP50Ms(1000L).build()))
      .disposition(AgentDisposition.builder().delegation(true).build())
      .build();
  when(definition.agentDescriptorFor("worker-1")).thenReturn(Optional.of(delegating));
  when(definition.getDecompositionStrategy()).thenReturn("llm");

  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  PlanItemRecord leafOnly = PlanItemRecord.primitive(
      caseUuid, "pi-1", "leaf-binding", TaskStatus.COMPLETED,
      java.time.Instant.now(), TargetType.CAPABILITY, null, "tenant-1",
      "leaf task", "worker-1", "Worker 1");
  when(planItemStore.findByCaseId(caseUuid, "tenant-1"))
      .thenReturn(java.util.List.of(leafOnly));

  recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.DELEGATION), eq(BehavioralSignal.VIOLATED));
}
```

- [ ] **Step 6: Run test to verify it passes** (implementation already handles this case)

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#delegationViolated_recordsViolatedSignal -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Write failing test — delegation not expected skips**

```java
@Test
void delegationNotExpected_skips() {
  // Default descriptor has no disposition set — delegation() returns false
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);
  when(definition.getDecompositionStrategy()).thenReturn("llm");

  recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.DELEGATION), any());
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#delegationNotExpected_skips -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 9: Write failing test — no decomposition infrastructure skips**

```java
@Test
void noDecompositionInfrastructure_skipsDelegation() {
  AgentDescriptor delegating = AgentDescriptor.builder()
      .agentId("agent-1")
      .name("Test Agent")
      .slot("test")
      .tenancyId("tenant-1")
      .capabilities(java.util.List.of(
          AgentCapability.builder().name("analysis").build()))
      .disposition(AgentDisposition.builder().delegation(true).build())
      .build();
  when(definition.agentDescriptorFor("worker-1")).thenReturn(Optional.of(delegating));
  when(definition.getDecompositionStrategy()).thenReturn(null);
  when(definition.getBindings()).thenReturn(java.util.List.of());

  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.DELEGATION), any());
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#noDecompositionInfrastructure_skipsDelegation -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 11: Write failing test — PlanItemStore unavailable skips**

```java
@SuppressWarnings("unchecked")
@Test
void planItemStoreUnavailable_skipsDelegation() {
  Instance<PlanItemStore> absentStore = mock(Instance.class);
  when(absentStore.isResolvable()).thenReturn(false);

  var recorderNoPlanItems = new BehavioralComplianceRecorder(
      storeInstance, registry, absentStore, vocabularyRegistry);

  AgentDescriptor delegating = AgentDescriptor.builder()
      .agentId("agent-1")
      .name("Test Agent")
      .slot("test")
      .tenancyId("tenant-1")
      .capabilities(java.util.List.of(
          AgentCapability.builder().name("analysis").build()))
      .disposition(AgentDisposition.builder().delegation(true).build())
      .build();
  when(definition.agentDescriptorFor("worker-1")).thenReturn(Optional.of(delegating));
  when(definition.getDecompositionStrategy()).thenReturn("llm");

  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  recorderNoPlanItems.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.DELEGATION), any());
}
```

- [ ] **Step 12: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#planItemStoreUnavailable_skipsDelegation -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 13: Run all existing tests to verify no regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS (existing latency/attestation tests must still pass with new constructor — setUp was updated)

Note: the existing `storeUnavailable_noOp` test creates its own recorder — update it to pass the new constructor params:
```java
var silentRecorder = new BehavioralComplianceRecorder(absent, registry, planItemStoreInstance, vocabularyRegistry);
```

Where `planItemStoreInstance` and `vocabularyRegistry` are the mocks from setUp. Also update any other test that constructs the recorder directly.

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorder.java runtime/src/test/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorderTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#645): add delegation compliance observation to BehavioralComplianceRecorder

Two-gate model: eidos disposition gate (delegationExpected) + engine
structural gate (decompositionStrategy or SubCaseTarget binding).
Evidence via PlanItemStore compound children query.

Refs #645"
```

### Task 2: Add escalation observation

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorder.java`
- Modify: `runtime/src/test/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorderTest.java`

**Interfaces:**
- Consumes: `BehavioralExpectations.escalationExpected(AgentDescriptor, VocabularyRegistry)` from eidos-api, `ComplianceDimension.ESCALATION` from eidos-api, `WorkerOutcome.Success.plannedAction()` from worker-api
- Produces: `recordEscalation(String, String, String, AgentDescriptor, WorkerOutcome<?>)` — private method, no external consumers

- [ ] **Step 1: Write failing test — escalation COMPLIANT when PlannedAction present**

```java
@Test
void escalationCompliant_plannedAction() {
  AgentDescriptor supervised = AgentDescriptor.builder()
      .agentId("agent-1")
      .name("Test Agent")
      .slot("test")
      .tenancyId("tenant-1")
      .capabilities(java.util.List.of(
          AgentCapability.builder().name("analysis").build()))
      .build();
  when(definition.agentDescriptorFor("worker-1")).thenReturn(Optional.of(supervised));

  // Mock escalationExpected to return true
  try (var mocked = org.mockito.MockedStatic.class.cast(null)) {
    // Use a real approach: mock at the VocabularyRegistry level
  }
  // Simpler: use mockStatic on BehavioralExpectations
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(
        caseInstance, "worker-1", "analysis",
        WorkerOutcome.success(PlannedAction.of("File report", "report.file", java.util.Map.of())),
        null);
  }

  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.ESCALATION), eq(BehavioralSignal.COMPLIANT));
}
```

Add import:
```java
import io.casehub.worker.api.PlannedAction;
```

- [ ] **Step 2: Run test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#escalationCompliant_plannedAction -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: FAIL — `recordEscalation` is a stub

- [ ] **Step 3: Implement recordEscalation**

Replace the stub `recordEscalation` in `BehavioralComplianceRecorder.java`:

```java
private void recordEscalation(
    String agentId,
    String tenancyId,
    String capabilityName,
    AgentDescriptor descriptor,
    WorkerOutcome<?> outcome) {
  if (!BehavioralExpectations.escalationExpected(descriptor, vocabularyRegistry)) {
    return;
  }
  if (outcome instanceof WorkerOutcome.Failed || outcome instanceof WorkerOutcome.Expired) {
    return;
  }
  boolean escalated =
      (outcome instanceof WorkerOutcome.Success<?> s && s.plannedAction() != null)
          || outcome instanceof WorkerOutcome.Declined;
  BehavioralSignal signal = escalated ? BehavioralSignal.COMPLIANT : BehavioralSignal.VIOLATED;
  signalStore
      .get()
      .record(agentId, tenancyId, capabilityName, ComplianceDimension.ESCALATION, signal);
  LOG.debugf(
      "Escalation %s: agent=%s capability=%s outcome=%s",
      signal, agentId, capabilityName, outcome.getClass().getSimpleName());
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#escalationCompliant_plannedAction -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 5: Write failing test — escalation COMPLIANT when Declined**

```java
@Test
void escalationCompliant_declined() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(
        caseInstance, "worker-1", "analysis",
        new WorkerOutcome.Declined<>("not my area"), null);
  }

  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.ESCALATION), eq(BehavioralSignal.COMPLIANT));
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#escalationCompliant_declined -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 7: Write failing test — escalation VIOLATED when autonomous success**

```java
@Test
void escalationViolated_autonomousSuccess() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);
  }

  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.ESCALATION), eq(BehavioralSignal.VIOLATED));
}
```

- [ ] **Step 8: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#escalationViolated_autonomousSuccess -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 9: Write failing test — escalation not expected skips**

```java
@Test
void escalationNotExpected_skips() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  // Default: escalationExpected returns false (no autonomy vocabulary)
  recorder.record(caseInstance, "worker-1", "analysis", WorkerOutcome.success(), null);

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.ESCALATION), any());
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#escalationNotExpected_skips -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 11: Write failing test — Failed/Expired skip escalation observation**

```java
@Test
void failedOutcome_noEscalationObservation() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(
        caseInstance, "worker-1", "analysis",
        new WorkerOutcome.Failed<>("error"), null);
  }

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.ESCALATION), any());
}

@Test
void expiredOutcome_noEscalationObservation() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(
        caseInstance, "worker-1", "analysis",
        new WorkerOutcome.Expired<>("timeout"), null);
  }

  verify(signalStore, never())
      .record(anyString(), anyString(), anyString(),
          eq(ComplianceDimension.ESCALATION), any());
}
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#failedOutcome_noEscalationObservation+expiredOutcome_noEscalationObservation -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 13: Write cross-dimension test — Declined is VIOLATED attestation AND COMPLIANT escalation**

```java
@Test
void declined_crossDimensionInteraction() {
  UUID caseUuid = UUID.randomUUID();
  when(caseInstance.getUuid()).thenReturn(caseUuid);

  try (var expectations = org.mockito.Mockito.mockStatic(BehavioralExpectations.class)) {
    expectations.when(() -> BehavioralExpectations.delegationExpected(any()))
        .thenReturn(false);
    expectations.when(() -> BehavioralExpectations.escalationExpected(any(AgentDescriptor.class), any(VocabularyRegistry.class)))
        .thenReturn(true);
    expectations.when(() -> BehavioralExpectations.latencyBound(any()))
        .thenCallRealMethod();

    recorder.record(
        caseInstance, "worker-1", "analysis",
        new WorkerOutcome.Declined<>("not my area"), null);
  }

  // Attestation: VIOLATED (didn't do the work)
  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.ATTESTATION_RATE), eq(BehavioralSignal.VIOLATED));
  // Escalation: COMPLIANT (correctly escalated by declining)
  verify(signalStore).record(
      eq("agent-1"), eq("tenant-1"), eq("analysis"),
      eq(ComplianceDimension.ESCALATION), eq(BehavioralSignal.COMPLIANT));
}
```

- [ ] **Step 14: Run test to verify it passes**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest#declined_crossDimensionInteraction -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: PASS

- [ ] **Step 15: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn test -pl runtime -Dtest=BehavioralComplianceRecorderTest -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: ALL PASS

- [ ] **Step 16: Compile full runtime module to check for wiring issues**

Run: `TESTCONTAINERS_RYUK_DISABLED=true /opt/homebrew/bin/mvn compile -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml`
Expected: BUILD SUCCESS (CDI will discover `Instance<PlanItemStore>` and `VocabularyRegistry` at runtime)

- [ ] **Step 17: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add runtime/src/main/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorder.java runtime/src/test/java/io/casehub/engine/internal/routing/BehavioralComplianceRecorderTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(#645): add escalation compliance observation to BehavioralComplianceRecorder

Eidos disposition gate (escalationExpected via autonomy vocabulary).
PlannedAction/Declined = COMPLIANT, autonomous success = VIOLATED.
Failed/Expired outcomes skip escalation observation.

Cross-dimension: Declined records VIOLATED attestation + COMPLIANT
escalation simultaneously (deliberate — reliability vs safety).

Closes #645"
```

## References

- [2026-08-17-delegation-escalation-compliance-design.md] — design spec this plan implements
- [BehavioralComplianceRecorder.java] — existing recorder with latency/attestation pattern
- [BehavioralComplianceRecorderTest.java] — existing test pattern
- [BehavioralExpectations.class] — eidos-api delegation/escalation gates
- [ComplianceDimension.class] — DELEGATION, ESCALATION constants
- [PlanItemRecord.java:40] — parentCompoundId field for delegation evidence
- [CaseInstance.java:53-55] — getUuid() for PlanItemStore query
- [GitHub #645] — focal issue
