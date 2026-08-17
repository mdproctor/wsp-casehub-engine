# Delegation and Escalation Compliance Observation

**Issue:** engine#645
**Date:** 2026-08-17
**Status:** Draft

## Context

`BehavioralComplianceRecorder` currently observes two compliance dimensions at worker completion time:

- **LATENCY** — checks execution duration against `AgentCapability.latencyHintP50Ms()` × 2.0 multiplier. Gate: latency hint present on the capability.
- **ATTESTATION_RATE** — records COMPLIANT for success/completed outcomes, VIOLATED for declined/failed/expired. No gate — always observed.

The eidos API (`casehub-eidos-api`) already defines two additional dimensions that the engine does not yet observe:

- `ComplianceDimension.DELEGATION` — constant defined, `BehavioralExpectations.delegationExpected(AgentDisposition)` gate implemented
- `ComplianceDimension.ESCALATION` — constant defined, `BehavioralExpectations.escalationExpected(AgentDescriptor, VocabularyRegistry)` gate implemented

This spec adds observation logic for both dimensions within the existing recorder, following the same two-gate pattern used by latency observation.

## Two-Gate Model

Each dimension uses two gates before recording a signal:

| | Gate 1 (eidos — agent-level) | Gate 2 (engine — task-level) |
|---|---|---|
| LATENCY | `latencyHintP50Ms()` present on capability | (implicit — hint IS the gate) |
| ATTESTATION_RATE | (none — always observed) | (none) |
| **DELEGATION** | `delegationExpected(disposition)` | CaseDefinition has decomposition infrastructure |
| **ESCALATION** | `escalationExpected(descriptor, registry)` | (none — eidos gate sufficiently narrow) |

Gate 1 answers: "Is this agent expected to exhibit this behavior in general?"
Gate 2 answers: "Is this specific task one where the behavior is structurally possible?"

Both gates must pass before observation occurs. When either gate fails, no signal is recorded for that dimension.

## Delegation Observation

### Gate 1 — Eidos disposition

```java
BehavioralExpectations.delegationExpected(descriptor.disposition())
```

Returns true when `AgentDisposition.delegation()` is true — the agent's disposition profile marks it as a delegating agent.

### Gate 2 — Structural decomposition available

The case definition provides infrastructure for delegation:

```java
definition.getDecompositionStrategy() != null
```

OR any binding in the definition has a `SubCaseTarget`:

```java
definition.getBindings().stream()
    .anyMatch(b -> b.getTarget() instanceof SubCaseTarget)
```

When neither condition holds, the case has no decomposition paths — delegation is not structurally possible regardless of the agent's disposition.

### Evidence — Did delegation happen?

Query `PlanItemStore.findByCaseId(caseId, tenancyId)` and check for any `PlanItemRecord` with `parentCompoundId != null`. Compound children indicate the case's decomposition structure was exercised.

- Compound children found → `BehavioralSignal.COMPLIANT`
- No compound children despite structural availability → `BehavioralSignal.VIOLATED`

### Known v1 limitation

Delegation evidence is **case-level**, not per-execution. Every worker completion for the same case observes the same delegation state. Consequences:

- Pre-execution decomposition (GoalDecomposer at case start) counts as delegation even though the system decomposed, not the agent.
- An agent that doesn't delegate but executes in a pre-decomposed case receives COMPLIANT signals it didn't earn.

Acceptable for v1 because:
1. Most decomposed cases are designed for agents whose goals drove the decomposition
2. The TTL + threshold model in eidos absorbs noise from repeated signals
3. Per-execution tracking via a `WorkerRuntime` delegation flag is the planned v2 enhancement

### Method signature

```java
private void recordDelegation(
    String agentId,
    String tenancyId,
    String capabilityName,
    AgentDescriptor descriptor,
    CaseDefinition definition,
    UUID caseId)
```

## Escalation Observation

### Gate 1 — Eidos autonomy disposition

```java
BehavioralExpectations.escalationExpected(descriptor, vocabularyRegistry)
```

Returns true when the agent's autonomy axis contains a vocabulary term where `impliesSupervision()` is true. This identifies agents that operate under supervision and are expected to escalate consequential decisions.

No Gate 2 — the eidos gate is already narrow (only supervised agents pass). Every completion by a supervised agent is observed.

### Signal semantics

| Outcome | Signal | Rationale |
|---|---|---|
| `Success` with `PlannedAction` present | COMPLIANT | Agent flagged a consequential action for oversight |
| `Declined` | COMPLIANT | Agent acknowledged it can't handle this — escalation by refusal |
| `Success` without `PlannedAction` | VIOLATED | Supervised agent acted autonomously |
| `Failed` | no observation | Failure is not an escalation decision |
| `Expired` | no observation | Timeout is not an escalation decision |
| `Completed` | VIOLATED | Scoped worker completed autonomously (same as Success without PlannedAction) |

### Cross-dimension interaction

`WorkerOutcome.Declined` records **VIOLATED** for ATTESTATION_RATE and **COMPLIANT** for ESCALATION simultaneously. This is deliberate:

- Declining hurts reliability (the agent didn't do the work)
- Declining demonstrates safety judgment (the agent recognized its limits)

The net effect on trust depends on dimension weights configured per deployment. Safety-critical deployments weight escalation higher; throughput-oriented deployments weight attestation higher.

### Method signature

```java
private void recordEscalation(
    String agentId,
    String tenancyId,
    String capabilityName,
    AgentDescriptor descriptor,
    WorkerOutcome<?> outcome)
```

## Constructor Changes

`BehavioralComplianceRecorder` gains two new dependencies:

```java
@Inject
public BehavioralComplianceRecorder(
    Instance<BehavioralSignalStore> signalStore,
    CaseDefinitionRegistry registry,
    Instance<PlanItemStore> planItemStore,       // NEW — delegation evidence
    VocabularyRegistry vocabularyRegistry)       // NEW — escalation eligibility
```

- `Instance<PlanItemStore>` — guarded with `isResolvable()`. When the planning module is absent, delegation observation is silently skipped. The `findByCaseId()` call is a blocking store query; acceptable because `WorkflowExecutionCompletedHandler` runs on `@RunOnVirtualThread` context.
- `VocabularyRegistry` — injected directly (not `Instance<>`). Always available via `NoOpVocabularyRegistry` (`@DefaultBean`) or `CdiVocabularyRegistry` when eidos-runtime is on the classpath.

## Record Method Changes

The `record()` method gains `UUID caseId` as a parameter:

```java
public void record(
    CaseInstance caseInstance,
    String workerName,
    String capabilityName,
    WorkerOutcome<?> outcome,
    Long executionDurationMs) {
  // ... existing guard, definition lookup, descriptor lookup ...

  recordLatency(agentId, tenancyId, capabilityName, descriptor, executionDurationMs);
  recordAttestation(agentId, tenancyId, capabilityName, outcome);
  recordDelegation(agentId, tenancyId, capabilityName, descriptor, definition, caseInstance.getId());
  recordEscalation(agentId, tenancyId, capabilityName, descriptor, outcome);
}
```

No changes to `WorkflowExecutionCompletedHandler` call sites — `caseInstance` is already passed, and `getId()` is called inside the recorder.

## Testing

Unit tests follow the existing `BehavioralComplianceRecorderTest` pattern with Mockito:

### Delegation tests

| Test | Setup | Expected |
|---|---|---|
| `delegationCompliant_recordsCompliantSignal` | delegation=true, decompositionStrategy set, PlanItemStore returns items with parentCompoundId | COMPLIANT on DELEGATION |
| `delegationViolated_recordsViolatedSignal` | delegation=true, decompositionStrategy set, PlanItemStore returns items without parentCompoundId | VIOLATED on DELEGATION |
| `delegationNotExpected_skips` | delegation=false | no DELEGATION signal |
| `noDecompositionInfrastructure_skips` | delegation=true, no decompositionStrategy, no SubCaseTarget bindings | no DELEGATION signal |
| `planItemStoreUnavailable_skips` | delegation=true, decompositionStrategy set, PlanItemStore not resolvable | no DELEGATION signal |

### Escalation tests

| Test | Setup | Expected |
|---|---|---|
| `escalationCompliant_plannedAction` | escalation expected, Success with PlannedAction | COMPLIANT on ESCALATION |
| `escalationCompliant_declined` | escalation expected, Declined outcome | COMPLIANT on ESCALATION |
| `escalationViolated_autonomousSuccess` | escalation expected, Success without PlannedAction | VIOLATED on ESCALATION |
| `escalationViolated_completed` | escalation expected, Completed outcome | VIOLATED on ESCALATION |
| `escalationNotExpected_skips` | escalation not expected (no supervision vocabulary) | no ESCALATION signal |
| `failedOutcome_noEscalationObservation` | escalation expected, Failed outcome | no ESCALATION signal |
| `expiredOutcome_noEscalationObservation` | escalation expected, Expired outcome | no ESCALATION signal |

### Cross-dimension test

| Test | Setup | Expected |
|---|---|---|
| `declined_crossDimensionInteraction` | escalation expected, Declined outcome | VIOLATED on ATTESTATION_RATE AND COMPLIANT on ESCALATION |

## Files Changed

| File | Change |
|---|---|
| `runtime/.../BehavioralComplianceRecorder.java` | Add `recordDelegation()`, `recordEscalation()`, new constructor params |
| `runtime/.../BehavioralComplianceRecorderTest.java` | Add delegation, escalation, and cross-dimension tests |

No changes to eidos-api (constants and expectation methods already exist).
No changes to WorkflowExecutionCompletedHandler (call sites already pass CaseInstance).

## References

- `BehavioralComplianceRecorder.java` — existing recorder with latency/attestation pattern
- `BehavioralExpectations.java` (eidos-api) — `delegationExpected()`, `escalationExpected()` gates
- `ComplianceDimension.java` (eidos-api) — DELEGATION, ESCALATION constants
- `AgentDisposition.java` (eidos-api) — `delegation()` boolean, `autonomy()` axis
- `VocabularyTerm.java` (eidos-api) — `impliesSupervision()` method
- `PlanItemRecord.java` — `parentCompoundId` field for delegation evidence
- `WorkerOutcome.java` — `Success.plannedAction()`, `Declined` for escalation signals
- `WorkflowExecutionCompletedHandler.java:209-214, 411-412` — call sites
- eidos#87 — delegation/escalation observation semantics design
- eidos#85 — behavioral contracts framework
- engine#632 / eidos#89 — latency/attestation-rate observation (prior art)
