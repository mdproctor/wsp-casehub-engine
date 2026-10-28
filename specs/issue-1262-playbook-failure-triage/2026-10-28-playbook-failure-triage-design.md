# Structured Playbook Failure Output + RecoveryCoordinator Triage Wiring

**Branch:** issue-1262-playbook-failure-triage
**Issues:** casehubio/engine#1262, casehubio/engine#1263
**Date:** 2026-10-28

---

## 1. Overview

Two complementary changes delivered in two batches on one branch:

**Batch 1 (#1263):** PlaybookWorkerFunctionProvider — enables playbook YAML (`steps:` blocks) as worker function definitions in case definitions, compiled via `PlaybookCompiler` to SWF `Workflow` objects.

**Batch 2 (#1262):** Structured failure output from playbook/flow executions, with per-step results. Wires `TriageHandler` (platform SPI) into `DefaultRecoveryCoordinator` for intelligent post-failure recovery.

---

## 2. Batch 1 — PlaybookWorkerFunctionProvider (#1263)

### 2.1 PlaybookWorkerFunction (engine-support-core)

New record wrapping a compiled playbook. Carries the `CompilationResult` (SourceMap + CoordinationSpec) alongside the `Workflow`:

```java
public record PlaybookWorkerFunction(
    Workflow workflow,
    CompilationResult compilationResult,
    Function<Map<String, Object>, PlannedAction> plannedActionFn)
    implements WorkerFunction<Map<String, Object>, Map<String, Object>> { }
```

- `compilationResult.sourceMap()` maps SWF task positions back to playbook step names — used by step capture (batch 2) for richer error reporting.
- `compilationResult.coordinationSpec()` carries signal/await/mutex primitives for future coordination handling.

### 2.2 PlaybookWorkerFunctionProvider (engine-support-core)

Handles `steps:` blocks in YAML case definitions:

```java
public class PlaybookWorkerFunctionProvider implements WorkerFunctionProvider {
    @Override
    public boolean handles(JsonNode rawWorkerNode) {
        return rawWorkerNode.has("steps");
    }

    @Override
    public WorkerFunction<?, ?> create(JsonNode rawWorkerNode) {
        // Convert JsonNode → Map for PlaybookCompiler
        // PlaybookCompiler.compile(content) → CompilationOutcome
        // Extract CompilationResult from CompilationOutcome
        // return new PlaybookWorkerFunction(result.workflow(), result, null)
    }
}
```

### 2.3 FlowWorkerFunctionHandler update

Update `supports()` to accept both function types:

```java
@Override
public boolean supports(WorkerFunction<?, ?> function) {
    return function instanceof FlowWorkerFunction
        || function instanceof PlaybookWorkerFunction;
}
```

In `execute()`, extract the `Workflow` from whichever function type is presented. The execution path is identical — both produce a `Workflow` that `WorkflowApplication` runs.

### 2.4 Module dependency

`engine-support-core` gains a dependency on `casehub-platform-playbook-compiler` (7.x, uses `io.serverlessworkflow:serverlessworkflow-types` 7.35.3.Final — compatible with engine's 7.25.1.Final).

---

## 3. Batch 2 — Structured Failure Output + Triage Wiring (#1262)

### 3.1 PlaybookExecutionResult (engine-api)

Sealed interface for structured playbook/flow execution outcomes. Placed in `engine-api` because consumers (domain repos, CBR, triage) need to inspect results.

```java
public sealed interface PlaybookExecutionResult {
    List<StepResult> steps();

    record Success(Object output, List<StepResult> steps)
        implements PlaybookExecutionResult {}

    record PartialSuccess(Object partialOutput, List<StepResult> steps)
        implements PlaybookExecutionResult {}

    record Failure(String rootCause, List<StepResult> steps)
        implements PlaybookExecutionResult {}
}

public record StepResult(
    String stepId,
    String stepName,
    StepStatus status,
    Duration elapsed,
    String error) {}

public enum StepStatus { SUCCEEDED, FAILED, SKIPPED }
```

### 3.2 Step capture — PlaybookExecutionListener (engine-support-core)

CDI bean implementing `WorkflowExecutionListener`. Auto-discovered by Quarkus and registered with `WorkflowApplication`.

**Lifecycle:**
1. `onTaskStarted` — record step start time, store in `WorkflowInstance` metadata via `addMetadataIfAbsent`.
2. `onTaskCompleted` — record step success with elapsed time.
3. `onTaskFailed` — record step failure with cause message and elapsed time.
4. After `FlowWorkerFunctionHandler.execute()` calls `join()`, it reads the accumulated `List<StepResult>` from instance metadata.
5. If the function is a `PlaybookWorkerFunction`, use `SourceMap.locate(taskPosition)` to map SWF task positions back to playbook step names.

**Identifying casehub executions:** `FlowExecutionRegistry` already registers instances with `caseId`, `workerName`, and `inputDataHash`. The listener checks the registry to filter — only casehub-managed executions get step recording.

**Building PlaybookExecutionResult:**
- All steps succeeded → `Success(output, steps)`
- Some steps succeeded, workflow failed → `PartialSuccess(partialOutput, steps)` (partialOutput is the last successful step's output)
- First step failed or no output → `Failure(rootCause, steps)`

### 3.3 HandlerResult enrichment

`HandlerResult` gains an optional `PlaybookExecutionResult`:

```java
public record HandlerResult(
    WorkerResult<?> result,
    Map<String, Object> protocolMetadata,
    PlaybookExecutionResult executionResult) {

    public HandlerResult(WorkerResult<?> result) {
        this(result, Map.of(), null);
    }

    public HandlerResult(WorkerResult<?> result, Map<String, Object> protocolMetadata) {
        this(result, protocolMetadata, null);
    }
}
```

`FlowWorkerFunctionHandler.execute()` populates `executionResult` on both success and failure paths.

### 3.4 Failure detail propagation to RecoveryContext

`RecoveryContext` gains a `PlaybookExecutionResult` field:

```java
public record RecoveryContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    String workerName,
    String capabilityName,
    WorkerOutcome<?> outcome,
    int attemptCount,
    UUID planItemId,
    PlaybookExecutionResult executionResult) {}
```

`RetryOrchestrator.applyRetryDecision()` passes the execution result through when building `RecoveryContext` on the `Exhaust` path.

### 3.5 TriageHandler wiring in DefaultRecoveryCoordinator

**Insertion point:** After `handleFailure()` returns `false` (all recovery levels exhausted), before `WorkerRetriesExhaustedEvent` is fired.

In `RetryOrchestrator.applyRetryDecision()`, the `Exhaust` branch currently:
1. Calls `recoveryCoordinator.handleFailure(recoveryCtx)` — if true, returns
2. If false, fires `WorkerRetriesExhaustedEvent`

New flow:
1. Calls `recoveryCoordinator.handleFailure(recoveryCtx)` — if true, returns
2. **NEW:** Calls triage loop (see §3.6)
3. If triage returns `Abandon`, fires `CoordinationExhaustedEvent` then `WorkerRetriesExhaustedEvent`

**Wiring:** `DefaultRecoveryCoordinator` injects `TriageHandler` via CDI (`Instance<TriageHandler>` for optional resolution). The `NoOpTriageHandler` from `casehub-platform-governance-core` serves as the default when no custom implementation is provided.

### 3.6 Triage loop

```
loop (attempt = 1..maxTriageAttempts):
    triageContext = buildTriageContext(recoveryContext)
    decision = triageHandler.evaluate(triageContext)
    match decision:
        Retry(modifications):
            apply modifications to case context working layer
            reset retry count for this binding (triage retries are separate)
            reschedule worker via RescheduleCallback
            return true
            // The rescheduled worker will either succeed (done) or fail
            // and re-enter RetryOrchestrator. If retries exhaust again,
            // triage fires again with updated failure detail and
            // attemptNumber incremented. The triage "loop" is implicit
            // across re-entries, not a synchronous wait.
        Abandon(reason):
            fire CoordinationExhaustedEvent
            return false (proceed to WorkerRetriesExhaustedEvent)
```

**Triage attempt tracking:** The triage attempt count is persisted on `CaseRecoveryState` (or `BindingRecoveryState`) — not held in a local loop variable. Each re-entry into the triage path increments the count and checks against `maxTriageAttempts`.

**Building TriageContext from RecoveryContext:**

| TriageContext field | Source |
|---|---|
| `operationId` | `recoveryContext.caseId() + ":" + recoveryContext.bindingName()` |
| `operationType` | `"worker-execution"` |
| `attemptNumber` | triage loop counter |
| `maxAttempts` | `recoveryPolicy.maxTriageAttempts()` |
| `failureSummary` | root cause from `PlaybookExecutionResult.Failure` |
| `failureDetail` | step results serialized as Map |
| `executionContext` | case context working layer snapshot |

**maxTriageAttempts configuration:** New field on `RecoveryPolicy` in `engine-api`, default 2.

### 3.7 Playbook modification application

When `TriageDecision.Retry(modifications)` is returned, the engine interprets well-known keys:

```java
public final class PlaybookTriageKeys {
    public static final String PLAYBOOK_NAME = "playbook.name";
    public static final String FAILED_STEP_ID = "playbook.failedStepId";
    public static final String FAILED_STEP_ERROR = "playbook.failedStepError";
    public static final String MODIFICATION_TYPE = "playbook.modificationType";
    // Modification types: skip-step, replace-step, update-parameter, reorder
}
```

Modifications are applied to the case context's working layer before re-dispatch. The worker execution path reads the updated context and adjusts its behavior accordingly.

### 3.8 PlaybookOutcomeRecorder (yaml-cbr)

Extends the `StepExecutionCbrBridge` pattern. Records all playbook executions (not just triage) with step-level data:

- **On every playbook execution:** step results, timing, success/failure
- **On triage attempts:** modifications applied, result of each triage attempt
- **On triage chain completion:** final outcome, linked by `executionChainId` in CbrPlanRecord features

Listens for the same `YamlStepExecutionEvent` that `StepExecutionCbrBridge` observes, plus a new `PlaybookTriageEvent` for triage-specific recording.

### 3.9 Spring parity

Generate Spring equivalents for new types in the existing Spring module pattern:
- `PlaybookExecutionResult` — pure record, no framework dependency, shared directly
- `PlaybookTriageKeys` — constants class, shared directly
- `StepResult`, `StepStatus` — pure records/enums, shared directly
- Any CDI-specific beans (`PlaybookExecutionListener`, triage wiring) get Spring `@Component` equivalents

---

## 4. Module placement summary

| Type | Module | Rationale |
|---|---|---|
| `PlaybookExecutionResult`, `StepResult`, `StepStatus` | engine-api | Consumer-visible definition types |
| `PlaybookTriageKeys` | engine-api | Well-known keys referenced by platform TriageHandler implementations |
| `PlaybookWorkerFunction` | engine-support-core | Function type alongside FlowWorkerFunction |
| `PlaybookWorkerFunctionProvider` | engine-support-core | Provider alongside FlowWorkerFunctionProvider |
| `PlaybookExecutionListener` | engine-support-core | CDI listener for step capture |
| `RecoveryContext` (updated) | common-core | Existing location, adds executionResult field |
| `RecoveryPolicy` (updated) | engine-api | Existing location, adds maxTriageAttempts field |
| Triage wiring | runtime (DefaultRecoveryCoordinator) | Existing recovery coordinator location |
| `PlaybookOutcomeRecorder` | yaml-cbr | CBR recording alongside StepExecutionCbrBridge |

---

## 5. Key design decisions

See [decisions.md](decisions.md) for full decision records.

1. **PlaybookExecutionResult is playbook-only** — not generalized across handler types (D1)
2. **#1263 folded into this branch** — PlaybookWorkerFunction provides the type distinction needed for rich step metadata (D2)
3. **CDI WorkflowExecutionListener** for step capture — auto-discovered, works for both Flow and Playbook functions (D3)
4. **Triage fires after recovery exhaustion** — semantic triage complements structural recovery, doesn't replace it (D4)
5. **maxTriageAttempts on RecoveryPolicy** — per-case-definition granularity, consistent with existing recovery config (D5)
6. **Modify context + re-dispatch** for triage retries — reuses existing worker execution path (D6)
7. **Record all playbook executions** in CBR — richer training data, not just triage events (D7)

---

## 6. Dependencies

- **casehub-platform-playbook-compiler** (7.x) — PlaybookCompiler, CompilationResult, SourceMap, CoordinationSpec
- **casehub-platform-api** (0.4-SNAPSHOT) — TriageHandler, TriageContext, TriageDecision, CoordinationExhaustedEvent
- **casehub-platform-governance-core** (0.4-SNAPSHOT) — NoOpTriageHandler (default)
- **io.serverlessworkflow:serverlessworkflow-impl-core** (7.25.1.Final) — WorkflowExecutionListener, TaskContext, TaskFailedEvent

---

## References

- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionHandler.java:43` — current handler, execution entry point
- `common-core/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java:44` — retry/recovery orchestration, triage insertion point
- `runtime/src/main/java/io/casehub/engine/runtime/recovery/DefaultRecoveryCoordinator.java:61` — existing 3-level recovery
- `common-core/src/main/java/io/casehub/engine/common/spi/recovery/RecoveryContext.java:21` — recovery context, gains executionResult field
- `yaml-cbr/src/main/java/io/casehub/engine/yamlcbr/StepExecutionCbrBridge.java:34` — CBR recording pattern to extend
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunctionProvider.java:34` — existing provider pattern for #1263
- `engine-support-core/src/main/java/io/casehub/engine/flow/FlowWorkerFunction.java:24` — existing function type
- `casehubio/engine#1262` — structured playbook failure output + triage wiring
- `casehubio/engine#1263` — PlaybookWorkerFunctionProvider
- `casehubio/platform#668` — platform TriageHandler SPI
- `casehubio/parent#536` — parent epic
- Protocol: `api-surface-parity.md` — PlaybookExecutionResult in engine-api ensures consumer visibility
- Protocol: `plan-type-module-boundary.md` — definition types in engine-api, execution types in common/runtime
