# db-scheduler as Default Worker Scheduler

**Issue:** casehubio/engine#813 (closed prematurely — work never merged; reopened for this design)
**Date:** 2026-08-21
**Status:** Design

## Context

The engine's worker scheduling is implemented solely by Quartz. The scheduler has two SPI levels:

- **`JobScheduler`** — timed/cron job scheduling (milestones, scheduled triggers, signal triggers)
- **`WorkerExecutionManager`** — worker execution dispatch (the hot path for capability-triggered workers)

Both are Quartz-only today. Quartz is configured with `quarkus.quartz.store-type=ram` — no JDBC persistence, no clustering. All scheduled tasks are ephemeral and recovered from EventLog on restart via `WorkerExecutionRecoveryService`.

The Quartz implementation has domain logic trapped in scheduler-specific classes:

| Class | Lines | Domain Logic |
|-------|-------|-------------|
| `QuartzWorkerExecutionJob` | ~346 | EventLog resolution, CaseInstance recovery, definition lookup, ContextBridge, WorkerExecutor delegation, scoped output handling |
| `QuartzRetryService` | ~289 | Failure persistence, retry policy resolution, backoff, RecoveryCoordinator gate |
| `ScheduledTriggerJob` | ~211 | Case loading, status check, CaseDefinition lookup, binding resolution, ScopedWorkerRegistry interaction, WorkerScheduleEvent construction |
| `ConditionalScheduledTriggerJob` | ~241 | All of ScheduledTriggerJob + expression evaluation via ExpressionEngineRegistry |
| `MilestoneSLATimeoutJob` | ~131 | Case loading, milestone lifecycle status from EventLog, SLA violation event |
| `ScheduledSignalJob` | ~118 | Case loading, conditional evaluation, signal payload parsing, ContextSignalEvent publishing |

Additionally, `ScheduledJobRequest.jobClass` is `Class<?>` — a Quartz-specific type leaked into the scheduler-agnostic SPI. Call sites (`SchedulerService`, `MilestoneActivatedEventHandler`) do not set `jobClass` directly — it is inferred internally by `QuartzJobScheduler.resolveJobClass()` from a `triggerType` string in the data map.

`QuartzWorkerExecutionJobListener` (147 lines) fires pre-execution lifecycle hooks: `WORKER_EXECUTION_STARTED` EventLog, `CaseLifecycleEvent` (consumed by ledger integration), and `workerStatusListener.onWorkerStarted()` (monitoring). These hooks must be preserved in any alternative implementation.

An earlier attempt (issue-813 branch, July 2026) extracted orchestrators and built a db-scheduler module but was never merged. Main has since evolved 203 commits. This design rebuilds against the current API surface.

### Why db-scheduler

Quartz with RAM store provides immediate dispatch but no persistence, no JDBC-backed durability, and carries a heavy dependency (XML config, complex API surface). db-scheduler provides:

- **Dual-mode storage** — H2 in-memory for RAM-equivalent ephemeral scheduling (default), PostgreSQL for durable scheduling (opt-in). Same API for both.
- **Simpler API** — single table, no XML, minimal configuration
- **Lighter dependency** — ~15 classes vs Quartz's hundreds
- **Same recovery model** — with H2 in-memory, tasks are ephemeral and recovered from EventLog on restart (identical to Quartz RAM store today). With PostgreSQL, db-scheduler persistence provides an additional recovery path.

A virtual thread `ScheduledExecutorService` was considered (zero dependency, zero latency) but lacks the storage abstraction needed for the H2/PostgreSQL dual-mode, has no built-in task persistence query (`getActiveWorkCount`, `getActiveCaseIds`), and would require building cancellation, group operations, and heartbeat monitoring from scratch.

## Goals

1. Extract scheduler-agnostic domain logic from all 6 Quartz job/service classes into reusable orchestrators
2. Fix the `jobClass` Quartz leak in the SPI
3. Add db-scheduler as a second `WorkerExecutionManager` and `JobScheduler` implementation
4. Make db-scheduler the default scheduler (higher `@Priority`)
5. Keep Quartz as an opt-in alternative

## Non-Goals

- Removing Quartz — it remains available for consumers who need its richer scheduling features (calendars, clustering, misfire policies)
- Changing the `CompositeWorkerExecutionManager` routing — the existing `@WorkerBackend` + `@Priority` mechanism works correctly
- Adding persistent scheduling as a feature — the default (H2) preserves current ephemeral semantics; PostgreSQL durability is available but not pushed

## Design

### Phase 1 — SPI Cleanup and Orchestrator Extraction

#### 1.1 WorkerExecutionOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts all domain logic from `QuartzWorkerExecutionJob`:

- Resolve EventLog by ID
- Recover CaseInstance from cache or repository
- Look up CaseDefinition, Worker, Capability from registries
- Resolve ContextBridge and deserialise typed input
- Build WorkerContext (channels, experiences, propagation context)
- **Pre-execution lifecycle hooks** (from `QuartzWorkerExecutionJobListener`): persist `WORKER_EXECUTION_STARTED` EventLog, fire `CaseLifecycleEvent`, call `workerStatusListener.onWorkerStarted()`
- Delegate to `WorkerExecutor.execute()`
- On success: publish `WORKER_EXECUTION_FINISHED` (WorkflowExecutionCompleted)
- On failure: delegate to retry handling via `RetryHandler`

After extraction, `QuartzWorkerExecutionJob.execute()` becomes a thin shim:
```java
@Override
public void execute(JobExecutionContext context) {
    WorkerTaskData taskData = WorkerTaskData.fromJobDataMap(context.getJobDetail().getJobDataMap());
    orchestrator.execute(taskData, retryHandler::handleFailure);
}
```

`QuartzWorkerExecutionJobListener` is retired — its logic moves into the orchestrator's pre-execution phase.

#### 1.2 RetryOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts all retry logic from `QuartzRetryService`:

- Persist `WORKER_EXECUTION_FAILED` EventLog
- Resolve retry policy from Worker's `ExecutionPolicy`
- Count prior failures from EventLog
- Call `RetryPolicies.evaluate()` → `RetryDecision`
- On `Retry(delay)`: invoke `RescheduleCallback` (scheduler-specific)
- On `Exhaust(reason)`: **call `RecoveryCoordinator.handleFailure()` first** — if the coordinator handles it (returns `true`), do NOT publish `WORKER_RETRIES_EXHAUSTED`. Only publish the exhausted event when the coordinator declines.

`RescheduleCallback` is a functional interface:
```java
@FunctionalInterface
public interface RescheduleCallback {
    void reschedule(WorkerTaskData taskData, Duration delay);
}
```

After extraction, `QuartzRetryService` becomes a thin adapter providing the Quartz-specific `RescheduleCallback`.

#### 1.3 ScheduledTriggerOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts shared domain logic from `ScheduledTriggerJob`, `ConditionalScheduledTriggerJob`, and `ScheduledSignalJob`:

- Load CaseInstance, check case status (skip if terminal)
- Look up CaseDefinition, resolve binding by name
- **ScopedWorkerRegistry interaction**: check for existing persistent or reinvoked scoped worker sessions. For persistent sessions, push `ContextEvent` to mailbox instead of dispatching a new worker. For reinvoked sessions, use the session's `executorName` for re-dispatch.
- Construct `WorkerScheduleEvent` / `ContextSignalEvent`
- For conditional triggers: evaluate condition via `ExpressionEngineRegistry`
- For signal triggers: parse signal payload, publish `ContextSignalEvent`

#### 1.4 MilestoneSLAOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts domain logic from `MilestoneSLATimeoutJob`:

- Load CaseInstance, resolve milestone lifecycle status from EventLog
- Check if milestone is still ACTIVE (skip if already completed/violated)
- Publish `MilestoneSLAViolatedEvent`

#### 1.5 WorkerTaskData

Record in `common/internal/executor/`:
```java
public record WorkerTaskData(
    Long eventLogId,
    String inputDataHash,
    UUID caseId,
    String workerId,
    String tenancyId,
    String bindingName,
    UUID signalId
) {
    public WorkerTaskData withBindingName(String bindingName) { ... }
    public WorkerTaskData withSignalId(UUID signalId) { ... }

    public Map<String, String> toMap() { ... }
    public static WorkerTaskData fromMap(Map<String, String> map) { ... }
}
```

`signalId` is `UUID` (matching downstream types `WorkflowExecutionCompleted`, `WorkerRetriesExhaustedEvent`). `eventLogId` is `Long` (the persisted type — `WorkerRetryContext`'s `String` was a Quartz serialization artifact).

#### 1.6 JobType Enum

Replace `jobClass: Class<?>` on `ScheduledJobRequest` with:
```java
public enum JobType {
    WORKER_EXECUTION,
    SCHEDULED_TRIGGER_UNCONDITIONAL,
    SCHEDULED_TRIGGER_CONDITIONAL,
    MILESTONE_SLA_TIMEOUT,
    SIGNAL_TRIGGER
}
```

`ScheduledJobRequest` gains `JobType jobType` field (replaces `jobClass`). Call sites (`SchedulerService`, `MilestoneActivatedEventHandler`) explicitly pass the appropriate `JobType` — this is a design change, not a description of current behavior. Currently, job type inference is internal to `QuartzJobScheduler.resolveJobClass()` via the `triggerType` string in the data map. Making `JobType` explicit at call sites gives compile-time exhaustiveness checking.

`QuartzJobScheduler` maps `JobType` → Quartz `Job` class internally.

#### 1.7 RetryHandler Interface

Functional interface in `common/internal/executor/`:
```java
@FunctionalInterface
public interface RetryHandler {
    void handleFailure(WorkerTaskData taskData, Throwable cause, String errorMessage);
}
```

Each scheduler provides its own `RetryHandler` that delegates to `RetryOrchestrator` with the scheduler-specific `RescheduleCallback`.

#### Phase 1 Validation

All existing Quartz tests must pass unchanged. The extraction is purely structural — no behavioral changes.

### Phase 2 — db-scheduler Module

#### 2.1 Module Structure

New Maven module `scheduler-dbscheduler`:

```
scheduler-dbscheduler/
  pom.xml
  src/main/java/io/casehub/engine/scheduler/dbscheduler/
    DbSchedulerJobScheduler.java          # implements JobScheduler
    DbSchedulerWorkerExecutionManager.java # @WorkerBackend, implements WorkerExecutionManager
    DbSchedulerRetryService.java          # wraps RetryOrchestrator
    DbSchedulerLifecycle.java             # CDI startup/shutdown
    WorkerExecutionTaskHandler.java       # db-scheduler ExecutionHandler for WORKER_EXECUTION
    ScheduledTriggerTaskHandler.java      # ExecutionHandler for SCHEDULED_TRIGGER_UNCONDITIONAL
    ConditionalScheduledTriggerTaskHandler.java # ExecutionHandler for SCHEDULED_TRIGGER_CONDITIONAL
    MilestoneSLATimeoutTaskHandler.java   # ExecutionHandler for MILESTONE_SLA_TIMEOUT
    ScheduledSignalTaskHandler.java       # ExecutionHandler for SIGNAL_TRIGGER
    ScheduledJobData.java                 # serialisation helper for task_data column
  src/main/resources/
    application.properties
  src/test/resources/
    application.properties
```

Dependencies: `casehub-engine-common`, `db-scheduler` (com.github.kagkarlsson:db-scheduler), `quarkus-jdbc-h2` (default datasource).

#### 2.2 DbSchedulerJobScheduler

Implements `JobScheduler`. Maps domain types to db-scheduler's API:

- `schedule(ScheduledJobRequest)` → `schedulerClient.schedule(taskInstance, executionTime)`
- `cancel(JobIdentifier)` → `schedulerClient.cancel(taskInstance)`
- `cancelGroup(String)` → batch cancel via `schedulerClient.fetchScheduledExecutions()` filtered by instance ID prefix, then `cancel()` each. Performance note: O(n) round-trips vs Quartz's single `deleteJobs()`. Acceptable for the current scale (tens of triggers per case, not thousands).
- `exists(JobIdentifier)` → `schedulerClient.getScheduledExecution(taskInstance)`

**Instance ID convention:** `"{group}:{name}"` maps `JobIdentifier`'s 2D key to db-scheduler's single string instance ID.

**Task name convention:** One db-scheduler `Task` per `JobType`:
- `"worker-execution"` → `WorkerExecutionTaskHandler`
- `"scheduled-trigger"` → `ScheduledTriggerTaskHandler`
- `"conditional-trigger"` → `ConditionalScheduledTriggerTaskHandler`
- `"milestone-sla-timeout"` → `MilestoneSLATimeoutTaskHandler`
- `"signal-trigger"` → `ScheduledSignalTaskHandler`

#### 2.3 DbSchedulerWorkerExecutionManager

`@WorkerBackend @Priority(10) @ApplicationScoped` — higher priority than Quartz's `@Priority(0)`, making it the default when both are on the classpath.

Implements `WorkerExecutionManager`:
- `submit()` → schedules a `"worker-execution"` task via `DbSchedulerJobScheduler`
- `supports()` → `true` (same as Quartz)
- `canExecute()` → delegates to `WorkerFunctionHandler` instances (same as Quartz)
- `getActiveWorkCount()` / `getActiveCaseIds()` → query `scheduled_tasks` table
- `supportsRecovery()` → `true`
- `schedulePersistedEvent()` → schedules from persisted EventLog (recovery path)

#### 2.4 Task Handlers

Each `ExecutionHandler` is a thin shim delegating to the appropriate orchestrator from Phase 1.

Worker execution example:
```java
public class WorkerExecutionTaskHandler implements ExecutionHandler<ScheduledJobData> {
    @Override
    public void execute(TaskInstance<ScheduledJobData> taskInstance, ExecutionContext context) {
        WorkerTaskData taskData = taskInstance.getData().toWorkerTaskData();
        orchestrator.execute(taskData, retryService::handleFailure);
    }
}
```

**Concurrency exclusion:** Quartz uses `@DisallowConcurrentExecution` on `ScheduledTriggerJob`, `ConditionalScheduledTriggerJob`, and `ScheduledSignalJob` to prevent overlapping cron executions. db-scheduler handles this differently: recurring tasks use the same `task_instance` ID, and a task can only be "picked" once (optimistic locking on `version` column). If a cron tick fires while the previous execution is still running (task is `picked=true`), the new execution is skipped. This provides equivalent semantics without an explicit annotation.

db-scheduler's built-in retry is NOT used — engine's `RetryOrchestrator` handles retry with its richer policy model (exponential backoff, max attempts, per-worker overrides). db-scheduler tasks are one-shot for worker execution; recurring for cron triggers.

#### 2.5 DbSchedulerRetryService

Wraps `RetryOrchestrator` with a db-scheduler `RescheduleCallback`:

```java
void handleFailure(WorkerTaskData taskData, Throwable cause, String errorMessage) {
    retryOrchestrator.handleFailure(
        taskData, cause, errorMessage,
        (data, delay) -> scheduler.schedule(/* reschedule as new one-shot task */));
}
```

#### 2.6 DbSchedulerLifecycle

`@ApplicationScoped` startup bean:
- Creates `Scheduler` instance with the module's `DataSource`
- Registers all `ExecutionHandler` task definitions
- Starts the scheduler on `@Observes StartupEvent`
- Stops on `@Observes ShutdownEvent`

#### 2.7 Dual-Mode Storage

**Default (H2 in-memory):** db-scheduler uses an H2 in-memory datasource. Tasks are ephemeral — identical to Quartz's current RAM store semantics. Recovery on restart uses the existing `WorkerExecutionRecoveryService` (EventLog replay). No Flyway migration needed. No dual-recovery conflict — only EventLog recovery is active.

```properties
# Default — H2 in-memory (no config needed, module provides defaults)
casehub.scheduler.dbscheduler.datasource=h2-scheduler
quarkus.datasource.h2-scheduler.db-kind=h2
quarkus.datasource.h2-scheduler.jdbc.url=jdbc:h2:mem:scheduler;DB_CLOSE_DELAY=-1
quarkus.datasource.h2-scheduler.username=sa
```

db-scheduler's `createIfNotExists(true)` creates the table in H2 automatically — no migration file needed for the default mode.

**Opt-in (PostgreSQL durable):** Consumer points db-scheduler at their PostgreSQL datasource and adds the scoped Flyway migration. Tasks survive restart. Recovery uses both EventLog replay AND db-scheduler persistence — `DbSchedulerWorkerExecutionManager` must coordinate with `WorkerExecutionRecoveryService` to prevent duplicate execution (idempotency via `inputDataHash`).

```properties
# Opt-in — PostgreSQL durable
casehub.scheduler.dbscheduler.datasource=default
quarkus.flyway.locations=classpath:db/migration,classpath:db/engine-scheduler/migration
```

Scoped Flyway migration at `db/engine-scheduler/migration/V1__Create_DbScheduler_Table.sql`:

```sql
CREATE TABLE IF NOT EXISTS scheduled_tasks (
    task_name       TEXT NOT NULL,
    task_instance   TEXT NOT NULL,
    task_data       BYTEA,
    execution_time  TIMESTAMPTZ NOT NULL,
    picked          BOOLEAN NOT NULL DEFAULT FALSE,
    picked_by       TEXT,
    last_success    TIMESTAMPTZ,
    last_failure    TIMESTAMPTZ,
    consecutive_failures INT,
    last_heartbeat  TIMESTAMPTZ,
    version         BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (task_name, task_instance)
);

CREATE INDEX idx_scheduled_tasks_execution_time
    ON scheduled_tasks (execution_time)
    WHERE picked = FALSE;
```

#### 2.8 Polling Configuration

```properties
casehub.scheduler.dbscheduler.threads=4
casehub.scheduler.dbscheduler.polling-interval=500ms
casehub.scheduler.dbscheduler.heartbeat-interval=5m
```

Polling interval defaults to 500ms — sub-second latency on the worker execution hot path. With H2 in-memory, the polling query is effectively free (no network round-trip). With PostgreSQL, the indexed `execution_time` query is lightweight.

#### 2.9 Default Selection

db-scheduler at `@Priority(10)` beats Quartz at `@Priority(0)`. `CompositeWorkerExecutionManager` + `FirstSupportedRoutingStrategy` naturally selects db-scheduler first.

To use Quartz instead: exclude `scheduler-dbscheduler` from the classpath. No configuration needed.

## Testing

### Phase 1
- Existing Quartz tests pass unchanged (structural extraction, no behavior change)
- New unit tests for all 4 orchestrators with mock dependencies
- `JobSchedulerContractTest` in common — abstract contract test that both implementations extend

### Phase 2
- `DbSchedulerJobSchedulerTest` extends `JobSchedulerContractTest`
- `DbSchedulerWorkerExecutionManagerTest` — integration test with H2 in-memory
- `DbSchedulerRetryServiceTest` — unit test with mock `RetryOrchestrator`
- End-to-end test: case start → worker dispatch → completion via db-scheduler
- Concurrency exclusion test: verify cron overlap prevention

## Migration Path for Consumers

1. Add `scheduler-dbscheduler` to classpath (it becomes the default automatically, using H2 in-memory)
2. No Flyway changes needed for default mode
3. For durable scheduling: set `casehub.scheduler.dbscheduler.datasource=default` and add Flyway location `classpath:db/engine-scheduler/migration`
4. Optionally remove `scheduler-quartz` if Quartz features aren't needed
5. No code changes required — the `@WorkerBackend` + `@Priority` mechanism handles selection

## References

- backup/issue-813-original — preserved original branch work
- `common/spi/scheduler/JobScheduler.java` — current SPI
- `common/spi/scheduler/WorkerExecutionManager.java` — current SPI
- `scheduler-quartz/` — current Quartz implementation (6 job classes, 1 job listener)
- `runtime/internal/engine/handler/WorkerScheduleEventHandler.java` — entry point for worker dispatch
- `runtime/internal/engine/recovery/WorkerRecoveryCoordinator.java` — recovery gate
- `common/internal/worker/scope/ScopedWorkerRegistry.java` — scoped worker interaction
- db-scheduler docs: https://github.com/kagkarlsson/db-scheduler
- CLAUDE.md "No Migration Tooling" section — scoped migration exception pattern
- Design review: `/Users/mdproctor/reviews/casehub-slots/issue-813-db-scheduler-20260821-065603/`
