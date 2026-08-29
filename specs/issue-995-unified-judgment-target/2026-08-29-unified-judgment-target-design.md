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
- `expiresIn` / `expiresInExpression` — deadline
- `evidenceRequirements` — required evidence keys
- `verifierStrategy` — post-response verification
- `resolutionType` — typed response validation

**Yield semantics (moved from HumanTaskTarget):**
- `title` / `titleExpression` — presentation label (useful for any caller, not just humans)
- `outcomes` — allowed response values (constrains response shape for any caller)
- `scope` / `scopeExpression` — SLA policy
- `priority` — request priority

**New fields:**
- `trustThreshold` — minimum trust level for caller selection (nullable Double, from #995 spec)
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

#### HumanTaskTarget deletion

`HumanTaskTarget` is removed from `BindingTarget` sealed permits:

```java
public sealed interface BindingTarget
    permits CapabilityTarget, SubCaseTarget, JudgmentTarget, SignalTarget, ExtensionTarget {}
```

- `HumanTaskTarget.java` — deleted
- `Binding.Builder.humanTask()` — deleted
- `case HumanTaskTarget` — removed from all 6 switch sites
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
    Instant expiresAtDeadline,
    String resolvedTitle,         // new — resolved from title/titleExpression
    String resolvedScope,         // new — resolved from scope/scopeExpression
    List<RetrievedExperience> experiences,    // new — from CBR retrieval
    Map<String, Double> candidateScores      // new — from HumanTaskRouting
) {}
```

The experiences and candidateScores fields are needed because the unified dispatch path now handles human yields that previously got these from `publishHumanTaskSchedule()`.

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

The existing `publishJudgmentSchedule()` in `CaseContextChangedEventHandler` expands to handle both human and non-human yields. When `routingConfig` is `HumanRoutingConfig`:
1. Resolves candidate groups/users via `CandidateSetSpec`
2. Expands group membership via `GroupMembershipProvider`
3. Runs `HumanTaskRoutingStrategy` (CBR, constraint) if configured
4. Resolves title, scope from expressions
5. Evaluates bridge validation for payloadType
6. Creates PlanItem (PENDING → DELEGATED via `markDelegated()`)

When `routingConfig` is null:
1. Evaluates input mapping
2. Resolves prompt expression
3. Creates PlanItem
4. Dispatches to `JudgmentScheduler`

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
1. Resolves `JudgmentEscalator` via `EngineStrategyResolver` from `JudgmentTarget.escalatorStrategy()`
2. Calls `escalator.escalate(context)`
3. On `ReYield`: re-publishes judgment request with feedback appended to prompt context
4. On `RouteHigher`: re-publishes with elevated trust threshold on the target
5. On `Fault`: marks PlanItem FAULTED, writes `_diagnostics`

#### CaseDefinition gains `maxEscalations`

`CaseDefinition.maxEscalations` (Integer, nullable — default 3). YAML: `spec: { maxEscalations: 3 }`.

### Part 3: DagNode Judgment Integration (#1000)

#### Approach

DagDriver's `Function<T, R>` executor is synchronous. A judgment node publishes the request and blocks the virtual thread on a `CompletableFuture<JudgmentResponse>` that resolves when `JudgmentCompletedHandler` receives the response.

#### JudgmentNodeExecutor

```java
public class JudgmentNodeExecutor {
    private final ConcurrentHashMap<String, CompletableFuture<JudgmentResponse>> pending;

    public JudgmentResponse execute(JudgmentTarget target, Map<String, Object> input,
                                     UUID caseId, String bindingName, Duration timeout) {
        CompletableFuture<JudgmentResponse> future = new CompletableFuture<>();
        String key = caseId + ":" + bindingName;
        pending.put(key, future);
        // Publish judgment request via event bus
        publishJudgmentYield(target, input, caseId, bindingName);
        try {
            return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            pending.remove(key);
            throw new JudgmentTimeoutException(bindingName, timeout);
        }
    }

    // Called by JudgmentCompletedHandler
    public void complete(UUID caseId, String bindingName, JudgmentResponse response) {
        String key = caseId + ":" + bindingName;
        CompletableFuture<JudgmentResponse> future = pending.remove(key);
        if (future != null) future.complete(response);
    }
}
```

This follows the `WorkerRuntime.awaitCase()` pattern — cheap blocking on virtual threads.

#### SWF integration

`call: casehub:judgment` callable task type in the flow module. `JudgmentCallableTaskBuilder implements CallableTaskBuilder<CallFunction>` handles the `casehub:judgment` call and delegates to `JudgmentNodeExecutor`.

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
- **Integration:** React execution end-to-end (#957)
- **Integration:** React audit trail (#957)

### Work slot tests (cross-repo)
- **Compile:** All consumer repos compile with unified JudgmentTarget
- **Integration:** Existing HumanTask tests in domain repos pass with migrated bindings
- **YAML:** Migrated YAML definitions parse correctly

## Module impact

| Module | Change |
|--------|--------|
| `engine-api` | JudgmentTarget gains fields, RoutingConfig/HumanRoutingConfig, JudgmentEscalator/EscalationDecision/EscalationContext, HumanTaskTarget DELETED, BindingTarget permits updated |
| `engine-common` | JudgmentScheduleRequest gains fields |
| `runtime` | DelegatingJudgmentScheduler replaces NoOp, publishJudgmentSchedule absorbs human dispatch, JudgmentEscalationHandler gains escalator resolution, EngineStrategyResolver gains Instance<JudgmentEscalator>, FaultEscalator, ReYieldEscalator, JudgmentNodeExecutor |
| `planning` | PlanningCasePlanModelSnapshotProvider switch update (remove HumanTaskTarget case) |
| `scheduler-quartz` | QuartzWorkerExecutionManager switch update |
| `work-cloudevent` | CloudEventHumanTaskScheduler unchanged (receives via delegation) |
| `react` | New integration tests (#957) |
| `flow` | JudgmentCallableTaskBuilder for `call: casehub:judgment` |

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
