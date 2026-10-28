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

### 2.1 SWFWorkerFunction sealed interface (engine-support-core)

Shared supertype for all function types that produce a SWF `Workflow` for `WorkflowApplication` to execute (I2):

```java
public sealed interface SWFWorkerFunction
    extends WorkerFunction<Map<String, Object>, Map<String, Object>>
    permits FlowWorkerFunction, PlaybookWorkerFunction {
    Workflow workflow();
}
```

`FlowWorkerFunction` implements `SWFWorkerFunction` (previously implemented `WorkerFunction` directly).

### 2.2 PlaybookWorkerFunction (engine-support-core)

New record wrapping a compiled playbook. Carries the `CompilationResult` (SourceMap + CoordinationSpec) alongside the `Workflow`:

```java
public record PlaybookWorkerFunction(
    Workflow workflow,
    CompilationResult compilationResult,
    Function<Map<String, Object>, PlannedAction> plannedActionFn)
    implements SWFWorkerFunction { }
```

- `compilationResult.sourceMap()` maps SWF task positions back to playbook step names — used by step capture (batch 2) for richer error reporting.
- `compilationResult.coordinationSpec()` carries signal/await/mutex primitives for future coordination handling.

### 2.3 PlaybookWorkerFunctionProvider (engine-support-core)

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

### 2.4 FlowWorkerFunctionHandler update

`supports()` becomes a single type check via the sealed interface:

```java
@Override
public boolean supports(WorkerFunction<?, ?> function) {
    return function instanceof SWFWorkerFunction;
}
```

In `execute()`, access the workflow via `((SWFWorkerFunction) function).workflow()`. The execution path is identical for both function types.

### 2.5 Version compatibility

`engine-support-core` gains a dependency on `casehub-platform-playbook-compiler` (7.x, uses `io.serverlessworkflow:serverlessworkflow-types` 7.35.3.Final — compatible with engine's 7.25.1.Final). A compatibility test (compile with 7.35 types, execute with 7.25 runtime) is required before merging batch 1. Engine should align to the compiler's SWF types version (7.35) or the compiler should target the engine's version — decision during implementation.

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

CDI bean implementing `WorkflowExecutionListener`. Auto-discovered by Quarkus via `quarkus-flow`'s `WorkflowApplicationCreator.injectCustomListeners()`, which discovers `@ApplicationScoped` beans implementing `WorkflowExecutionListener` via `Instance<WorkflowExecutionListener>` injection.

**Step data propagation (I1):** Step results accumulate in `FlowExecutionRegistry` — not in SWF `WorkflowInstance` metadata. `FlowExecutionRegistry` already maps instance IDs to `FlowExecution` context (caseId, workerName, inputDataHash). Step data is stored in a **separate** `ConcurrentHashMap<String, List<StepResult>>` in `FlowExecutionRegistry`, keyed by instance ID — `FlowExecution` remains an immutable record. This avoids coupling to SWF SDK internals and eliminates the need to cast `WorkflowContextData` to reach `WorkflowMutableInstance.addMetadataIfAbsent()`.

**Lifecycle:**
1. `onTaskStarted` — look up execution in `FlowExecutionRegistry` by instance ID. If found (casehub execution), record step start time.
2. `onTaskCompleted` — record step success with elapsed time.
3. `onTaskFailed` — record step failure with cause message and elapsed time.
4. After `FlowWorkerFunctionHandler.execute()` calls `join()`, it reads the accumulated `List<StepResult>` from `FlowExecutionRegistry`.
5. If the function is a `PlaybookWorkerFunction`, use `SourceMap.locate(taskPosition)` to map SWF task positions back to playbook step names.

**Identifying casehub executions:** The listener uses a new `FlowExecutionRegistry.find(instanceId)` method (returns `Optional<FlowExecution>`) — the existing `get()` throws `IllegalStateException` on miss. Non-casehub executions return empty — O(1) ConcurrentHashMap lookup, no wasted work.

**Building PlaybookExecutionResult:**
- All steps succeeded → `Success(output, steps)`
- Some steps succeeded, workflow failed → `PartialSuccess(partialOutput, steps)` (partialOutput is the last successful step's output)
- First step failed or no output → `Failure(rootCause, steps)`

### 3.3 Failure propagation mechanism

**Problem:** When a workflow fails, `wfInstance.start().join()` throws. The exception propagates through `WorkerExecutionOrchestrator` → `RetryHandler.handleFailure(taskData, Throwable, message)` → `RetryOrchestrator`. There is no parameter for `PlaybookExecutionResult` on this path.

**Solution:** `FlowWorkerFunctionHandler.execute()` catches workflow exceptions, builds `PlaybookExecutionResult` from step data accumulated in `FlowExecutionRegistry`, and wraps both in a `PlaybookExecutionException` (new type in engine-support-core):

```java
public class PlaybookExecutionException extends RuntimeException {
    private final PlaybookExecutionResult executionResult;
    // constructor, getter
}
```

On the success path, `HandlerResult` gains an optional `PlaybookExecutionResult`:

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

On the failure path, `RetryOrchestrator` extracts the `PlaybookExecutionResult` from the `Throwable` if it is a `PlaybookExecutionException`. This minimizes blast radius — no interface changes to `RetryHandler` or its callers (Quartz, db-scheduler backends).

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

`RetryOrchestrator.applyRetryDecision()` extracts `PlaybookExecutionResult` from `PlaybookExecutionException` (if present) and passes it through when building `RecoveryContext` on the `Exhaust` path. Non-playbook handler failures pass `null`.

### 3.5 TriageHandler wiring in DefaultRecoveryCoordinator

**Insertion point:** Inside `DefaultRecoveryCoordinator.handleFailure()`, after all three structural recovery levels (TRANSIENT → REASONING → FUNDAMENTAL) have been tried and failed. The coordinator's boolean return value now means "recovered via either structural recovery or triage."

`RetryOrchestrator` calls `recoveryCoordinator.handleFailure()` unchanged — no module boundary changes needed. Triage is an internal concern of `DefaultRecoveryCoordinator`, not visible to the retry orchestration layer.

**Wiring:** `DefaultRecoveryCoordinator` injects `TriageHandler` via CDI (`Instance<TriageHandler>` for optional resolution). The `NoOpTriageHandler` from `casehub-platform-governance-core` serves as the default when no custom implementation is provided.

**Per-binding triage overrides:** `RecoveryOverride` gains `skipTriage` (default false) and `maxTriageAttempts` (nullable, falls through to `RecoveryPolicy` default). `skipRecovery=true` skips structural recovery but does NOT skip triage unless `skipTriage=true` independently — a binding with `skipRecovery=true, skipTriage=false` goes directly to triage after mechanical retries exhaust.

**handleFailure restructuring:** The existing `skipRecovery` check returns `false` immediately (line ~108 of DefaultRecoveryCoordinator), which exits before reaching any triage code. The method must be restructured: `skipRecovery` now skips only the structural recovery levels (TRANSIENT/REASONING/FUNDAMENTAL) and falls through to the triage section. The new flow is: (1) check skipRecovery → if true, skip to triage section; (2) classify and escalate through levels; (3) if all levels fail (or skipped), check skipTriage → if false, invoke triage loop.

### 3.6 Triage flow

The triage "loop" is implicit across re-entries through the execution pipeline, not a synchronous wait:

```
// Inside DefaultRecoveryCoordinator.handleFailure(), after structural recovery fails:
if (triageHandler not resolvable || skipTriage): return false
if (bindingState.triageAttemptCount >= maxTriageAttempts): return false

triageContext = buildTriageContext(recoveryContext, bindingState.previousTriageModifications)
decision = triageHandler.evaluate(triageContext)

match decision:
    Retry(modifications):
        apply typed modifications to case context working layer
        record modifications in bindingState.previousTriageModifications
        increment bindingState.triageAttemptCount
        publish CaseContextChangedEvent
        // CaseContextChangedEventHandler.evaluateAndDispatch() re-evaluates
        // all bindings and dispatches workers for satisfied bindings.
        // Same pattern as Level 3 recovery.
        return true

    Abandon(reason):
        fire CoordinationExhaustedEvent
        return false
```

**Re-dispatch mechanism:** The coordinator publishes `CaseContextChangedEvent` to the event bus — the same event-driven pattern used by Level 3 recovery (line 270 of DefaultRecoveryCoordinator). `CaseContextChangedEventHandler.evaluateAndDispatch()` evaluates all bindings against the modified context and dispatches workers for satisfied bindings.

**Retry multiplication:** With `maxAttempts=N` and `maxTriageAttempts=T`, total worker executions = `N × (1 + T)`. Example: `maxAttempts=3, maxTriageAttempts=2` → 9 executions. Intentional — triage-modified executions still need mechanical retries for transient failures orthogonal to the semantic modification.

**Building TriageContext from RecoveryContext:**

| TriageContext field | Source |
|---|---|
| `operationId` | `recoveryContext.caseId() + ":" + recoveryContext.bindingName()` |
| `operationType` | `"worker-execution"` |
| `attemptNumber` | `bindingState.triageAttemptCount` |
| `maxAttempts` | `recoveryPolicy.maxTriageAttempts()` (or binding override) |
| `failureSummary` | root cause from `PlaybookExecutionResult.Failure` |
| `failureDetail` | step results serialized as Map |
| `executionContext` | case context working layer snapshot + `previousTriageModifications` |

**maxTriageAttempts configuration:** New field on `RecoveryPolicy` in `engine-api`, default 2. Per-binding override on `RecoveryOverride`.

### 3.7 Triage modifications — Map convention with engine-side typed conversion

**Platform SPI reality:** `TriageDecision.Retry` carries `Map<String, Object> modifications` — an untyped map. `TriageModification` does not exist in platform-api. The typed hierarchy lives entirely in the engine.

**Convention:** `TriageHandler` implementations populate the map with well-known keys. The engine deserializes map entries into a sealed `PlaybookTriageModification` hierarchy:

```java
// engine-support-core — sealed within one compilation unit
public sealed interface PlaybookTriageModification
    permits SkipStep, ReplaceStepInput, UpdateParameter {
    record SkipStep(String stepId) implements PlaybookTriageModification {}
    record ReplaceStepInput(String stepId, Map<String, Object> newInput)
        implements PlaybookTriageModification {}
    record UpdateParameter(String key, Object value)
        implements PlaybookTriageModification {}
}
```

**Map key convention:** `TriageHandler` implementations populate `modifications` with entries like:
```
"playbook.skipSteps" → ["step-3"]
"playbook.replaceInput.step-2" → { "url": "/api/v2/contacts" }
"playbook.updateParams.timeout" → 30000
```

`PlaybookTriageModificationParser` (engine-support-core) converts map entries → `List<PlaybookTriageModification>` using the `playbook.*` namespace. Unknown keys are ignored (forward-compatible).

**How modifications are applied during re-execution:** Modifications are written to a `triage` namespace in the case context's working layer (e.g., `triage.skipSteps`, `triage.replaceInput.*`). When `FlowWorkerFunctionHandler.execute()` runs, the `PlaybookExecutionListener` reads triage directives from the case context:
- `SkipStep` — listener intercepts `onTaskStarted` for the matching step ID and marks it skipped (no execution)
- `ReplaceStepInput` — handler modifies the workflow input data for the matching step before execution
- `UpdateParameter` — handler modifies the workflow-level input data

**Re-dispatch scope:** `CaseContextChangedEvent` triggers `evaluateAndDispatch()` for ALL bindings. For triage, only the binding under triage has modifications in its context. Other bindings evaluate normally — if their conditions aren't affected by the triage context changes, they won't re-dispatch. This is the same behavior as Level 3 recovery.

**Modification history:** `BindingRecoveryState` tracks `List<Map<String, Object>> previousTriageModifications`. Each entry is the raw modification map from one triage attempt. `TriageContext.executionContext` includes this history so the handler sees what was already tried.

### 3.7.1 Retry count partitioning

**Problem:** `RetryOrchestrator.countFailedAttempts()` counts ALL `WORKER_EXECUTION_FAILED` event logs for the worker + inputDataHash. After triage modifies context and re-dispatches, the previous failure count carries over — the re-dispatched worker may immediately exhaust.

**Solution:** Include a `triageRound` discriminator in event log metadata. The first execution round (pre-triage) is round 0. After each triage attempt, the round increments. `countFailedAttempts()` filters by the current triage round. `RetryOrchestrator` receives the current `triageRound` from `BindingRecoveryState` (accessed via `RecoveryContext`).

### 3.8 PlaybookOutcomeRecorder (yaml-cbr)

Dual-granularity CBR recording alongside `StepExecutionCbrBridge`:

- **StepExecutionCbrBridge** (existing) — per-action granularity. Records individual `YamlStepExecutionEvent` outcomes. One `CbrPlanRecord` per action with `actionName`, `durationMs`, `bindingType`, `success`, and domain-specific features. Answers: "should step X succeed in this context?"
- **PlaybookOutcomeRecorder** (new) — per-execution granularity. Records full execution arcs: all steps in order, which succeeded/failed, overall outcome, triage chain context. Answers: "have we seen this failure pattern before, and what triage modifications resolved it?"

Triage-specific data:
- Modifications applied per attempt
- Result of each triage attempt
- Final outcome (success after triage, or abandon)
- Linked by `executionChainId` in CbrPlanRecord features

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
| `PlaybookExecutionException` | engine-support-core | Carries PlaybookExecutionResult on failure path |
| `PlaybookTriageModificationParser` | engine-support-core | Converts Map entries to typed modifications |
| `SWFWorkerFunction` | engine-support-core | Sealed interface for workflow-based functions (I2) |
| `PlaybookWorkerFunction` | engine-support-core | Function type alongside FlowWorkerFunction |
| `PlaybookWorkerFunctionProvider` | engine-support-core | Provider alongside FlowWorkerFunctionProvider |
| `PlaybookTriageModification` | engine-support-core | Sealed sub-hierarchy for playbook modifications |
| `PlaybookExecutionListener` | engine-support-core | CDI listener for step capture |
| `RecoveryContext` (updated) | common-core | Existing location, adds executionResult field |
| `RecoveryPolicy` (updated) | engine-api | Existing location, adds maxTriageAttempts field |
| `RecoveryOverride` (updated) | engine-api | Existing location, adds skipTriage + maxTriageAttempts |
| Triage wiring | runtime (DefaultRecoveryCoordinator) | Existing recovery coordinator location |
| `PlaybookOutcomeRecorder` | yaml-cbr | CBR recording alongside StepExecutionCbrBridge |

---

## 5. Key design decisions

See [decisions.md](decisions.md) for full decision records.

1. **PlaybookExecutionResult is playbook-only** — not generalized across handler types (D1)
2. **#1263 folded into this branch** — PlaybookWorkerFunction provides the type distinction needed for rich step metadata (D2)
3. **CDI WorkflowExecutionListener + FlowExecutionRegistry** for step capture — accumulates in engine-owned data structure, no SWF SDK coupling (D3, I1)
4. **Triage inside DefaultRecoveryCoordinator** — after structural recovery levels exhaust, re-dispatch via CaseContextChangedEvent (D4)
5. **maxTriageAttempts on RecoveryPolicy + per-binding overrides** — consistent with existing recovery config pattern (D5)
6. **Map-convention modifications with engine-side typed conversion** — platform SPI uses `Map<String, Object>`, engine deserializes to sealed `PlaybookTriageModification` hierarchy via `playbook.*` key namespace. Modification history tracked on `BindingRecoveryState` (D6)
7. **Dual-granularity CBR recording** — per-action (StepExecutionCbrBridge) + per-execution (PlaybookOutcomeRecorder) (D7)
8. **Sealed SWFWorkerFunction** — shared supertype for FlowWorkerFunction and PlaybookWorkerFunction (I2)

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
