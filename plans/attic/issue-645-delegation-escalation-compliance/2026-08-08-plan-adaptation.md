# Plan Adaptation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #803 — Plan adaptation — revise active plans based on new observations
**Issue group:** #803, #806, #805, #808

**Goal:** Enable the engine to revise decomposed plans after worker completions,
using orthogonal trigger and revision strategy SPIs.

**Architecture:** Two NamedStrategy SPIs (`AdaptationTrigger`, `PlanRevisionStrategy`)
in engine-api. Orchestrator (`DefaultPlanAdaptationEvaluator`) in planning module.
Compound replacement via new `CasePlanModel.replaceCompound()`. Bounded concurrency
via semaphore. `AdaptationConfig` record on `CaseDefinition`.

**Tech Stack:** Quarkus CDI, Vert.x EventBus, Smallrye Mutiny, Jackson, JQ (jackson-jq)

## Global Constraints

- Linear chain plans only (same as #802)
- `TaskStatus.OBSOLETE` already exists in the enum
- Plan-definition types → engine-api; execution types → engine-common (PP-20260727-5267d2)
- CasePlanModel is authoritative source of truth for definition state
- Planning module already depends on runtime via EngineStrategyResolver injection
- EngineStrategyResolver auto-discovers NamedStrategy implementations; explicit injection is best-practice

---

### Task 1: Foundation SPI types (engine-api)

**Files:**
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/AdaptationTrigger.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/AdaptationSignal.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/AdaptationCause.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/AdaptationContext.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/CompletedStep.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/PlanStepDescriptor.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/PlanRevisionStrategy.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/RevisionContext.java`
- Create: `api/src/main/java/io/casehub/engine/plan/adaptation/RevisedPlan.java`
- Create: `api/src/main/java/io/casehub/api/model/AdaptationConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/CaseDefinition.java` — add `adaptationConfig` field + getter/setter + builder method
- Modify: `api/src/main/java/io/casehub/api/model/event/CaseHubEventType.java` — add `PLAN_ADAPTED`
- Test: `api/src/test/java/io/casehub/engine/plan/adaptation/AdaptationTypesTest.java`
- Test: `api/src/test/java/io/casehub/api/model/AdaptationConfigTest.java`

**Interfaces:**
- Consumes: `NamedStrategy` (platform-api), `TaskStatus` (api), `Capability` (worker-api), `RetrievedMemory` (api)
- Produces: All adaptation SPI types consumed by Tasks 2–6. `AdaptationTrigger.evaluate(AdaptationContext) → AdaptationSignal`. `PlanRevisionStrategy.revise(RevisionContext) → Uni<RevisedPlan>`. `AdaptationConfig(String trigger, String revision)`. `PlanStepDescriptor(String id, String description, String capabilityName)`.

- [ ] **Step 1: Write tests for SPI types**

```java
// AdaptationTypesTest.java
@Test void planStepDescriptorRejectsNullId() {
    assertThrows(NullPointerException.class,
        () -> new PlanStepDescriptor(null, "desc", "cap"));
}
@Test void planStepDescriptorRejectsNullCapabilityName() {
    assertThrows(NullPointerException.class,
        () -> new PlanStepDescriptor("id", "desc", null));
}
@Test void completedStepRejectsNullFields() {
    assertThrows(NullPointerException.class,
        () -> new CompletedStep(null, "cap", "desc", Map.of(), Instant.now()));
}
@Test void completedStepOutputIsImmutable() {
    var step = new CompletedStep("id", "cap", "desc", Map.of("k","v"), Instant.now());
    assertThrows(UnsupportedOperationException.class,
        () -> step.output().put("x", "y"));
}
@Test void adaptationSignalProceedIsSealed() {
    AdaptationSignal signal = AdaptationSignal.PROCEED;
    assertInstanceOf(AdaptationSignal.class, signal);
}
@Test void adaptationSignalSkipIsSealed() {
    AdaptationSignal signal = AdaptationSignal.SKIP;
    assertInstanceOf(AdaptationSignal.class, signal);
}
@Test void adaptationCauseStepCompleted() {
    var cause = new AdaptationCause.StepCompleted("s1", "cap", Map.of("r","v"));
    assertEquals("s1", cause.stepId());
    assertEquals("cap", cause.capabilityName());
}
@Test void adaptationCauseStepFailed() {
    var cause = new AdaptationCause.StepFailed("s1", "timeout");
    assertEquals("s1", cause.stepId());
    assertEquals("timeout", cause.reason());
}
@Test void revisedPlanStepsImmutable() {
    var plan = new RevisedPlan(
        List.of(new PlanStepDescriptor("id","desc","cap")), "reason");
    assertThrows(UnsupportedOperationException.class,
        () -> plan.steps().add(new PlanStepDescriptor("x","y","z")));
}
// AdaptationConfigTest.java
@Test void adaptationConfigRejectsNullTrigger() {
    assertThrows(NullPointerException.class,
        () -> new AdaptationConfig(null, "forward-replan"));
}
@Test void adaptationConfigRejectsNullRevision() {
    assertThrows(NullPointerException.class,
        () -> new AdaptationConfig("every-step", null));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=AdaptationTypesTest,AdaptationConfigTest -q`
Expected: compilation failures (types don't exist yet)

- [ ] **Step 3: Implement SPI types**

Create all records and interfaces listed in Files. Key implementation details:

`PlanStepDescriptor`: record with null-check compact constructor, `Objects.requireNonNull` on `id`, `description`, `capabilityName`.

`CompletedStep`: record with null-checks on `stepId`, `capabilityName`, `description`, `completedAt`. `output` defaults to `Map.of()` if null, wrapped in `Map.copyOf()`.

`AdaptationSignal`: sealed interface with two enum-like constants `PROCEED` and `SKIP` (records implementing the interface).

`AdaptationCause`: sealed interface with two record permits `StepCompleted(String stepId, String capabilityName, Map<String, Object> output)` and `StepFailed(String stepId, String reason)`.

`AdaptationContext`: record with all fields from spec. `adaptationGeneration` is `int`.

`AdaptationTrigger`: interface extending `NamedStrategy`, `evaluate(AdaptationContext) → AdaptationSignal`, default `id()` returns `"every-step"`.

`PlanRevisionStrategy`: interface extending `NamedStrategy`, `revise(RevisionContext) → Uni<RevisedPlan>`, default `id()` returns `"forward-replan"`.

`RevisionContext`: record wrapping `AdaptationContext` + `AdaptationCause` + `List<Capability>` + `List<RetrievedMemory>`.

`RevisedPlan`: record with `List<PlanStepDescriptor> steps` (wrapped in `List.copyOf()`) + nullable `String rationale`.

`AdaptationConfig`: record with `String trigger`, `String revision`. Both non-null.

`CaseDefinition`: add `private AdaptationConfig adaptationConfig` field (line ~80), `getAdaptationConfig()`/`setAdaptationConfig()` methods, `Builder.adaptationConfig(AdaptationConfig)` method, wire in `build()`.

`CaseHubEventType`: add `PLAN_ADAPTED` constant after `GOAL_DECOMPOSED`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=AdaptationTypesTest,AdaptationConfigTest -q`
Expected: all PASS

- [ ] **Step 5: Commit**

```
feat(#803): add plan adaptation SPI types — AdaptationTrigger, PlanRevisionStrategy, AdaptationConfig

Refs #803
```

---

### Task 2: PlanAdaptationEvaluator SPI + CasePlanModel.replaceCompound()

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/spi/PlanAdaptationEvaluator.java`
- Modify: `planning/src/main/java/io/casehub/engine/planning/plan/CasePlanModel.java` — add `replaceCompound()` and `getAdaptationGeneration()` methods
- Modify: `planning/src/main/java/io/casehub/engine/planning/plan/DefaultCasePlanModel.java` — implement `replaceCompound()`, add `adaptationGenerations` map
- Test: `planning/src/test/java/io/casehub/engine/planning/plan/CasePlanModelReplaceCompoundTest.java`

**Interfaces:**
- Consumes: `TaskStatus` (api), `CaseInstance` (common), `PlanItemDefinition.Compound` (planning)
- Produces: `PlanAdaptationEvaluator.evaluateAdaptation(UUID, String, String, TaskStatus)`. `CasePlanModel.replaceCompound(String, Compound, int)`. `CasePlanModel.getAdaptationGeneration(String) → int`.

- [ ] **Step 1: Write tests for replaceCompound()**

```java
// CasePlanModelReplaceCompoundTest.java
@Test void replaceCompoundUnregistersOldChildren() {
    // Build compound with 3 children, register it
    // Replace with compound with 2 different children
    // Assert old children removed from getChildrenOf(), getDefinition(), getDefinitionStatus()
}
@Test void replaceCompoundRegistersNewChildren() {
    // Replace compound
    // Assert new children appear in getChildrenOf(), getDefinition()
    // Assert new children have PENDING definition status
}
@Test void replaceCompoundUpdatesParentIndex() {
    // Replace compound
    // Assert old children no longer in getParentOf()
    // Assert new children have correct parent
}
@Test void replaceCompoundUpdatesScopedBindings() {
    // Replace with compound that has different scopedBindings
    // Assert getAllCompounds() returns compound with new bindings
}
@Test void replaceCompoundPreservesCompletedPlanItems() {
    // Add PlanItem for child A, mark COMPLETED
    // Replace compound (keeping A in new compound)
    // Assert A's PlanItem still in CasePlanModel
}
@Test void replaceCompoundRemovesPendingPlanItems() {
    // Add PlanItem for child B, leave PENDING
    // Replace compound (B not in new compound)
    // Assert B's PlanItem removed
}
@Test void replaceCompoundIncrementsGeneration() {
    // Initial generation is 0
    // After replaceCompound with generation=1, getAdaptationGeneration returns 1
}
@Test void replaceCompoundThrowsOnUnknownCompoundId() {
    assertThrows(IllegalArgumentException.class,
        () -> plan.replaceCompound("nonexistent", newCompound, 1));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl planning -Dtest=CasePlanModelReplaceCompoundTest -q`
Expected: FAIL (methods don't exist)

- [ ] **Step 3: Implement replaceCompound() and SPI**

`PlanAdaptationEvaluator` (common/spi/): single method `evaluateAdaptation(UUID caseId, String tenancyId, String completedBindingName, TaskStatus completedStatus)`.

`CasePlanModel` interface: add `void replaceCompound(String compoundId, PlanItemDefinition.Compound newCompound, int newGeneration)` and `int getAdaptationGeneration(String compoundId)` as default methods (throwing `UnsupportedOperationException` — SPI evolution pattern).

`DefaultCasePlanModel`: add `ConcurrentHashMap<String, AtomicInteger> adaptationGenerations` field.

`replaceCompound()` implementation:
1. Validate compoundId exists in `definitions`
2. Get old compound's children from `childrenIndex`
3. For each old child: remove from `definitions`, `definitionStates`, `parentIndex`. If its PlanItem exists and is not terminal/RUNNING, remove from `itemsById`, `agenda`, `latestByBinding`.
4. Remove old compound from `definitions`, `definitionStates`, `childrenIndex`
5. Call `registerDefinition(newCompound)` — populates all index structures
6. Update `adaptationGenerations.computeIfAbsent(compoundId, k -> new AtomicInteger()).set(newGeneration)`

`getAdaptationGeneration()`: returns `adaptationGenerations.getOrDefault(compoundId, new AtomicInteger(0)).get()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl planning -Dtest=CasePlanModelReplaceCompoundTest -q`
Expected: all PASS

- [ ] **Step 5: Commit**

```
feat(#803): add PlanAdaptationEvaluator SPI and CasePlanModel.replaceCompound()

Refs #803
```

---

### Task 3: Built-in trigger and revision strategy implementations

**Files:**
- Create: `planning/src/main/java/io/casehub/engine/planning/adaptation/EveryStepTrigger.java`
- Create: `planning/src/main/java/io/casehub/engine/planning/adaptation/OnFailureTrigger.java`
- Create: `planning/src/main/java/io/casehub/engine/planning/adaptation/ForwardReplanRevision.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/adaptation/AdaptationTriggerTest.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/adaptation/ForwardReplanRevisionTest.java`

**Interfaces:**
- Consumes: `AdaptationTrigger` (api), `PlanRevisionStrategy` (api), `AdaptationContext` (api), `RevisionContext` (api), `ChatModelProvider` (api)
- Produces: `EveryStepTrigger` (id=`"every-step"`), `OnFailureTrigger` (id=`"on-failure"`), `ForwardReplanRevision` (id=`"forward-replan"`)

- [ ] **Step 1: Write trigger tests**

```java
// AdaptationTriggerTest.java
@Test void everyStepTriggerAlwaysProceeds() {
    var trigger = new EveryStepTrigger();
    var ctx = buildContext(TaskStatus.COMPLETED);
    assertEquals(AdaptationSignal.PROCEED, trigger.evaluate(ctx));
}
@Test void everyStepTriggerProceedsOnFaulted() {
    var trigger = new EveryStepTrigger();
    var ctx = buildContext(TaskStatus.FAULTED);
    assertEquals(AdaptationSignal.PROCEED, trigger.evaluate(ctx));
}
@Test void everyStepTriggerIdIsEveryStep() {
    assertEquals("every-step", new EveryStepTrigger().id());
}
@Test void onFailureTriggerSkipsOnCompleted() {
    var trigger = new OnFailureTrigger();
    var ctx = buildContext(TaskStatus.COMPLETED);
    assertEquals(AdaptationSignal.SKIP, trigger.evaluate(ctx));
}
@Test void onFailureTriggerProceedsOnFaulted() {
    var trigger = new OnFailureTrigger();
    assertEquals(AdaptationSignal.PROCEED,
        trigger.evaluate(buildContext(TaskStatus.FAULTED)));
}
@Test void onFailureTriggerProceedsOnRejected() {
    var trigger = new OnFailureTrigger();
    assertEquals(AdaptationSignal.PROCEED,
        trigger.evaluate(buildContext(TaskStatus.REJECTED)));
}
@Test void onFailureTriggerProceedsOnCancelled() {
    var trigger = new OnFailureTrigger();
    assertEquals(AdaptationSignal.PROCEED,
        trigger.evaluate(buildContext(TaskStatus.CANCELLED)));
}
@Test void onFailureTriggerIdIsOnFailure() {
    assertEquals("on-failure", new OnFailureTrigger().id());
}
```

- [ ] **Step 2: Run trigger tests to verify they fail**

Run: `mvn test -pl planning -Dtest=AdaptationTriggerTest -q`
Expected: FAIL (classes don't exist)

- [ ] **Step 3: Implement triggers**

`EveryStepTrigger`: `@ApplicationScoped`, `id()` returns `"every-step"`, `evaluate()` always returns `PROCEED`.

`OnFailureTrigger`: `@ApplicationScoped`, `id()` returns `"on-failure"`, `evaluate()` returns `PROCEED` when `context.latestStatus()` is `FAULTED`, `REJECTED`, `OBSOLETE`, or `CANCELLED`; `SKIP` otherwise.

- [ ] **Step 4: Run trigger tests to verify they pass**

Run: `mvn test -pl planning -Dtest=AdaptationTriggerTest -q`
Expected: all PASS

- [ ] **Step 5: Write ForwardReplanRevision tests**

```java
// ForwardReplanRevisionTest.java
@Test void producesRevisedPlanFromLlmResponse() {
    // Mock ChatModelProvider returning canned JSON with 2 steps
    // Call revise() with context containing 1 completed step
    // Assert RevisedPlan has 2 PlanStepDescriptor entries
}
@Test void includesCompletedStepHistoryInPrompt() {
    // Capture the prompt sent to ChatModel
    // Assert it contains "Completed steps:" section with step descriptions and outputs
}
@Test void returnsFailureWhenNoChatModelProvider() {
    // Instance<ChatModelProvider>.isUnsatisfied() = true
    // Assert revise() returns failed Uni
}
@Test void returnsFailureOnInvalidJson() {
    // Mock ChatModelProvider returning non-JSON
    // Assert revise() returns Uni failure with AgentException
}
@Test void handlesEmptyStepsResponse() {
    // Mock ChatModelProvider returning {"steps": []}
    // Assert revise() returns Uni failure (empty plan is an error)
}
@Test void idIsForwardReplan() {
    assertEquals("forward-replan", new ForwardReplanRevision().id());
}
```

- [ ] **Step 6: Run revision tests to verify they fail**

Run: `mvn test -pl planning -Dtest=ForwardReplanRevisionTest -q`
Expected: FAIL

- [ ] **Step 7: Implement ForwardReplanRevision**

`@ApplicationScoped`, `@Inject Instance<ChatModelProvider>`. Prompt structure from spec. Parses JSON response to `List<PlanStepDescriptor>`. Uses `Agent.builder()` for LLM interaction (same pattern as `LlmDecompositionStrategy`).

- [ ] **Step 8: Run revision tests to verify they pass**

Run: `mvn test -pl planning -Dtest=ForwardReplanRevisionTest -q`
Expected: all PASS

- [ ] **Step 9: Commit**

```
feat(#803): add EveryStepTrigger, OnFailureTrigger, ForwardReplanRevision strategies

Refs #803
```

---

### Task 4: DefaultPlanAdaptationEvaluator orchestrator

**Files:**
- Create: `planning/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java`
- Test: `planning/src/test/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluatorTest.java`

**Interfaces:**
- Consumes: `PlanAdaptationEvaluator` (common/spi), `AdaptationTrigger` (api), `PlanRevisionStrategy` (api), `CasePlanModel.replaceCompound()` (planning), `EngineStrategyResolver` (runtime), `BlackboardRegistry` (planning), `PlanItemStore` (common), `EventLogRepository` (common), `CaseInstanceRepository` (common), `CaseDefinitionRegistry` (common), `AgentMemoryRetriever` (runtime)
- Produces: Orchestrated adaptation — resolves instance/definition, evaluates trigger, calls revision, applies compound replacement, writes EventLog.

- [ ] **Step 1: Write orchestrator tests**

```java
// DefaultPlanAdaptationEvaluatorTest.java — unit tests with mocked dependencies

@Test void skipsWhenBindingNotInDecomposedCompound() {
    // PlanItemStore returns items with no parentCompoundId
    // Assert: trigger never called, no EventLog written
}
@Test void skipsWhenAdaptationConfigNull() {
    // CaseDefinition has null adaptationConfig
    // Assert: trigger never called
}
@Test void callsTriggerAndSkipsOnSkipSignal() {
    // Trigger returns SKIP
    // Assert: revision strategy never called
}
@Test void callsTriggerProceedThenRevision() {
    // Trigger returns PROCEED
    // Revision returns RevisedPlan with 2 steps
    // Assert: replaceCompound called, EventLog written
}
@Test void marksPendingPlanItemsObsolete() {
    // 2 pending PlanItems in compound
    // Revision returns different steps
    // Assert: PlanItemStore.updateStatus called with OBSOLETE for both
}
@Test void leavesRunningPlanItemsUntouched() {
    // 1 RUNNING PlanItem
    // Assert: its status not changed, included in new compound bindings
}
@Test void leavesCompletedPlanItemsUntouched() {
    // 1 COMPLETED PlanItem
    // Assert: its status not changed
}
@Test void writesEventLogWithCorrectMetadata() {
    // Successful adaptation
    // Assert: EventLog has PLAN_ADAPTED type with goalName, compoundId,
    //   triggerStrategy, revisionStrategy, counts, obsoletedSteps, materializedSteps
}
@Test void lockKeyIncludesCaseId() {
    // Two cases with same goal name adapt concurrently
    // Assert: both proceed (no cross-case contention)
}
@Test void generationCounterPreventsRedundantAdaptation() {
    // First adaptation increments generation to 1
    // Second event still has generation 0
    // Assert: second adaptation skipped after acquiring lock
}
@Test void semaphoreBoundsMaxConcurrentAdaptations() {
    // Configure semaphore with 1 permit
    // Submit 2 concurrent adaptations
    // Assert: second blocks until first completes
}
@Test void timeoutResultsInGracefulDegradation() {
    // Revision Uni times out
    // Assert: existing plan unchanged, warning logged, no EventLog written
}
@Test void exceptionIsolationOnRevisionFailure() {
    // Revision throws exception
    // Assert: existing plan unchanged, warning logged
}
@Test void constructsCorrectAdaptationCauseForSuccess() {
    // Step completed with COMPLETED status
    // Assert: AdaptationCause.StepCompleted passed to RevisionContext
}
@Test void constructsCorrectAdaptationCauseForFailure() {
    // Step with FAULTED status
    // Assert: AdaptationCause.StepFailed passed to RevisionContext
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl planning -Dtest=DefaultPlanAdaptationEvaluatorTest -q`
Expected: FAIL

- [ ] **Step 3: Implement the orchestrator**

`@ApplicationScoped`, implements `PlanAdaptationEvaluator`. Full implementation following spec's orchestrator section. Key implementation details:

- Semaphore: `new Semaphore(maxConcurrent)` where `maxConcurrent` from `@ConfigProperty(name = "casehub.engine.adaptation.max-concurrent", defaultValue = "3")`
- Lock map: `ConcurrentHashMap<String, ReentrantLock>` keyed by `caseId + ":" + compoundId`
- Resolves `CaseInstance` from `CaseInstanceRepository`, `CaseDefinition` from `CaseDefinitionRegistry`
- Builds `AdaptationContext` from `PlanItemStore.findByCaseId()` (separate completed/pending/running) and `EventLogRepository` (completed step outputs)
- Calls `trigger.evaluate(context)` → if SKIP, return
- Constructs `AdaptationCause` from `completedStatus` and binding info
- Calls `revision.revise(revisionContext).await().atMost(timeout)`
- Builds new `Compound` with updated `scopedBindings` (completed + running + new steps)
- Calls `casePlanModel.replaceCompound()`
- Updates `PlanItemStore` (OBSOLETE old pending, save new)
- Writes `PLAN_ADAPTED` EventLog

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl planning -Dtest=DefaultPlanAdaptationEvaluatorTest -q`
Expected: all PASS

- [ ] **Step 5: Commit**

```
feat(#803): add DefaultPlanAdaptationEvaluator — orchestrates trigger + revision + compound replacement

Refs #803
```

---

### Task 5: Call site integration + YAML mapping

**Files:**
- Modify: `planning/src/main/java/io/casehub/engine/planning/handler/PlanItemCompletionHandler.java` — inject `Instance<PlanAdaptationEvaluator>`, call before `compoundCompletionEvaluator`
- Modify: `planning/src/main/java/io/casehub/engine/planning/handler/WorkerRetryExhaustionHandler.java` — inject `Instance<PlanAdaptationEvaluator>`, call before `compoundCompletionEvaluator`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` — parse `adaptation:` block
- Test: `api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperAdaptationTest.java`

**Interfaces:**
- Consumes: `PlanAdaptationEvaluator` (common/spi), `CaseDefinitionYamlMapper` patterns
- Produces: Wired call sites on both success and failure paths. YAML parsing for `adaptation:` block.

- [ ] **Step 1: Write YAML mapper tests**

```java
// CaseDefinitionYamlMapperAdaptationTest.java
@Test void parsesExplicitAdaptationConfig() {
    // YAML with adaptation: {trigger: every-step, revision: forward-replan}
    // Assert adaptationConfig != null, trigger = "every-step", revision = "forward-replan"
}
@Test void parsesAdaptivePreset() {
    // YAML with adaptation: adaptive
    // Assert trigger = "every-step", revision = "forward-replan"
}
@Test void parsesConservativePreset() {
    // YAML with adaptation: conservative
    // Assert trigger = "on-failure", revision = "forward-replan"
}
@Test void missingAdaptationReturnsNull() {
    // YAML with no adaptation block
    // Assert adaptationConfig is null
}
@Test void partialExplicitConfigUsesDefaults() {
    // YAML with adaptation: {trigger: on-failure}  (no revision)
    // Assert trigger = "on-failure", revision = "forward-replan"
}
@Test void unknownPresetThrows() {
    // YAML with adaptation: unknown-preset
    assertThrows(IllegalArgumentException.class, ...);
}
@Test void adaptationWithoutDecompositionStrategyWarns() {
    // YAML with adaptation but no decompositionStrategy
    // Assert: definition built successfully (warning logged, not thrown)
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperAdaptationTest -q`
Expected: FAIL

- [ ] **Step 3: Implement YAML mapping**

In `CaseDefinitionYamlMapper`, after the `memoryRetrieval` block (~line 835), add:

```java
final JsonNode adaptationNode = specNode != null ? specNode.get("adaptation") : null;
if (adaptationNode != null) {
    if (adaptationNode.isTextual()) {
        String preset = adaptationNode.asText();
        switch (preset) {
            case "adaptive" -> def.setAdaptationConfig(
                new AdaptationConfig("every-step", "forward-replan"));
            case "conservative" -> def.setAdaptationConfig(
                new AdaptationConfig("on-failure", "forward-replan"));
            case "off" -> {} // null = disabled
            default -> throw new IllegalArgumentException(
                "Unknown adaptation preset: " + preset);
        }
    } else if (adaptationNode.isObject()) {
        String trigger = adaptationNode.has("trigger")
            ? adaptationNode.get("trigger").asText() : "every-step";
        String revision = adaptationNode.has("revision")
            ? adaptationNode.get("revision").asText() : "forward-replan";
        def.setAdaptationConfig(new AdaptationConfig(trigger, revision));
    }
    if (def.getAdaptationConfig() != null && def.getDecompositionStrategy() == null) {
        LOG.warnf("adaptation configured without decompositionStrategy — "
            + "adaptation requires initial decomposition");
    }
}
```

- [ ] **Step 4: Run YAML tests to verify they pass**

Run: `mvn test -pl api -Dtest=CaseDefinitionYamlMapperAdaptationTest -q`
Expected: all PASS

- [ ] **Step 5: Wire call sites**

In `PlanItemCompletionHandler.completePlanItemByBindingName()`, after `item.markCompleted()` and before `compoundCompletionEvaluator.evaluate()`:

```java
if (planAdaptationEvaluator.isResolvable()) {
    planAdaptationEvaluator.get().evaluateAdaptation(
        caseId, tenancyId, bindingName, TaskStatus.COMPLETED);
}
```

In `WorkerRetryExhaustionHandler.onWorkerRetriesExhausted()`, after `item.markFaulted()` and before `compoundCompletionEvaluator.evaluate()`:

```java
if (planAdaptationEvaluator.isResolvable()) {
    planAdaptationEvaluator.get().evaluateAdaptation(
        event.caseId(), event.tenancyId(), event.bindingName(), TaskStatus.FAULTED);
}
```

Both inject `@Inject Instance<PlanAdaptationEvaluator> planAdaptationEvaluator`.

- [ ] **Step 6: Run full planning module tests**

Run: `mvn test -pl planning -q`
Expected: all PASS (existing tests unaffected)

- [ ] **Step 7: Commit**

```
feat(#803): wire plan adaptation into completion handlers + YAML adaptation: block

Refs #803
```

---

### Task 6: Integration test + lock lifecycle cleanup

**Files:**
- Create: `planning/src/test/java/io/casehub/engine/planning/adaptation/PlanAdaptationIntegrationTest.java`
- Modify: `planning/src/main/java/io/casehub/engine/planning/adaptation/DefaultPlanAdaptationEvaluator.java` — add lock cleanup listener

**Interfaces:**
- Consumes: All previous tasks
- Produces: Full end-to-end verification; lock lifecycle management

- [ ] **Step 1: Write integration test**

```java
// PlanAdaptationIntegrationTest.java — @QuarkusTest
@Test void fullAdaptationFlow() {
    // 1. Create CaseDefinition with decompositionStrategy: "llm",
    //    adaptationConfig: new AdaptationConfig("every-step", "forward-replan")
    // 2. Mock ChatModelProvider to return:
    //    - Initial plan: [step-A (gather), step-B (analyse), step-C (report)]
    //    - Revised plan: [step-D (deep-analyse), step-E (report)]
    // 3. Start case → GoalDecomposer creates compound with 3 steps
    // 4. Complete step-A via WorkflowExecutionCompleted
    // 5. Assert: adaptation fires, compound replaced with new bindings
    // 6. Assert: step-B and step-C marked OBSOLETE
    // 7. Assert: step-D and step-E created as new PlanItems
    // 8. Assert: EventLog has PLAN_ADAPTED entry with correct metadata
    // 9. Complete step-D, then step-E
    // 10. Assert: compound completes
}
@Test void adaptationSkippedWhenTriggerReturnsSkip() {
    // Configure with on-failure trigger
    // Complete step successfully
    // Assert: no PLAN_ADAPTED EventLog, compound continues normally
}
@Test void obsoleteStepsNotRedispatched() {
    // After adaptation, publish CONTEXT_CHANGED
    // Assert: OBSOLETE steps not picked up by PlanningStrategyLoopControl
}
```

- [ ] **Step 2: Run integration test to verify it fails**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest=PlanAdaptationIntegrationTest -q`
Expected: FAIL (integration not wired yet or test infrastructure issues)

- [ ] **Step 3: Add lock lifecycle cleanup**

In `DefaultPlanAdaptationEvaluator`, add listener for `CompoundCompletedEvent`:

```java
@ConsumeEvent(value = EventBusAddresses.COMPOUND_COMPLETED, blocking = true)
public void onCompoundCompleted(CompoundCompletedEvent event) {
    String key = event.caseId() + ":" + event.compoundId();
    compoundLocks.remove(key);
    // adaptationGenerations cleanup handled by CasePlanModel lifecycle
}
```

Also listen for case terminal status to clean all locks for that case:

```java
// Clean locks when case completes
public void cleanLocksForCase(UUID caseId) {
    compoundLocks.keySet().removeIf(k -> k.startsWith(caseId + ":"));
}
```

- [ ] **Step 4: Fix and iterate on integration test until passing**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -Dtest=PlanAdaptationIntegrationTest -q`
Expected: all PASS

- [ ] **Step 5: Run full module test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl planning -q`
Expected: all PASS

- [ ] **Step 6: Run api module tests (YAML mapper + SPI types)**

Run: `mvn test -pl api -q`
Expected: all PASS

- [ ] **Step 7: Commit**

```
feat(#803): plan adaptation integration test + lock lifecycle cleanup

Closes #803
```
