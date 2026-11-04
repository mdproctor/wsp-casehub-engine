# CDI Event Dispatch Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #798 — Replace Vert.x @ConsumeEvent with CDI @ObservesAsync for internal engine events

**Goal:** Eliminate stale event bus consumer bugs by migrating all internal engine events from Vert.x `@ConsumeEvent` + `eventBus.publish()` to CDI `@ObservesAsync` + `Event.fireAsync()`.

**Architecture:** CDI observers are lifecycle-managed — destroyed with the CDI context, recreated on restart. This eliminates the root cause: Vert.x event bus consumers outliving their CDI bean instances after CDI context restarts (test, dev mode). The migration is mechanical because every event bus address already maps to a unique typed event class.

**Tech Stack:** Quarkus CDI (ArC), Jakarta CDI Events (`Event.fireAsync()`, `@ObservesAsync`), virtual threads via `@VirtualThreads ExecutorService`

## Global Constraints

- Every `eventBus.publish(ADDRESS, event)` becomes `cdiEvent.fireAsync(event)` — fire-and-forget semantics preserved
- Every `@ConsumeEvent(ADDRESS) void handle(T e)` becomes `void handle(@ObservesAsync T e)`
- `@RunOnVirtualThread` is not supported with `@ObservesAsync` — observers that need virtual thread execution wrap their body in `virtualThreads.submit()`
- `blocking = true` (Vert.x concept) has no CDI equivalent — CDI async observers already run off the caller thread. Handlers that need transactional context acquire it explicitly via `QuarkusTransaction.requiringNew()` or `@Transactional` on an injected helper
- `EventBusAddresses` constants and the `EventBus` injection are removed incrementally — constants are deleted once all publish/consume sites for that address are migrated
- Existing event types (`CaseContextChangedEvent`, `WorkerScheduleEvent`, etc.) are reused as-is — no new types needed
- `CaseEvaluationSerializer` is preserved — it serializes per-case evaluation regardless of dispatch mechanism
- Tests that use `eventBus.publish()` to trigger handlers migrate to injecting `Event<T>` and calling `fireAsync()`
- Layer-changed events (`EventBusAddresses.layerChanged(name)`) use dynamic string addresses — these stay on the Vert.x event bus until a CDI-native alternative is designed (no consumers in production code; only used for monitoring)

---

### Task 1: Migrate CONTEXT_CHANGED — the immediate fix

The event that triggers the bug. Two consumers, 12 publish sites.

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseContextChangedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/milestone/MilestoneLifecycleManager.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStartedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/SignalReceivedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/WorkflowExecutionCompletedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/CaseStatusChangedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneCompletedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneActivatedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneSLAViolatedEventHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ActionGateRejectedHandler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/ActionGateExpiredHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/PlanItemCompletionHandler.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/handler/WorkerOutcomeResolvedHandler.java`
- Test: `common/src/test/java/io/casehub/engine/common/internal/model/CaseKeyRegistrySimulationTest.java` (already exists from investigation)
- Test: `runtime/src/test/java/io/casehub/engine/internal/engine/RegistryLookupPipelineTest.java` (already exists from investigation)

**Interfaces:**
- Consumes: `CaseContextChangedEvent` (existing record, unchanged)
- Produces: Pattern for all subsequent tasks — `@Inject Event<T>` + `fireAsync()` on publisher side, `@ObservesAsync T` on consumer side

- [ ] **Step 1: Convert consumers — remove @ConsumeEvent, add @ObservesAsync**

`CaseContextChangedEventHandler`: replace `@RunOnVirtualThread @ConsumeEvent(value = EventBusAddresses.CONTEXT_CHANGED)` with just `@ObservesAsync`. The method already delegates to `evaluationSerializer.submit()` which runs on virtual threads.

```java
// Before
@RunOnVirtualThread
@ConsumeEvent(value = EventBusAddresses.CONTEXT_CHANGED)
public void onCaseStateContextChangedEventHandler(final CaseContextChangedEvent event) {

// After
public void onCaseStateContextChangedEventHandler(@ObservesAsync final CaseContextChangedEvent event) {
```

`MilestoneLifecycleManager`: same pattern.

```java
// Before
@RunOnVirtualThread
@ConsumeEvent(value = EventBusAddresses.CONTEXT_CHANGED)
public void onContextChanged(CaseContextChangedEvent event) {

// After
public void onContextChanged(@ObservesAsync CaseContextChangedEvent event) {
```

Remove the `EventBus` injection from `CaseContextChangedEventHandler` ONLY if no other addresses are published from this class. (It still publishes `layerChanged` and other events — keep `EventBus` for now, remove `CONTEXT_CHANGED` import.)

- [ ] **Step 2: Convert publishers — replace eventBus.publish(CONTEXT_CHANGED, ...) with Event.fireAsync()**

For each publisher, add `@Inject Event<CaseContextChangedEvent> contextChangedEvents;` and replace:

```java
// Before
eventBus.publish(EventBusAddresses.CONTEXT_CHANGED, new CaseContextChangedEvent(...));

// After
contextChangedEvents.fireAsync(new CaseContextChangedEvent(...));
```

Publisher sites (12):
1. `CaseStartedEventHandler:140`
2. `SignalReceivedEventHandler:220` (onSignalReceived)
3. `SignalReceivedEventHandler:299` (onBulkSignalReceived)
4. `SignalReceivedEventHandler:363` (onTypedSignalReceived)
5. `WorkflowExecutionCompletedHandler:207`
6. `CaseStatusChangedHandler:173`
7. `MilestoneCompletedEventHandler:159`
8. `MilestoneActivatedEventHandler:174`
9. `MilestoneSLAViolatedEventHandler:111`
10. `ActionGateRejectedHandler:152`
11. `ActionGateExpiredHandler:137`
12. `PlanItemCompletionHandler:104` (blackboard)
13. `WorkerOutcomeResolvedHandler:108` (blackboard)

- [ ] **Step 3: Update tests that publish CONTEXT_CHANGED via eventBus**

`MilestoneLifecycleTest:236` uses `eventBus.publish(EventBusAddresses.CONTEXT_CHANGED, event)`. Replace with `@Inject Event<CaseContextChangedEvent>` + `fireAsync()`.

- [ ] **Step 4: Run AML Layer 5 test to verify the fix**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -f <AML>/pom.xml -pl app -Dtest=AmlLayer5InvestigationTest -o
```

Expected: all 5 tests pass. The stale-consumer bug is eliminated because CDI observers are lifecycle-managed.

- [ ] **Step 5: Commit**

```
fix(#798): migrate CONTEXT_CHANGED from Vert.x event bus to CDI events

Refs #798
```

---

### Task 2: Migrate WORKER_EXECUTION_FINISHED

**Files:**
- Modify: `runtime/.../handler/WorkflowExecutionCompletedHandler.java` (consumer)
- Modify: `blackboard/.../handler/PlanItemCompletionHandler.java` (consumer)
- Modify: `scheduler-quartz/.../QuartzWorkerExecutionJob.java` (publisher)
- Modify: `runtime/.../bridge/QhorusMessageSignalBridge.java` (publisher)
- Modify: `testing/.../WorkResultSubmitter.java` (publisher)

**Interfaces:**
- Event type: `WorkflowExecutionCompleted` (existing)

- [ ] **Step 1: Convert consumers**

`WorkflowExecutionCompletedHandler`: `@RunOnVirtualThread @ConsumeEvent(WORKER_EXECUTION_FINISHED)` → `@ObservesAsync`

`PlanItemCompletionHandler`: `@ConsumeEvent(WORKER_EXECUTION_FINISHED, blocking = true)` → `@ObservesAsync`. Note: `blocking = true` meant the handler ran on a worker thread, not the event loop. `@ObservesAsync` already runs off-caller-thread. If the handler needs `@Transactional`, add it to the method or a helper.

- [ ] **Step 2: Convert publishers**

Add `@Inject Event<WorkflowExecutionCompleted> workerFinishedEvents` to each publisher. Replace `eventBus.publish(WORKER_EXECUTION_FINISHED, event)` with `workerFinishedEvents.fireAsync(event)`.

- [ ] **Step 3: Commit**

```
refactor(#798): migrate WORKER_EXECUTION_FINISHED to CDI events

Refs #798
```

---

### Task 3: Migrate CASE_STATUS_CHANGED

**Files:**
- Modify: `runtime/.../handler/CaseStatusChangedHandler.java` (publisher — publishes to CASE_STATUS_CHANGED)
- Modify: `runtime/.../routing/CbrCacheEvictionHandler.java` (consumer)
- Modify: `blackboard/.../handler/CaseEvictionHandler.java` (consumer)
- Modify: `runtime/.../handler/GoalReachedEventHandler.java` (publisher)

**Interfaces:**
- Event type: `CaseStatusChanged` (existing)

- [ ] **Step 1: Convert 3 consumers to @ObservesAsync**
- [ ] **Step 2: Convert publishers to Event.fireAsync()**
- [ ] **Step 3: Commit**

```
refactor(#798): migrate CASE_STATUS_CHANGED to CDI events

Refs #798
```

---

### Task 4: Migrate WORKER_SCHEDULE and WORKER_RETRIES_EXHAUSTED

**Files:**
- Modify: `runtime/.../handler/WorkerScheduleEventHandler.java` (consumer)
- Modify: `runtime/.../handler/WorkerRetriesExhaustedEventHandler.java` (consumer)
- Modify: `blackboard/.../handler/WorkerRetryExhaustionHandler.java` (consumer)
- Modify: `resilience/.../deadletter/DeadLetterEventHandler.java` (consumer)
- Modify: `runtime/.../handler/CaseContextChangedEventHandler.java` (publisher — WORKER_SCHEDULE)
- Modify: `scheduler-quartz/.../QuartzRetryService.java` (publisher — WORKER_RETRIES_EXHAUSTED)
- Modify: `resilience/.../timeout/CaseTimeoutEnforcer.java` (publisher)

**Interfaces:**
- Event types: `WorkerScheduleEvent`, `WorkerRetriesExhaustedEvent` (both existing)

- [ ] **Step 1: Convert consumers**
- [ ] **Step 2: Convert publishers**
- [ ] **Step 3: Commit**

```
refactor(#798): migrate WORKER_SCHEDULE + WORKER_RETRIES_EXHAUSTED to CDI events

Refs #798
```

---

### Task 5: Migrate milestone events (REACHED, ACTIVATED, COMPLETED, SLA_VIOLATED)

**Files:**
- Modify: `runtime/.../handler/MilestoneReachedEventHandler.java`
- Modify: `runtime/.../handler/MilestoneActivatedEventHandler.java`
- Modify: `runtime/.../handler/MilestoneCompletedEventHandler.java`
- Modify: `runtime/.../handler/MilestoneSLAViolatedEventHandler.java`
- Modify: `blackboard/.../handler/MilestoneAchievementHandler.java` (2 consumers)
- Modify: `runtime/.../milestone/MilestoneLifecycleManager.java` (publisher)
- Modify: `scheduler-quartz/.../MilestoneSLATimeoutJob.java` (publisher)

**Interfaces:**
- Event types: `MilestoneReachedEvent`, `MilestoneActivatedEvent`, `MilestoneCompletedEvent`, `MilestoneSLAViolatedEvent` (all existing)

- [ ] **Step 1: Convert 6 consumers**
- [ ] **Step 2: Convert publishers**
- [ ] **Step 3: Commit**

```
refactor(#798): migrate milestone events to CDI events

Refs #798
```

---

### Task 6: Migrate GOAL_REACHED, SUBCASE_SCHEDULE, WORKER_OUTCOME_RESOLVED, AGENT_ROUTING_ESCALATION

**Files:**
- Modify: `runtime/.../handler/GoalReachedEventHandler.java` (consumer + publisher)
- Modify: `blackboard/.../subcase/SubCaseExecutionHandler.java` (consumer)
- Modify: `blackboard/.../handler/WorkerOutcomeResolvedHandler.java` (consumer)
- Modify: `runtime/.../handler/AgentRoutingEscalationHandler.java` (consumer)
- Modify: `runtime/.../handler/CaseContextChangedEventHandler.java` (publishers)
- Modify: `runtime/.../orchestration/DefaultWorkOrchestrator.java` (publisher)

**Interfaces:**
- Event types: `GoalReachedEvent`, `SubCaseScheduleEvent`, `WorkerOutcomeResolvedEvent`, `AgentRoutingEscalationEvent` (all existing)

- [ ] **Step 1: Convert 4 consumers**
- [ ] **Step 2: Convert publishers**
- [ ] **Step 3: Commit**

```
refactor(#798): migrate goal/subcase/outcome/escalation events to CDI events

Refs #798
```

---

### Task 7: Migrate action gate events and blackboard-internal events

**Files:**
- Modify: `runtime/.../handler/ActionGateApprovedHandler.java`
- Modify: `runtime/.../handler/ActionGateRejectedHandler.java`
- Modify: `runtime/.../handler/ActionGateExpiredHandler.java`
- Modify: `blackboard/.../handler/ActionGateExpiredPlanItemHandler.java`
- Modify: `blackboard/.../handler/ActionGateRejectedPlanItemHandler.java`
- Modify: `blackboard/.../handler/StageResetOutcomesCleaner.java`
- Modify: `blackboard/.../handler/PlanItemCompletionHandler.java` (SUBCASE_EXECUTION_COMPLETED consumer)
- Modify: `blackboard/.../control/StageLifecycleEvaluator.java` (publisher — STAGE_ACTIVATED)
- Modify: `blackboard/.../subcase/SubCaseCompletionService.java` (publisher — SUBCASE_EXECUTION_COMPLETED)

**Interfaces:**
- Event types: `ActionGateApprovedEvent`, `ActionGateRejectedEvent`, `ActionGateExpiredEvent`, `ActionGateWorkerFaultedEvent`, `StageActivatedEvent`, `SubCaseExecutionCompleted` (all existing)

- [ ] **Step 1: Convert 7 consumers**
- [ ] **Step 2: Convert publishers**
- [ ] **Step 3: Commit**

```
refactor(#798): migrate action gate + blackboard-internal events to CDI events

Refs #798
```

---

### Task 8: Clean up EventBusAddresses and remove EventBus injection

**Files:**
- Modify: `common/.../event/EventBusAddresses.java` — remove migrated constants; keep only `layerChanged()` and `HUMAN_TASK_SCHEDULE` (if still used externally)
- Modify: `blackboard/.../event/BlackboardEventBusAddresses.java` — remove migrated constants
- Modify: all handler files — remove `@Inject EventBus eventBus` where no longer needed
- Modify: `runtime/src/test/resources/application.properties` — remove any event-bus-specific config
- Test: verify full test suite passes

- [ ] **Step 1: Audit remaining eventBus.publish() calls**

Search for any remaining `eventBus.publish()` in production code. These should only be `layerChanged()` dynamic addresses or external events.

- [ ] **Step 2: Remove unused constants from EventBusAddresses**

Keep: `layerChanged()` method (dynamic addresses still on event bus), `HUMAN_TASK_SCHEDULE` (consumed by work-adapter, external module).

Remove: all other constants (now dispatched via CDI).

- [ ] **Step 3: Remove unused EventBus injection**

For each handler that no longer calls `eventBus.publish()`, remove `@Inject EventBus eventBus`.

- [ ] **Step 4: Full test run**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime,blackboard,resilience,scheduler-quartz
```

Then run AML:
```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -f <AML>/pom.xml -pl app
```

- [ ] **Step 5: Commit**

```
chore(#798): remove migrated EventBusAddresses constants and unused EventBus injection

Closes #798
```
