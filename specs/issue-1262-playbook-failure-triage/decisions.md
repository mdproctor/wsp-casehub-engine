# Decisions — issue-1262-playbook-failure-triage

## D1: PlaybookExecutionResult scope

**Choice:** Playbook-only — only FlowWorkerFunctionHandler produces step-level results
**Alternatives:**
- General HandlerResult extension — adds optional structured-result field to HandlerResult for any handler type. Over-generalizes when other handlers (MCP, ReAct, A2A, SyncAgent) are single-operation.
- Sealed per-handler-type results — each handler gets its own result subtype. Most complexity for little benefit.
**Rationale:** Only flow/playbook workers have step-level execution. Other handlers don't have steps. Generalize later if needed.
**Trade-offs:** If a future handler type (e.g., multi-tool MCP chains) needs step-level results, we'd need to extend then.
**Sources:** FlowWorkerFunctionHandler.java:43, WorkerFunctionHandler.java:34, HandlerResult.java:21
**Exploration:** quick
**Status:** captured

## D2: Include #1263 (PlaybookWorkerFunctionProvider) in this branch

**Choice:** Yes — fold #1263 into this branch as batch 1
**Alternatives:**
- Keep them separate — delays playbook-as-worker-definition, misses the type distinction that enables richer step metadata.
**Rationale:** #1263 establishes PlaybookWorkerFunction with CompilationResult (SourceMap, CoordinationSpec). #1262's step capture and triage wiring builds on these types. The platform compiler is on 7.x (serverlessworkflow-types 7.35.3.Final), compatible with engine's 7.25.1.Final.
**Trade-offs:** Larger branch scope (S/Low + L/High).
**Sources:** casehubio/engine#1263, casehubio/platform playbook-compiler/pom.xml (serverlessworkflow-types 7.35.3.Final), engine pom.xml (version.io.serverlessworkflow=7.25.1.Final)
**Exploration:** deep-analysis
**Status:** captured

## D3: Step capture mechanism

**Choice:** CDI WorkflowExecutionListener bean
**Alternatives:**
- Per-execution listener in handler — couples to SDK internals, requires manual registration per call.
- Wrap CompletableFuture with try-catch — loses step timing and partial-success data.
**Rationale:** WorkflowExecutionListener provides onTaskStarted/onTaskCompleted/onTaskFailed callbacks with TaskContext (taskName, startedAt, completedAt, cause). CDI auto-discovery in Quarkus. Accumulates StepResults into WorkflowInstance metadata. FlowWorkerFunctionHandler reads them after join(). Works for both Flow and Playbook functions.
**Trade-offs:** Listener is global (sees all workflow executions, not just casehub ones). Need to filter by instance metadata to identify casehub executions.
**Sources:** WorkflowExecutionListener (SWF SDK), TaskContext (taskName, startedAt, completedAt, position), TaskFailedEvent (cause), FlowExecutionRegistry.java:26
**Exploration:** quick
**Status:** captured

## D4: Triage insertion point

**Choice:** After recovery exhaustion — between DefaultRecoveryCoordinator failure and WorkerRetriesExhaustedEvent
**Alternatives:**
- Replace DefaultRecoveryCoordinator — simpler pipeline but loses classification-driven escalation.
- Inside each recovery level — premature at TRANSIENT level, too complex.
**Rationale:** Triage is semantic (interprets WHY steps failed, suggests modifications). Recovery is structural (reclassifies errors, re-plans). They complement each other. Triage fires only after all structural recovery options (TRANSIENT/REASONING/FUNDAMENTAL) are exhausted.
**Trade-offs:** Triage never gets a chance if recovery succeeds at an earlier level. This is intentional — structural fixes are cheaper than semantic triage.
**Sources:** RetryOrchestrator.java:108-121, DefaultRecoveryCoordinator.java:106-158, TriageHandler SPI (platform-api)
**Exploration:** quick
**Status:** captured

## D5: Triage loop cap configuration

**Choice:** On RecoveryPolicy in CaseDefinition
**Alternatives:**
- Global config property — no per-case customization.
- On TriageHandler SPI itself — engine loses control over loop termination.
**Rationale:** Consistent with how maxRetries and recovery levels are already configured in RecoveryPolicy. Per-case-definition granularity allows different triage budgets for different case types.
**Trade-offs:** Requires schema change to RecoveryPolicy (new maxTriageAttempts field with default).
**Sources:** RecoveryPolicy (engine-api), DefaultRecoveryCoordinator.java:117-119
**Exploration:** quick
**Status:** captured

## D6: Triage retry mechanism

**Choice:** Modify context + re-dispatch via RetryOrchestrator
**Alternatives:**
- Direct re-invocation from RecoveryCoordinator — bypasses retry counts, scheduling, and execution pipeline.
- CDI event (TriageRetryRequestedEvent) — adds event machinery, harder to follow.
**Rationale:** Apply well-known modification keys (skip-step, update-parameter, etc.) to the case context's working layer, then re-dispatch through the existing worker execution path. Reuses retry infrastructure. The next execution sees the modified context.
**Trade-offs:** Retry count management needs care — triage retries should be counted separately from mechanical retries.
**Sources:** RetryOrchestrator.java:86-135, TriageDecision.Retry (platform-api)
**Exploration:** quick
**Status:** captured

## D7: CBR recording scope

**Choice:** All playbook executions — extend StepExecutionCbrBridge pattern
**Alternatives:**
- Triage-only — focused on failure/recovery patterns but misses baseline successful executions.
**Rationale:** Richer training data for CBR. Records every playbook execution with step-level data, plus triage attempts when they occur. The existing StepExecutionCbrBridge records step events — PlaybookOutcomeRecorder adds structured results and triage chain context.
**Trade-offs:** Higher CBR storage volume. Mitigated by CBR's existing retention policies.
**Sources:** StepExecutionCbrBridge.java:34, CbrRecordStore SPI
**Exploration:** quick
**Status:** captured
