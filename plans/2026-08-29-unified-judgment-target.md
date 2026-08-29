# Unified JudgmentTarget Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #995 — feat: JudgmentTarget type — generalised yield target in engine-api
**Issue group:** #995, #999, #1000, #957

**Goal:** Unify HumanTaskTarget into JudgmentTarget with RoutingConfig sealed interface, add JudgmentEscalator SPI, enable judgment yields in DAG/SWF execution, and add React module integration tests.

**Architecture:** JudgmentTarget becomes the only yield target in BindingTarget. HumanTaskTarget retained as internal scheduler-layer data carrier (stripped of BindingTarget). RoutingConfig sealed interface carries caller-type-specific routing. DelegatingJudgmentScheduler bridges to existing HumanTaskScheduler. JudgmentEscalator as NamedStrategy with ReYield/RouteHigher/Fault. DagNode judgment via CompletableFuture blocking. Cross-repo migration deferred to a separate work slot.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x event bus, CDI, Jackson, virtual threads

## Global Constraints

- All `@ConsumeEvent` handlers: `@RunOnVirtualThread` + `void` return (PP-20260723-c4c1cf)
- SPI types in `engine-api`; execution types in `engine-common` (PP-20260727-5267d2)
- All code operations via IntelliJ MCP — never bash grep/Edit on .java files
- HumanTaskScheduler SPI and implementations preserved unchanged
- Cross-repo consumer migration in a separate multi-repo work slot (not this branch)
- No Flyway migrations

---

## Batch 1: Unified target types (engine-api)

### Task 1: Expand JudgmentTarget + RoutingConfig + strip HumanTaskTarget from BindingTarget

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/RoutingConfig.java`
- Create: `api/src/main/java/io/casehub/api/model/HumanRoutingConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/JudgmentTarget.java` — add title, titleExpression, outcomes, scope, scopeExpression, priority, trustThreshold, escalatorStrategy, routingConfig, expiresAtExpression fields + builder methods + human() convenience
- Modify: `api/src/main/java/io/casehub/api/model/BindingTarget.java` — remove HumanTaskTarget from sealed permits
- Modify: `api/src/main/java/io/casehub/api/model/HumanTaskTarget.java` — remove `implements BindingTarget`
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java` — delete `humanTask()` builder method
- Modify: `api/src/main/java/io/casehub/api/model/converter/deser/BindingDeserializer.java` — update `deserializeHumanTask()` to produce JudgmentTarget with HumanRoutingConfig; rename to `deserializeHumanJudgment()`; update `deserializeJudgment()` to parse `human:` sub-block
- Modify: all 9 switch/instanceof sites — remove `case HumanTaskTarget` branches, ensure `case JudgmentTarget` handles both human and non-human yields
- Test: `api/src/test/java/io/casehub/api/model/JudgmentTargetTest.java` — add tests for new fields, routingConfig, human() builder
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperJudgmentTest.java` — add test for `judgment:` with `human:` sub-block
- Test: `api/src/test/resources/yaml/judgment-human-test.yaml`

**Interfaces:**
- Consumes: `JudgmentTarget` (existing from #996), `HumanTaskTarget` (existing — being stripped), `CandidateSetSpec` (existing)
- Produces: `RoutingConfig` sealed interface, `HumanRoutingConfig` record, expanded `JudgmentTarget` with all unified fields, `JudgmentTarget.Builder.human(HumanRoutingConfig)` convenience

- [ ] **Step 1: Create RoutingConfig and HumanRoutingConfig**

Create `api/src/main/java/io/casehub/api/model/RoutingConfig.java`:
```java
package io.casehub.api.model;

public sealed interface RoutingConfig permits HumanRoutingConfig {}
```

Create `api/src/main/java/io/casehub/api/model/HumanRoutingConfig.java`:
```java
package io.casehub.api.model;

import io.casehub.api.spi.routing.CandidateSetSpec;
import org.jspecify.annotations.Nullable;

public record HumanRoutingConfig(
    @Nullable String templateRef,
    @Nullable CandidateSetSpec candidateGroups,
    @Nullable CandidateSetSpec candidateUsers,
    @Nullable Integer claimDeadlineHours,
    @Nullable Class<?> payloadType) {}
```

- [ ] **Step 2: Write failing tests for new JudgmentTarget fields**

Add to `JudgmentTargetTest.java`:
```java
@Test
void builder_withHumanRoutingConfig_builds() {
  HumanRoutingConfig hrc = new HumanRoutingConfig(
      "template-1", null, null, 2, null);
  JudgmentTarget target = JudgmentTarget.builder()
      .prompt("Review this")
      .title("Review Task")
      .outcomes(Set.of("approve", "reject"))
      .priority("high")
      .trustThreshold("senior")
      .human(hrc)
      .build();
  assertThat(target.title()).isEqualTo("Review Task");
  assertThat(target.outcomes()).containsExactlyInAnyOrder("approve", "reject");
  assertThat(target.routingConfig()).isInstanceOf(HumanRoutingConfig.class);
  assertThat(((HumanRoutingConfig) target.routingConfig()).templateRef()).isEqualTo("template-1");
}

@Test
void builder_withoutRoutingConfig_buildsNullConfig() {
  JudgmentTarget target = JudgmentTarget.builder().prompt("test").build();
  assertThat(target.routingConfig()).isNull();
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=JudgmentTargetTest -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: FAIL — methods don't exist

- [ ] **Step 4: Add fields to JudgmentTarget**

Using `ide_insert_member` and `ide_replace_text_in_file`, add to `JudgmentTarget`:
- Fields: `title`, `titleExpression`, `outcomes` (Set<String>), `scope`, `scopeExpression`, `priority`, `trustThreshold`, `escalatorStrategy`, `routingConfig` (RoutingConfig), `expiresAtExpression`
- Constructor assignments
- Getters
- Builder fields + setter methods
- `Builder.human(HumanRoutingConfig)` convenience that sets `routingConfig`
- `outcomes` uses `Set.copyOf()` in constructor (immutable)

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=JudgmentTargetTest -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 6: Strip HumanTaskTarget from BindingTarget**

Using `ide_replace_text_in_file`:
1. `BindingTarget.java` — remove `HumanTaskTarget` from sealed permits
2. `HumanTaskTarget.java` — remove `implements BindingTarget` (retain class as data carrier)
3. `Binding.java` — delete `humanTask(HumanTaskTarget)` builder method via `ide_edit_member`

- [ ] **Step 7: Update all 9 switch/instanceof sites**

Remove `case HumanTaskTarget` from each site. The `case JudgmentTarget` branch (added in #996) now handles all yields. Use `ide_replace_text_in_file` for each:

1. `CaseContextChangedEventHandler:379` — remove HumanTaskTarget case (publishHumanTaskSchedule absorbs into publishJudgmentSchedule in Task 2)
2. `PlanningCasePlanModelSnapshotProvider:182` — remove `case HumanTaskTarget h -> "HUMAN"` (JudgmentTarget case returns "HUMAN" when routingConfig is HumanRoutingConfig, else existing value)
3. `BindingExecutorResolver:48` — remove `case HumanTaskTarget`
4. `QuartzWorkerExecutionManager` — remove `case HumanTaskTarget`
5. `SchedulerService:126` — remove `case HumanTaskTarget`
6. `SchedulerService:248` — remove `case HumanTaskTarget`
7-9. `PlanItemCompletionApplier` (2 sites) and `CbrCaseRetainObserver` — remove HumanTaskTarget references

- [ ] **Step 8: Update YAML parsing**

In `BindingDeserializer`:
1. Update `deserializeJudgment()` to parse `human:` sub-block into `HumanRoutingConfig` and set on builder via `.human()`
2. Update `resolveTarget()` — `humanTask:` block now produces JudgmentTarget: call `deserializeHumanJudgment()` which creates JudgmentTarget with HumanRoutingConfig
3. Add new fields parsing: `title`, `titleExpression`, `outcomes`, `scope`, `scopeExpression`, `priority`, `trustThreshold`, `escalatorStrategy`

Create YAML test fixture `api/src/test/resources/yaml/judgment-human-test.yaml`:
```yaml
spec:
  bindings:
    - name: human-judgment
      judgment:
        prompt: "Review this transaction"
        title: "Compliance Review"
        outcomes: [approve, reject]
        expiresIn: PT4H
        human:
          candidateGroups: ["compliance-team"]
          templateRef: review-template
          claimDeadlineHours: 2
      on:
        contextChange: ".transaction != null"
```

Add YAML test to `CaseDefinitionYamlMapperJudgmentTest`:
```java
@Test
void judgmentBinding_humanRoutingConfig_parsedFromYaml() {
  CaseDefinition def = loadDefinition("judgment-human-test.yaml");
  Binding binding = def.getBindings().stream()
      .filter(b -> b.getName().equals("human-judgment"))
      .findFirst().orElseThrow();
  JudgmentTarget target = (JudgmentTarget) binding.target();
  assertThat(target.prompt()).isEqualTo("Review this transaction");
  assertThat(target.title()).isEqualTo("Compliance Review");
  assertThat(target.outcomes()).containsExactlyInAnyOrder("approve", "reject");
  assertThat(target.routingConfig()).isInstanceOf(HumanRoutingConfig.class);
  HumanRoutingConfig hrc = (HumanRoutingConfig) target.routingConfig();
  assertThat(hrc.templateRef()).isEqualTo("review-template");
  assertThat(hrc.claimDeadlineHours()).isEqualTo(2);
}
```

- [ ] **Step 9: Build and run all api tests**

Run: `mvn install -pl api -DskipTests -Dcheckstyle.skip=true -Dspotless.check.skip=true -q && mvn test -pl api -Dtest="JudgmentTargetTest,CaseDefinitionYamlMapperJudgmentTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: ALL PASS

- [ ] **Step 10: Verify compilation across all modules**

Run: `ide_build_project` — expect only pre-existing errors, no new errors from our changes

- [ ] **Step 11: Commit**

```bash
git add api/src/ common/src/ runtime/src/ planning/src/ scheduler-quartz/src/
git commit -m "feat(#995): unify JudgmentTarget — RoutingConfig, strip HumanTaskTarget from BindingTarget  Refs #995"
```

## Batch 2: Handler unification + DelegatingJudgmentScheduler

### Task 2: DelegatingJudgmentScheduler + unified publishJudgmentSchedule

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/DelegatingJudgmentScheduler.java`
- Delete: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpJudgmentScheduler.java` (via `ide_refactor_safe_delete`)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java` — expand `publishJudgmentSchedule()` to handle HumanRoutingConfig (candidate resolution, routing strategy, bridge validation)
- Modify: `common/src/main/java/io/casehub/engine/common/spi/JudgmentScheduleRequest.java` — add new fields (caseBudgetDeadline, resolvedTitle, resolvedScope, experiences, candidateScores)
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/DelegatingJudgmentSchedulerTest.java`
- Test: update `JudgmentTargetDispatchTest.java` — add human routing dispatch test

**Interfaces:**
- Consumes: `JudgmentTarget` with `RoutingConfig` (Task 1), `HumanTaskScheduler` (existing SPI), `HumanTaskScheduleRequest` (existing), `HumanTaskTarget` (data carrier)
- Produces: `DelegatingJudgmentScheduler`, expanded `publishJudgmentSchedule()`, updated `JudgmentScheduleRequest`

- [ ] **Step 1: Write DelegatingJudgmentScheduler test**

```java
@Test
void delegatesToHumanTaskScheduler_whenHumanRoutingConfig() {
  // Create a JudgmentScheduleRequest with HumanRoutingConfig
  // Verify humanTaskScheduler.schedule() is called
  // Verify the HumanTaskScheduleRequest has correct fields mapped
}

@Test
void noOp_whenNullRoutingConfig() {
  // Create a JudgmentScheduleRequest with null routingConfig
  // Verify no exception, no humanTaskScheduler call
}
```

- [ ] **Step 2: Implement DelegatingJudgmentScheduler**

```java
@DefaultBean
@ApplicationScoped
public class DelegatingJudgmentScheduler implements JudgmentScheduler {
  @Inject Instance<HumanTaskScheduler> humanTaskScheduler;

  @Override
  public void schedule(JudgmentScheduleRequest request) {
    if (request.target().routingConfig() instanceof HumanRoutingConfig hrc
        && humanTaskScheduler.isResolvable()) {
      humanTaskScheduler.get().schedule(toHumanRequest(request, hrc));
      return;
    }
  }

  private HumanTaskScheduleRequest toHumanRequest(JudgmentScheduleRequest req, HumanRoutingConfig hrc) {
    // Build HumanTaskTarget from JudgmentTarget + HumanRoutingConfig fields
    // Map all fields to HumanTaskScheduleRequest
  }
}
```

- [ ] **Step 3: Update JudgmentScheduleRequest with new fields**

Add: `caseBudgetDeadline`, `resolvedTitle`, `resolvedScope`, `experiences`, `candidateScores`. Update all construction sites.

- [ ] **Step 4: Expand publishJudgmentSchedule() in CaseContextChangedEventHandler**

Absorb the human-specific logic from `publishHumanTaskSchedule()`:
- When `routingConfig instanceof HumanRoutingConfig`: resolve candidates, run HumanTaskRoutingStrategy, validate bridge
- Resolve title, scope, caseBudgetDeadline for all yields
- Delete `publishHumanTaskSchedule()` method
- Remove `Instance<HumanTaskScheduler>` field (delegation handles this)

- [ ] **Step 5: Run integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest="JudgmentTargetDispatchTest,DelegatingJudgmentSchedulerTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add runtime/src/ common/src/
git commit -m "feat(#995): DelegatingJudgmentScheduler + unified publishJudgmentSchedule  Refs #995"
```

## Batch 3: JudgmentEscalator SPI (#999)

### Task 3: JudgmentEscalator SPI + handler integration

**Files:**
- Create: `api/src/main/java/io/casehub/api/spi/judgment/JudgmentEscalator.java`
- Create: `api/src/main/java/io/casehub/api/spi/judgment/EscalationDecision.java`
- Create: `api/src/main/java/io/casehub/api/spi/judgment/EscalationContext.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/FaultEscalator.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/worker/ReYieldEscalator.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/JudgmentEscalationHandler.java` — add escalator resolution and decision execution
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/EngineStrategyResolver.java` — add `Instance<JudgmentEscalator>`
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/FaultEscalatorTest.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/worker/ReYieldEscalatorTest.java`

**Interfaces:**
- Consumes: `JudgmentTarget.escalatorStrategy()` (Task 1), `VerificationResult` (from #997), `JudgmentResponse` (from #996)
- Produces: `JudgmentEscalator` SPI, `EscalationDecision` sealed type, `EscalationContext` record, `FaultEscalator`, `ReYieldEscalator`

- [ ] **Step 1: Create SPI types (EscalationDecision, EscalationContext, JudgmentEscalator)**

Three new files in `api/src/main/java/io/casehub/api/spi/judgment/`. Code as specified in the design spec.

- [ ] **Step 2: Write FaultEscalator and ReYieldEscalator tests**

```java
// FaultEscalatorTest
@Test
void alwaysReturnsFault() {
  var escalator = new FaultEscalator();
  var ctx = buildContext(0, 3);
  assertThat(escalator.escalate(ctx)).isInstanceOf(EscalationDecision.Fault.class);
}

// ReYieldEscalatorTest
@Test
void underMax_returnsReYield() {
  var escalator = new ReYieldEscalator();
  var ctx = buildContext(1, 3);
  assertThat(escalator.escalate(ctx)).isInstanceOf(EscalationDecision.ReYield.class);
}

@Test
void atMax_returnsFault() {
  var escalator = new ReYieldEscalator();
  var ctx = buildContext(3, 3);
  assertThat(escalator.escalate(ctx)).isInstanceOf(EscalationDecision.Fault.class);
}
```

- [ ] **Step 3: Implement FaultEscalator and ReYieldEscalator**

```java
@DefaultBean @ApplicationScoped
public class FaultEscalator implements JudgmentEscalator {
  public EscalationDecision escalate(EscalationContext ctx) {
    return new EscalationDecision.Fault("Verification failed: " + ctx.verificationResult());
  }
  public String id() { return "fault"; }
}

@ApplicationScoped
public class ReYieldEscalator implements JudgmentEscalator {
  public EscalationDecision escalate(EscalationContext ctx) {
    if (ctx.escalationCount() < ctx.maxEscalations()) {
      String feedback = switch (ctx.verificationResult()) {
        case VerificationResult.InsufficientEvidence ie -> ie.feedback();
        case VerificationResult.TrustTooLow ttl -> "Trust level too low: " + ttl.requiredLevel();
        default -> "Verification failed";
      };
      return new EscalationDecision.ReYield(feedback);
    }
    return new EscalationDecision.Fault("Max escalations reached (" + ctx.maxEscalations() + ")");
  }
  public String id() { return "re-yield"; }
}
```

- [ ] **Step 4: Update JudgmentEscalationHandler with escalator resolution**

Wire escalator: resolve from EngineStrategyResolver, build EscalationContext with escalationCount from EventLog, execute decision. Add `Instance<JudgmentEscalator>` to EngineStrategyResolver.

- [ ] **Step 5: Run tests**

Run: `mvn install -pl api,common -DskipTests -Dcheckstyle.skip=true -Dspotless.check.skip=true -q && mvn test -pl runtime -Dtest="FaultEscalatorTest,ReYieldEscalatorTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add api/src/ runtime/src/
git commit -m "feat(#999): JudgmentEscalator SPI with FaultEscalator and ReYieldEscalator  Closes #999"
```

## Batch 4: DAG/SWF judgment integration (#1000)

### Task 4: JudgmentNodeExecutor + SWF callable

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/JudgmentNodeExecutor.java`
- Create: `runtime/src/main/java/io/casehub/engine/internal/engine/JudgmentNodeResult.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/JudgmentCompletedHandler.java` — inject JudgmentNodeExecutor, enqueue Completed/Faulted
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/JudgmentEscalationHandler.java` — inject JudgmentNodeExecutor, enqueue ReYielded/Faulted
- Create: `flow/src/main/java/io/casehub/flow/judgment/JudgmentCallableDispatcher.java`
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/JudgmentNodeExecutorTest.java`

**Interfaces:**
- Consumes: `JudgmentCompletedHandler` (from #996), `JudgmentEscalationHandler` (Task 3), `DagDriver` (existing)
- Produces: `JudgmentNodeExecutor` (blocking executor for DAG threads), `JudgmentCallableDispatcher` (SWF `call: casehub:judgment`)

- [ ] **Step 1: Create JudgmentNodeResult sealed type**

```java
package io.casehub.engine.internal.engine;

import io.casehub.engine.common.spi.JudgmentResponse;

public sealed interface JudgmentNodeResult {
  record Completed(JudgmentResponse response) implements JudgmentNodeResult {}
  record ReYielded() implements JudgmentNodeResult {}
  record Faulted(String reason) implements JudgmentNodeResult {}
}
```

- [ ] **Step 2: Write JudgmentNodeExecutor test**

```java
@Test
void execute_completesWhenResponseArrives() {
  var executor = new JudgmentNodeExecutor();
  // Simulate: start execute in a virtual thread, then complete it
  var target = JudgmentTarget.builder().prompt("test").build();
  var caseId = UUID.randomUUID();
  CompletableFuture<JudgmentResponse> resultFuture = CompletableFuture.supplyAsync(() ->
      executor.execute(target, Map.of(), caseId, "binding", Duration.ofSeconds(5)));
  // Simulate response
  Thread.sleep(100);
  executor.enqueue(caseId, "binding",
      new JudgmentNodeResult.Completed(new JudgmentResponse(caseId, "binding", "t", "approve", Map.of(), null, null)));
  assertThat(resultFuture.get(2, TimeUnit.SECONDS).decision()).isEqualTo("approve");
}

@Test
void execute_throwsOnTimeout() {
  var executor = new JudgmentNodeExecutor();
  var target = JudgmentTarget.builder().prompt("test").build();
  assertThatThrownBy(() -> executor.execute(target, Map.of(), UUID.randomUUID(), "b", Duration.ofMillis(100)))
      .isInstanceOf(JudgmentTimeoutException.class);
}
```

- [ ] **Step 3: Implement JudgmentNodeExecutor**

As specified in the design spec — BlockingQueue per pending judgment, poll loop with per-cycle timeout.

- [ ] **Step 4: Wire handlers to enqueue results**

In `JudgmentCompletedHandler`: after output mapping (Accepted path) or on Rejected, check `pending.containsKey(key)` and enqueue `Completed` or `Faulted`. Place OUTSIDE the `instanceof JudgmentTarget` conditional for SWF support.

In `JudgmentEscalationHandler`: after escalator decision, enqueue `ReYielded` (for ReYield/RouteHigher) or `Faulted` (for Fault).

- [ ] **Step 5: Create JudgmentCallableDispatcher for SWF**

```java
@ApplicationScoped
public class JudgmentCallableDispatcher implements CallableDispatcher {
  @Inject JudgmentNodeExecutor executor;
  // Delegates to executor.execute() with target and input from call arguments
}
```

Register in `CallableDispatchRegistry` for `"casehub:judgment"`.

- [ ] **Step 6: Run tests**

Run: `mvn install -DskipTests -Dcheckstyle.skip=true -Dspotless.check.skip=true -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=JudgmentNodeExecutorTest -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add runtime/src/ flow/src/
git commit -m "feat(#1000): JudgmentNodeExecutor + SWF casehub:judgment callable  Closes #1000"
```

## Batch 5: React module integration tests (#957)

### Task 5: React module @QuarkusTest integration tests

**Files:**
- Create: `react/src/test/java/io/casehub/engine/react/ReActExecutionIntegrationTest.java`
- Create: `react/src/test/java/io/casehub/engine/react/ReActAuditTrailTest.java`

**Interfaces:**
- Consumes: `ReActWorkerFunctionHandler` (existing), `CaseHubEventType.REACT_CYCLE` (existing), mock `ChatModelProvider`
- Produces: Two integration test classes

- [ ] **Step 1: Write ReActExecutionIntegrationTest**

`@QuarkusTest` with inner CaseHub subclass declaring a react worker. Mock `ChatModelProvider` returning canned tool-use then final-answer responses. Start case, await completion, verify case reaches COMPLETED.

- [ ] **Step 2: Write ReActAuditTrailTest**

`@QuarkusTest` verifying:
- `REACT_CYCLE` EventLog entries exist, ordered by cycleIndex
- Each entry contains `reasoningText`, `toolCalls`, `tokenUsage`
- `WORKER_EXECUTION_COMPLETED` EventLog carries `reactCycleCount`, `reactTotalInputTokens`, `reactTotalOutputTokens`

- [ ] **Step 3: Run tests**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl react -Dtest="ReActExecutionIntegrationTest,ReActAuditTrailTest" -Dcheckstyle.skip=true -Dspotless.check.skip=true -q`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add react/src/
git commit -m "feat(#957): add @QuarkusTest integration tests for react module  Closes #957"
```

## References

- [2026-08-29-unified-judgment-target-design.md] — design spec this plan implements
- `HumanTaskTarget.java` — type being stripped from BindingTarget
- `CaseContextChangedEventHandler.java:685-815` — publishHumanTaskSchedule (logic absorbed)
- `DagDriver.java:74-93` — execute method (Function<T,R> contract)
- `JudgmentCompletedHandler.java` — verification gate + DAG notification point
- `JudgmentEscalationHandler.java` — escalator integration point
- `EngineStrategyResolver.java` — needs Instance<JudgmentEscalator>
- PP-20260723-c4c1cf — virtual-thread-handler-convention
- PP-20260727-5267d2 — plan-type-module-boundary
- GitHub #995, #999, #1000, #957
