# Apply Triage Modifications to Playbook Re-Execution

**Issue:** casehubio/engine#1264
**Date:** 2026-10-30
**Status:** Draft

## Problem

When `DefaultRecoveryCoordinator` triggers a triage retry, `TriageDecision.Retry.modifications()` is recorded in `BindingRecoveryState` and a `CaseContextChangedEvent` re-dispatches the worker. But nothing reads the recorded modifications and applies them to the playbook before re-execution. The worker runs the identical playbook and produces the same failure.

## Solution Overview

Apply triage modifications at the handler level in `FlowWorkerFunctionHandler.execute()`, just before the workflow instance is created. A thin SPI in `common-core` bridges the module boundary between the handler (`engine-support-core`) and the recovery state registry (`runtime-core`).

Three components:

1. **`TriageModificationProvider`** — SPI in `common-core` for retrieving raw modifications by caseId + bindingName
2. **`PlaybookTriageModificationParser.mergeAll()`** — cumulative merge of modifications across triage rounds
3. **`PlaybookTriageModificationApplicator`** — transforms a compiled `Workflow` + merged modifications into a modified `Workflow` copy

## Component Design

### TriageModificationProvider (common-core)

```java
package io.casehub.engine.common.spi.recovery;

public interface TriageModificationProvider {
    List<Map<String, Object>> getModifications(UUID caseId, String bindingName);
}
```

Returns the raw modification maps from `BindingRecoveryState.previousTriageModifications()`. Raw `Map<String, Object>` keeps the interface in `common-core` without depending on `PlaybookTriageModification` (which is in `engine-support-core`).

### DefaultTriageModificationProvider (runtime-core)

```java
@ApplicationScoped
public class DefaultTriageModificationProvider implements TriageModificationProvider {
    @Inject CaseRecoveryStateRegistry stateRegistry;

    public List<Map<String, Object>> getModifications(UUID caseId, String bindingName) {
        CaseRecoveryState caseState = stateRegistry.getOrCreate(caseId);
        BindingRecoveryState bindingState = caseState.find(bindingName);
        if (bindingState == null) return List.of();
        return bindingState.previousTriageModifications();
    }
}
```

### PlaybookTriageModificationParser.mergeAll() (engine-support-core)

New static method on the existing parser. Takes the list of per-round modification maps and produces a single merged list with cumulative semantics:

- **SkipStep:** union — all skipped steps from all rounds accumulate
- **ReplaceStepInput:** last-writer-wins per stepId — later rounds override earlier ones
- **UpdateParameter:** last-writer-wins per key — later rounds override earlier ones

### PlaybookTriageModificationApplicator (engine-support-core)

New class. Takes a `Workflow` and a merged `List<PlaybookTriageModification>`, returns a `ModificationResult`:

```java
public record ModificationResult(Workflow workflow, Map<String, Object> parameterOverrides) {}
```

**SkipStep(stepId):** Recursively traverse the Workflow task tree. Find the `TaskItem` whose `getName()` matches `stepId`. Replace it with a new `TaskItem` of the same name containing an empty `SetTask` (no-op). Preserves position in the flow.

**ReplaceStepInput(stepId, newInput):** Find the matching `TaskItem`. Wrap it in a `DoTask` block where the first task is a `SetTask` that injects the `newInput` map into the workflow model, followed by the original task.

**UpdateParameter(key, value):** Collected into `parameterOverrides` on `ModificationResult`. The handler merges these into the workflow input map before creating the instance. Not applied to the Workflow structure itself.

**Recursive traversal:** Covers `DoTask.getDo()`, `TryTask.getTry()`, `ForEachTask.getDo()`, `SwitchTask` branches, and `ParallelTask` branches. Bounded by the compiler's own step types.

**Unknown stepId:** Gracefully ignored — no crash, log a WARN so triage handlers can debug misconfigurations. The triage handler may reference steps that were already removed by a prior SkipStep or that don't exist in this playbook version.

**No mutation:** The original `Workflow` is never modified. A new `Workflow` is constructed with the modified task list.

### FlowWorkerFunctionHandler Integration (engine-support-core)

New constructor dependency: `Instance<TriageModificationProvider>` (optional CDI).

In `execute()`, between constructing `mapInput` and creating the workflow instance:

1. Guard: only for `PlaybookWorkerFunction`, only when provider is resolvable, only when `metadata.bindingName()` is non-null
2. Query provider for raw modifications
3. If non-empty: `mergeAll()` → `apply()` → use `effectiveWorkflow` and `effectiveInput`
4. Otherwise: original workflow and input pass through unchanged

Non-playbook execution (`FlowWorkerFunction`, `McpWorkerFunction`, `ReActWorkerFunction`, etc.) is completely untouched.

## Cumulative Modification Semantics

Modifications from multiple triage rounds accumulate. `BindingRecoveryState.previousTriageModifications()` stores one `Map<String, Object>` per round. `mergeAll()` processes them in order (round 1 first, round N last).

Example with two rounds:
- Round 1: `{playbook.skipSteps: ["validate_input"], playbook.updateParams.retryMode: "lenient"}`
- Round 2: `{playbook.skipSteps: ["check_permissions"], playbook.replaceInput.fetch_data: {source: "backup"}}`

Merged result:
- SkipStep("validate_input"), SkipStep("check_permissions") — union
- ReplaceStepInput("fetch_data", {source: "backup"}) — from round 2
- UpdateParameter("retryMode", "lenient") — from round 1

## Module Boundary Summary

| Artifact | Module | Purpose |
|----------|--------|---------|
| `TriageModificationProvider` | common-core | SPI interface |
| `DefaultTriageModificationProvider` | runtime-core | Implementation backed by `CaseRecoveryStateRegistry` |
| `PlaybookTriageModificationParser.mergeAll()` | engine-support-core | Cumulative merge logic |
| `PlaybookTriageModificationApplicator` | engine-support-core | Workflow tree transformation |
| `FlowWorkerFunctionHandler` (modified) | engine-support-core | Integration point |

No cross-repo changes. All new code is within casehub-engine.

## Test Plan

### Unit Tests

1. **`PlaybookTriageModificationParserTest` (extend):**
   - `mergeAll` with empty list → empty result
   - `mergeAll` with single round → same as `parse`
   - `mergeAll` cumulative SkipSteps across rounds → union
   - `mergeAll` ReplaceStepInput last-writer-wins per stepId
   - `mergeAll` UpdateParameter last-writer-wins per key

2. **`PlaybookTriageModificationApplicatorTest` (new):**
   - SkipStep on top-level step → replaced with no-op SetTask
   - SkipStep on nested step (inside DoTask block) → found and replaced
   - SkipStep on step inside TryTask → found and replaced
   - ReplaceStepInput → input injection wrapper created around target step
   - UpdateParameter → collected in parameterOverrides, Workflow unchanged
   - Unknown stepId → gracefully ignored, Workflow unchanged
   - Multiple modifications applied together
   - Empty modifications → Workflow returned as-is

3. **`FlowWorkerFunctionHandlerTest` (extend):**
   - No modifications → workflow executes unchanged
   - Modifications exist → modified workflow used, original untouched
   - Provider not resolvable → no modification attempted
   - Non-playbook function (FlowWorkerFunction) → no modification attempted
   - bindingName is null → no modification attempted

4. **`DefaultTriageModificationProviderTest` (new):**
   - Returns modifications for known binding
   - Returns empty list for unknown binding
   - Returns empty list for known case with no triage state

### Integration Test

Full triage loop (in `runtime` module): worker fails → `DefaultRecoveryCoordinator` triggers triage → `TriageHandler` returns `Retry` with `SkipStep` modification → `CaseContextChangedEvent` re-dispatches → `FlowWorkerFunctionHandler` applies modification → playbook executes with step skipped → success.

## Acceptance Criteria Mapping

| Criterion | Implementation |
|-----------|---------------|
| Worker re-execution applies modifications from BindingRecoveryState | FlowWorkerFunctionHandler queries TriageModificationProvider, applies via Applicator |
| SkipStep prevents step from executing | Applicator replaces TaskItem with no-op SetTask |
| ReplaceStepInput substitutes step input | Applicator wraps TaskItem with SetTask input injection |
| UpdateParameter modifies playbook parameters | Handler merges parameterOverrides into mapInput |
| Integration test: fail → triage → re-dispatch → modified execution | Full-loop test in runtime module |
| Non-playbook execution unaffected | instanceof PlaybookWorkerFunction guard |

## References

- `FlowWorkerFunctionHandler.java:46-130` — handler being modified
- `PlaybookTriageModification.java:20-31` — sealed modification hierarchy
- `PlaybookTriageModificationParser.java:22-56` — existing parser, extended with mergeAll
- `BindingRecoveryState.java:23-70` — stores raw modifications per binding
- `CaseRecoveryStateRegistry.java:22-33` — case-level registry (runtime-core)
- `DefaultRecoveryCoordinator.java:291-346` — attemptTriage records modifications and publishes CaseContextChangedEvent
- `PlaybookWorkerFunction.java:25-53` — carries CompilationResult with SourceMap
- `CompilationResult.java:5` — record(Workflow, CoordinationSpec, SourceMap)
- `PlaybookCompiler.java:31-247` — compiler producing the Workflow task tree
- `WorkerExecutionOrchestrator.java:278-289` — constructs ExecutionMetadata with bindingName
- `ExecutionMetadata.java:27-45` — already carries bindingName
- Issue casehubio/engine#1264
- Issue casehubio/engine#1262 — triage infrastructure (prerequisite, already landed)
