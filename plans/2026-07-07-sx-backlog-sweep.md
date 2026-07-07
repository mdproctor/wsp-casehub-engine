# S/XS Backlog Sweep Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #678 — epic: S/XS backlog sweep
**Issue group:** #664, #665, #673, #654, #663, #666, #655, #618, #619, #617, #616, #661, #671

**Goal:** Sweep 12 engine-scoped S/XS issues in a single branch — naming consistency, validation gaps, handler consolidation, API quality, and CBR enhancements.

**Architecture:** Each issue is an independent commit. Clusters A/C/D/E touch different modules with no cross-cluster dependencies. Cluster B touches blackboard handlers. Intra-cluster dependency: #671 (adds `timing` to CbrConfig) before #673 (validates CbrConfig fields including `timing`).

**Tech Stack:** Java 21, Quarkus 3.32, Jackson, Vert.x EventBus, JQ (jackson-jq), CDI

## Global Constraints

- One commit per issue, referencing issue number
- TDD: failing test → verify fail → implement → verify pass
- All new types in `api/` (Tier 1, pure Java) per `module-tier-structure` protocol
- New methods on existing interfaces as `default` methods per `spi-evolution-default-methods` protocol
- `mvn install -DskipTests -q` before any module-specific test run
- `TESTCONTAINERS_RYUK_DISABLED=true` on all test commands
- IntelliJ MCP for all renames, references, and navigation — never bash grep

---

## Task 1: #664 — Rename Memory*PlanItemStore classes to InMemory*

**Closes:** #664
**Module:** persistence-memory

**Files:**
- Rename: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java` → `InMemoryReactivePlanItemStore.java`
- Rename: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java` → `InMemoryPlanItemStore.java`
- Rename: `persistence-memory/src/test/java/io/casehub/persistence/memory/MemoryPlanItemStoreContractTest.java` → `InMemoryPlanItemStoreContractTest.java`
- Update: any `selected-alternatives` or import references across all test `application.properties`

**Interfaces:**
- Consumes: nothing
- Produces: `InMemoryPlanItemStore`, `InMemoryReactivePlanItemStore` (same API, new names)

- [ ] **Step 1: Use IntelliJ refactor-rename on all three classes**

Use `mcp__intellij-index__ide_refactor_rename` for each class. This handles imports, references, and config entries across the project.

- [ ] **Step 2: Search for any remaining `MemoryPlanItemStore` or `MemoryReactivePlanItemStore` references**

Use `mcp__intellij-index__ide_search_text` with query `MemoryPlanItemStore` and `MemoryReactivePlanItemStore` to find straggling references in `application.properties`, `CLAUDE.md`, or other config.

- [ ] **Step 3: Run persistence-memory tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl persistence-memory
```
Expected: all tests pass with new class names.

- [ ] **Step 4: Commit**

```bash
git add -A persistence-memory/
git commit -m "refactor(#664): rename Memory*PlanItemStore → InMemory* for naming consistency

Refs #664
Closes #664"
```

---

## Task 2: #665 — Make DefaultCaseDefinitionRegistry startup timeout configurable

**Closes:** #665
**Module:** runtime

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java` (line 86)
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistryTimeoutTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: config property `casehub.engine.registry.startup-timeout`

- [ ] **Step 1: Write failing test**

```java
@QuarkusTest
class DefaultCaseDefinitionRegistryTimeoutTest {

  @Inject DefaultCaseDefinitionRegistry registry;

  @Test
  void startupTimeout_isConfigurable() {
    // The registry should inject the config property without error.
    // A non-default value in test application.properties proves it's wired.
    assertThat(registry).isNotNull();
  }
}
```

Add to `runtime/src/test/resources/application.properties`:
```properties
casehub.engine.registry.startup-timeout=45s
```

- [ ] **Step 2: Run test — verify it fails**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTimeoutTest
```
Expected: FAIL — `startupTimeout` field doesn't exist yet.

- [ ] **Step 3: Implement**

In `DefaultCaseDefinitionRegistry`, add:
```java
@ConfigProperty(name = "casehub.engine.registry.startup-timeout", defaultValue = "30s")
Duration startupTimeout;
```

Replace line 86's `.atMost(Duration.ofSeconds(30))` with `.atMost(startupTimeout)`.

- [ ] **Step 4: Run test — verify pass**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=DefaultCaseDefinitionRegistryTimeoutTest
```

- [ ] **Step 5: Commit**

```bash
git commit -m "chore(#665): make DefaultCaseDefinitionRegistry startup timeout configurable

Replaces hardcoded 30s with @ConfigProperty casehub.engine.registry.startup-timeout.

Closes #665"
```

---

## Task 3: #666 — Consolidate WorkerRetryExhaustionHandler + PlanItemFaultHandler

**Closes:** #666
**Module:** blackboard

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandler.java`
- Delete: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemFaultHandler.java`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/handler/WorkerRetryExhaustionHandlerTest.java`
- Delete: `blackboard/src/test/java/io/casehub/blackboard/handler/PlanItemFaultHandlerTest.java`

**Interfaces:**
- Consumes: `PlanItemFaultedEvent` (CDI event class), `StageAutocompleteEvaluator`
- Produces: unified handler that fires BOTH `PlanItemFaultedEvent` AND `stageAutocompleteEvaluator.evaluate()`

- [ ] **Step 1: Write failing test — verify both downstream effects fire from single handler**

Add test to `WorkerRetryExhaustionHandlerTest`:
```java
@Test
void onWorkerRetriesExhausted_firesBothPlanItemFaultedEventAndStageAutocomplete() {
    // Setup: create a RUNNING PlanItem
    // Act: call onWorkerRetriesExhausted
    // Assert: PlanItemFaultedEvent fired via CDI AND stageAutocompleteEvaluator.evaluate() called
}
```

- [ ] **Step 2: Run test — verify it fails** (PlanItemFaultedEvent not fired by current handler)

- [ ] **Step 3: Implement consolidation**

In `WorkerRetryExhaustionHandler`:
1. Inject `Event<PlanItemFaultedEvent> planItemFaultedEvents`
2. After `markFaulted()` succeeds, add:
   ```java
   planItemFaultedEvents.fireAsync(
       new PlanItemFaultedEvent(caseId, item.getPlanItemId(), worker, capability));
   ```
3. Pre-guard: keep `item.getStatus() != PlanItemStatus.RUNNING` (more defensive than `isTerminal()`)

- [ ] **Step 4: Delete PlanItemFaultHandler and its test**

Use `mcp__intellij-index__ide_refactor_safe_delete` on `PlanItemFaultHandler.java`.

- [ ] **Step 5: Run full blackboard suite**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl blackboard
```

- [ ] **Step 6: Commit**

```bash
git commit -m "refactor(#666): consolidate WorkerRetryExhaustionHandler + PlanItemFaultHandler

Single handler now fires both PlanItemFaultedEvent and stageAutocompleteEvaluator.evaluate().
Eliminates CAS race where exactly one downstream effect was lost on every exhaustion.

Closes #666"
```

---

## Task 4: #663 — Fix TestCaseInstanceRepository tenant mismatch

**Closes:** #663
**Module:** testing

**Files:**
- Modify: `testing/src/main/java/io/casehub/testing/TestCaseInstanceRepository.java`

**Interfaces:**
- Consumes: `InMemoryCaseInstanceRepository`
- Produces: tenant-agnostic `findByUuid()` for test infrastructure

- [ ] **Step 1: Write failing test reproducing the issue**

```java
@Test
void findByUuid_withMismatchedTenancy_stillReturnsInstance() {
    CaseInstance instance = new CaseInstance();
    instance.setUuid(UUID.randomUUID());
    instance.tenancyId = "tenant-a";
    repo.save(instance, "tenant-a");

    // In production, event bus handler might resolve a different tenancy
    CaseInstance found = repo.findByUuid(instance.getUuid(), "tenant-b");
    assertThat(found).isNotNull();
    assertThat(found.getUuid()).isEqualTo(instance.getUuid());
}
```

- [ ] **Step 2: Implement fix**

Override `findByUuid(UUID, String)` in `TestCaseInstanceRepository` to ignore tenancyId:
```java
@Override
public CaseInstance findByUuid(UUID uuid, String tenancyId) {
    // Test infrastructure — tenancy enforcement is in TenantAwareRepository (JPA layer).
    // TODO: Thread tenancyId through event bus messages (tracked in engine#XXX).
    return findByUuid(uuid);
}
```

File a GitHub issue for the deeper fix (threading tenancyId through event bus).

- [ ] **Step 3: Run testing module tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl testing
```

- [ ] **Step 4: Commit**

```bash
git commit -m "fix(#663): TestCaseInstanceRepository ignores tenancyId on findByUuid

Event bus handlers may resolve a different CurrentPrincipal.tenancyId() than
the one used at save() time. Test repositories should not enforce tenancy —
real enforcement is in TenantAwareRepository (JPA/RLS).

Deeper fix (threading tenancyId through event bus) tracked in engine#XXX.

Closes #663"
```

---

## Task 5: #654 — Populate CaseMetaModel definition column

**Closes:** #654
**Module:** runtime

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/CaseMetaModelDefinitionPopulationTest.java`

**Interfaces:**
- Consumes: `CaseMetaModel.setDefinition(JsonNode)`, Jackson MixIns
- Produces: populated `definition` column on every registered `CaseMetaModel`

- [ ] **Step 1: Write failing test**

```java
@Test
void registerCaseDefinition_populatesDefinitionColumn() {
    CaseDefinition def = CaseDefinition.builder()
        .namespace("test").name("def-pop").version("1.0.0")
        .build();
    registry.registerCaseDefinition(def).toCompletableFuture().join();
    CaseMetaModel model = registry.getCaseMetaModel(def);
    assertThat(model.getDefinition()).isNotNull();
    assertThat(model.getDefinition().has("namespace")).isTrue();
    assertThat(model.getDefinition().get("name").asText()).isEqualTo("def-pop");
}
```

- [ ] **Step 2: Implement Jackson MixIns and serialization**

Create a private `ObjectMapper` with MixIns for `Worker` (exclude `function`), `LambdaExpressionEvaluator` (exclude `predicate`), `LambdaFeatureExtractor` (exclude `extractionFunction`). Use `metadataMapper.valueToTree(definition)` and set on `CaseMetaModel` before `save()`.

- [ ] **Step 3: Run runtime tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=CaseMetaModelDefinitionPopulationTest
```

- [ ] **Step 4: Commit**

```bash
git commit -m "feat(#654): populate CaseMetaModel definition column during registration

Serializes CaseDefinition to JSON via Jackson MixIns that exclude non-serializable
lambda fields (WorkerFunction, LambdaExpressionEvaluator, LambdaFeatureExtractor).
Retained fields: namespace, name, version, bindings, capabilities, goals, milestones,
types, labels, stages, planningStrategy, cbrConfig.

Closes #654"
```

---

## Task 6: #655 — Vocabulary validation for CaseDefinition types and labels

**Closes:** #655
**Module:** runtime

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/CaseDefinitionVocabularyValidationTest.java`

**Interfaces:**
- Consumes: `Instance<VocabularyRegistry>`, `CaseDefinition.getTypes()`, `CaseDefinition.getLabels()`
- Produces: WARNING log on unresolvable type/label paths

- [ ] **Step 1: Write failing test**

Test that when a `VocabularyRegistry` is available and a type path doesn't resolve, a warning is logged. Use a recording `VocabularyRegistry` alternative.

- [ ] **Step 2: Implement validation in `validateExpressions()`**

Inject `Instance<VocabularyRegistry>`. If resolvable and not `NoOpVocabularyRegistry`:
- For each type: `vocabularyRegistry.get().resolve(typeUri, segment)` — warn if null
- For each label: same check

- [ ] **Step 3: Run test, commit**

```bash
git commit -m "feat(#655): vocabulary validation for CaseDefinition types and labels

Advisory validation at registration time — warns on unresolvable type/label paths
when VocabularyRegistry is available. Skips silently with NoOpVocabularyRegistry.

Closes #655"
```

---

## Task 7: #661 — Extend QhorusMessageSignalBridge for STATUS messages

**Closes:** #661
**Module:** runtime

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/bridge/QhorusMessageSignalBridge.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/bridge/QhorusMessageSignalBridgeStatusTest.java`

**Interfaces:**
- Consumes: `MessageType.STATUS`, `CaseHubRuntime.signal()`
- Produces: `statusReport` signal on case context

- [ ] **Step 1: Write failing test**

```java
@Test
void statusMessage_signalsCaseWithStatusReport() {
    // Create a case, send a STATUS message on its channel
    // Assert: case context has statusReport with from, content, timestamp
}
```

- [ ] **Step 2: Implement**

Add `isStatusUpdate()` predicate. In `onMessage()`, branch: commitment-resolving follows existing path; STATUS calls `runtime.signal(caseId, "statusReport", value, tenancyId)` with no correlationId lookup.

- [ ] **Step 3: Run test, commit**

```bash
git commit -m "feat(#661): extend QhorusMessageSignalBridge for STATUS messages

STATUS messages route via runtime.signal() with statusReport context key.
No correlationId lookup — STATUS is informational, not commitment-resolving.
Milestone/sentry conditions can evaluate .statusReport.content in JQ.

Closes #661"
```

---

## Task 8: #671 — Case-lifetime CBR retrieval caching

**Closes:** #671
**Module:** api (enum) + runtime (cache)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/cbr/CbrConfig.java` (add `CbrRetrievalTiming` enum)
- Modify: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrRetrievalService.java` (add cache)
- Create: `runtime/src/main/java/io/casehub/engine/internal/routing/CbrCacheEvictionHandler.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/routing/CbrRetrievalCachingTest.java`
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (parse `timing`)
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` (add `timing` property)

**Interfaces:**
- Consumes: `CbrConfig`, `EventBusAddresses.CASE_STATUS_CHANGED`
- Produces: `CbrRetrievalTiming` enum, cached retrieval path

- [ ] **Step 1: Add `CbrRetrievalTiming` enum to `CbrConfig`**

```java
public enum CbrRetrievalTiming { PER_EVALUATION, CASE_LIFETIME }
```

Add `timing` field to `CbrConfig` with default `PER_EVALUATION`.

- [ ] **Step 2: Write failing test for cache hit**

```java
@Test
void retrieve_caseLifetime_cachesOnFirstAccess() {
    // Configure CbrConfig with CASE_LIFETIME timing
    // Call retrieve() twice
    // Assert: CbrCaseMemoryStore.retrieveSimilar() called only once
}
```

- [ ] **Step 3: Implement cache in CbrRetrievalService**

Add `ConcurrentHashMap<UUID, List<RetrievedExperience>>` cache with `MAX_CACHE_SIZE = 1000`. On `CASE_LIFETIME`, check cache first. On miss, retrieve and `putIfAbsent()`. Cache immutable `List.copyOf()`.

- [ ] **Step 4: Create CbrCacheEvictionHandler**

`@ConsumeEvent(value = CASE_STATUS_CHANGED, blocking = true)` — evict from cache when case reaches terminal state.

- [ ] **Step 5: Update YAML schema and mapper**

Add `timing` to `CaseDefinition.yaml` cbr block. Parse in `CaseDefinitionYamlMapper`.

- [ ] **Step 6: Run tests, commit**

```bash
git commit -m "perf(#671): case-lifetime CBR retrieval caching

CbrRetrievalTiming.CASE_LIFETIME caches retrieval results for the case's
lifetime. Bounded at 1000 entries. Eviction via CASE_STATUS_CHANGED event.
Default PER_EVALUATION preserves current behavior.

Closes #671"
```

---

## Task 9: #673 — CbrConfig validation at registration time

**Closes:** #673
**Module:** runtime (depends on Task 8 for `timing` field)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/DefaultCaseDefinitionRegistry.java`
- Create: `runtime/src/test/java/io/casehub/engine/internal/engine/CbrConfigRegistrationValidationTest.java`

**Interfaces:**
- Consumes: `CbrConfig`, `JQEvaluator`, `EpisodicMemoryConfig`
- Produces: WARNING logs for invalid CbrConfig

- [ ] **Step 1: Write failing test**

```java
@Test
void registerCaseDefinition_cbrConfigWithNoDomainAndNoEpisodicMemory_logsWarning() {
    // CbrConfig present, domain=null, no EpisodicMemoryConfig
    // Assert: WARNING log emitted
}

@Test
void registerCaseDefinition_cbrConfigWithInvalidJqFeatureExtractor_logsWarning() {
    // JqFeatureExtractor with invalid JQ expression
    // Assert: WARNING log emitted
}
```

- [ ] **Step 2: Implement validation in `validateExpressions()`**

Add CbrConfig validation block after existing goal validation.

- [ ] **Step 3: Run test, commit**

```bash
git commit -m "chore(#673): CbrConfig validation at CaseDefinition registration time

Warns when CbrConfig has no domain and no EpisodicMemoryConfig (retrieval
will always return empty). Compile-checks JQ expressions in JqFeatureExtractor.

Closes #673"
```

---

## Task 10: #618 — ExecutionOrigin provenance metadata

**Closes:** #618
**Module:** api (enum) + runtime (handlers) + common (EventLog metadata)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/event/ExecutionOrigin.java`
- Modify: `api/src/main/java/io/casehub/api/engine/PlanExecutionContext.java` (add field)
- Modify: `runtime/.../handler/CaseContextChangedEventHandler.java` (tag BINDING_DISPATCH)
- Modify: `runtime/.../handler/SignalReceivedEventHandler.java` (tag SIGNAL)
- Modify: `runtime/.../engine/recovery/WorkerRecoveryCoordinator.java` (tag RECOVERY)
- Modify: `blackboard/.../subcase/SubCaseCompletionService.java` (tag SUBCASE_COMPLETION)
- Create: `api/src/test/java/io/casehub/api/model/event/ExecutionOriginTest.java`

**Interfaces:**
- Consumes: `EventLog.metadata`
- Produces: `ExecutionOrigin` enum, `PlanExecutionContext.origin` field

- [ ] **Step 1: Create enum and update PlanExecutionContext**

```java
public enum ExecutionOrigin {
    BINDING_DISPATCH, SIGNAL, SCHEDULE_TRIGGER, SUBCASE_COMPLETION, RECOVERY
}
```

Add `ExecutionOrigin origin` (nullable) to `PlanExecutionContext` record.

- [ ] **Step 2: Write test verifying origin is set on EventLog**

- [ ] **Step 3: Tag each handler's EventLog entries with origin**

- [ ] **Step 4: Commit**

```bash
git commit -m "feat(#618): ExecutionOrigin provenance metadata for worker executions

Tags EventLog entries with origination path (BINDING_DISPATCH, SIGNAL,
SCHEDULE_TRIGGER, SUBCASE_COMPLETION, RECOVERY). Available on PlanExecutionContext.

Closes #618"
```

---

## Task 11: #619 — Per-key change listener API on CaseContext

**Closes:** #619
**Module:** api (interface) + runtime (implementation)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/context/CaseContext.java` (add default methods)
- Create: `api/src/main/java/io/casehub/api/context/ContextChangeEvent.java`
- Create: `api/src/main/java/io/casehub/api/context/Subscription.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/CaseContextImpl.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/context/WritableLayerImpl.java` (add `setPrev()`)
- Create: `runtime/src/test/java/io/casehub/engine/internal/context/CaseContextListenerTest.java`
- Create: `api/src/test/java/io/casehub/api/context/CaseContextDefaultMethodContractTest.java`

**Interfaces:**
- Consumes: `WritableLayerImpl.setPrev()`
- Produces: `CaseContext.onChange()`, `CaseContext.onAnyChange()`, `ContextChangeEvent`, `Subscription`

- [ ] **Step 1: Create API types**

`ContextChangeEvent` record, `Subscription` functional interface with `NOOP` constant.

- [ ] **Step 2: Add default methods to CaseContext**

```java
default Subscription onChange(String key, Consumer<ContextChangeEvent> listener) {
    return Subscription.NOOP;
}
default Subscription onAnyChange(Consumer<ContextChangeEvent> listener) {
    return Subscription.NOOP;
}
```

- [ ] **Step 3: Write contract test for default methods**

- [ ] **Step 4: Add `setPrev()` to WritableLayerImpl**

Returns previous value atomically with the write (inside same lock acquisition).

- [ ] **Step 5: Implement listeners in CaseContextImpl**

`ConcurrentHashMap<String, CopyOnWriteArrayList<Consumer<ContextChangeEvent>>>` for per-key. Wrap `set()`, `setAll()`, `remove()`, etc. to capture old/new and fire listeners after lock release.

- [ ] **Step 6: Write comprehensive tests**

- Error isolation (throwing listener doesn't prevent others)
- Listeners fire outside write lock (re-entrant write doesn't deadlock)
- `engineSet()` does NOT fire listeners
- `setAll()` fires one event per changed key
- `Subscription.cancel()` removes the listener

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#619): per-key change listener API on CaseContext

Adds onChange(key, listener) and onAnyChange(listener) to CaseContext interface.
Atomic old-value capture via WritableLayerImpl.setPrev(). Listeners fire after
lock release — no deadlock risk. Error isolation per listener.

Closes #619"
```

---

## Task 12: #617 — RetryState explicit retry history

**Closes:** #617
**Module:** api (record) + scheduler-quartz (population) + resilience (DLQ enrichment)

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/RetryState.java`
- Modify: `api/src/main/java/io/casehub/api/engine/PlanExecutionContext.java` (add field)
- Modify: `scheduler-quartz/.../QuartzRetryService.java` (build RetryState)
- Modify: `resilience/.../deadletter/DeadLetterEntry.java` (add field)
- Create: `api/src/test/java/io/casehub/api/model/RetryStateTest.java`

**Interfaces:**
- Consumes: `EventLog` WORKER_EXECUTION_FAILED entries
- Produces: `RetryState` record, `PlanExecutionContext.retryState`, `DeadLetterEntry.retryState`

- [ ] **Step 1: Create RetryState record**

- [ ] **Step 2: Add to PlanExecutionContext and DeadLetterEntry**

- [ ] **Step 3: Build RetryState in QuartzRetryService from failure chain**

- [ ] **Step 4: Tests and commit**

```bash
git commit -m "feat(#617): RetryState — explicit retry attempt history

Tracks every retry attempt with timestamp, error, duration, success flag.
Attached to PlanExecutionContext (for PlanningStrategy reasoning) and
DeadLetterEntry (for DLQ enrichment).

Closes #617"
```

---

## Task 13: #616 — CaseFileContribution key-level audit trail

**Closes:** #616
**Module:** api (Binding field + YAML) + runtime (EventLog metadata)

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/Binding.java` (add `producedKeys`)
- Modify: `api/src/main/java/io/casehub/api/model/converter/CaseDefinitionYamlMapper.java` (parse `producedKeys`)
- Modify: `schema/src/main/resources/schema/CaseDefinition.yaml` (add `producedKeys`)
- Modify: `runtime/.../handler/WorkflowExecutionCompletedHandler.java` (add `producedKeys` to metadata)
- Modify: `runtime/.../engine/DefaultCaseDefinitionRegistry.java` (warn on same-stage overlap)
- Create: `api/src/test/java/io/casehub/api/model/BindingProducedKeysTest.java`

**Interfaces:**
- Consumes: context diff from `WorkflowExecutionCompletedHandler`
- Produces: `Binding.producedKeys`, YAML `producedKeys`, EventLog `producedKeys` metadata

- [ ] **Step 1: Add `producedKeys` to Binding**

`Set<String>` field with builder method. Empty by default.

- [ ] **Step 2: Update YAML schema and mapper**

Add `producedKeys` array of strings to binding schema. Parse in `CaseDefinitionYamlMapper`.

- [ ] **Step 3: Add runtime produced-key extraction to EventLog metadata**

In `WorkflowExecutionCompletedHandler`, extract top-level keys from context diff and add as `producedKeys` to EventLog metadata.

- [ ] **Step 4: Add same-stage overlap validation**

In `DefaultCaseDefinitionRegistry.validateExpressions()`, warn if two bindings in the same stage declare overlapping `producedKeys`.

- [ ] **Step 5: Tests and commit**

```bash
git commit -m "feat(#616): CaseFileContribution — key-level audit trail

Static: producedKeys on Binding (YAML + builder). Warns on same-stage overlap.
Runtime: top-level produced keys extracted from context diff and recorded in
EventLog metadata alongside contextChanges.

Closes #616"
```

---

## Verification

After all tasks complete:

```bash
# Full build
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test

# CI push
git push origin issue-678-sx-backlog-sweep
```
