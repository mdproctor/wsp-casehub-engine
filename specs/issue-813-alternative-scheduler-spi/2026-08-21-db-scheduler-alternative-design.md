# db-scheduler as Default Worker Scheduler

**Issue:** casehubio/engine#813
**Date:** 2026-08-21
**Status:** Design

## Context

The engine's worker scheduling is implemented solely by Quartz. The scheduler has two SPI levels:

- **`JobScheduler`** — timed/cron job scheduling (milestones, scheduled triggers, signal triggers)
- **`WorkerExecutionManager`** — worker execution dispatch (the hot path for capability-triggered workers)

Both are Quartz-only today. `QuartzWorkerExecutionJob` contains ~250 lines of domain logic (EventLog resolution, CaseInstance recovery, definition lookup, context bridge, worker execution, output publishing). `QuartzRetryService` contains ~220 lines of retry logic (failure persistence, policy resolution, backoff computation). None of this logic is Quartz-specific — it's engine domain logic trapped in Quartz classes.

Additionally, `ScheduledJobRequest.jobClass` is `Class<?>` — a Quartz-specific type (`org.quartz.Job`) leaked into the scheduler-agnostic SPI.

An earlier attempt (issue-813 branch, July 2026) extracted orchestrators and built a db-scheduler module but was never merged. Main has since evolved 203 commits. This design rebuilds the work against the current API surface, incorporating the original branch's sound architectural decisions.

## Goals

1. Extract scheduler-agnostic domain logic from Quartz into reusable orchestrators
2. Fix the `jobClass` Quartz leak in the SPI
3. Add db-scheduler as a second `WorkerExecutionManager` and `JobScheduler` implementation
4. Make db-scheduler the default scheduler (higher `@Priority`)
5. Keep Quartz as an opt-in alternative

## Non-Goals

- Removing Quartz — it remains available for consumers who need its richer scheduling features (calendars, clustering, misfire policies)
- Changing the `CompositeWorkerExecutionManager` routing — the existing `@WorkerBackend` + `@Priority` mechanism works correctly
- Adding new scheduling capabilities — this is a backend swap, not a feature change

## Design

### Phase 1 — SPI Cleanup and Orchestrator Extraction

#### 1.1 WorkerExecutionOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts all domain logic from `QuartzWorkerExecutionJob`:

- Resolve EventLog by ID
- Recover CaseInstance from cache or repository
- Look up CaseDefinition, Worker, Capability from registries
- Resolve ContextBridge and deserialise typed input
- Build WorkerContext (channels, experiences, propagation context)
- Delegate to `WorkerExecutor.execute()`
- On success: publish `WORKER_EXECUTION_FINISHED` (WorkflowExecutionCompleted)
- On failure: delegate to retry handling

After extraction, `QuartzWorkerExecutionJob.execute()` becomes a ~10-line shim:
```java
@Override
public void execute(JobExecutionContext context) {
    WorkerTaskData taskData = WorkerTaskData.fromJobDataMap(context.getJobDetail().getJobDataMap());
    orchestrator.execute(taskData, this::onFailure);
}
```

#### 1.2 RetryOrchestrator

New `@ApplicationScoped` bean in `common/internal/executor/`. Extracts all retry logic from `QuartzRetryService`:

- Persist `WORKER_EXECUTION_FAILED` EventLog
- Resolve retry policy from Worker's `ExecutionPolicy`
- Count prior failures from EventLog
- Call `RetryPolicies.evaluate()` → `RetryDecision`
- On `Retry(delay)`: invoke `RescheduleCallback` (scheduler-specific)
- On `Exhaust(reason)`: publish `WORKER_RETRIES_EXHAUSTED`

`RescheduleCallback` is a functional interface:
```java
@FunctionalInterface
public interface RescheduleCallback {
    void reschedule(WorkerTaskData taskData, Duration delay);
}
```

After extraction, `QuartzRetryService` becomes a thin adapter that provides the Quartz-specific `RescheduleCallback`.

#### 1.3 WorkerTaskData

Record in `common/internal/executor/`:
```java
public record WorkerTaskData(
    Long eventLogId,
    String inputDataHash,
    UUID caseId,
    String workerId,
    String tenancyId,
    String bindingName,
    String signalId
) {
    // Copy methods for optional fields
    public WorkerTaskData withBindingName(String bindingName) { ... }
    public WorkerTaskData withSignalId(String signalId) { ... }

    // Scheduler-specific conversion helpers
    public Map<String, String> toMap() { ... }
    public static WorkerTaskData fromMap(Map<String, String> map) { ... }
}
```

#### 1.4 JobType Enum

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

`ScheduledJobRequest` gains `JobType jobType` field (replaces `jobClass`). `QuartzJobScheduler.resolveJobClass()` maps `JobType` → Quartz `Job` class internally. Call sites in `SchedulerService` and `MilestoneActivatedEventHandler` pass the appropriate `JobType`.

#### 1.5 RetryHandler Interface

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

Dependencies: `casehub-engine-common`, `db-scheduler` (com.github.kagkarlsson:db-scheduler).

#### 2.2 DbSchedulerJobScheduler

Implements `JobScheduler`. Maps domain types to db-scheduler's API:

- `schedule(ScheduledJobRequest)` → `schedulerClient.schedule(taskInstance, executionTime)`
- `cancel(JobIdentifier)` → `schedulerClient.cancel(taskInstance)`
- `cancelGroup(String)` → query + cancel by instance ID prefix
- `exists(JobIdentifier)` → `schedulerClient.getScheduledExecution(taskInstance)`

**Instance ID convention:** `"{group}:{name}"` maps `JobIdentifier`'s 2D key to db-scheduler's single string instance ID. Group-based operations use prefix matching.

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

Each `ExecutionHandler` is a thin shim. Example for worker execution:

```java
public class WorkerExecutionTaskHandler implements ExecutionHandler<ScheduledJobData> {
    @Override
    public void execute(TaskInstance<ScheduledJobData> taskInstance, ExecutionContext context) {
        WorkerTaskData taskData = taskInstance.getData().toWorkerTaskData();
        orchestrator.execute(taskData, retryService::handleFailure);
    }
}
```

db-scheduler's built-in retry is NOT used — engine's `RetryOrchestrator` handles retry with its richer policy model (exponential backoff, max attempts, per-worker overrides). db-scheduler tasks are one-shot.

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
- Creates `Scheduler` instance with the application's `DataSource`
- Registers all `ExecutionHandler` task definitions
- Starts the scheduler on `@Observes StartupEvent`
- Stops on `@Observes ShutdownEvent`

Configuration via Quarkus config properties:
```properties
casehub.scheduler.dbscheduler.threads=4
casehub.scheduler.dbscheduler.polling-interval=5s
casehub.scheduler.dbscheduler.heartbeat-interval=5m
```

#### 2.7 Schema

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

Consumers add to their Flyway config:
```properties
quarkus.flyway.locations=classpath:db/migration,classpath:db/engine-scheduler/migration
```

#### 2.8 Default Selection

db-scheduler at `@Priority(10)` beats Quartz at `@Priority(0)`. `CompositeWorkerExecutionManager` + `FirstSupportedRoutingStrategy` naturally selects db-scheduler first.

To use Quartz instead: exclude `scheduler-dbscheduler` from the classpath. No configuration needed — the `@WorkerBackend` discovery handles it automatically.

## Testing

### Phase 1
- Existing Quartz tests pass unchanged (structural extraction, no behavior change)
- New unit tests for `WorkerExecutionOrchestrator` and `RetryOrchestrator` with mock dependencies
- `JobSchedulerContractTest` in common — abstract contract test that both implementations extend

### Phase 2
- `DbSchedulerJobSchedulerTest` extends `JobSchedulerContractTest`
- `DbSchedulerWorkerExecutionManagerTest` — integration test with Testcontainers PostgreSQL
- `DbSchedulerRetryServiceTest` — unit test with mock `RetryOrchestrator`
- End-to-end test: case start → worker dispatch → completion via db-scheduler

## Migration Path for Consumers

1. Add `scheduler-dbscheduler` to classpath (it becomes the default automatically)
2. Add Flyway location: `classpath:db/engine-scheduler/migration`
3. Optionally remove `scheduler-quartz` if Quartz features aren't needed
4. No code changes required — the `@WorkerBackend` + `@Priority` mechanism handles selection

## References

- backup/issue-813-original — preserved original branch work
- `common/spi/scheduler/JobScheduler.java` — current SPI
- `common/spi/scheduler/WorkerExecutionManager.java` — current SPI
- `scheduler-quartz/` — current Quartz implementation
- `runtime/internal/engine/handler/WorkerScheduleEventHandler.java` — entry point for worker dispatch
- db-scheduler docs: https://github.com/kagkarlsson/db-scheduler
- CLAUDE.md "No Migration Tooling" section — scoped migration exception pattern
