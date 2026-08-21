# db-scheduler Alternative Scheduler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #813 — Investigate and implement alternative worker scheduler behind SPI
**Issue group:** #813

**Goal:** Extract scheduler-agnostic domain logic into reusable orchestrators, then add db-scheduler as the default scheduler implementation with H2 in-memory storage.

**Architecture:** Two-phase approach. Phase 1 extracts domain logic from 6 Quartz classes into 4 orchestrators in `common/`, then slims Quartz classes to thin adapters. Phase 2 adds `scheduler-dbscheduler` module implementing both `JobScheduler` and `WorkerExecutionManager` SPIs, defaulting to H2 in-memory with opt-in PostgreSQL durability.

**Tech Stack:** Java 21, Quarkus 3.32.2, db-scheduler (com.github.kagkarlsson:db-scheduler), H2 (in-memory default)

**Reuse source:** `backup/issue-813-original` branch — ~40% reusable with minor adaptation. Use `git show backup/issue-813-original:<path>` to reference original code.

## Global Constraints

- All existing Quartz tests must continue to pass after Phase 1
- No Mutiny/Uni — all scheduler methods return `void` (virtual thread model)
- `ScheduledJobRequest.jobClass` (the Quartz leak) must be replaced with `JobType` enum
- db-scheduler's built-in retry is NOT used — engine's `RetryOrchestrator` handles retry
- Default storage is H2 in-memory (RAM-equivalent to Quartz's current `store-type=ram`)
- Scoped Flyway migration for PostgreSQL opt-in follows `db/engine-ledger/migration/` pattern

---

## Batch 1: SPI Foundation Types

### Task 1: JobType enum + ScheduledJobRequest migration

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/scheduler/JobType.java`
- Modify: `common/src/main/java/io/casehub/engine/common/internal/scheduler/ScheduledJobRequest.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzJobScheduler.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/scheduler/SchedulerService.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/engine/handler/MilestoneActivatedEventHandler.java`
- Test: `common/src/test/java/io/casehub/engine/common/internal/scheduler/ScheduledJobRequestTest.java`

**Interfaces:**
- Produces: `JobType` enum with values `WORKER_EXECUTION`, `SCHEDULED_TRIGGER_UNCONDITIONAL`, `SCHEDULED_TRIGGER_CONDITIONAL`, `MILESTONE_SLA_TIMEOUT`, `SIGNAL_TRIGGER`
- Produces: `ScheduledJobRequest.getJobType()` replacing `getJobClass()`

- [ ] **Step 1: Write JobType enum test**

```java
@Test
void allJobTypesAreDefined() {
    assertEquals(5, JobType.values().length);
    assertNotNull(JobType.WORKER_EXECUTION);
    assertNotNull(JobType.SCHEDULED_TRIGGER_UNCONDITIONAL);
    assertNotNull(JobType.SCHEDULED_TRIGGER_CONDITIONAL);
    assertNotNull(JobType.MILESTONE_SLA_TIMEOUT);
    assertNotNull(JobType.SIGNAL_TRIGGER);
}
```

- [ ] **Step 2: Create JobType enum**

```java
package io.casehub.engine.common.internal.scheduler;

public enum JobType {
    WORKER_EXECUTION,
    SCHEDULED_TRIGGER_UNCONDITIONAL,
    SCHEDULED_TRIGGER_CONDITIONAL,
    MILESTONE_SLA_TIMEOUT,
    SIGNAL_TRIGGER
}
```

- [ ] **Step 3: Write ScheduledJobRequest test for jobType**

```java
@Test
void builderAcceptsJobType() {
    var request = ScheduledJobRequest.builder()
        .jobId(JobIdentifier.of("test", "group"))
        .schedule(new ScheduleStrategy.DelaySchedule(1000))
        .jobType(JobType.WORKER_EXECUTION)
        .data(Map.of())
        .build();
    assertEquals(JobType.WORKER_EXECUTION, request.getJobType());
    assertNull(request.getJobClass()); // deprecated
}
```

- [ ] **Step 4: Add `jobType` field to ScheduledJobRequest**

Add `JobType jobType` field alongside `jobClass` (keep `jobClass` temporarily for backward compat during migration). Add `jobType(JobType)` to Builder. `getJobClass()` stays but is deprecated.

- [ ] **Step 5: Update QuartzJobScheduler.resolveJobClass() to use JobType**

Change `resolveJobClass()` to check `request.getJobType()` first (switch expression), falling back to the existing `triggerType` string lookup from data map only when `jobType` is null. This preserves backward compatibility during migration.

- [ ] **Step 6: Update SchedulerService call sites to pass JobType**

Replace the `triggerType` string in data map with explicit `JobType`:
- `scheduleWorker()` → `.jobType(JobType.SCHEDULED_TRIGGER_UNCONDITIONAL)`
- `scheduleConditionalWorker()` → `.jobType(JobType.SCHEDULED_TRIGGER_CONDITIONAL)`
- `scheduleSignal()` → `.jobType(JobType.SIGNAL_TRIGGER)`

- [ ] **Step 7: Update MilestoneActivatedEventHandler to pass JobType**

Add `.jobType(JobType.MILESTONE_SLA_TIMEOUT)` to the `ScheduledJobRequest.builder()` call.

- [ ] **Step 8: Run full test suite, verify green**

Run: `mvn clean test -pl common,scheduler-quartz,runtime -q`

- [ ] **Step 9: Commit**

```
feat(#813): add JobType enum, replace jobClass on ScheduledJobRequest

Refs #813
```

### Task 2: WorkerTaskData record + functional interfaces

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/WorkerTaskData.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/RescheduleCallback.java`
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/RetryHandler.java`
- Test: `common/src/test/java/io/casehub/engine/common/internal/executor/WorkerTaskDataTest.java`

**Interfaces:**
- Produces: `WorkerTaskData` record — scheduler-agnostic job data carrier
- Produces: `RescheduleCallback` — `void reschedule(WorkerTaskData, Duration)`
- Produces: `RetryHandler` — `void handleFailure(WorkerTaskData, Throwable, String)`

**Reuse:** Adapt from `backup/issue-813-original:common/src/main/java/io/casehub/engine/common/internal/executor/WorkerTaskData.java`. Fix: `signalId` String→UUID, `eventLogId` stays Long (correct on branch).

- [ ] **Step 1: Write WorkerTaskData test**

```java
@Test
void roundTripsViaMap() {
    var original = new WorkerTaskData(1L, "hash123",
        UUID.randomUUID(), "worker1", "tenant1", "binding1",
        UUID.randomUUID());
    Map<String, String> map = original.toMap();
    WorkerTaskData restored = WorkerTaskData.fromMap(map);
    assertEquals(original, restored);
}

@Test
void withMethodsReturnNewInstance() {
    var original = new WorkerTaskData(1L, "hash", UUID.randomUUID(),
        "w", "t", null, null);
    var withBinding = original.withBindingName("b1");
    assertNull(original.bindingName());
    assertEquals("b1", withBinding.bindingName());
}
```

- [ ] **Step 2: Implement WorkerTaskData**

```java
package io.casehub.engine.common.internal.executor;

import java.time.Duration;
import java.util.Map;
import java.util.UUID;

public record WorkerTaskData(
    Long eventLogId,
    String inputDataHash,
    UUID caseId,
    String workerId,
    String tenancyId,
    String bindingName,
    UUID signalId
) {
    public WorkerTaskData withBindingName(String bindingName) {
        return new WorkerTaskData(eventLogId, inputDataHash, caseId, workerId, tenancyId, bindingName, signalId);
    }

    public WorkerTaskData withSignalId(UUID signalId) {
        return new WorkerTaskData(eventLogId, inputDataHash, caseId, workerId, tenancyId, bindingName, signalId);
    }

    public Map<String, String> toMap() {
        var map = new java.util.HashMap<String, String>();
        map.put("eventLogId", String.valueOf(eventLogId));
        map.put("inputDataHash", inputDataHash);
        map.put("caseId", caseId.toString());
        map.put("workerId", workerId);
        map.put("tenancyId", tenancyId);
        if (bindingName != null) map.put("bindingName", bindingName);
        if (signalId != null) map.put("signalId", signalId.toString());
        return Map.copyOf(map);
    }

    public static WorkerTaskData fromMap(Map<String, String> map) {
        return new WorkerTaskData(
            Long.parseLong(map.get("eventLogId")),
            map.get("inputDataHash"),
            UUID.fromString(map.get("caseId")),
            map.get("workerId"),
            map.get("tenancyId"),
            map.get("bindingName"),
            map.containsKey("signalId") ? UUID.fromString(map.get("signalId")) : null
        );
    }
}
```

- [ ] **Step 3: Create RescheduleCallback and RetryHandler**

```java
package io.casehub.engine.common.internal.executor;

import java.time.Duration;

@FunctionalInterface
public interface RescheduleCallback {
    void reschedule(WorkerTaskData taskData, Duration delay);
}
```

```java
package io.casehub.engine.common.internal.executor;

@FunctionalInterface
public interface RetryHandler {
    void handleFailure(WorkerTaskData taskData, Throwable cause, String errorMessage);
}
```

- [ ] **Step 4: Run tests, verify green**

Run: `mvn clean test -pl common -q`

- [ ] **Step 5: Commit**

```
feat(#813): add WorkerTaskData, RescheduleCallback, RetryHandler

Refs #813
```

---

## Batch 2: Worker Execution Orchestrator Extraction

### Task 3: WorkerExecutionOrchestrator

Extract domain logic from `QuartzWorkerExecutionJob` (~346 lines) and `QuartzWorkerExecutionJobListener` (~147 lines) into a single orchestrator.

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java` — slim to ~15 lines
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJobListener.java` — remove domain logic (lifecycle hooks move to orchestrator)
- Modify: `common/pom.xml` — add dependencies needed by orchestrator (if any missing)
- Test: `common/src/test/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestratorTest.java`

**Interfaces:**
- Consumes: `WorkerTaskData`, `RetryHandler`
- Produces: `WorkerExecutionOrchestrator.execute(WorkerTaskData, RetryHandler)`

**Reuse:** Start from `backup/issue-813-original:common/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java` (244 lines). Add pre-execution lifecycle hooks from `QuartzWorkerExecutionJobListener`.

- [ ] **Step 1: Read the current QuartzWorkerExecutionJob and QuartzWorkerExecutionJobListener**

Use `git show origin/main:scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionJob.java` and the JobListener. Identify all domain logic vs Quartz-specific glue.

- [ ] **Step 2: Read the backup branch orchestrator**

Use `git show backup/issue-813-original:common/src/main/java/io/casehub/engine/common/internal/executor/WorkerExecutionOrchestrator.java`. Identify what's missing (lifecycle hooks, current API changes).

- [ ] **Step 3: Write orchestrator unit test**

Test the core flow: given a valid eventLogId, the orchestrator resolves EventLog, CaseInstance, CaseDefinition, Worker, Capability, and delegates to WorkerExecutor. Mock all dependencies. Verify pre-execution lifecycle event is fired, success publishes WorkflowExecutionCompleted, failure delegates to RetryHandler.

- [ ] **Step 4: Implement WorkerExecutionOrchestrator**

`@ApplicationScoped` bean in `common/internal/executor/`. Inject: `CrossTenantEventLogRepository`, `CaseInstanceCache`, `WorkerExecutionRecoveryService`, `CaseDefinitionRegistry`, `BridgeResolver`, `WorkerExecutor`, `EventBus`, `Event<CaseLifecycleEvent>`, `WorkerStatusListener`, `LedgerTraceIdProvider`. Method: `void execute(WorkerTaskData taskData, RetryHandler retryHandler)`.

Pre-execution phase (from JobListener):
1. Persist `WORKER_EXECUTION_STARTED` EventLog
2. Fire `CaseLifecycleEvent` (fire-and-forget `.invoke()`)
3. Call `workerStatusListener.onWorkerStarted()`

Execution phase (from QuartzWorkerExecutionJob):
1. Resolve EventLog → CaseInstance → CaseDefinition → Worker → Capability
2. Resolve ContextBridge, deserialise typed input
3. Build WorkerContext
4. Delegate to `WorkerExecutor.execute()`
5. On success: publish `WORKER_EXECUTION_FINISHED`
6. On failure: call `retryHandler.handleFailure()`

- [ ] **Step 5: Run test, verify pass**

- [ ] **Step 6: Slim QuartzWorkerExecutionJob to delegate**

Replace the body of `execute(JobExecutionContext)` with:
```java
WorkerTaskData taskData = WorkerTaskData.fromMap(
    context.getJobDetail().getJobDataMap().getWrappedMap());
orchestrator.execute(taskData, retryHandler::handleFailure);
```

- [ ] **Step 7: Remove domain logic from QuartzWorkerExecutionJobListener**

Lifecycle hooks are now in the orchestrator. The listener becomes a thin Quartz adapter or is removed entirely (its `@Observes StartupEvent` registration moves to `QuartzWorkerExecutionManager`).

- [ ] **Step 8: Run full Quartz test suite**

Run: `mvn clean test -pl common,scheduler-quartz,runtime -q`
All existing tests must pass unchanged.

- [ ] **Step 9: Commit**

```
refactor(#813): extract WorkerExecutionOrchestrator from Quartz classes

Refs #813
```

### Task 4: RetryOrchestrator

Extract retry logic from `QuartzRetryService` (~289 lines).

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzRetryService.java` — slim to thin adapter
- Test: `common/src/test/java/io/casehub/engine/common/internal/executor/RetryOrchestratorTest.java`

**Interfaces:**
- Consumes: `WorkerTaskData`, `RescheduleCallback`
- Produces: `RetryOrchestrator.handleFailure(WorkerTaskData, Throwable, String, RescheduleCallback)`

**Reuse:** Start from `backup/issue-813-original:common/src/main/java/io/casehub/engine/common/internal/executor/RetryOrchestrator.java` (217 lines). **Critical fix:** add `RecoveryCoordinator.handleFailure()` gate before publishing `WORKER_RETRIES_EXHAUSTED`.

- [ ] **Step 1: Write test — retry decision delegates to RescheduleCallback**

Mock `RetryPolicies.evaluate()` → `Retry(Duration.ofSeconds(5))`. Verify `RescheduleCallback.reschedule()` is called with the delay.

- [ ] **Step 2: Write test — exhaustion with RecoveryCoordinator handling**

Mock `RecoveryCoordinator.handleFailure()` → returns `true`. Verify `WORKER_RETRIES_EXHAUSTED` is NOT published.

- [ ] **Step 3: Write test — exhaustion without RecoveryCoordinator handling**

Mock `RecoveryCoordinator.handleFailure()` → returns `false`. Verify `WORKER_RETRIES_EXHAUSTED` IS published.

- [ ] **Step 4: Implement RetryOrchestrator**

`@ApplicationScoped` bean. Inject: `CrossTenantEventLogRepository`, `CaseDefinitionRegistry`, `EventBus`, `RecoveryCoordinator`. Method: `void handleFailure(WorkerTaskData, Throwable, String, RescheduleCallback)`.

Flow:
1. Persist `WORKER_EXECUTION_FAILED` EventLog
2. Resolve Worker → ExecutionPolicy → RetryPolicy
3. Count prior failures from EventLog
4. `RetryPolicies.evaluate()` → `RetryDecision`
5. `Retry(delay)` → call `rescheduleCallback.reschedule(taskData, delay)`
6. `Exhaust(reason)` → call `recoveryCoordinator.handleFailure()` → if NOT handled, publish `WORKER_RETRIES_EXHAUSTED`

- [ ] **Step 5: Slim QuartzRetryService**

Replace with thin adapter:
```java
public void handleFailure(WorkerTaskData taskData, Throwable cause, String message) {
    retryOrchestrator.handleFailure(taskData, cause, message,
        (data, delay) -> workerExecutionScheduler.rescheduleJob(data, delay));
}
```

- [ ] **Step 6: Run full test suite**

- [ ] **Step 7: Commit**

```
refactor(#813): extract RetryOrchestrator with RecoveryCoordinator gate

Refs #813
```

---

## Batch 3: Scheduled Job Orchestrator Extraction

### Task 5: ScheduledTriggerOrchestrator

Extract domain logic from `ScheduledTriggerJob` (~211 lines), `ConditionalScheduledTriggerJob` (~241 lines), and `ScheduledSignalJob` (~118 lines).

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/ScheduledTriggerOrchestrator.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ScheduledTriggerJob.java` — slim to delegate
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ConditionalScheduledTriggerJob.java` — slim to delegate
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/ScheduledSignalJob.java` — slim to delegate
- Test: `common/src/test/java/io/casehub/engine/common/internal/executor/ScheduledTriggerOrchestratorTest.java`

**Interfaces:**
- Produces: `executeUnconditional(UUID caseId, String bindingName)`
- Produces: `executeConditional(UUID caseId, String bindingName)`
- Produces: `executeSignal(UUID caseId, String bindingName, String signalPayload)`

**Reuse:** From scratch — these orchestrators don't exist on the backup branch.

- [ ] **Step 1: Read the three Quartz job classes**

Identify shared domain logic: case loading, status check, CaseDefinition lookup, binding resolution, ScopedWorkerRegistry interaction.

- [ ] **Step 2: Write tests for unconditional trigger**

Test: given valid caseId + bindingName, orchestrator loads CaseInstance, resolves binding, publishes WorkerScheduleEvent. Test: skip if case is terminal.

- [ ] **Step 3: Write tests for conditional trigger**

Test: condition evaluates true → publishes WorkerScheduleEvent. Test: condition evaluates false → no event published.

- [ ] **Step 4: Write tests for signal trigger**

Test: publishes ContextSignalEvent with parsed payload.

- [ ] **Step 5: Write test for ScopedWorkerRegistry interaction**

Test: when persistent scoped session exists for binding, push ContextEvent to mailbox instead of dispatching new worker.

- [ ] **Step 6: Implement ScheduledTriggerOrchestrator**

`@ApplicationScoped` bean. Inject: `CrossTenantCaseInstanceRepository`, `CaseInstanceCache`, `CaseDefinitionRegistry`, `ScopedWorkerRegistry`, `ExpressionEngineRegistry`, `EventBus`. Three public methods for each trigger type.

- [ ] **Step 7: Slim three Quartz job classes**

Each becomes ~10 lines extracting job data and delegating to the orchestrator.

- [ ] **Step 8: Run full test suite**

- [ ] **Step 9: Commit**

```
refactor(#813): extract ScheduledTriggerOrchestrator from 3 Quartz jobs

Refs #813
```

### Task 6: MilestoneSLAOrchestrator

Extract domain logic from `MilestoneSLATimeoutJob` (~131 lines).

**Files:**
- Create: `common/src/main/java/io/casehub/engine/common/internal/executor/MilestoneSLAOrchestrator.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/MilestoneSLATimeoutJob.java` — slim to delegate
- Test: `common/src/test/java/io/casehub/engine/common/internal/executor/MilestoneSLAOrchestratorTest.java`

**Interfaces:**
- Produces: `executeSLATimeout(UUID caseId, String milestoneName)`

**Reuse:** From scratch.

- [ ] **Step 1: Read MilestoneSLATimeoutJob**

- [ ] **Step 2: Write test — SLA violation fires event when milestone is ACTIVE**

- [ ] **Step 3: Write test — skip when milestone is already COMPLETED**

- [ ] **Step 4: Implement MilestoneSLAOrchestrator**

- [ ] **Step 5: Slim MilestoneSLATimeoutJob to delegate**

- [ ] **Step 6: Run full test suite**

- [ ] **Step 7: Commit**

```
refactor(#813): extract MilestoneSLAOrchestrator from Quartz job

Refs #813
```

---

## Batch 4: JobScheduler Contract Test

### Task 7: JobSchedulerContractTest

Abstract contract test that both Quartz and db-scheduler implementations will extend.

**Files:**
- Create: `common/src/test/java/io/casehub/engine/common/spi/scheduler/JobSchedulerContractTest.java`
- Modify: existing Quartz `QuartzJobScheduler` tests to extend the contract test

**Interfaces:**
- Produces: abstract `JobSchedulerContractTest` with `schedule`, `cancel`, `cancelGroup`, `exists` test methods

**Reuse:** Adapt from `backup/issue-813-original:common/src/test/java/io/casehub/engine/common/spi/scheduler/JobSchedulerContractTest.java`. **Fix:** remove all `Uni`/`.await().indefinitely()` — methods return `void`.

- [ ] **Step 1: Write abstract contract test**

Abstract method: `protected abstract JobScheduler createScheduler()`. Test methods: `scheduleAndExists()`, `cancelRemovesJob()`, `cancelGroupRemovesAllInGroup()`, `cancelNonExistentReturnsFalse()`.

- [ ] **Step 2: Create Quartz subclass extending contract test**

`QuartzJobSchedulerContractTest extends JobSchedulerContractTest` — creates `QuartzJobScheduler` with embedded Quartz Scheduler.

- [ ] **Step 3: Run test, verify pass**

- [ ] **Step 4: Commit**

```
test(#813): add JobSchedulerContractTest in common

Refs #813
```

---

## Batch 5: db-scheduler Module Scaffold + Core Implementation

### Task 8: Module scaffold + lifecycle + core implementations

**Files:**
- Create: `scheduler-dbscheduler/pom.xml`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/DbSchedulerLifecycle.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/ScheduledJobData.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/DbSchedulerJobScheduler.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/DbSchedulerWorkerExecutionManager.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/DbSchedulerRetryService.java`
- Create: `scheduler-dbscheduler/src/main/resources/application.properties`
- Create: `scheduler-dbscheduler/src/test/resources/application.properties`
- Create: `db/engine-scheduler/migration/V1__Create_DbScheduler_Table.sql`
- Modify: `pom.xml` (root) — add `scheduler-dbscheduler` module
- Test: `scheduler-dbscheduler/src/test/java/.../DbSchedulerJobSchedulerContractTest.java` (extends contract test)
- Test: `scheduler-dbscheduler/src/test/java/.../DbSchedulerWorkerExecutionManagerTest.java`

**Interfaces:**
- Consumes: `JobScheduler`, `WorkerExecutionManager`, `WorkerExecutionOrchestrator`, `RetryOrchestrator`
- Produces: `DbSchedulerJobScheduler implements JobScheduler`, `DbSchedulerWorkerExecutionManager implements WorkerExecutionManager` (`@WorkerBackend @Priority(10)`)

**Reuse:** Adapt `pom.xml`, `DbSchedulerLifecycle`, `DbSchedulerWorkerExecutionManager`, `ScheduledJobData` from backup branch (minor adaptation). Major rewrite: `DbSchedulerJobScheduler` (strip Uni), `DbSchedulerRetryService` (wire RecoveryCoordinator). Flyway migration reusable as-is.

- [ ] **Step 1: Create pom.xml**

Dependencies: `casehub-engine-common`, `com.github.kagkarlsson:db-scheduler`, `io.quarkus:quarkus-jdbc-h2`, `io.quarkus:quarkus-agroal`. Test: `io.quarkus:quarkus-junit5`, `io.quarkus:quarkus-jdbc-h2`.

Reference `backup/issue-813-original:scheduler-dbscheduler/pom.xml` for structure.

- [ ] **Step 2: Add module to root pom.xml**

Add `<module>scheduler-dbscheduler</module>` after `scheduler-quartz`.

- [ ] **Step 3: Create ScheduledJobData**

Serialisation helper for db-scheduler's `task_data` column. Wraps `WorkerTaskData` fields + `JobType`.

- [ ] **Step 4: Create DbSchedulerLifecycle**

`@ApplicationScoped` startup bean. Creates db-scheduler `Scheduler` with H2 in-memory datasource (default). Registers task definitions. Starts on `@Observes StartupEvent`, stops on `@Observes ShutdownEvent`.

Default config:
```properties
casehub.scheduler.dbscheduler.threads=4
casehub.scheduler.dbscheduler.polling-interval=500ms
casehub.scheduler.dbscheduler.heartbeat-interval=5m
```

`createIfNotExists(true)` for H2 auto-table creation.

- [ ] **Step 5: Create DbSchedulerJobScheduler**

`@ApplicationScoped` implementing `JobScheduler`. Maps `JobType` → db-scheduler task names. Instance ID convention: `"{group}:{name}"`. All methods return `void` (no Uni).

- [ ] **Step 6: Create contract test subclass**

`DbSchedulerJobSchedulerContractTest extends JobSchedulerContractTest` — uses H2 in-memory.

- [ ] **Step 7: Run contract test**

- [ ] **Step 8: Create DbSchedulerWorkerExecutionManager**

`@WorkerBackend @Priority(10) @ApplicationScoped`. Delegates submit/recovery to `DbSchedulerJobScheduler`. `getActiveWorkCount`/`getActiveCaseIds` query the `scheduled_tasks` table via `schedulerClient.fetchScheduledExecutions()`.

- [ ] **Step 9: Create DbSchedulerRetryService**

Wraps `RetryOrchestrator` with db-scheduler `RescheduleCallback`:
```java
(data, delay) -> jobScheduler.schedule(
    ScheduledJobRequest.builder()
        .jobId(JobIdentifier.of(data.workerId(), data.caseId().toString()))
        .schedule(new ScheduleStrategy.DelaySchedule(delay.toMillis()))
        .jobType(JobType.WORKER_EXECUTION)
        .data(data.toMap()))
```

- [ ] **Step 10: Create Flyway migration**

`db/engine-scheduler/migration/V1__Create_DbScheduler_Table.sql` — standard db-scheduler schema with PostgreSQL types. Only used when consumer opts into PostgreSQL mode.

- [ ] **Step 11: Write integration test**

`DbSchedulerWorkerExecutionManagerTest` — submit a task, verify it executes via the orchestrator. Uses H2 in-memory.

- [ ] **Step 12: Run full test suite across all modules**

Run: `mvn clean test -q`

- [ ] **Step 13: Commit**

```
feat(#813): add scheduler-dbscheduler module with H2 default

Refs #813
```

---

## Batch 6: Task Handlers + End-to-End Verification

### Task 9: db-scheduler task handlers

Create the 5 `ExecutionHandler` implementations that delegate to orchestrators.

**Files:**
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/WorkerExecutionTaskHandler.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/ScheduledTriggerTaskHandler.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/ConditionalScheduledTriggerTaskHandler.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/MilestoneSLATimeoutTaskHandler.java`
- Create: `scheduler-dbscheduler/src/main/java/io/casehub/engine/scheduler/dbscheduler/ScheduledSignalTaskHandler.java`
- Modify: `scheduler-dbscheduler/src/main/java/.../DbSchedulerLifecycle.java` — register all task definitions

**Interfaces:**
- Consumes: `WorkerExecutionOrchestrator`, `RetryOrchestrator` (via `DbSchedulerRetryService`), `ScheduledTriggerOrchestrator`, `MilestoneSLAOrchestrator`

**Reuse:** Adapt from backup branch — each handler needs major rewrite to delegate to orchestrators instead of containing domain logic.

- [ ] **Step 1: Create WorkerExecutionTaskHandler**

```java
public class WorkerExecutionTaskHandler implements ExecutionHandler<ScheduledJobData> {
    private final WorkerExecutionOrchestrator orchestrator;
    private final DbSchedulerRetryService retryService;

    @Override
    public void execute(TaskInstance<ScheduledJobData> taskInstance, ExecutionContext context) {
        WorkerTaskData taskData = taskInstance.getData().toWorkerTaskData();
        orchestrator.execute(taskData, retryService::handleFailure);
    }
}
```

- [ ] **Step 2: Create remaining 4 task handlers**

Each is a thin shim: extract data from `TaskInstance`, delegate to the appropriate orchestrator method.

- [ ] **Step 3: Register all task definitions in DbSchedulerLifecycle**

Add task definitions for each task name → handler mapping.

- [ ] **Step 4: Run full test suite**

Run: `mvn clean test -q`

- [ ] **Step 5: Commit**

```
feat(#813): add db-scheduler task handlers for all 5 job types

Refs #813
```

### Task 10: End-to-end verification + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` — add db-scheduler module documentation
- Modify: `docs/contributor-guide.md` — document scheduler backend selection

- [ ] **Step 1: Verify db-scheduler is selected by default**

Write a test that creates a `CompositeWorkerExecutionManager` with both Quartz (`@Priority(0)`) and db-scheduler (`@Priority(10)`) on the classpath. Verify db-scheduler is selected first.

- [ ] **Step 2: Verify Quartz-only mode**

Run existing Quartz tests without db-scheduler on classpath — all pass unchanged.

- [ ] **Step 3: Update CLAUDE.md**

Add `scheduler-dbscheduler` module documentation alongside existing `scheduler-quartz` section.

- [ ] **Step 4: Update contributor guide**

Document scheduler backend selection: db-scheduler default (H2), Quartz opt-in, PostgreSQL durable opt-in.

- [ ] **Step 5: Final full test suite run**

Run: `mvn clean test -q`

- [ ] **Step 6: Commit**

```
feat(#813): db-scheduler as default scheduler — end-to-end verified

Closes #813
```

---

## References

- [2026-08-21-db-scheduler-alternative-design.md] — design spec this plan implements
- `common/spi/scheduler/JobScheduler.java` — current SPI interface
- `common/spi/scheduler/WorkerExecutionManager.java` — current SPI interface
- `scheduler-quartz/` — current Quartz implementation (6 job classes, 1 job listener)
- `runtime/internal/engine/handler/WorkerScheduleEventHandler.java` — worker dispatch entry point
- `runtime/internal/engine/recovery/WorkerRecoveryCoordinator.java` — recovery gate
- `common/internal/worker/scope/ScopedWorkerRegistry.java` — scoped worker interaction
- `backup/issue-813-original` — preserved original branch for code reuse
- Design review: `/Users/mdproctor/reviews/casehub-slots/issue-813-db-scheduler-20260821-065603/`
- GitHub #813 — focal issue
