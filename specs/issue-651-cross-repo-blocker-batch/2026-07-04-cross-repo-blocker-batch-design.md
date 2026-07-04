# Cross-Repo Blocker Batch — Design Spec

**Date:** 2026-07-04
**Issues:** #651, #650, #583, #644, #640, #626
**Branch:** `issue-651-cross-repo-blocker-batch`

---

## Problem

Six issues block downstream consumers. Each is a gap in engine-api's boundary types or propagation:

- Routing context missing tenancyId (#651)
- Routing result missing rationale (#650)
- Repository SPIs missing blocking variants, naming inconsistent (#640)
- Case lifecycle missing event write path for external consumers (#626)
- Consumer implementations broken after SPI evolution (#644)
- CI not triggering downstream builds for new repos (#583)

## Changes

### 1. AgentRoutingContext — add tenancyId (#651)

Add `String tenancyId` as the 4th field:

```java
public record AgentRoutingContext(
    UUID caseId, String capabilityName, JsonNode caseContext, String tenancyId) {}
```

**Production construction sites** (2):
- `CaseContextChangedEventHandler.publishWorkerSchedule()` — pass `caseInstance.tenancyId`
- `DefaultWorkOrchestrator.doSubmit()` — pass `instance.tenancyId`

Strategy implementations receive tenancyId via the context parameter. No signature changes to `AgentRoutingStrategy.select()`.

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

**Rationale propagation:** `TrustCandidateClassifier.ScoredCandidate` gains a `String rationale` field. Each strategy sets the rationale when constructing `ScoredCandidate`. `TrustCandidateClassifier.decide()` uses `best.rationale()` for `Assigned` outcomes and generates its own for non-Assigned outcomes.

**Assigned rationale strings (per strategy, per phase):**
- `LeastLoadedAgentStrategy`: `"selected %s: load %d (vs next: %s load %d)"` — 2+ candidates; `"selected %s: load %d (sole candidate)"` — 1 candidate
- `TrustWeightedAgentStrategy` QUALIFIED: `"selected %s: trust %.2f, blended %.2f (threshold %.2f)"`
- `TrustWeightedAgentStrategy` BOOTSTRAP: `"selected %s: availability %.2f (bootstrap)"`
- `SemanticAgentRoutingStrategy` QUALIFIED: `"selected %s: semantic %.2f, trust %.2f, blended %.2f"`
- `SemanticAgentRoutingStrategy` BOOTSTRAP: `"selected %s: availability %.2f (bootstrap)"`
- `DispositionAwareRoutingStrategy` QUALIFIED: `"selected %s: trust %.2f, blended %.2f × disposition %.2f = %.2f"`
- `DispositionAwareRoutingStrategy` BOOTSTRAP: `"selected %s: availability %.2f × disposition %.2f = %.2f"`

**Unresolvable rationale strings:**
- Empty candidates (all strategies): `"no candidates available"`
- All excluded — `TrustCandidateClassifier.decide()`: `"all candidates excluded for capability '%s'"`

**EscalateToOversight rationale strings:**
- Bootstrap guard — `TrustWeightedAgentStrategy`/`SemanticAgentRoutingStrategy`/`DispositionAwareRoutingStrategy`: `"bootstrap only — no qualified agents for capability '%s'"`
- Borderline stalemate — `TrustCandidateClassifier.decide()`: `"all candidates borderline for capability '%s' — oversight required"`

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

**Blocking CrossTenantCaseInstanceRepository:**

```java
public interface CrossTenantCaseInstanceRepository {
  CaseInstance findByUuid(UUID caseId);
}
```

**Blocking CrossTenantEventLogRepository:**

```java
public interface CrossTenantEventLogRepository {
  List<EventLog> findByTypes(Collection<CaseHubEventType> types);
  List<EventLog> findByCaseAndTypes(UUID caseId, Collection<CaseHubEventType> types);
  List<String> findSubmittedWorkWithoutCompletion();
  List<EventLog> findByWorkerAndTypeAcrossTenants(String workerId, CaseHubEventType type);
  EventLog findById(Long id);
  List<EventLog> findByCaseAndWorkerAndType(UUID caseId, String workerId, CaseHubEventType type);
}
```

**Implementations (two classes per persistence layer — a single class cannot implement both interfaces because methods differ only in return type):**

| Module | Blocking class | Reactive class | Canonical |
|--------|---------------|----------------|-----------|
| persistence-memory | `InMemoryCaseInstanceRepository` | `InMemoryReactiveCaseInstanceRepository` | Blocking; reactive injects blocking delegate, wraps with `Uni.createFrom().item(...)` |
| persistence-memory | `InMemoryEventLogRepository` | `InMemoryReactiveEventLogRepository` | Blocking; reactive injects blocking delegate |
| persistence-memory | `InMemoryCrossTenantCaseInstanceRepository` | `InMemoryReactiveCrossTenantCaseInstanceRepository` | Blocking; reactive wraps |
| persistence-memory | `InMemoryCrossTenantEventLogRepository` | `InMemoryReactiveCrossTenantEventLogRepository` | Blocking; reactive wraps |
| persistence-jpa | `JpaCaseInstanceRepository` | `JpaReactiveCaseInstanceRepository` | Reactive (Panache); blocking injects reactive delegate, awaits via `.await().indefinitely()` |
| persistence-jpa | `JpaEventLogRepository` | `JpaReactiveEventLogRepository` | Reactive (Panache); blocking injects reactive delegate |
| persistence-jpa | `JpaCrossTenantCaseInstanceRepository` | `JpaReactiveCrossTenantCaseInstanceRepository` | Reactive (Panache); blocking awaits |
| persistence-jpa | `JpaCrossTenantEventLogRepository` | `JpaReactiveCrossTenantEventLogRepository` | Reactive (Panache); blocking awaits |
| testing | `TestCaseInstanceRepository` | `TestReactiveCaseInstanceRepository` | Blocking; reactive wraps |
| testing | `TestEventLogRepository` | `TestReactiveEventLogRepository` | Blocking; reactive wraps |

**Rename execution:** IntelliJ semantic refactoring across the open workspace (26 repos). All import sites, injection points, `quarkus.arc.selected-alternatives`, and `quarkus.arc.exclude-types` entries update automatically.

### 4. CaseEventRecorder — event write SPI (#626)

**Request record** in `api/spi/`:

```java
public record CaseEventRequest(
    UUID caseId, CaseHubEventType type, EventStreamType stream,
    String workerId, String tenancyId, JsonNode payload, JsonNode metadata) {}
```

tenancyId is explicit — consistent with every other SPI in the codebase (no implicit tenant from CDI scope).

**Blocking interface** in `api/spi/`:

```java
public interface CaseEventRecorder {
  void record(CaseEventRequest request);
  Long recordAndReturnId(CaseEventRequest request);
}
```

**Reactive interface** in `api/spi/`:

```java
public interface ReactiveCaseEventRecorder {
  Uni<Void> record(CaseEventRequest request);
  Uni<Long> recordAndReturnId(CaseEventRequest request);
}
```

Follows the `PlanItemStore` / `ReactivePlanItemStore` dual-stack convention: unqualified = blocking, `Reactive` prefix = Uni-based.

**Engine implementation:** `DefaultReactiveCaseEventRecorder` in `runtime/internal/engine/` (`@ApplicationScoped`) — reactive is canonical. Injects `ReactiveEventLogRepository`. Constructs `EventLog` domain objects internally — consumers never import `EventLog`. `DefaultCaseEventRecorder` in `runtime/internal/engine/` — blocking delegates via `.await().indefinitely()`.

**No-op defaults:** `NoOpCaseEventRecorder` and `NoOpReactiveCaseEventRecorder` in `api/` (`@DefaultBean @ApplicationScoped`). Active when `engine-runtime` is not on the classpath (e.g., standalone worker deployments). Follows the `NoOpPlanItemStore` / `NoOpReactivePlanItemStore` pattern — the no-ops live in a module that consumers always depend on, separate from the real implementations.

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

**Event type semantic layers** — AGENT_* and ORCHESTRATION_* operate at the routing/orchestration layer, distinct from existing WORKER_* events at the execution layer:

| Layer | Events | Fires when |
|-------|--------|------------|
| Orchestration lifecycle | `ORCHESTRATION_STARTED`, `ORCHESTRATION_COMPLETED`, `ORCHESTRATION_ESCALATED` | WorkOrchestrator begins/ends an orchestration cycle, or escalates to human oversight |
| Routing decision | `AGENT_ROUTED` | AgentRoutingStrategy selects an agent — captures strategy id, selected worker, rationale |
| Agent dispatch | `AGENT_DISPATCHED`, `AGENT_COMPLETED`, `AGENT_FAILED` | Orchestration layer dispatches/acknowledges agent work — one dispatch may involve multiple worker retries |
| Execution | `WORKER_SCHEDULED`, `WORKER_EXECUTION_*` | Engine Quartz layer creates/runs/completes a job — mechanical execution lifecycle |

Key distinction: `AGENT_DISPATCHED` is the orchestration intent to assign work to a selected agent. `WORKER_SCHEDULED` is the mechanical Quartz job creation. One agent dispatch may produce multiple `WORKER_SCHEDULED` events (retries, re-scheduling). `AGENT_COMPLETED` fires when the orchestration layer acknowledges completion, regardless of how many worker attempts occurred.

The event types are defined in engine-api so consumers can subscribe/filter. The firing code paths are in engine-runtime and will be wired into the orchestration handlers as part of the WorkOrchestrator implementation.

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

**Direct construction sites (devtown, clinical, life, soc, iot):** Every `new RiskDecision.GateRequired(reason, reversible, List.of(...), expiresIn, scope)` becomes `new RiskDecision.GateRequired(reason, reversible, StaticSetStrategy.of(...), expiresIn, scope)`. Add `import io.casehub.api.spi.routing.StaticSetStrategy`.

| Repo | Class |
|---|---|
| devtown | `DevtownActionRiskClassifier` |
| clinical | `ClinicalActionRiskClassifier` |
| life | `LifeActionRiskClassifier` |
| soc | `SocActionRiskClassifier` |
| iot | `IoTActionRiskClassifier` |

**AML (indirect via enum):** AML delegates through `AmlActionType.candidateGroups()` — the `List.of(...)` values are in the `AmlActionType` enum, not in the classifier. Migration:
1. `AmlActionType` enum field type: `List<String>` → `CandidateSetStrategy`
2. All enum constants: `List.of(AmlGroups.MLRO)` → `StaticSetStrategy.of(AmlGroups.MLRO)`, etc.
3. `candidateGroups()` return type: `List<String>` → `CandidateSetStrategy`
4. `AmlActionRiskClassifier.gate()` and `missingContext()` — no change needed (already passes `type.candidateGroups()` to `GateRequired`)

**C. quarkmind `DispositionAwareRoutingStrategy` — full migration:**

`id()` method:

| Repo | Class | id() |
|---|---|---|
| quarkmind | `DispositionAwareRoutingStrategy` | `"quarkmind-disposition"` |

`AgentAssignment` factory method signature changes (3 production sites):
1. `AgentAssignment.unresolvable()` (line 77) → `AgentAssignment.unresolvable("no candidates available")`
2. `AgentAssignment.escalate(capability, EscalationReason.NO_QUALIFIED_AGENT)` (line 91) → `AgentAssignment.escalate(capability, EscalationReason.NO_QUALIFIED_AGENT, "bootstrap only — no qualified agents for capability '%s'".formatted(capability))`
3. `new ScoredCandidate(cc, trustScore * multiplier)` (line 110) → `new ScoredCandidate(cc, trustScore * multiplier, rationale)` — rationale per phase (see §2)

`AgentRoutingContext` 4th field (test sites — 8+ constructors):
All `new AgentRoutingContext(caseId, capability, context)` in `DispositionAwareRoutingStrategyTest` → `new AgentRoutingContext(caseId, capability, context, tenancyId)`

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
- Unit tests for all strategy rationale strings (Assigned, Unresolvable, EscalateToOversight — all phases)
- Blocking/reactive parity tests for `CaseInstanceRepository` and `EventLogRepository` (following `PlanItemStoreContractTest` pattern)
- Existing test suites across consumer repos must compile and pass after migration

## Issues to file

- Naming consistency cleanup for `CaseMetaModelRepository` and `SubCaseGroupRepository` (also reactive-only, same inconsistency) — deferred to separate branch
