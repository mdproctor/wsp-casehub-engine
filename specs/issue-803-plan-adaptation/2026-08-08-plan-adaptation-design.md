# Plan Adaptation — Revise Active Plans Based on New Observations

**Issue:** engine#803
**Epic:** engine#800 (Sub-epic B — Agent Reflection & Planning)
**Depends on:** engine#802 (hierarchical planning — landed)

## Problem

`GoalDecomposer` decomposes agent goals into `DagPlan<LeafTask>` at case start
via `LlmDecompositionStrategy`. Plans are materialized as compound
`PlanItemDefinition`s and executed by the existing planning dispatch loop
(CompoundStrategyDispatcher + CHOREOGRAPHED). But once materialized, plans are
static — new observations (worker outcomes, context changes, reflection insights)
may invalidate assumptions the plan was based on. Agents need to detect when
their current plan is no longer valid and revise it, keeping completed steps and
replacing pending ones.

## Design Principles

1. **Orthogonal composable foundations.** Trigger evaluation (should we replan?)
   and plan revision (what should the new plan be?) are genuinely independent
   dimensions. Separate SPIs, separately configurable, independently testable.
2. **DSL wrappers for common presets.** YAML supports both explicit two-field
   config and string shorthands for common combinations.
3. **Worker-outcome driven.** v1 triggers after worker completions within
   decomposed compounds. Future triggers (reflection, signals) use the same SPI
   without changes.
4. **Compound replacement, not mutation.** Initial decomposition and plan
   adaptation are structurally different operations. Initial decomposition creates
   a complete compound via `registerDefinition()` (populates scopedBindings,
   childrenIndex, definitions at once). Adaptation replaces the compound
   definition — unregisters the old, registers a new one with updated bindings.
   This is a definition-layer replacement, not an additive child operation.

## Architecture

### Lifecycle phases

Plan adaptation is "plan monitoring and repair" — four phases:

1. **Monitor** — detects that a step completed (infrastructure, always runs)
2. **Evaluate** — given what happened, is the current plan still valid?
   (configurable `AdaptationTrigger`)
3. **Revise** — produce a replacement plan for remaining work (configurable
   `PlanRevisionStrategy`)
4. **Apply** — materialise the revision in the runtime (infrastructure, fixed)

Phases 1 and 4 are engine infrastructure. Phases 2 and 3 are the orthogonal
configurable strategies.

### Call site

```
Worker completes within decomposed compound
  → PlanItemCompletionHandler marks PlanItem COMPLETED (success path)
    OR WorkerRetryExhaustionHandler marks PlanItem FAULTED (failure path)
  → PlanAdaptationEvaluator.evaluateAdaptation(...)   ← NEW (both paths)
      1. Resolve CaseInstance, CaseDefinition, CaseContext from caseId
      2. Is this step in a decomposed compound? (parentCompoundId != null)
      3. Acquire per-compound Semaphore (bounded concurrency)
      4. Build AdaptationContext (completed steps + outputs, pending, running)
      5. Resolve AdaptationTrigger → evaluate(context) → AdaptationSignal
      6. If Skip → release permit, return
      7. Resolve PlanRevisionStrategy → revise(revisionContext) → RevisedPlan
      8. Apply: replace compound definition, obsolete pending PlanItems, create new
      9. Write PLAN_ADAPTED EventLog
      10. Release permit
  → CompoundCompletionEvaluator evaluates (with revised compound definition)
  → CONTEXT_CHANGED published
```

Adaptation fires AFTER step status change but BEFORE compound completion
evaluation. This ensures the compound evaluates the revised plan, not the stale
one. Both success and failure paths trigger adaptation — `OnFailureTrigger`
relies on the failure path; `EveryStepTrigger` fires on both.

**Synchronous execution with bounded concurrency:** Adaptation must be
synchronous — `CompoundCompletionEvaluator` must evaluate the revised plan,
not the stale one. To prevent worker thread pool starvation during LLM calls,
a `Semaphore` bounds concurrent adaptations (default: 3). Excess requests
queue on the semaphore. This trades latency for correctness — a queued
adaptation delays its compound's completion evaluation but never produces a
stale evaluation.

### SPI signature — thin call site

The `PlanAdaptationEvaluator` SPI accepts only data available at both call
sites:

```java
public interface PlanAdaptationEvaluator {
    void evaluateAdaptation(
        UUID caseId,
        String tenancyId,
        String completedBindingName,
        TaskStatus completedStatus);
}
```

The implementation resolves `CaseInstance`, `CaseDefinition`, and
`MutableCaseContext` internally from its injected dependencies
(`CaseInstanceRepository`, `CaseDefinitionRegistry`). This keeps call sites
thin — `PlanItemCompletionHandler` and `WorkerRetryExhaustionHandler` pass
only what they already have.

## Foundation SPIs (engine-api)

### AdaptationTrigger

```java
public interface AdaptationTrigger extends NamedStrategy {
    AdaptationSignal evaluate(AdaptationContext context);

    @Override
    default String id() { return "every-step"; }
}
```

**`AdaptationSignal`** — sealed interface:
- `Proceed` — adaptation warranted
- `Skip` — current plan is still valid

The signal is a pure decision — proceed or skip. `AdaptationCause` is
constructed by the orchestrator (not the trigger) from the event data it
already has. This separates the decision concern (trigger) from the
description concern (orchestrator). The cause is passed directly to
`RevisionContext`, not returned by the trigger.

**`AdaptationCause`** — sealed interface describing what triggered adaptation:
- `StepCompleted(String stepId, String capabilityName, Map<String, Object> output)`
- `StepFailed(String stepId, String reason)`

Extension points for future triggers (not implemented in v1):
- `ContextDiverged(Set<String> changedKeys)`
- `InsightReceived(String insight)`

**`AdaptationContext`** — record carrying evaluation state:

```java
public record AdaptationContext(
    UUID caseId,
    String tenancyId,
    String compoundId,
    String goalName,
    List<CompletedStep> completedSteps,
    List<PlanStepDescriptor> pendingSteps,
    List<PlanStepDescriptor> runningSteps,
    JsonNode currentContext,
    CaseDefinition definition,
    TaskStatus latestStatus,
    String latestBindingName,
    int adaptationGeneration
)
```

`adaptationGeneration` is a monotonically increasing counter per compound,
incremented on each adaptation. Used for idempotency — after acquiring the
semaphore, the evaluator re-checks the generation. If it's changed since the
event was triggered, the adaptation was superseded and is skipped.

**`PlanStepDescriptor`** — engine-api record for plan steps in SPI types.
Replaces `GoalStep` in all api-level types to avoid the engine-api → planning
module dependency:

```java
public record PlanStepDescriptor(
    String id,
    String description,
    String capabilityName
)
```

`GoalStep` (planning module) maps to `PlanStepDescriptor` at the SPI boundary.

**`CompletedStep`** — record for completed step history:

```java
public record CompletedStep(
    String stepId,
    String capabilityName,
    String description,
    Map<String, Object> output,
    Instant completedAt
)
```

### PlanRevisionStrategy

```java
public interface PlanRevisionStrategy extends NamedStrategy {
    Uni<RevisedPlan> revise(RevisionContext context);

    @Override
    default String id() { return "forward-replan"; }
}
```

**`RevisionContext`** — carries everything the revision strategy needs:

```java
public record RevisionContext(
    AdaptationContext adaptationContext,
    AdaptationCause cause,
    List<Capability> capabilities,
    List<RetrievedMemory> memories
)
```

**`RevisedPlan`** — record carrying the revision result:

```java
public record RevisedPlan(
    List<PlanStepDescriptor> steps,
    String rationale
)
```

`steps` is the complete forward plan — all pending steps to materialise. The
orchestrator replaces the compound definition entirely and creates new
PlanItems. `rationale` is the LLM's explanation (nullable, for audit).

### Relationship to DecompositionStrategy

`PlanRevisionStrategy` does NOT delegate to `DecompositionStrategy`. They are
separate concerns with different prompt requirements:

- **DecompositionStrategy**: "Given a goal and capabilities, plan from scratch"
- **PlanRevisionStrategy**: "Given completed steps and their outputs, revised
  context, and remaining capabilities, plan the remaining work"

`ForwardReplanRevision` constructs its own prompt and interacts with
`ChatModelProvider` directly.

## Built-in Strategy Implementations (planning module)

### Triggers

**`EveryStepTrigger`** (id=`"every-step"`, default)
Always returns `Proceed` after any step completion. Maximum responsiveness.

**`OnFailureTrigger`** (id=`"on-failure"`)
Returns `Proceed` only when `latestStatus` is a non-success terminal state
(`FAULTED`, `REJECTED`, `OBSOLETE`, `CANCELLED`). Returns `Skip` on
`COMPLETED`. More conservative — only replans when something went wrong.

### Revision strategies

**`ForwardReplanRevision`** (id=`"forward-replan"`, default)
Re-invokes LLM with completed step history + current context + available
capabilities. Asks for remaining steps only. Prompt structure:

```
System: "You are a planning assistant. A plan is in progress. Some steps have
completed. Given the current state and remaining capabilities, produce an
updated plan for the remaining work. Each step must reference exactly one
capability."

User: "Goal: {goalName}
Completed steps:
  1. {step.description} → Output: {step.output}
  2. ...
Current context: {contextSnapshot}
Available capabilities: {capabilities}
Produce the remaining steps as a JSON 'steps' array."
```

Response parsing reuses the same JSON structure as `LlmDecompositionStrategy`
(steps array with id, description, capabilityName, dependsOn). Maps to
`PlanStepDescriptor` instances.

Injected dependencies:
- `Instance<ChatModelProvider>` — transparent no-op when absent
- Returns `Uni.createFrom().failure()` when no ChatModelProvider available

## Compound Replacement — Apply Phase

The apply phase is adaptation-specific infrastructure, not shared with initial
decomposition. Initial decomposition builds a compound from scratch via
`registerDefinition()`. Adaptation replaces a live compound.

### `CasePlanModel.replaceCompound()` (new method on DefaultCasePlanModel)

```java
void replaceCompound(String compoundId,
                     PlanItemDefinition.Compound newCompound,
                     int newGeneration)
```

Atomic compound replacement:
1. Unregisters old compound's children from `childrenIndex`, `definitions`,
   `definitionStates`, `parentIndex`
2. Removes old children's PlanItems from `itemsById`, `agenda`,
   `latestByBinding`
3. Registers new compound definition via `registerDefinition()` — populates
   all index structures (scopedBindings, childrenIndex, definitions,
   definitionStates, parentIndex) in one operation
4. Stores the new `adaptationGeneration` on the compound state

The compound definition itself is replaced — the new `Compound` record has
updated `scopedBindings` reflecting the new plan steps. This ensures
`PlanningStrategyLoopControl` sees the new bindings in its dispatch loop.

The `CasePlanModel` is the authoritative source of truth for definition state.
`PlanItemStore` is updated for persistence/recovery but does not drive dispatch
or completion evaluation.

### PlanItem lifecycle during replacement

1. **COMPLETED PlanItems** — untouched. Their binding names stay in the new
   compound's `scopedBindings` (the compound knows what's already done).
2. **RUNNING PlanItems** — untouched. Their binding names are included in the
   new compound's `scopedBindings`. When they complete, another adaptation
   cycle fires.
3. **PENDING PlanItems** — marked `OBSOLETE` via `PlanItemStore.updateStatus()`
   and removed from `CasePlanModel`.
4. **New PlanItems** — created from `RevisedPlan.steps()`, saved via
   `PlanItemStore.save()`, added to `CasePlanModel` via the compound
   registration.

`PlanItemStore` writes happen after `CasePlanModel` mutation. On failure
between the two, recovery reconstructs `CasePlanModel` from
`PlanItemStore` (same as initial decomposition recovery).

### Failure degradation

On adaptation timeout or LLM failure:
- **Success-triggered** (`EveryStepTrigger` on COMPLETED): existing plan
  continues unmodified. Warning logged.
- **Failure-triggered** (`OnFailureTrigger` on FAULTED): the faulted step
  is already handled by the existing failure cascade
  (`WorkerOutcomeResolvedHandler` → `OutcomePolicy`). Adaptation failure
  does not add a second fault path — the plan continues with the faulted
  step handled by the binding's own reroute/fault policy.

## Orchestrator — DefaultPlanAdaptationEvaluator (planning module)

`@ApplicationScoped`, injected dependencies:
- `EngineStrategyResolver` — resolves trigger + revision strategies
- `BlackboardRegistry` — access CasePlanModel
- `PlanItemStore` — query PlanItems for completed step list, persist changes
- `EventLogRepository` — query completed step outputs, write audit
- `CaseInstanceRepository` — resolve CaseInstance from caseId
- `CaseDefinitionRegistry` — resolve CaseDefinition from CaseMetaModel
- `Instance<AgentMemoryRetriever>` — optional memory retrieval (planning
  already depends on runtime via `EngineStrategyResolver` — no new
  dependency direction)

**Bounded concurrency:** `Semaphore` with configurable permits
(`casehub.engine.adaptation.max-concurrent`, default 3). Acquired before
CaseInstance resolution, released in a finally block. `tryAcquire` with
timeout equal to `casehub.engine.decomposition.timeout-ms` — if the
semaphore can't be acquired in time, adaptation is skipped with a warning.

**Per-compound serialization:** `ConcurrentHashMap<String, ReentrantLock>`
keyed by `caseId + ":" + compoundId`. Prevents concurrent adaptations
within the same compound when two steps complete near-simultaneously.

**Lock map lifecycle:** Entries are removed when:
- The compound reaches a terminal status (via
  `CompoundCompletedEvent` listener)
- The case reaches a terminal status (via `CaseStatusChangedHandler`)

**Idempotency via generation counter:** Each compound tracks an
`adaptationGeneration` (int, starts at 0, incremented on each adaptation).
After acquiring the per-compound lock, the evaluator re-checks the
generation against the value captured when the event was triggered. If
changed, a concurrent adaptation already ran — skip to avoid redundant
LLM calls and PlanItem churn.

**Completed step output reconstruction:** Queries `EventLogRepository` for
events with `eventType = WORKER_EXECUTION_COMPLETED` matching the case ID.
Extracts output from EventLog payload. The `WORKER_EXECUTION_COMPLETED`
event type and its payload structure are shared via `CaseHubEventType`
(common module) — no runtime-module dependency for parsing.

**Timeout:** Reuses `casehub.engine.decomposition.timeout-ms` (default 30000).

## CaseDefinition Configuration

`CaseDefinition` gains `adaptationConfig` (nullable `AdaptationConfig`):

```java
public record AdaptationConfig(
    String trigger,
    String revision
)
```

Null means adaptation is disabled. Groups the two strategy IDs as a single
conceptual unit, following the established pattern of `ReflectionTriggerConfig`,
`MemoryRetrievalConfig`, `CbrConfig`.

Builder:
```java
CaseDefinition.builder()
    .decompositionStrategy("llm")
    .adaptationConfig(new AdaptationConfig("every-step", "forward-replan"))
    .build();
```

### YAML — explicit configuration

```yaml
spec:
  decompositionStrategy: llm
  adaptation:
    trigger: every-step
    revision: forward-replan
```

### YAML — preset shorthands

```yaml
spec:
  decompositionStrategy: llm
  adaptation: adaptive
```

Presets:
- `adaptive` → trigger: `every-step` + revision: `forward-replan`
- `conservative` → trigger: `on-failure` + revision: `forward-replan`
- `off` or absent → no adaptation

`CaseDefinitionYamlMapper` detects whether `adaptation` is a string (preset) or
object (explicit config). Missing `trigger` or `revision` fields in explicit
config fall back to defaults (`every-step` and `forward-replan` respectively).

### Validation

- `adaptation` without `decompositionStrategy` → warning log at parse time
  (adaptation requires initial decomposition to produce a compound to adapt)
- Unknown preset name → `IllegalArgumentException` at build time
- Absent `adaptation` → no adaptation (default, opt-in)

## EventLog Audit

New event type: `CaseHubEventType.PLAN_ADAPTED`

Metadata:
- `goalName` — which goal's plan was revised
- `compoundId` — which compound was adapted
- `triggerStrategy` — which trigger strategy fired (e.g., `"every-step"`)
- `cause` — structured cause (e.g., `{type: "StepCompleted", stepId: "...",
  capabilityName: "..."}`)
- `revisionStrategy` — which revision strategy produced the new plan
- `previousStepCount` — number of pending steps before revision
- `newStepCount` — number of new steps materialised
- `obsoletedSteps` — list of planItemIds marked OBSOLETE
- `materializedSteps` — list of new planItemIds created
- `adaptationGeneration` — the generation counter after this adaptation
- `rationale` — LLM's explanation for the revision (nullable)

## Module Placement

| Type | Module | Rationale (PP-20260727-5267d2) |
|------|--------|-------------------------------|
| `AdaptationTrigger` | engine-api | Consumer-implementable NamedStrategy SPI |
| `PlanRevisionStrategy` | engine-api | Consumer-implementable NamedStrategy SPI |
| `AdaptationSignal`, `AdaptationCause` | engine-api | SPI result types |
| `AdaptationContext`, `CompletedStep` | engine-api | SPI parameter types |
| `PlanStepDescriptor` | engine-api | SPI-level plan step representation |
| `RevisionContext`, `RevisedPlan` | engine-api | SPI parameter/result types |
| `AdaptationConfig` | engine-api | CaseDefinition config record |
| `PlanAdaptationEvaluator` (SPI) | common/spi | Cross-module SPI |
| `DefaultPlanAdaptationEvaluator` | planning | Execution infrastructure |
| `EveryStepTrigger` | planning | Built-in strategy |
| `OnFailureTrigger` | planning | Built-in strategy |
| `ForwardReplanRevision` | planning | Built-in strategy |
| `PLAN_ADAPTED` event type | common | Event constant |

**`EngineStrategyResolver`:** The catch-all `@Any Instance<NamedStrategy>`
with `registerRemainingStrategies()` auto-discovers new `NamedStrategy`
implementations. Explicit per-type `Instance<>` injections are a best
practice for build-time pruning reliability but not functionally required.
Add explicit injections for `AdaptationTrigger` and `PlanRevisionStrategy`
as a best-practice addition, not a blocking prerequisite.

## Testing

### Unit tests

1. **`AdaptationTriggerTest`** — tests for each trigger:
   - `EveryStepTrigger` always returns `Proceed`
   - `OnFailureTrigger` returns `Proceed` for FAULTED, REJECTED, CANCELLED
   - `OnFailureTrigger` returns `Skip` for COMPLETED
   - Both handle all TaskStatus values without exceptions

2. **`ForwardReplanRevisionTest`** — LLM interaction:
   - Produces RevisedPlan from structured response
   - Includes completed step history in prompt
   - Handles empty response (no steps needed)
   - No-op when ChatModelProvider absent
   - Invalid JSON → Uni failure
   - Unknown capabilities filtered

3. **`CasePlanModelReplaceCompoundTest`** — compound replacement:
   - `replaceCompound()` unregisters old children from all index structures
   - `replaceCompound()` registers new compound with updated scopedBindings
   - New children visible to `PlanningStrategyLoopControl` dispatch
   - Completed children preserved across replacement
   - Running children preserved across replacement
   - Generation counter incremented
   - `CompoundCompletionEvaluator` evaluates revised children correctly

4. **`DefaultPlanAdaptationEvaluatorTest`** — orchestrator logic:
   - Skips when completedBindingName is not in a decomposed compound
   - Skips when adaptation not configured (null adaptationConfig)
   - Calls trigger → Skip → no revision called
   - Calls trigger → Proceed → calls revision → applies result
   - Compound definition replaced (not mutated) with new scopedBindings
   - Pending PlanItems marked OBSOLETE
   - Running PlanItems untouched
   - Completed PlanItems untouched
   - Writes PLAN_ADAPTED EventLog with correct metadata
   - Per-compound lock prevents concurrent adaptation (caseId:compoundId key)
   - Lock key does not collide across cases with same goal name
   - Semaphore bounds concurrent adaptations
   - Semaphore timeout → adaptation skipped with warning
   - Generation counter prevents redundant LLM calls
   - Timeout → graceful degradation (existing plan continues)
   - Exception isolation → existing plan continues, warning logged
   - Lock map entries cleaned up on compound/case completion

5. **`CaseDefinitionYamlMapperTest`** — YAML parsing:
   - Parses explicit adaptation config (trigger + revision)
   - Parses preset shorthand (`adaptation: adaptive`)
   - Parses preset shorthand (`adaptation: conservative`)
   - Missing adaptation → null AdaptationConfig
   - Adaptation without decompositionStrategy → warning
   - Unknown preset → error
   - Partial explicit config → defaults for missing fields

6. **`AdaptationConfigTest`**, **`PlanStepDescriptorTest`**,
   **`CompletedStepTest`**, **`RevisedPlanTest`**:
   - Record validation, null checks, immutability

### Integration test

7. **`PlanAdaptationIntegrationTest`** (`@QuarkusTest`):
   - Full flow: case with goals + LLM strategy + adaptation → start → step
     completes → adaptation fires → compound replaced → new steps dispatch
     → compound completes
   - Mock `ChatModelProvider` with canned JSON (initial plan + revised plan)
   - EventLog contains both `GOAL_DECOMPOSED` and `PLAN_ADAPTED`
   - Compound completes when revised plan finishes
   - Verify OBSOLETE steps are not re-dispatched
   - Verify new steps visible to dispatch loop via updated scopedBindings

## Scope Boundaries

**In scope:**
- `AdaptationTrigger` + `PlanRevisionStrategy` SPIs (engine-api)
- `PlanStepDescriptor` (engine-api) — SPI-level plan step type
- `AdaptationConfig` (engine-api) — config record
- `PlanAdaptationEvaluator` SPI + `DefaultPlanAdaptationEvaluator`
- `EveryStepTrigger`, `OnFailureTrigger`, `ForwardReplanRevision`
- `CasePlanModel.replaceCompound()` — compound replacement
- CaseDefinition config + YAML + presets
- EventLog audit (`PLAN_ADAPTED`)
- Bounded concurrency (semaphore + per-compound lock)
- Generation-based idempotency
- Lock map lifecycle cleanup
- In-flight step handling

**v1 constraints (deliberate):**
- Linear chain plans only (same as #802)
- Single adaptation per compound per step completion (serialized)
- Full pending-step replacement (no diffing/merging)
- No adaptation during initial decomposition

**Out of scope (future work):**
- `ContextDiffTrigger` — adapt based on context key changes
- `IncrementalRevision` — LLM returns targeted modifications
- `FullReplanRevision` — LLM replans everything, engine diffs
- Reflection-triggered adaptation (#808 may drive this)
- Parallel plan adaptation (non-linear compounds)
- Adaptation cost budgeting (limit LLM calls per case)
- Adaptation history visualization / REST endpoints

## Review Findings Addressed

Summary of design-review findings incorporated in this revision:

| Finding | Resolution |
|---------|-----------|
| Compound immutability (Rob R1-01, R1-02) | `replaceCompound()` — atomic compound replacement |
| GoalStep in engine-api (Coh R1-01, Str R1-01) | `PlanStepDescriptor` — clean api-level type |
| Call site data availability (all dimensions) | Thin SPI: `(caseId, tenancyId, bindingName, status)` |
| AdaptationCause in wrong place (Str R1-04) | Orchestrator constructs cause; trigger returns Proceed/Skip |
| Config pattern (Str R1-05) | `AdaptationConfig` record follows established pattern |
| Lock key collision (Rob R1-04) | Key = `caseId + ":" + compoundId` |
| Thread starvation (Rob R1-05, XC R1-01) | Semaphore-bounded concurrency, synchronous execution |
| Lock map unbounded (Str R1-06, Rob R1-08) | Lifecycle cleanup on compound/case completion |
| Idempotency (Rob R1-06) | Generation counter per compound |
| OnFailureTrigger terminology (Coh R1-04) | Uses actual TaskStatus enum values |
| Shared materialisation unsound (XC R1-03) | Replaced with compound replacement; no shared PlanMaterializer |
| AgentMemoryRetriever circularity (Coh R1-03) | Invalid — planning already depends on runtime (XC R1-06) |
| EngineStrategyResolver (XC R1-07) | Auto-discovers; explicit injection is best-practice only |
| Failure degradation (Rob R1-07) | Failure-triggered: existing OutcomePolicy handles faulted step |
