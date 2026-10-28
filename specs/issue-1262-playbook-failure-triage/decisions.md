# Decisions — issue-1262-playbook-failure-triage

## D1: PlaybookExecutionResult scope

**Choice:** Playbook-only — only FlowWorkerFunctionHandler produces step-level results
**Alternatives:**
- General HandlerResult extension — adds optional structured-result field to HandlerResult for any handler type. Over-generalizes when other handlers (MCP, ReAct, A2A, SyncAgent) are single-operation.
- Sealed per-handler-type results — each handler gets its own result subtype. Most complexity for little benefit.
- Handler-agnostic `StructuredExecutionResult` sealed interface with common `steps()` accessor (R1-02) — forces false equivalence between fundamentally different execution models: SWF tasks are named/positioned/modifiable, ReAct cycles are fire-and-forget to EventDispatcher, A2A statusTransitions are protocol state changes with no actionable step identity.
**Rationale:** Only flow/playbook workers have step-level execution with named, positioned tasks that triage can address (skip step, replace input). A2A tracks protocol state transitions (working → completed) in protocolMetadata — not individually addressable steps. ReAct dispatches per-cycle events to EventDispatcher — not accumulated in HandlerResult. Generalize later if needed.
**Trade-offs:** If a future handler type needs step-level triage, extend then. The TriageHandler SPI in platform-api is handler-agnostic as an interface; initial implementation is correctly playbook-specific.
**Sources:** FlowWorkerFunctionHandler.java:43, WorkerFunctionHandler.java:34, HandlerResult.java:21, A2AWorkerFunctionHandler.java:202-214, ReActWorkerFunctionHandler.java:111-131
**Exploration:** quick
**Status:** captured

## D2: Include #1263 (PlaybookWorkerFunctionProvider) in this branch

**Choice:** Yes — fold #1263 into this branch as batch 1
**Alternatives:**
- Keep them separate — delays playbook-as-worker-definition, misses the type distinction that enables richer step metadata.
**Rationale:** #1263 establishes PlaybookWorkerFunction with CompilationResult (SourceMap, CoordinationSpec). #1262's step capture and triage wiring builds on these types. The platform compiler is on 7.x (serverlessworkflow-types 7.35.3.Final), compatible with engine's 7.25.1.Final.
**Version compatibility note (from R1-04):** The compatibility boundary is the SWF JSON/YAML spec wire format, not the Java API — the compiler produces a Workflow document, the engine executes it. A compatibility test (compile with 7.35 types, execute with 7.25 runtime) is required before merging batch 1. Engine should align to the compiler's SWF types version (7.35) or the compiler should target the engine's version — decision to be made during implementation.
**Trade-offs:** Larger branch scope (S/Low + L/High).
**Sources:** casehubio/engine#1263, casehubio/platform playbook-compiler/pom.xml (serverlessworkflow-types 7.35.3.Final), engine pom.xml (version.io.serverlessworkflow=7.25.1.Final)
**Exploration:** deep-analysis
**Status:** revised (R1-04: added version compatibility note and test requirement)

## D3: Step capture mechanism

**Choice:** CDI WorkflowExecutionListener bean with step accumulation in FlowExecutionRegistry
**Alternatives:**
- Per-execution listener in handler — couples to SDK internals, requires manual registration per call.
- Wrap CompletableFuture with try-catch — loses step timing and partial-success data.
- Accumulate in SWF WorkflowInstance metadata (original choice) — couples to SWF SDK internals, requires casting WorkflowContextData to WorkflowContext to reach WorkflowMutableInstance.addMetadataIfAbsent(). FlowExecutionRegistry is cleaner.
**Rationale:** WorkflowExecutionListener provides onTaskStarted/onTaskCompleted/onTaskFailed callbacks with TaskContext (taskName, startedAt, completedAt, cause). CDI auto-discovery works in Quarkus via quarkus-flow's WorkflowApplicationCreator.injectCustomListeners() which discovers @ApplicationScoped beans implementing WorkflowExecutionListener via Instance<WorkflowExecutionListener> injection and adds them to the builder. The listener looks up the execution by instance ID in FlowExecutionRegistry (which already maps instance IDs to FlowExecution context) and accumulates StepResults there. FlowWorkerFunctionHandler reads accumulated results from the same registry after join(). No SWF metadata coupling, no casts.
**Trade-offs:** Listener is global (sees all workflow executions, not just casehub ones). Filtered by FlowExecutionRegistry lookup — non-casehub executions get O(1) ConcurrentHashMap.get() returning null.
**Sources:** WorkflowExecutionListener (SWF SDK), TaskContext (taskName, startedAt, completedAt, position), TaskFailedEvent (cause), FlowExecutionRegistry.java:26, WorkflowApplicationCreator.java:73-74 (quarkus-flow 0.13.0)
**Exploration:** quick
**Status:** revised (R1-07/R1-20: step accumulation moved from SWF metadata to FlowExecutionRegistry; R1-06: CDI registration mechanism documented precisely)

## D4: Triage insertion point

**Choice:** Inside DefaultRecoveryCoordinator.handleFailure(), after all three structural recovery levels exhaust; re-dispatch via CaseContextChangedEvent
**Alternatives:**
- After recovery exhaustion in RetryOrchestrator's Exhaust branch (original choice) — ambiguous module boundary, would require common-core to depend on TriageHandler from platform-api.
- Replace DefaultRecoveryCoordinator — simpler pipeline but loses classification-driven escalation.
- Inside each recovery level — premature at TRANSIENT level, too complex.
- As a fourth RecoveryLevel (reviewer option 2) — conflates error classification (what kind of failure?) with recovery strategy (what to do?). Triage is a strategy, not a classification level.
- Extend RecoveryCoordinator SPI with RescheduleCallback (R2-03 option 1) — unnecessary SPI change; coordinator already injects EventBus for Level 3.
- New TriageRetryEvent type (R2-03 option 2) — unnecessary; CaseContextChangedEvent already triggers full binding re-evaluation; a separate event type is only warranted when re-dispatch needs different handling from normal context-change re-evaluation.
- PlanItem reset (R2-03 option 3) — over-specified at the design level; whether a new PlanItem is created or the faulted one is reset is an implementation detail of the blackboard module.
**Rationale:** DefaultRecoveryCoordinator already lives in the runtime module, injects CDI beans, and has access to CaseDefinition. It injects TriageHandler (from platform-api) and invokes it after all three structural recovery levels (TRANSIENT → REASONING → FUNDAMENTAL) have been tried and failed. The coordinator's boolean return value now means "recovered via either structural recovery or triage." RetryOrchestrator (common-core) calls recoveryCoordinator.handleFailure() unchanged — no module boundary violation.
**Re-dispatch mechanism (from R2-03):** The coordinator applies triage modifications to the case context's working layer, records the triage attempt in BindingRecoveryState, and publishes `CaseContextChangedEvent` to the event bus. This follows the same event-driven pattern as Level 3 recovery (which publishes `CaseContextChangedEvent` after replan). The `CaseContextChangedEventHandler.evaluateAndDispatch()` pipeline evaluates all bindings against the modified context and dispatches workers for satisfied bindings.
**Retry multiplication (from R1-11):** With maxAttempts=N and maxTriageAttempts=T, total worker executions = N × (1 + T). Example: maxAttempts=3, maxTriageAttempts=2 → 9 executions. This is intentional — triage-modified executions still need mechanical retries for transient failures that are orthogonal to the semantic modification.
**Trade-offs:** Triage never gets a chance if recovery succeeds at an earlier level. This is intentional — structural fixes are cheaper than semantic triage.
**Spec sync (from R2-04):** Spec §3.5–3.6 text still references RetryOrchestrator as the triage call site — needs updating to reflect DefaultRecoveryCoordinator during implementation.
**Sources:** RetryOrchestrator.java:108-121, DefaultRecoveryCoordinator.java:106-158, DefaultRecoveryCoordinator.java:260-273 (Level 3 CaseContextChangedEvent pattern), CaseContextChangedEventHandler.java:264-316 (evaluateAndDispatch), TriageHandler SPI (platform-api)
**Exploration:** quick
**Status:** revised (R1-10: moved from RetryOrchestrator to DefaultRecoveryCoordinator; R1-11: documented retry multiplication; R2-03: specified CaseContextChangedEvent re-dispatch mechanism; R2-04: noted spec text sync needed)

## D5: Triage loop cap configuration

**Choice:** On RecoveryPolicy in CaseDefinition, with per-binding overrides on RecoveryOverride
**Alternatives:**
- Global config property — no per-case customization.
- On TriageHandler SPI itself — engine loses control over loop termination.
- Case-level only, no per-binding overrides (original choice) — inconsistent with existing RecoveryOverride pattern for structural recovery.
**Rationale:** Consistent with how maxRetries and recovery levels are already configured in RecoveryPolicy. Per-case-definition granularity allows different triage budgets for different case types. Per-binding overrides on RecoveryOverride mirror the existing pattern: `skipTriage` (default false) disables triage for a binding, `maxTriageAttempts` (nullable, falls through to RecoveryPolicy default) overrides the case-level budget.
**skipRecovery interaction (from R1-13):** `skipRecovery=true` skips structural recovery but does NOT skip triage unless `skipTriage=true` independently. A binding with `skipRecovery=true, skipTriage=false` goes directly to triage after mechanical retries exhaust. This is valid for bindings where structural recovery is unhelpful but semantic triage adds value.
**Trade-offs:** Requires schema change to both RecoveryPolicy (new maxTriageAttempts field) and RecoveryOverride (new skipTriage and maxTriageAttempts fields).
**Sources:** RecoveryPolicy (engine-api), RecoveryOverride.java:20-46, DefaultRecoveryCoordinator.java:117-119
**Exploration:** quick
**Status:** revised (R1-13: added per-binding triage overrides on RecoveryOverride; documented skipRecovery interaction)

## D6: Triage retry mechanism

**Choice:** Apply typed TriageModification hierarchy + re-dispatch via CaseContextChangedEvent, with modification history tracking
**Alternatives:**
- Direct re-invocation from RecoveryCoordinator — bypasses retry counts, scheduling, and execution pipeline.
- CDI event (TriageRetryRequestedEvent) — adds event machinery, harder to follow.
- Stringly-typed Map<String, Object> modifications (original choice) — violates typed-preference-keys protocol (PP-20260517-2cd5f0).
- Sealed TriageModification in platform-api with permits clause (previous revision) — doesn't compile: sealed `permits` requires compile-time visibility, creating a dependency inversion from platform-api to engine-support-core.
**Rationale:** TriageDecision.Retry carries `List<TriageModification>` where TriageModification is a **non-sealed marker interface** in platform-api. Handler-specific subtypes use sealed hierarchies in their own modules:
```
// platform-api — non-sealed marker, no permits clause
public interface TriageModification { }

// engine-support-core — sealed within one compilation unit
public sealed interface PlaybookTriageModification extends TriageModification
    permits SkipStep, ReplaceStepInput, UpdateParameter {
    record SkipStep(String stepId) implements PlaybookTriageModification {}
    record ReplaceStepInput(String stepId, Map<String, Object> newInput) implements PlaybookTriageModification {}
    record UpdateParameter(String key, Object value) implements PlaybookTriageModification {}
}
```
The engine casts to `PlaybookTriageModification` and does exhaustive pattern matching on the sealed sub-hierarchy. Top-level exhaustiveness is not needed — the handler always works with handler-specific subtypes. Future handler types add their own sealed sub-hierarchies without platform-api changes.
**Modification history (from R1-16):** BindingRecoveryState tracks `List<List<TriageModification>> previousTriageModifications`. Each entry is the modification set applied in one triage attempt. TriageContext includes this history so the handler sees what was already tried and avoids repeating failed modifications.
**Trade-offs:** Retry count management needs care — triage retries counted separately from mechanical retries. Non-sealed marker loses compile-time exhaustiveness at the TriageDecision level, but per-handler sealed hierarchies provide it where it matters.
**Sources:** RetryOrchestrator.java:86-135, TriageDecision.Retry (platform-api), typed-preference-keys protocol (PP-20260517-2cd5f0)
**Exploration:** quick
**Status:** revised (R1-15: replaced Map<String, Object> with typed TriageModification hierarchy; R1-16: added modification history tracking; R2-02: TriageModification changed from sealed to non-sealed marker interface — sealed permits clause doesn't compile across module boundaries)

## D7: CBR recording scope

**Choice:** All playbook executions — dual-granularity with StepExecutionCbrBridge (per-action) and PlaybookOutcomeRecorder (per-execution)
**Alternatives:**
- Triage-only — focused on failure/recovery patterns but misses baseline successful executions.
- Replace StepExecutionCbrBridge with PlaybookOutcomeRecorder — loses per-action CBR granularity.
**Rationale:** Dual recording serves different CBR retrieval use cases. StepExecutionCbrBridge records individual YamlStepExecutionEvent outcomes — one CbrPlanRecord per action with actionName, durationMs, bindingType, success, and domain-specific features. PlaybookOutcomeRecorder records full execution arcs — all steps in order, which succeeded/failed, overall outcome, and triage chain context. Per-action recording answers "should step X succeed in this context?" Per-execution recording answers "have we seen this failure pattern before, and what triage modifications resolved it?"
**Trade-offs:** Higher CBR storage volume. Mitigated by CBR's existing retention policies.
**Sources:** StepExecutionCbrBridge.java:34, CbrRecordStore SPI
**Exploration:** quick
**Status:** captured

## I1: Step data propagation channel

**Choice:** FlowExecutionRegistry — engine-owned, instance-keyed, thread-safe data structure
**Alternatives:**
- SWF WorkflowInstance metadata side-channel — listener and handler communicate through a third party's mutable state. Requires casting WorkflowContextData to WorkflowContext to reach WorkflowMutableInstance.addMetadataIfAbsent(). Couples to SWF SDK internals.
**Rationale:** FlowExecutionRegistry already exists as an engine-owned ConcurrentHashMap mapping workflow instance IDs to FlowExecution context. The listener looks up the execution by instance ID and accumulates step results there. The handler reads accumulated results from the same registry after join(). No SWF metadata coupling, no casts, no assumption about WorkflowContextData implementation type.
**Trade-offs:** FlowExecution changes from a record to a class (or a record with a mutable step accumulator field). Minor — the additional complexity is justified by eliminating SDK coupling.
**Sources:** FlowExecutionRegistry.java:26, FlowExecution.java:26, WorkflowContext.java:18, WorkflowMutableInstance.java:40
**Exploration:** surfaced by reviewer (R1-07, R1-20)
**Status:** captured

## I2: PlaybookWorkerFunction and FlowWorkerFunction type relationship

**Choice:** Sealed interface SWFWorkerFunction with workflow() accessor
**Alternatives:**
- instanceof branching in FlowWorkerFunctionHandler.supports() — works but obscures the type relationship. Two function types with identical execution semantics (both produce a Workflow for WorkflowApplication) have no shared supertype.
**Rationale:** A sealed interface SWFWorkerFunction with a workflow() accessor makes the type relationship explicit. FlowWorkerFunctionHandler.supports() becomes a single `instanceof SWFWorkerFunction` check. The execute() method accesses the workflow via the shared accessor.
**Trade-offs:** Minor API change — FlowWorkerFunction and PlaybookWorkerFunction both implement SWFWorkerFunction.
**Sources:** FlowWorkerFunctionHandler.java:60-64, FlowWorkerFunction, PlaybookWorkerFunction
**Exploration:** surfaced by reviewer (R1-21)
**Status:** captured
