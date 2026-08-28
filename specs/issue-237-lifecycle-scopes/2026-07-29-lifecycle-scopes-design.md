# Lifecycle Scopes for Workers — engine#237

**Date:** 2026-07-29
**Issue:** casehubio/engine#237
**Status:** Design

## Summary

Workers in the engine currently have no structured lifecycle scope — they are activated per binding trigger and complete independently. This design introduces first-class lifecycle scoping: workers can be declared to persist for the duration of a Compound or an entire Case, receiving context changes throughout their scope's lifetime.

Three scopes: **BINDING** (current default — one activation, one execution), **COMPOUND** (worker lives for the owning Compound's lifetime), and **CASE** (worker persists for the full case lifetime).

Two execution strategies: **PERSISTENT** (long-running thread with a mailbox, receives context events) and **REINVOKED** (re-invoked on each context change with accumulated state from prior invocations).

Two participation roles: **PARTICIPANT** (counts toward scope completion) and **COMPANION** (excluded from completion, terminated when scope ends — the Kubernetes sidecar pattern).

Two activation modes: **scope-activated** (starts when the scope activates) and **binding-triggered** (starts when binding condition first matches, persists for the scope).

## Prerequisites

- Unified execution model (blocks#60) — Compound replaces Stage, sealed PlanItemDefinition hierarchy. **Delivered.**

## Design Decisions

### Scope declared on Binding, not Worker or PlanItemDefinition

The Binding is the dispatch control point — it already governs trigger conditions, input projection, outcome policy, and agent routing. Scope is a natural extension of what the binding controls: how long the dispatched worker persists.

A Worker is a reusable definition (foundation-tier record in `casehubio/worker`). The same worker can be used with BINDING scope in one case definition and COMPOUND scope in another. Scope is a property of deployment, not definition.

PlanItemDefinition was considered (Approach 2 — Compound-native). Rejected because it creates a parallel dispatch path that must reimplement or bridge every feature the binding system already provides (routing, projections, outcome handling, CBR retrieval).

### Sidecar model for completion semantics

Research across CMMN, Kubernetes, actor models (Akka/Erlang), and BPMN multi-instance confirms that the cleanest pattern separates scope lifetime from completion semantics:

- **Kubernetes sidecars** (GA v1.33): init containers with `restartPolicy: Always` run for the Pod's lifetime but don't extend Pod lifetime. Main containers determine completion.
- **CMMN required flag:** Plan items within a stage can be marked required or not. Non-required items don't block stage completion.
- **BPMN completion condition:** A boolean expression evaluated after each instance completes. Remaining active instances are terminated when it fires.
- **Akka supervision:** Parent termination cascades to all children unconditionally.

The common thread: the container's completion is determined by a subset of its contents, and the rest is terminated when the container completes.

Applied here: `Participation.COMPANION` workers are excluded from `evaluateCompletion` and terminated after the scope completes. `Participation.PARTICIPANT` workers count toward completion and must signal "done" for the scope to complete.

### One dispatch path, not two

Scope-activated workers flow into the existing scheduling infrastructure after their initial trigger. `CompoundLifecycleEvaluator` fires the initial dispatch via `WorkerScheduleEvent`, which enters the same `WorkerScheduleEventHandler` → `WorkerExecutionManager` → `QuartzWorkerExecutionJob` path as all other workers.

Binding-triggered scoped workers go through the existing `CaseContextChangedEventHandler.publishWorkerSchedule()` path with an interception point: the handler checks `ScopedWorkerRegistry` before creating a new PlanItem. If a session exists, context is routed to the existing session.

One path means all existing features (agent routing, input projection, outcome handling, CBR, action risk classification) work for scoped workers automatically on first activation.

The Quartz job's completion behavior changes for scoped workers — see §Scoped dispatch metadata below.

## Core Types

### LifecycleScope

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum LifecycleScope {
    BINDING,    // one activation, one execution (current default)
    COMPOUND,   // lives for the owning Compound's lifetime
    CASE        // lives for the entire case lifetime
}
```

### Participation

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum Participation {
    PARTICIPANT,  // counts toward completion (default)
    COMPANION     // excluded from completion, terminated when scope ends
}
```

### ExecutionMode

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum ExecutionMode {
    TRANSIENT,   // fire-and-forget, no session continuity (current default)
    PERSISTENT,  // long-running virtual thread, receives context events via mailbox
    REINVOKED    // re-invoked on each context change with accumulated state
}
```

### ScopeActivatedTrigger

Record in `engine-api`, package `io.casehub.api.model`.

```java
public record ScopeActivatedTrigger() implements Trigger {}
```

Fires when the owning Compound activates (transitions to RUNNING). For CASE scope, fires when the case starts. The binding's `when` condition is still respected as a guard.

### Binding changes

`Binding` gains three fields, all defaulting to current behavior:

```java
public class Binding {
    // ... existing fields ...
    private LifecycleScope lifecycleScope;    // default BINDING
    private Participation participation;       // default PARTICIPANT
    private ExecutionMode executionMode;        // default TRANSIENT
}
```

Builder methods: `.lifecycleScope(LifecycleScope)`, `.participation(Participation)`, `.executionMode(ExecutionMode)`.

### Validation rules (build-time, in `Binding.Builder.build()`)

- `BINDING` scope requires `ExecutionMode.TRANSIENT`
- `COMPOUND` scope requires the binding to be in a Compound's `scopedBindings`
- `CASE` scope is valid on any binding (case-global, no compound membership required)
- `CASE` scope requires `Participation.COMPANION` — case completion is goal-based (`GoalBasedCompletion`), so PARTICIPANT has no mechanism to block it; use compound-level composition to gate case completion
- `COMPANION` participation requires `COMPOUND` or `CASE` scope
- `ScopeActivatedTrigger` requires `COMPOUND` or `CASE` scope
- `lifecycleScope != BINDING` requires `CapabilityTarget` — SubCase and HumanTask targets have their own lifecycle models incompatible with scoped sessions; `publishByTarget()` dispatches them through separate paths (`publishSubCaseSchedule`, `publishHumanTaskSchedule`)

## Runtime Infrastructure

### ScopedWorkerSession

Sealed interface in `runtime`, package `io.casehub.engine.internal.worker.scope`.

```java
public sealed interface ScopedWorkerSession
    permits ScopedWorkerSession.Persistent, ScopedWorkerSession.Reinvoked {

    String bindingName();
    UUID caseId();
    String planItemId();
    LifecycleScope scope();
    Participation participation();

    record Persistent(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        BlockingQueue<ContextEvent> mailbox,
        Future<?> workerThread
    ) implements ScopedWorkerSession {}

    record Reinvoked(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        AtomicReference<Map<String, Object>> accumulatedState
    ) implements ScopedWorkerSession {}
}
```

**Persistent sessions** have a mailbox (`BlockingQueue<ContextEvent>`) and a running virtual thread. The worker's `Persistent` handler receives a `PersistentScope` backed by the mailbox — `nextEvent()` blocks on the queue, `emit()` applies output to the case context. The engine manages the virtual thread lifecycle.

#### `PersistentScope.emit()` semantics

`emit()` is asynchronous and non-blocking from the worker's perspective:

1. Applies output to the case context (direct context mutation)
2. Publishes a `CaseContextChangedEvent` via the event bus (fire-and-forget)
3. Returns immediately to the worker

The `CaseContextChangedEvent` is consumed by `CaseContextChangedEventHandler` on a separate virtual thread and enters `CaseEvaluationSerializer` like any other context change. The worker's virtual thread never enters the serializer — it is decoupled from the evaluation pipeline. This avoids head-of-line blocking: the serializer uses a non-blocking coalescing pattern (lock + pending evaluator), but the worker doesn't participate in that at all.

#### `PersistentScope.nextEvent()` shutdown signaling

`nextEvent()` throws `ScopeTerminatedException` (unchecked, extends `RuntimeException`) when the engine sends a shutdown signal. This cleanly distinguishes:

- **Worker-initiated exit:** handler returns normally → PlanItem → COMPLETED (PARTICIPANT done)
- **Engine-initiated shutdown:** `nextEvent()` throws `ScopeTerminatedException` → engine catches it, treats as normal termination (scope ended)

Worker loop pattern:
```java
new WorkerFunction.Persistent<>(MyInput.class, scope -> {
    var state = initState();
    while (true) {
        MyInput event = scope.nextEvent(); // throws ScopeTerminatedException on shutdown
        state = process(state, event);
        scope.emit(state.output());
        if (state.isDone()) return; // worker-initiated exit → COMPLETED
    }
    // ScopeTerminatedException propagates naturally — engine catches it
});
```

#### Persistent worker fault detection

The engine wraps the `Persistent` handler in a try/catch on the managed virtual thread. If the handler throws an unhandled exception (not `ScopeTerminatedException`):

1. Publishes `WorkflowExecutionCompleted` with `WorkerOutcome.Failed` outcome
2. Removes the session from `ScopedWorkerRegistry`
3. Applies `OutcomePolicy` for first-invocation faults, or logs for post-activation faults

This mirrors the TRANSIENT fault detection path in `QuartzWorkerExecutionJob.executeInternal()`.

**Reinvoked sessions** hold accumulated state from prior invocations (`AtomicReference<Map<String, Object>>`). On each context change, the engine re-invokes the worker function with projected input of type `T`. Prior output is accessible via `WorkerScope.accumulatedState()` — a new method returning `Map<String, Object>` (empty for TRANSIENT workers). Output `R` is serialized and stored as the new accumulated state.

#### REINVOKED accumulated state access

Accumulated state is NOT merged with the typed input `T`. Input type `T` (from projection) and output type `R` are distinct types that cannot be safely merged. Instead:

- The worker receives projected input `T` on every invocation (consistent contract, per R1-04)
- Prior invocation output is accessible via `scope.accumulatedState()` as `Map<String, Object>`
- The worker combines current input with prior state as it sees fit
- Output `R` is serialized to `Map<String, Object>` and stored as the new accumulated state

`WorkerScope.accumulatedState()` is added to `casehub-worker-api`. Returns empty map for TRANSIENT workers; returns the prior invocation's serialized output for REINVOKED workers.

### ContextEvent

Record in `runtime`, package `io.casehub.engine.internal.worker.scope`.

Mailbox message type for persistent workers. Carries the context snapshot, change metadata, and a shutdown sentinel.

### ScopedWorkerRegistry

`@ApplicationScoped` in `runtime`, package `io.casehub.engine.internal.worker.scope`.

```java
@ApplicationScoped
public class ScopedWorkerRegistry {

    private final ConcurrentHashMap<ScopeKey, ScopedWorkerSession> sessions;

    public Optional<ScopedWorkerSession> get(UUID caseId, String bindingName);
    public void register(ScopeKey key, ScopedWorkerSession session);
    public void terminateByScope(UUID caseId, String compoundId);
    public void terminateByCase(UUID caseId);
    public List<ScopedWorkerSession> getCompanions(UUID caseId, String compoundId);

    record ScopeKey(UUID caseId, String bindingName) {}
}
```

Keyed by `(caseId, bindingName)` — one scoped session per binding per case instance.

`terminateByScope(caseId, compoundId)` finds sessions whose binding is owned by the compound and terminates them.

`terminateByCase(caseId)` terminates all sessions for a case.

### Termination protocol

**Persistent:** Poison pill (`ContextEvent.SHUTDOWN`) on the mailbox. Worker loop reads it and exits cleanly. If the worker doesn't exit within `ExecutionPolicy.timeoutMs`, the thread is interrupted.

**Reinvoked:** Session removed from registry. Any in-flight invocation completes normally but output is discarded if scope has ended.

PlanItem transitions to COMPLETED (COMPANION) or is left in current state (PARTICIPANT — should already be terminal).

## Dispatch and Lifecycle Integration

### Scope-activated dispatch

`CompoundLifecycleEvaluator.activatePendingCompounds()` transitions compounds to RUNNING (no change to its current responsibilities). A new `CompoundActivatedEvent(caseId, tenancyId, compoundId, compoundName)` — symmetric to the existing `CompoundCompletedEvent` — is published by the caller when compounds transition. A new `CompoundActivatedEventHandler` (in `runtime`, which has access to `CaseDefinition`, bindings, `EventBus`, and the full dispatch infrastructure) consumes the event and dispatches scope-activated bindings via the normal `WorkerScheduleEvent` path.

This follows the existing architectural pattern: lifecycle evaluators produce state transitions, event handlers perform side effects.

For CASE-scoped scope-activated bindings, `CaseStartedEventHandler` dispatches them at case start.

### ScheduleTrigger interaction with scoped bindings

`ScheduleTrigger` (cron or delay) is valid with scoped bindings. The interaction follows the same registry-check pattern as `ContextChangeTrigger`:

- **First fire:** creates PlanItem, registers scoped session, dispatches worker normally.
- **Subsequent fires:** checks `ScopedWorkerRegistry`. If session exists, routes to it (mailbox for PERSISTENT, re-invocation for REINVOKED). No new PlanItem.
- **Delay-based (`ScheduleTrigger.delay`):** the delay governs initial activation timing; after first fire, the scoped session persists for the scope lifetime.
- **Cron-based (`ScheduleTrigger.cron`):** each cron tick routes to the existing session. Useful for periodic workers that accumulate state (e.g., scheduled health checks).

### Scoped dispatch metadata

`WorkerScheduleEvent` gains two nullable fields: `LifecycleScope lifecycleScope` and `ExecutionMode executionMode`. Null = TRANSIENT (backward compatible — existing constructors pass null).

`QuartzWorkerExecutionJob` branches on these fields to suppress premature PlanItem completion:

- **`executionMode == null` or `TRANSIENT`:** current behavior — `onSuccess()` publishes `WorkflowExecutionCompleted`, `PlanItemCompletionHandler` transitions PlanItem → COMPLETED.

- **`PERSISTENT` (initial invocation):** the Quartz job creates the `ScopedWorkerSession.Persistent`, starts the virtual thread, registers the session in `ScopedWorkerRegistry`, and returns. It does NOT publish `WorkflowExecutionCompleted`. The PlanItem stays RUNNING. The virtual thread's fault detection wrapper (§Persistent worker fault detection) handles eventual completion or failure.

- **`REINVOKED` (initial and subsequent invocations):** the Quartz job executes the worker function normally, applies output to the case context, and stores output as accumulated state in the session. On `WorkerOutcome.Success`: does NOT publish `WorkflowExecutionCompleted` — PlanItem stays RUNNING. On `WorkerOutcome.Completed`: publishes `WorkflowExecutionCompleted` with `Completed` outcome — `PlanItemCompletionHandler` transitions PlanItem → COMPLETED. On failure outcomes (`Declined`/`Failed`/`Expired`): applies `OutcomePolicy` for first invocation; logs and preserves session for subsequent invocations.

This generalizes the "re-invocation flag" previously mentioned: there is no separate flag. `executionMode` on the schedule event determines behavior for both initial and subsequent invocations.

### Binding-triggered scoped dispatch

`CaseContextChangedEventHandler.publishWorkerSchedule()` — before creating a new PlanItem:

1. Check `binding.lifecycleScope()` — if BINDING, proceed as current (no change).
2. If COMPOUND or CASE: check `scopedWorkerRegistry.get(caseId, bindingName)`.
3. If session exists and active: route context to existing session, return.
4. If no session: proceed with normal dispatch, then register a new `ScopedWorkerSession`.

**Routing context to an existing session:**

- Persistent: put `ContextEvent` on session's mailbox.
- Reinvoked: publish `WorkerScheduleEvent` with `executionMode = REINVOKED`. `QuartzWorkerExecutionJob` reads accumulated state from session, invokes worker function with projected input (accumulated state accessible via `WorkerScope.accumulatedState()`), stores output as new accumulated state, applies output to case context. PlanItem stays RUNNING (see §Scoped dispatch metadata for completion suppression).

### Scope termination

`CompoundCompletionEvaluator.evaluate()` — after a compound transitions to COMPLETED, a new `ScopedWorkerTerminationHandler` (in `runtime` module) consumes `CompoundCompletedEvent` via the event bus and calls `scopedWorkerRegistry.terminateByScope(caseId, compoundId)`. Placed in `runtime` alongside `CompoundActivatedEventHandler` — both consume planning events and interact with worker infrastructure.

`CaseStatusChangedHandler` — on terminal case state (COMPLETED, FAULTED, CANCELLED), calls `scopedWorkerRegistry.terminateByCase(caseId)`. `ScopedWorkerRegistry` is injected into `CaseStatusChangedHandler` — both are in the `runtime` module, so the dependency is module-internal. The call is placed in the `isTerminalState(newState)` block alongside `schedulerService.cancelAllTriggers()` and channel closure.

Termination ordering matches Kubernetes: COMPANION workers terminate AFTER the compound completes, not during.

### PlanItem lifecycle for scoped workers

One PlanItem per scoped worker for the entire scope lifetime:
- Created at first activation
- Transitions to RUNNING immediately
- Stays RUNNING for scope duration — `QuartzWorkerExecutionJob` suppresses `WorkflowExecutionCompleted` on `Success` for non-TRANSIENT `executionMode` (see §Scoped dispatch metadata)
- Intermediate output applied via `emit()` (persistent) or accumulated state merge (reinvoked)
- Transitions to COMPLETED when scope ends (COMPANION) or when worker signals "done" (PARTICIPANT)

### How PARTICIPANT workers signal "done"

- **Persistent:** Worker handler returns normally (exits the `PersistentScope.nextEvent()` loop). Engine transitions PlanItem → COMPLETED.
- **Reinvoked:** Worker returns `WorkerResult.completed(output)` — a new factory method producing `WorkerOutcome.Completed`. Engine detects `Completed` outcome, transitions PlanItem → COMPLETED. Session stays alive (can still receive events) but no longer blocks completion.

`WorkerOutcome.Completed` is a new permit in the sealed `WorkerOutcome<R>` hierarchy in `casehub-worker-api`. This is type-safe (compile-time checked by exhaustive switch), discoverable (autocomplete, API docs), and consistent with the existing outcome model. The context-key approach (`_lifecycle.<bindingName>.done`) is rejected — it violates the context layer model and is invisible to the compiler.

## Completion Semantics

### Compound.scopedBindings carries Participation metadata

`PlanItemDefinition.Compound.scopedBindings` changes from `Set<String>` to `Map<String, Participation>`. `Compound.Builder` gains a two-arg overload `binding(String, Participation)`. The existing no-arg `binding(String)` overload is retained, defaulting to `Participation.PARTICIPANT`:

```java
public Builder binding(String name) { return binding(name, Participation.PARTICIPANT); }
public Builder binding(String name, Participation p) { ... }
```

This keeps participation metadata in the definition where it belongs — `evaluateCompletion` needs no external lookups to distinguish COMPANION from PARTICIPANT.

### CompoundCompletionEvaluator changes

`evaluateCompletion` in `DefaultCasePlanModel` filters COMPANION bindings before counting toward completion:

```java
Map<String, Participation> scoped = compound.scopedBindings();

long scopedTerminal = scoped.entrySet().stream()
    .filter(e -> e.getValue() == Participation.PARTICIPANT)
    .map(e -> {
        PlanItem pi = latestByBinding.get(e.getKey());
        return pi != null ? pi.getStatus() : TaskStatus.PENDING;
    })
    .filter(TaskStatus::isTerminal)
    .count();

long scopedParticipantCount = scoped.values().stream()
    .filter(p -> p == Participation.PARTICIPANT)
    .count();

long totalCount = children.size() + scopedParticipantCount;
```

COMPANION PlanItems are excluded from the count entirely. PARTICIPANT PlanItems count toward `CompletionSemantics` (All/MOfN/FirstWins) like any other child.

### No CASE-level completion gating

CASE scope is restricted to COMPANION participation (see validation rules). Case completion is goal-based (`GoalBasedCompletion` evaluated by `GoalReachedEventHandler`), which checks named goal events in the event log — it has no concept of plan item state gating. Workers that need to block case completion should be COMPOUND-scoped PARTICIPANTs of a compound that gates the case's completion goal.

### Completion truth table

| Participation | Compound completing | Worker done | Worker running |
|---------------|--------------------|----|---------|
| PARTICIPANT | Compound waits | Terminal PlanItem counts toward CompletionSemantics | Compound cannot complete |
| COMPANION | Compound ignores | No effect on completion | Worker terminated after compound completes |

### Edge cases

| Scenario | Behavior |
|----------|----------|
| COMPANION faults on first activation | `OutcomePolicy` applies normally (REROUTE or FAULT). Rerouted COMPANION inherits COMPANION participation from the binding and is registered as a new scoped session. |
| COMPANION faults after first activation | Logged, session removed. Does NOT fault the compound. No reroute — `OutcomePolicy` governs first activation only. |
| PARTICIPANT faults | Normal fault handling — `OutcomePolicy` applies (REROUTE or FAULT). |
| Compound exits via exit condition | All scoped workers (both roles) terminated immediately. |
| Case cancelled | All scoped workers across all scopes terminated. |
| Repeatable compound resets | `ScopedWorkerRegistry.register()` uses replace-on-register semantics: if a session exists for the same `ScopeKey`, it is terminated before the new session is registered. This eliminates the race window between asynchronous `CompoundCompletedEvent` termination and re-activation dispatch. |
| Case suspension (SUSPENDED) | Scoped sessions stay registered. Persistent workers receive no new mailbox events (no `CaseContextChangedEvent` fires while suspended). Reinvoked workers are not re-invoked. On resume (RUNNING), `CaseStatusChangedHandler` publishes a `CaseContextChangedEvent` which flows through normal dispatch — persistent workers receive it via their mailbox, reinvoked workers are re-invoked. No session teardown or recreation. |

## Interaction with Existing Features

| Feature | Impact |
|---------|--------|
| Agent routing | First activation only — not on re-invocations |
| Outcome policy | First activation only. Re-invocation failures: logged, session state preserved |
| Input projection | Applied on every invocation (first and re-invocations) — worker type contract `T` must be consistent across all invocations |
| CBR retrieval | First activation only |
| Signal settlement | Scoped workers don't participate — they outlive individual signals |
| Action risk classifier | Per output application, same as current |

## YAML Schema

```yaml
bindings:
  - name: case-monitor
    capability: monitoring
    trigger: scope-activated
    lifecycleScope: CASE
    participation: COMPANION
    executionMode: PERSISTENT

  - name: analyst
    capability: analysis
    trigger:
      contextChange: ".request != null"
    lifecycleScope: COMPOUND
    participation: PARTICIPANT
    executionMode: REINVOKED

  - name: normal-worker
    capability: processing
    trigger:
      contextChange: ".input != null"
    # all defaults: BINDING / PARTICIPANT / TRANSIENT
```

`CaseDefinitionYamlMapper` parses `lifecycleScope`, `participation`, `executionMode` from binding nodes. `trigger: scope-activated` creates `ScopeActivatedTrigger`. All default to current behavior when absent.

## Persistence

### In-memory sessions (v1)

Scoped worker sessions are in-memory only, consistent with `CaseInstanceCache` and `BlackboardRegistry`.

On JVM restart:
- Persistent sessions lost. Recovery: `WorkerRecoveryCoordinator` detects RUNNING PlanItems with scoped bindings, re-creates sessions, re-dispatches workers (fresh start, no mailbox history).
- Reinvoked sessions lose accumulated state. Recovery re-creates with empty state. Next context change re-invokes with current context only.
- Durable accumulated state is a follow-on — requires `CaseContextStore` durability (engine#732).

### PlanItem persistence

One new field on `PlanItemRecord`/`PlanItemEntity`: `lifecycle_scope` (VARCHAR, nullable — null = BINDING).

`execution_mode` and `participation` are NOT persisted on `PlanItemRecord`. They are definitional properties of the `Binding` in `CaseDefinition`, derivable during recovery via the already-persisted `bindingName` and `CaseDefinitionRegistry`. `lifecycle_scope` serves as a recovery marker for filtering queries (`WHERE lifecycle_scope IS NOT NULL AND status = 'RUNNING'`). Case definitions are immutable once registered — no stale-data risk from deriving at recovery time.

### EventLog metadata

Worker schedule events for scoped workers carry:
- `lifecycleScope`: COMPOUND or CASE
- `participation`: COMPANION or PARTICIPANT
- `executionMode`: PERSISTENT or REINVOKED
- `scopeId`: compound ID or case ID

## Module Placement

| Type | Module | Package |
|------|--------|---------|
| `LifecycleScope` | `engine-api` | `io.casehub.api.model` |
| `Participation` | `engine-api` | `io.casehub.api.model` |
| `ExecutionMode` | `engine-api` | `io.casehub.api.model` |
| `ScopeActivatedTrigger` | `engine-api` | `io.casehub.api.model` |
| `ScopedWorkerSession` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ScopedWorkerRegistry` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ContextEvent` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ScopedWorkerTerminationHandler` | `runtime` | `io.casehub.engine.internal.engine.handler` |
| `CompoundActivatedEvent` | `planning` | `io.casehub.engine.planning.event` |
| `CompoundActivatedEventHandler` | `runtime` | `io.casehub.engine.internal.engine.handler` |

### Changes to `casehubio/worker`

The persistent execution model requires a new `WorkerFunction` variant and a new `WorkerOutcome` permit:

```java
// New WorkerFunction variant for persistent execution
record Persistent<T>(Class<T> inputType,
    Consumer<PersistentScope<T>> handler) implements WorkerFunction<T, Void> {
    @Override public Class<Void> outputType() { return Void.class; }
}

// PersistentScope extends WorkerScope with mailbox access and shutdown signaling
interface PersistentScope<T> extends WorkerScope {
    T nextEvent() throws ScopeTerminatedException;  // blocking take; throws on shutdown
    void emit(Map<String, Object> output);           // async fire-and-forget to event bus
}

// Unchecked exception thrown by nextEvent() on engine-initiated shutdown
class ScopeTerminatedException extends RuntimeException {}

// New WorkerOutcome permit for lifecycle completion signaling
record Completed<R>() implements WorkerOutcome<R> {}

// New method on WorkerScope for REINVOKED accumulated state access
// In WorkerScope interface:
Map<String, Object> accumulatedState();  // empty for TRANSIENT, prior output for REINVOKED
```

`WorkerFunction.Persistent` workers run on a virtual thread managed by the engine. The handler receives a `PersistentScope` with `nextEvent()` (blocking take; throws `ScopeTerminatedException` on shutdown) and `emit()` (async fire-and-forget to the event bus). Natural return signals lifecycle completion.

`WorkerOutcome.Completed` extends the sealed `WorkerOutcome` hierarchy. `TRANSIENT` workers returning `Completed` is a validation error. Exhaustive switches on `WorkerOutcome` will require a new case — the breakage is intentional.

#### Cascading changes from `WorkerOutcome.Completed`

- `OutcomeKind` (engine-api): add `COMPLETED` value. `fromWorkerOutcome()` maps `WorkerOutcome.Completed` → `OutcomeKind.COMPLETED`. `isTerminal()` returns `false` for `COMPLETED` (same as `SUCCESS` — it is not a failure).
- `WorkflowExecutionCompletedHandler.handleSemanticFailure()` (runtime): add `case WorkerOutcome.Completed` → throw `IllegalStateException` (same pattern as `Success` — `Completed` should not reach failure handling).
- `PlanItemCompletionHandler.onWorkerFinished()` (planning): extend the `Success` check to also accept `Completed` — both trigger PlanItem completion. The PlanItem's `TaskStatus.COMPLETED` is the authoritative completion signal; `evaluateCompletion` reads it via `latestByBinding.get(bindingName).getStatus()`. No separate session-level flag is needed.

No changes to `casehub-engine-common`, `casehubio/blocks`, or `casehubio/platform`.

## Cross-Repo Impact

| Repo | Impact |
|------|--------|
| `casehubio/engine` | All engine changes |
| `casehubio/worker` | `WorkerFunction.Persistent`, `PersistentScope`, `ScopeTerminatedException`, `WorkerOutcome.Completed`, `WorkerScope.accumulatedState()` |
| `casehubio/blocks` | None |
| `casehubio/platform` | None |
| Consumer repos | None until they opt into scoped bindings |

## Follow-On Issues

Filed as GitHub issues against `casehubio/engine`:

- **engine#TBD** — Durable accumulated state for reinvoked sessions (depends on engine#732)
- **engine#TBD** — Persistent session recovery with mailbox replay from EventLog
- **engine#TBD** — External worker (WorkerFunction.None) lifecycle scope — Qhorus channel lifetime scoping
- **engine#TBD** — YAML schema validation tooling for scope/participation/trigger consistency
