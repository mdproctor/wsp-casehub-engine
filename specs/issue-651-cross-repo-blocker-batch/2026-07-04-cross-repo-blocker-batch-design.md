# Cross-Repo Blocker Batch — Design Spec

**Date:** 2026-07-04
**Issues:** #651, #650, #583, #644, #640, #626
**Branch:** `issue-651-cross-repo-blocker-batch`

---

## Problem

Six issues block downstream consumers. Each is a gap in engine-api's boundary types or propagation:

- Routing context missing tenantId (#651)
- Routing result missing rationale (#650)
- Repository SPIs missing blocking variants, naming inconsistent (#640)
- Case lifecycle missing event write path for external consumers (#626)
- Consumer implementations broken after SPI evolution (#644)
- CI not triggering downstream builds for new repos (#583)

## Changes

### 1. AgentRoutingContext — add tenantId (#651)

Add `String tenantId` as the 4th field:

```java
public record AgentRoutingContext(
    UUID caseId, String capabilityName, JsonNode caseContext, String tenantId) {}
```

**Production construction sites** (2):
- `CaseContextChangedEventHandler.publishWorkerSchedule()` — pass `caseInstance.tenancyId`
- `DefaultWorkOrchestrator.doSubmit()` — pass `instance.tenancyId`

Strategy implementations receive tenantId via the context parameter. No signature changes to `AgentRoutingStrategy.select()`.

### 2. AgentAssignment — mandatory rationale (#650)

Add mandatory `String rationale` to all three sealed variants. Remove old factory methods — breaking is intentional, forces every caller to be explicit:

```java
public sealed interface AgentAssignment ... {
  record Assigned(String workerId, String rationale) implements AgentAssignment {}
  record Unresolvable(String rationale) implements AgentAssignment {}
  record EscalateToOversight(String capabilityName, EscalationReason reason,
      String rationale) implements AgentAssignment {}

  static AgentAssignment assign(String workerId, String rationale) { ... }
  static AgentAssignment unresolvable(String rationale) { ... }
  static AgentAssignment escalate(String capabilityName, EscalationReason reason,
      String rationale) { ... }
}
```

**Strategy rationale strings:**
- `LeastLoadedAgentStrategy`: `"selected %s: load %d (vs next: %s load %d)"`
- `TrustWeightedAgentStrategy`: `"selected %s: trust %.2f (threshold %.2f)"`
- `SemanticAgentRoutingStrategy`: `"selected %s: semantic %.2f, trust %.2f"`

Callers log rationale at INFO level via `a.rationale()`.

### 3. Repository dual-stack rename (#640 + EventLogRepository)

Full rename sweep following the `PlanItemStore` / `ReactivePlanItemStore` convention where unqualified = blocking:

| Current name | New name (reactive) | New name (blocking) |
|---|---|---|
| `CaseInstanceRepository` | `ReactiveCaseInstanceRepository` | `CaseInstanceRepository` |
| `CrossTenantCaseInstanceRepository` | `ReactiveCrossTenantCaseInstanceRepository` | `CrossTenantCaseInstanceRepository` |
| `EventLogRepository` | `ReactiveEventLogRepository` | `EventLogRepository` |
| `CrossTenantEventLogRepository` | `ReactiveCrossTenantEventLogRepository` | `CrossTenantEventLogRepository` |

**Blocking CaseInstanceRepository:**

```java
public interface CaseInstanceRepository {
  CaseInstance save(CaseInstance instance, String tenancyId);
  CaseInstance update(CaseInstance instance, String tenancyId);
  CaseInstance findByUuid(UUID uuid, String tenancyId);
  void updateStateAndAppendEvent(CaseInstance instance, EventLog eventLog, String tenancyId);
  default List<CaseInstance> findByStatus(CaseStatus status, String tenancyId) { return List.of(); }
  default List<CaseInstance> findAll(String tenancyId) { return List.of(); }
  default List<CaseInstance> findByNamespaceAndName(String ns, String name, String tenancyId) { return List.of(); }
}
```

**Blocking EventLogRepository:**

```java
public interface EventLogRepository {
  void append(EventLog eventLog, String tenancyId);
  Long appendAndReturnId(EventLog eventLog, String tenancyId);
  EventLog findById(Long id, String tenancyId);
  List<EventLog> findSchedulingEvents(UUID caseId, String workerId, Instant after, String tenancyId);
  default List<EventLog> findSchedulingEvents(UUID caseId, String workerId, String tenancyId) {
    return findSchedulingEvents(caseId, workerId, null, tenancyId);
  }
  List<EventLog> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types, String tenancyId);
  List<EventLog> findByCaseAndWorkerAndType(UUID caseId, String workerId, CaseHubEventType type, String tenancyId);
  List<EventLog> findByWorkerAndType(String workerId, CaseHubEventType type, String tenancyId);
  List<EventLog> findByCaseWithFilters(UUID caseId, Collection<CaseHubEventType> eventTypes,
      Collection<EventStreamType> streamTypes, String tenancyId);
}
```

**Blocking cross-tenant mirrors follow the same pattern.**

**Implementations:**
- `InMemoryCaseInstanceRepository` / `InMemoryEventLogRepository` — implement both blocking and reactive interfaces. Blocking methods are the canonical implementation; reactive methods delegate via `Uni.createFrom().item(() -> blockingMethod(...))`.
- `JpaCaseInstanceRepository` / `JpaEventLogRepository` — reactive is canonical (Panache reactive); blocking delegates via `.await().indefinitely()`.
- `TestCaseInstanceRepository` / `TestEventLogRepository` — implement both interfaces.

**Rename execution:** IntelliJ semantic refactoring across the open workspace (26 repos). All import sites, injection points, `quarkus.arc.selected-alternatives`, and `quarkus.arc.exclude-types` entries update automatically.

### 4. CaseEventRecorder — event write SPI (#626)

New interface in `api/spi/`:

```java
public interface CaseEventRecorder {
  CompletionStage<Void> recordEvent(
      UUID caseId, CaseHubEventType type, EventStreamType stream,
      String workerId, String tenancyId, JsonNode payload, JsonNode metadata);

  CompletionStage<Long> recordEventAndReturnId(
      UUID caseId, CaseHubEventType type, EventStreamType stream,
      String workerId, String tenancyId, JsonNode payload, JsonNode metadata);
}
```

tenancyId is explicit — consistent with every other SPI in the codebase (no implicit tenant from CDI scope).

**Engine implementation:** `DefaultCaseEventRecorder` in `runtime/internal/engine/` (`@ApplicationScoped`). Injects `ReactiveEventLogRepository`. Constructs `EventLog` domain objects internally — consumers never import `EventLog`.

**No-op default:** `NoOpCaseEventRecorder` in `runtime/internal/worker/` (`@DefaultBean @ApplicationScoped`). Returns completed futures. Follows the established SPI no-op pattern.

**New orchestration event types** on `CaseHubEventType`:

```java
ORCHESTRATION_STARTED,
ORCHESTRATION_COMPLETED,
AGENT_ROUTED,
AGENT_DISPATCHED,
AGENT_COMPLETED,
AGENT_FAILED,
ORCHESTRATION_ESCALATED,
```

**New stream type** on `EventStreamType`:

```java
ORCHESTRATION
```

### 5. Consumer repo migration (#644)

Two change types across 8 repos. All repos verified on `main` before any change.

**A. TrustRoutingPolicyProvider — add `id()` method:**

| Repo | Class | id() |
|---|---|---|
| devtown | `DevtownTrustRoutingPolicyProvider` | `"devtown"` |
| aml | `AmlTrustRoutingPolicyProvider` | `"aml"` |
| clinical | `ClinicalTrustRoutingPolicyProvider` | `"clinical"` |
| life | `LifeTrustRoutingPolicyProvider` | `"life"` |
| quarkmind | `QuarkMindTrustRoutingPolicyProvider` | `"quarkmind"` |
| ops | `DeploymentTrustRoutingPolicyProvider` | `"ops-deployment"` |

**B. ActionRiskClassifier — `List<String>` → `StaticSetStrategy.of(...)`:**

Every `new RiskDecision.GateRequired(reason, reversible, List.of(...), expiresIn, scope)` becomes `new RiskDecision.GateRequired(reason, reversible, StaticSetStrategy.of(...), expiresIn, scope)`. Add `import io.casehub.api.spi.routing.StaticSetStrategy`.

| Repo | Class |
|---|---|
| devtown | `DevtownActionRiskClassifier` |
| aml | `AmlActionRiskClassifier` |
| clinical | `ClinicalActionRiskClassifier` |
| life | `LifeActionRiskClassifier` |
| soc | `SocActionRiskClassifier` |
| iot | `IoTActionRiskClassifier` |

**C. AgentRoutingStrategy — add `id()` method:**

| Repo | Class | id() |
|---|---|---|
| quarkmind | `DispositionAwareRoutingStrategy` | `"quarkmind-disposition"` |

### 6. CI dispatch list (#583)

Add 6 repos that depend on engine packages to the downstream trigger in `publish.yml`:

```bash
for repo in scaffold openclaw claudony workers aml devtown life blocks soc iot clinical quarkmind ops; do
```

## Execution order

1. **Engine-api types** (#651, #650, #626 types) — foundation; everything else depends on these
2. **Repository rename** (#640) — IntelliJ refactoring across workspace
3. **CaseEventRecorder implementation** (#626 impl) — depends on renamed repos
4. **Consumer migration** (#644) — depends on engine-api changes being committed
5. **CI dispatch** (#583) — independent, can land any time

## Testing

- Contract tests for `AgentRoutingContext` (4th field), `AgentAssignment` (rationale), `CaseEventRecorder`
- Unit tests for all 3 strategy rationale strings
- Blocking/reactive parity tests for `CaseInstanceRepository` and `EventLogRepository` (following `PlanItemStoreContractTest` pattern)
- Existing test suites across consumer repos must compile and pass after migration

## Issues to file

- Naming consistency cleanup for `CaseMetaModelRepository` and `SubCaseGroupRepository` (also reactive-only, same inconsistency) — deferred to separate branch
