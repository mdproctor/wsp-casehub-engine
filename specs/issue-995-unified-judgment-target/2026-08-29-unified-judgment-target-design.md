# Unified JudgmentTarget, JudgmentEscalator SPI, and DAG Integration

**Issues:** #995 (unified target), #999 (escalator SPI), #1000 (DAG/SWF integration), #957 (react tests)
**Parent:** #994 (governed yield epic)
**Depends on:** #996 (JudgmentScheduler SPI — landed), #997 (JudgmentVerifier SPI — landed)
**Date:** 2026-08-29

## Problem

The engine has two parallel yield mechanisms — `HumanTaskTarget` (humans via WorkItems) and `JudgmentTarget` (any caller via JudgmentScheduler). These share the same control flow (formulate request → route → wait → verify → apply) but have separate dispatch paths, separate schedulers, separate completion handlers, and no shared verification or provenance. The governed yield vision (#994) says "the engine doesn't care who the caller is" — but today it cares deeply, with caller type baked into the BindingTarget type hierarchy.

Additionally, the escalation path (#999) logs but takes no action, and judgment yields cannot participate in DAG execution (#1000).

## Architecture

### Part 1: Unified JudgmentTarget (#995)

#### Design principle

Every yield separates three concerns:
1. **WHAT** to ask — prompt, evidence requirements, verification strategy, mappings, deadlines (yield semantics)
2. **WHO** to ask — candidate groups, template refs, trust thresholds (routing hints)
3. **HOW** to verify — verifier strategy, escalator strategy (post-response processing)

`HumanTaskTarget` mixed all three. The unified design separates them.

#### JudgmentTarget — the unified yield target

`JudgmentTarget` becomes the ONLY yield target in `BindingTarget`. All fields:

**Yield semantics (already present from #996/#997):**
- `prompt` / `promptExpression` — what to ask
- `inputMapping` / `outputMapping` — data flow
- `expiresIn` / `expiresInExpression` — relative deadline (Duration)
- `expiresAtExpression` — absolute deadline from case context (JQ → Instant)
- `evidenceRequirements` — required evidence keys
- `verifierStrategy` — post-response verification
- `resolutionType` — typed response validation

**Yield semantics (moved from HumanTaskTarget):**
- `title` / `titleExpression` — presentation label (useful for any caller, not just humans)
- `outcomes` — allowed response values (constrains response shape for any caller)
- `scope` / `scopeExpression` — SLA policy
- `priority` — request priority

**New fields:**
- `trustThreshold` — minimum trust level for caller selection (nullable String — ordinal category, consistent with `VerificationResult.TrustTooLow.requiredLevel` and `EscalationDecision.RouteHigher.minimumTrustLevel`)
- `escalatorStrategy` — post-verification-failure handling (nullable String, NamedStrategy ID)
- `routingConfig` — caller-type-specific routing hints (nullable `RoutingConfig`)

#### RoutingConfig — sealed interface for caller-specific routing

```java
public sealed interface RoutingConfig permits HumanRoutingConfig {}

public record HumanRoutingConfig(
    String templateRef,
    CandidateSetSpec candidateGroups,
    CandidateSetSpec candidateUsers,
    Integer claimDeadlineHours,
    Class<?> payloadType
) {}
```

- `JudgmentTarget` with `HumanRoutingConfig` → human answers
- `JudgmentTarget` with null routingConfig → scheduler picks caller
- Future sealed permits: `LlmRoutingConfig`, `A2ARoutingConfig` — no target changes

#### HumanTaskTarget deletion from BindingTarget

`HumanTaskTarget` is removed from `BindingTarget` sealed permits and `implements BindingTarget` is stripped:

```java
public sealed interface BindingTarget
    permits CapabilityTarget, SubCaseTarget, JudgmentTarget, SignalTarget, ExtensionTarget {}
```

`HumanTaskTarget` is **retained as an internal data carrier** for the scheduler layer. `HumanTaskScheduleRequest` keeps its `HumanTaskTarget target` field, and all `HumanTaskScheduler` implementations (`CloudEventHumanTaskScheduler`, `HumanTaskScheduleHandler`) remain unchanged. `DelegatingJudgmentScheduler.toHumanRequest()` constructs a `HumanTaskTarget` from `JudgmentTarget` + `HumanRoutingConfig` fields.

Changes:
- `HumanTaskTarget.java` — `implements BindingTarget` removed; class retained as scheduler-layer data carrier
- `Binding.Builder.humanTask()` — deleted
- `case HumanTaskTarget` — removed from all 9 switch/instanceof sites:
  1. `CaseContextChangedEventHandler:379` — publishByTarget dispatch
  2. `PlanningCasePlanModelSnapshotProvider:180` — mapTargetType
  3. `BindingExecutorResolver:48` — resolve
  4. `QuartzWorkerExecutionManager:372` — createTriggerJobData
  5. `PlanItemCompletionApplier:219` (planning) — applyOutputMapping
  6. `PlanItemCompletionApplier:190` (engine-adapter) — applyOutputMapping
  7. `SchedulerService:124` — registerScheduledTriggers
  8. `SchedulerService:244` — createJobData
  9. `CbrCaseRetainObserver:266` — buildRoutingKeyMap
- `BindingDeserializer:226` — `deserializeHumanTask()` updated to produce `JudgmentTarget` with `HumanRoutingConfig`
- `publishHumanTaskSchedule()` — logic absorbed into `publishJudgmentSchedule()`
- `humanTask:` YAML block — replaced by `judgment:` with `human:` sub-block
- `HumanTaskTargetTest.java`, `HumanTaskTargetDispatchTest.java` — migrated to JudgmentTarget

#### DelegatingJudgmentScheduler

Replaces `NoOpJudgmentScheduler`. `@DefaultBean @ApplicationScoped`:

```java
public class DelegatingJudgmentScheduler implements JudgmentScheduler {
    @Inject Instance<HumanTaskScheduler> humanTaskScheduler;

    @Override
    public void schedule(JudgmentScheduleRequest request) {
        if (request.target().routingConfig() instanceof HumanRoutingConfig hrc
            && humanTaskScheduler.isResolvable()) {
            humanTaskScheduler.get().schedule(toHumanRequest(request, hrc));
            return;
        }
        // No generic scheduler configured — log and skip
    }
}
```

`HumanTaskScheduler` SPI and all its implementations (`CloudEventHumanTaskScheduler`, work-adapter's `HumanTaskScheduleHandler`) are preserved unchanged. They receive requests through the delegating scheduler.

`toHumanRequest()` maps `JudgmentScheduleRequest` + `HumanRoutingConfig` → `HumanTaskScheduleRequest` by extracting the human-specific fields.

#### JudgmentScheduleRequest update

Gains fields from the unified target that the scheduler needs:

```java
public record JudgmentScheduleRequest(
    UUID caseId,
    String tenancyId,
    String bindingName,
    JudgmentTarget target,        // full target — scheduler reads routingConfig, prompt, etc.
    Map<String, Object> inputData,
    String resolutionTypeName,
    @Nullable Instant expiresAtDeadline,
    @Nullable Instant caseBudgetDeadline,    // new — from PropagationContext
    @Nullable String resolvedTitle,          // new — resolved from title/titleExpression
    @Nullable String resolvedScope,          // new — resolved from scope/scopeExpression
    List<RetrievedExperience> experiences,    // new — from CBR retrieval
    Map<String, Double> candidateScores      // new — from HumanTaskRouting
) {}
```

`caseBudgetDeadline` is propagated from `PropagationContext.getDeadline()` — schedulers use it to select the earliest of task and budget deadlines. The experiences and candidateScores fields are needed because the unified dispatch path now handles human yields that previously got these from `publishHumanTaskSchedule()`.

#### YAML schema

```yaml
# Human yield (replaces humanTask: block)
judgment:
  prompt: "Review this transaction for compliance"
  title: "Compliance Review"
  outcomes: [approve, reject, escalate]
  expiresIn: PT4H
  evidenceRequirements: [rationale]
  verifierStrategy: evidence-presence
  human:
    candidateGroups: ["compliance-team"]
    templateRef: compliance-review-template
    claimDeadlineHours: 2

# Non-human yield (unchanged from #996)
judgment:
  prompt: "Assess risk level"
  verifierStrategy: evidence-presence
  evidenceRequirements: [riskScore, rationale]
```

#### publishJudgmentSchedule() — unified dispatch

The existing `publishJudgmentSchedule()` in `CaseContextChangedEventHandler` expands to handle both human and non-human yields. The method always:
1. Evaluates `inputMapping` if present → `inputData`
2. Resolves `promptExpression` if present → `resolvedPrompt`
3. Resolves deadline — `expiresIn`/`expiresInExpression` (relative, mutually exclusive) and `expiresAtExpression` (absolute) are **independent concerns**. When both produce an Instant, pick the earliest. This is consistent with `caseBudgetDeadline` resolution in the existing `CloudEventHumanTaskScheduler`. The builder enforces mutual exclusivity only between `expiresIn` and `expiresInExpression` (same concept, different expression modes); `expiresAtExpression` is composable with either. → `expiresAtDeadline`
4. Resolves `caseBudgetDeadline` from `PropagationContext.getDeadline()`
5. Resolves `title` / `titleExpression` → `resolvedTitle`
6. Resolves `scope` / `scopeExpression` → `resolvedScope`

When `target.routingConfig() instanceof HumanRoutingConfig hrc`, additionally:
7. Resolves `hrc.candidateGroups()` / `hrc.candidateUsers()` via `CandidateSetSpec` → `resolvedGroups`, `resolvedUsers`
8. Runs `HumanTaskRoutingStrategy` from `caseDefinition.getHumanTaskRouting()` → `HumanTaskRoutingResult` → `finalGroups`, `finalUsers`, `candidateScores`
9. Evaluates bridge validation for `hrc.payloadType()` against `inputData`

All computed values are passed to `JudgmentScheduleRequest`:
```java
new JudgmentScheduleRequest(
    caseId, tenancyId, bindingName, target, inputData,
    resolutionTypeName, expiresAtDeadline, caseBudgetDeadline,
    resolvedTitle, resolvedScope, experiences, candidateScores)
```

The `DelegatingJudgmentScheduler` then maps from `JudgmentScheduleRequest` → `HumanTaskScheduleRequest` when `routingConfig instanceof HumanRoutingConfig`, populating the `HumanTaskTarget` data carrier from `JudgmentTarget` + `HumanRoutingConfig` fields.

When `routingConfig` is null:
1. Steps 1–6 above, then dispatches to `JudgmentScheduler`

#### Cross-repo migration

Executed in a **multi-repo work slot** with IntelliJ workspace covering: engine, work, clinical, devtown, life, soc, examples, fsitrading. All refactoring via `ide_refactor_rename`, `ide_find_references`, `ide_move_file` — never grep/find-and-replace.

Consumer changes:
- Java DSL: `Binding.builder().humanTask(HumanTaskTarget.inline()...)` → `Binding.builder().judgment(JudgmentTarget.builder().human(HumanRoutingConfig...)...)`
- YAML: `humanTask:` → `judgment:` with `human:` sub-block

### Part 2: JudgmentEscalator SPI (#999)

#### JudgmentEscalator — NamedStrategy SPI

**Package:** `io.casehub.api.spi.judgment`

```java
public interface JudgmentEscalator extends NamedStrategy {
    EscalationDecision escalate(EscalationContext context);

    @Override default String id() { return "fault"; }
}
```

#### EscalationDecision — sealed result

```java
public sealed interface EscalationDecision {
    record ReYield(String feedback) implements EscalationDecision {}
    record RouteHigher(String minimumTrustLevel) implements EscalationDecision {}
    record Fault(String reason) implements EscalationDecision {}
}
```

#### EscalationContext

```java
public record EscalationContext(
    UUID caseId,
    String tenancyId,
    String bindingName,
    JudgmentTarget target,
    JudgmentResponse failedResponse,
    VerificationResult verificationResult,
    int escalationCount,
    int maxEscalations,
    CaseDefinition definition
) {}
```

`escalationCount` prevents infinite re-yield loops. `maxEscalations` defaults to 3, configurable per case.

#### Built-in strategies

**`FaultEscalator`** — `@DefaultBean @ApplicationScoped`, id=`"fault"`. Always returns `Fault(reason)`. Safe default.

**`ReYieldEscalator`** — `@ApplicationScoped`, id=`"re-yield"`. Returns `ReYield(feedback)` when `escalationCount < maxEscalations`, `Fault` when exhausted.

#### Handler integration

`JudgmentEscalationHandler` (already exists from #997) gains escalator resolution:
1. Queries `eventLogRepository.findByCaseAndTypes(caseId, [JUDGMENT_ESCALATED], tenancyId)`, filters by `bindingName` in metadata → `escalationCount`
2. Reads `maxEscalations` from `CaseDefinitionSpec` (default 3 if null)
3. Resolves `JudgmentEscalator` via `EngineStrategyResolver` from `JudgmentTarget.escalatorStrategy()`
4. Calls `escalator.escalate(context)` with `escalationCount` and `maxEscalations`
5. Writes `JUDGMENT_ESCALATED` EventLog with decision outcome in metadata (after escalator decision, not before — so escalationCount reflects prior events only)
6. On `ReYield`: calls `planItem.tryMarkReDispatching()` (DELEGATED → DISPATCHING), then re-publishes judgment request with feedback appended to prompt context through `JudgmentScheduler.schedule()` — the scheduler's `markDelegated()` succeeds from DISPATCHING as normal
7. On `RouteHigher`: same PlanItem transition as ReYield, then re-publishes with elevated trust threshold on the target
8. On `Fault`: marks PlanItem FAULTED, writes `_diagnostics`

**PlanItem re-dispatch transition:** `PlanItem.tryMarkReDispatching()` — CAS-based `DELEGATED → DISPATCHING` transition. Follows the existing revert pattern (`revertDispatching()`: DISPATCHING → PENDING, `revertEscalated()`: ESCALATED → PENDING). Semantically correct: the PlanItem was delegated, but the delegation is being retried with updated parameters. The intermediate DISPATCHING state allows downstream schedulers to call `markDelegated()` normally without weakening the state machine's safety guarantees.

**Escalation count tracking:** `escalationCount` is derived from `EventLogRepository` — count of `JUDGMENT_ESCALATED` entries for `(caseId, bindingName)`. Durable across restarts, consistent with the event-sourced audit trail. The EventLog write (step 5) occurs after the escalator decision (step 4), so the count passed to the escalator reflects only prior escalations. With `maxEscalations=3`: the `ReYieldEscalator` checks `escalationCount < maxEscalations`, allowing escalation counts 0, 1, 2 (three re-yields total) and faulting at count 3.

#### CaseDefinitionSpec gains `maxEscalations`

`CaseDefinitionSpec.maxEscalations` (Integer, nullable — default 3). YAML: `spec: { maxEscalations: 3 }`. Placed on `CaseDefinitionSpec` alongside peer operational limits `maxDecompositionDepth` and `maxAdaptations`.

### Part 3: DagNode Judgment Integration (#1000)

#### Approach

DagDriver's `Function<T, R>` executor is synchronous. A judgment node publishes the request and blocks the virtual thread on a `CompletableFuture<JudgmentResponse>` that resolves when `JudgmentCompletedHandler` receives the response.

#### JudgmentNodeExecutor

`@ApplicationScoped`. Uses a `BlockingQueue<JudgmentNodeResult>` per pending judgment to support per-cycle timeouts across escalation re-yields.

```java
@ApplicationScoped
public class JudgmentNodeExecutor {
    private final ConcurrentHashMap<String, BlockingQueue<JudgmentNodeResult>> pending;

    public JudgmentResponse execute(JudgmentTarget target, Map<String, Object> input,
                                     UUID caseId, String bindingName, Duration perCycleTimeout) {
        BlockingQueue<JudgmentNodeResult> queue = new LinkedBlockingQueue<>();
        String key = caseId + ":" + bindingName;
        pending.put(key, queue);
        publishJudgmentYield(target, input, caseId, bindingName);
        try {
            while (true) {
                JudgmentNodeResult result = queue.poll(perCycleTimeout.toMillis(), MILLISECONDS);
                if (result == null) {
                    throw new JudgmentTimeoutException(bindingName, perCycleTimeout);
                }
                switch (result) {
                    case JudgmentNodeResult.Completed(var response) -> { return response; }
                    case JudgmentNodeResult.ReYielded() -> { /* timeout resets — loop */ }
                    case JudgmentNodeResult.Faulted(var reason) ->
                        throw new JudgmentFaultException(bindingName, reason);
                }
            }
        } finally {
            pending.remove(key);
        }
    }
}
```

`JudgmentNodeResult` is a sealed interface:
```java
sealed interface JudgmentNodeResult {
    record Completed(JudgmentResponse response) implements JudgmentNodeResult {}
    record ReYielded() implements JudgmentNodeResult {}
    record Faulted(String reason) implements JudgmentNodeResult {}
}
```

This follows the `WorkerRuntime.awaitCase()` pattern — cheap blocking on virtual threads — but with per-cycle timeout that resets on each escalation re-yield.

#### Completion protocol

`JudgmentNodeExecutor` is `@Inject`ed into both `JudgmentCompletedHandler` and `JudgmentEscalationHandler`. The protocol:

| Verification outcome | Handler action | JudgmentNodeExecutor notification |
|---|---|---|
| `Accepted` | Apply output mapping → fire CONTEXT_CHANGED | Enqueue `Completed(response)` — **after** output mapping is applied |
| `InsufficientEvidence` | Publish JUDGMENT_ESCALATED | (escalation handler enqueues `ReYielded()` on re-yield) |
| `TrustTooLow` | Publish JUDGMENT_ESCALATED | (escalation handler enqueues `ReYielded()` on re-yield) |
| `Rejected` | Write `_diagnostics` → fire CONTEXT_CHANGED | Enqueue `Faulted(reason)` |
| (no binding / not JudgmentTarget) | Write event log → fire CONTEXT_CHANGED | Enqueue `Completed(response)` |

The last row covers **SWF/DAG judgment yields** where `call: casehub:judgment` runs inside a worker's Serverless Workflow. The enclosing binding is a `CapabilityTarget` (not `JudgmentTarget`), so binding lookup either finds no binding or a non-judgment binding. In this case, verification is not run at the binding level — the response is accepted directly and the DAG thread is unblocked. The `pending.containsKey(key)` check and `Completed(response)` enqueue must be placed **outside** the `instanceof JudgmentTarget` conditional in `JudgmentCompletedHandler`, so SWF judgment responses always reach the DAG executor.

Escalation handler notifications:
| Escalation decision | JudgmentNodeExecutor notification |
|---|---|
| `ReYield` | Enqueue `ReYielded()` — per-cycle timeout resets |
| `RouteHigher` | Enqueue `ReYielded()` — per-cycle timeout resets |
| `Fault` | Enqueue `Faulted(reason)` |

The handlers check `pending.containsKey(key)` before enqueuing — if no DAG thread is waiting (non-DAG judgment yield), the notification is a no-op. This keeps the DAG integration transparent to non-DAG code paths.

#### SWF integration

`call: casehub:judgment` callable task type in the flow module. A `JudgmentCallableDispatcher implements CallableDispatcher` is registered in `CallableDispatchRegistry` for the `"casehub:judgment"` call name. The existing `CasehubCallableTaskBuilder` already accepts all `CallFunction` instances and delegates to `CallableDispatchRegistry` — no new `CallableTaskBuilder` is needed. `JudgmentCallableDispatcher` delegates to `JudgmentNodeExecutor`.

### Part 4: React Module Integration Tests (#957)

Two `@QuarkusTest` classes, independent of the judgment generalization:

**`ReActExecutionIntegrationTest`:**
- Case with react worker → start → handler runs → tool calls dispatch → final answer → case completes
- Mock `ChatModelProvider` with canned tool-use responses
- Verify case reaches COMPLETED

**`ReActAuditTrailTest`:**
- Verify `REACT_CYCLE` EventLog entries are queryable, ordered, and contain complete reasoning traces
- Verify `WORKER_EXECUTION_COMPLETED` carries react protocol metadata with aggregated token counts

Test pattern follows `casehub-engine-a2a` integration tests. Test config at `react/src/test/resources/application.properties` already exists.

## Test strategy

### Engine-side tests (this branch)
- **Unit:** JudgmentTarget builder with all new fields (title, outcomes, scope, priority, trustThreshold, escalatorStrategy, routingConfig)
- **Unit:** HumanRoutingConfig record construction
- **Unit:** RoutingConfig sealed type
- **Unit:** DelegatingJudgmentScheduler — delegates to HumanTaskScheduler on HumanRoutingConfig, no-ops on null
- **Unit:** EscalationDecision sealed type, EscalationContext record
- **Unit:** FaultEscalator — always Fault
- **Unit:** ReYieldEscalator — ReYield under max, Fault when exhausted
- **YAML:** Parse `judgment:` with `human:` sub-block → JudgmentTarget with HumanRoutingConfig
- **YAML:** Parse `judgment:` without `human:` → JudgmentTarget with null routingConfig
- **Integration:** JudgmentTarget with HumanRoutingConfig dispatches to HumanTaskScheduler via DelegatingJudgmentScheduler
- **Integration:** JudgmentEscalationHandler resolves escalator, executes ReYield
- **Unit:** JudgmentNodeExecutor — Completed unblocks thread, ReYielded resets timeout and loops, Faulted throws JudgmentFaultException, timeout throws JudgmentTimeoutException, cleanup in finally block
- **Unit:** JudgmentCallableDispatcher — dispatch delegates to JudgmentNodeExecutor, CompletableFuture result mapping
- **Unit:** JudgmentNodeResult sealed type exhaustiveness
- **Integration:** SWF judgment call end-to-end — `call: casehub:judgment` → response → executor unblocks
- **Integration:** Non-binding completion path — SWF judgment response reaches executor without JudgmentTarget binding (R2-04 path)
- **Integration:** Escalation re-yield through JudgmentNodeExecutor — re-yield resets per-cycle timeout, PlanItem re-dispatch transition
- **Integration:** escalationCount via EventLog — count tracks correctly across re-yield cycles
- **Integration:** React execution end-to-end (#957)
- **Integration:** React audit trail (#957)

### Work slot tests (cross-repo)
- **Compile:** All consumer repos compile with unified JudgmentTarget
- **Integration:** Existing HumanTask tests in domain repos pass with migrated bindings
- **YAML:** Migrated YAML definitions parse correctly

## Module impact

| Module | Change |
|--------|--------|
| `engine-api` | JudgmentTarget gains fields (title, outcomes, scope, priority, trustThreshold, escalatorStrategy, routingConfig, expiresAtExpression), RoutingConfig/HumanRoutingConfig, JudgmentEscalator/EscalationDecision/EscalationContext, HumanTaskTarget `implements BindingTarget` removed (class retained as scheduler data carrier), BindingTarget permits updated |
| `engine-common` | JudgmentScheduleRequest gains caseBudgetDeadline, resolvedTitle, resolvedScope, experiences, candidateScores |
| `runtime` | DelegatingJudgmentScheduler replaces NoOp, publishJudgmentSchedule absorbs human dispatch (conditional on HumanRoutingConfig), JudgmentEscalationHandler gains escalator resolution, EngineStrategyResolver gains `@Any Instance<JudgmentEscalator>` constructor parameter + `registerStrategies(judgmentEscalators)` call, FaultEscalator, ReYieldEscalator, JudgmentNodeExecutor (@ApplicationScoped), JudgmentNodeResult, JudgmentCallableDispatcher |
| `planning` | PlanningCasePlanModelSnapshotProvider: `case JudgmentTarget jt -> jt.routingConfig() instanceof HumanRoutingConfig ? "HUMAN" : "JUDGMENT"`, PlanItem gains `tryMarkReDispatching()` (DELEGATED → DISPATCHING) |
| `scheduler-quartz` | QuartzWorkerExecutionManager switch update (remove HumanTaskTarget case) |
| `work-cloudevent` | CloudEventHumanTaskScheduler unchanged (receives via delegation) |
| `react` | New integration tests (#957) |
| `flow` | JudgmentCallableDispatcher registered in CallableDispatchRegistry for `"casehub:judgment"` |

## Execution plan

1. **This branch:** Engine-side changes — unified JudgmentTarget, delete HumanTaskTarget, JudgmentEscalator SPI, JudgmentNodeExecutor, DelegatingJudgmentScheduler, React tests. All engine tests pass.
2. **Work slot:** Multi-repo slot with engine + work + clinical + devtown + life + soc + examples + fsitrading. IntelliJ workspace refactoring migrates all consumers.

## Out of scope

- LlmRoutingConfig / A2ARoutingConfig — future sealed permits when those caller types ship
- Trust score integration for trustThreshold — qhorus E4
- Context-aware redistribution escalator — qhorus E8
- Consensus verification (M-of-N) — blocks concern
- ActionGate unification into JudgmentTarget — separate analysis needed (different lifecycle: pre-action vs post-response)

## References

- `HumanTaskTarget.java` — type being deleted (16+ fields, 390 lines)
- `CaseContextChangedEventHandler.java:685-815` — publishHumanTaskSchedule (logic absorbed)
- `BindingTarget.java:27` — sealed permits
- `DagDriver.java:74-93` — execute method (Function<T,R> contract)
- `WorkerRuntime.awaitCase()` — CompletableFuture blocking pattern
- `JudgmentVerifier.java` — NamedStrategy SPI precedent (for JudgmentEscalator)
- `JudgmentCompletedHandler.java` — verification gate (escalation integration point)
- `JudgmentEscalationHandler.java` — current log-only handler (gains escalator resolution)
- Cross-repo impact: work (7 files), clinical (10), devtown (11), life (27), soc (2), examples (3), fsitrading (3)
- PP-20260723-c4c1cf — virtual-thread-handler-convention
- PP-20260727-5267d2 — plan-type-module-boundary
- Epic #994 — governed yield vision
- #995, #999, #1000, #957 — this branch's issues
