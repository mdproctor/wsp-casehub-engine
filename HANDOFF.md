# Handoff — 2026-06-16 (session 2)

**Branch:** `issue-463-function-worker-design` — MID-WORK, branch stays open
**Spec:** `docs/specs/2026-06-16-worker-execution-redesign.md` (rev 4, approved after 4 review rounds)

## What's Done

### Earlier this session
- PR#499 merged (signal API #493, implementation routing #476, repeatable stage #482, auto-registration #497)
- engine#498 closed (CDI protocol updated in garden)
- PR#496 reviewed and merged (RLS cross-tenant guard)

### This branch (#463)
- Design spec written, 4 review rounds, approved
- **WorkerFunction sealed type** — replaces `WorkerFunctionHolder<T>`. Three variants: `Sync`, `AgentExec`, `Flow`. `WorkerFunctionHolder` deleted. All 10 call sites updated. Worker constructors updated (legacy `CaseContext` function type and `File` overload removed).
- **WorkerExecutor interface** — `common/internal/executor/`. `DefaultWorkerExecutor` in `runtime/` using `@VirtualThreads ExecutorService`. `ExecutionMetadata` record for flow lineage.
- **RetryPolicies static utility** — `common/internal/executor/`. `RetryDecision` sealed type. Backoff computation moved from `QuartzWorkerExecutionJobListener`.
- **WorkflowExecutor signature change** — `CaseInstance` → `UUID caseId`. `FlowExecutionRegistry`, `FlowExecution`, `CasehubDispatch` (injects `CaseInstanceCache`), `NoOpWorkflowExecutor`, `ServerlessWorkflowExecutor` all updated.
- **quarkus-virtual-threads** dependency added to `runtime/pom.xml`

## What's Next — Task 11

**Refactor QuartzWorkerExecutionJob to thin fire-and-forget adapter.** This is the final implementation step:

1. Extract `QuartzRetryService` (`@ApplicationScoped`) from `QuartzWorkerExecutionJobListener` — owns `handleFailure()`, `resolveRetryPolicy()`, `countFailedAttempts()`, `rescheduleWorker()`
2. `QuartzWorkerExecutionJob.execute()` becomes fire-and-forget: subscribe to `workerExecutor.execute()`, success callback publishes `WORKER_EXECUTION_FINISHED` (with PlannedAction enrichment), failure callback calls `retryService.handleFailure()`
3. `QuartzWorkerExecutionJobListener` simplifies to `jobToBeExecuted()` only
4. Remove `WORKFLOW_EXECUTION_FAILED` event bus address, `WorkflowExecutionFailed` event type, `handleWorkflowFailure()`, `onWorkflowExecutionFailed()`
5. Move `WorkerExecutionConfig` from `scheduler-quartz` to `common/internal/executor/`

After Task 11: code review, commit, CLAUDE.md update, protocol PP-20260531 update, PR.

## Commits on branch (5)
```
b7251f88 docs: revise spec (rev 4)
fee7f4df docs: revise spec (rev 3)
ccb51934 docs: correct durability section
ecea907b feat: WorkerExecutor SPI, RetryPolicies, WorkflowExecutor UUID signature
818db760 feat: sealed WorkerFunction replaces WorkerFunctionHolder
```

## Test status
249 tests pass across api, common, flow, blackboard. Runtime and scheduler-quartz @QuarkusTest suites not yet run (will run after Task 11 completion).

## Issues filed this session
- parent#259 — write-content blog entries should link across sessions for same branch
