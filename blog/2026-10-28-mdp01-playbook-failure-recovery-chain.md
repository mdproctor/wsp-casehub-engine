---
layout: post
title: "Why Your Playbook Steps Couldn't Tell You What Went Wrong"
date: 2026-10-28
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [recovery, triage, playbook, workflow, design]
series: issue-1262-playbook-failure-triage
---

# Why Your Playbook Steps Couldn't Tell You What Went Wrong

A playbook with five steps fails on step three. What does the engine tell you? "Workflow execution failed." Which step? What was it doing? What succeeded before the failure? Nothing — just a bare exception propagating up from `CompletableFuture.join()`.

This is the gap that drove the design work behind structured playbook failure output and the triage wiring for CaseHub's recovery coordinator.

## The information loss problem

The Serverless Workflow SDK has all the data. `WorkflowExecutionListener` fires `onTaskStarted`, `onTaskCompleted`, `onTaskFailed` for every task in a workflow. Each event carries the task name, start time, completion time, and on failure the cause. The data exists — it just evaporates before reaching the recovery system.

`FlowWorkerFunctionHandler.execute()` calls `wfInstance.start().join()`. On failure, the exception propagates through `WorkerExecutionOrchestrator` into `RetryOrchestrator.handleFailure(taskData, Throwable, message)`. By this point, step-level data is gone. `RetryOrchestrator` builds a `RecoveryContext` with `null` for the capability name, the outcome, and the plan item ID. The actual Throwable — which carried at least a root-cause message — is discarded before `DefaultRecoveryCoordinator` ever sees it.

The recovery coordinator classifies failures into three levels: TRANSIENT (retry the same agent), REASONING (exclude the agent, try another), FUNDAMENTAL (replan from scratch). None of these levels can make an informed decision because they don't know which step failed or what the successful steps produced.

## Two problems, one design

The design splits into two concerns that feed each other.

First, structured step capture. A CDI `WorkflowExecutionListener` bean accumulates `StepResult` entries in `FlowExecutionRegistry` — not in the SWF SDK's own metadata, which would require casting through `WorkflowContextData` to reach `WorkflowMutableInstance.addMetadataIfAbsent()`. The registry already maps instance IDs to execution context; a separate `ConcurrentHashMap` for step data keeps the immutable `FlowExecution` record untouched while avoiding SDK coupling.

After `join()`, the handler reads the accumulated steps and builds a `PlaybookExecutionResult` — a sealed interface with `Success`, `PartialSuccess`, and `Failure` variants, each carrying the full step trace. On failure, this gets wrapped in a `PlaybookExecutionException` that rides the existing exception propagation path. No interface changes to `RetryHandler`. No blast radius into the scheduler backends.

Second, triage. `TriageHandler` is a platform SPI — `evaluate(TriageContext) → TriageDecision`, where the decision is either `Retry(modifications)` or `Abandon(reason)`. It sits inside `DefaultRecoveryCoordinator`, after all three structural recovery levels have been exhausted. Structural recovery answers "what kind of failure is this?" Triage answers "what should we change about the playbook to make it work?"

The insertion point matters. Triage fires only after TRANSIENT, REASONING, and FUNDAMENTAL recovery have all failed. Structural fixes are cheaper than semantic triage — reclassifying the error and trying a different agent costs less than analysing step-by-step failure patterns and modifying the playbook. The triage attempt count is separate from retry counts and configured on `RecoveryPolicy`, consistent with how `maxRetries` and recovery levels are already specified.

## The sealed interface for workflow functions

The work also introduced `SWFWorkerFunction` — a sealed interface that both `FlowWorkerFunction` (raw SWF `do:` blocks) and `PlaybookWorkerFunction` (compiled playbook `steps:` blocks) implement. Before this, `FlowWorkerFunctionHandler` checked `instanceof FlowWorkerFunction`. Now it checks `instanceof SWFWorkerFunction` and accesses the workflow through a shared `workflow()` accessor. One handler, two function types, identical execution path.

`PlaybookWorkerFunction` carries the `CompilationResult` from `casehub-platform-playbook-compiler` — which includes a `SourceMap` mapping SWF task positions back to playbook step names. When the step capture listener records a failure at SWF task position `/transform-data`, the source map resolves it back to the playbook's `transform-data` step with its original line number. The error message stops being opaque.

## Where the modifications live

The platform `TriageDecision.Retry` carries `Map<String, Object>` — an untyped map. I considered proposing a typed `TriageModification` marker interface in `platform-api`, but sealed `permits` clauses don't compile across module boundaries, and a non-sealed marker loses the exhaustiveness we want. Instead, modifications use a map key convention (`playbook.skipSteps`, `playbook.replaceInput.*`, `playbook.updateParams.*`) and the engine deserialises them into a sealed `PlaybookTriageModification` hierarchy locally. The platform SPI stays generic; the engine gets type safety where it matters.

Modifications are written to a `triage` namespace in the case context's working layer. Re-dispatch uses `CaseContextChangedEvent` — the same mechanism Level 3 recovery uses for full replans. The execution listener reads triage directives from the context on re-execution and applies them: skip a step, replace a step's input, update a parameter.

## What retry multiplication means

With `maxAttempts=3` and `maxTriageAttempts=2`, total possible executions are `3 × (1 + 2) = 9`. Each triage-modified execution gets its own fresh retry budget. This is intentional — a triage modification might fix the semantic failure but the re-execution could still hit a transient network error. Counting previous retries against a modified playbook penalises the modification for something it didn't cause.

Retry counting partitions on a `triageRound` discriminator in event log metadata. Round 0 is pre-triage. Each triage attempt increments the round. `countFailedAttempts()` filters by the current round, so the re-dispatched worker starts with a clean count.

## What landed

All four batches are complete and on main — 14 tasks squashed into 5 commits.

Batch 1 delivered the `SWFWorkerFunction` sealed hierarchy and `PlaybookWorkerFunctionProvider`. Batch 2 added structured step capture: `PlaybookExecutionListener` accumulates `StepResult` entries in `FlowExecutionRegistry`, and `PlaybookExecutionException` carries the full step trace through the existing exception propagation path.

Batch 3 was the most architecturally interesting. `DefaultRecoveryCoordinator.handleFailure()` was restructured so that `skipRecovery` skips structural recovery levels but falls through to triage. A binding with `skipRecovery=true, skipTriage=false` bypasses the classification-and-escalation dance and goes straight to semantic triage — a valid configuration for bindings where the error type is irrelevant but the playbook modification is everything. The `TriageHandler` injection, `TriageContext` construction, retry/abandon dispatch, and `PlaybookTriageModification` parser all landed here, along with retry count partitioning by triage round.

Batch 4 added `PlaybookOutcomeRecorder` for per-execution CBR recording alongside the existing per-step `StepExecutionCbrBridge`.

Code review caught a race condition in `FlowWorkerFunctionHandler` — the `CompletableFuture.whenComplete` callback was removing step data from the registry before the error handler could read it, which would have produced empty step lists in every `PlaybookExecutionException`. Removed the callback; cleanup is handled explicitly in each code path.

The platform-side triage infrastructure (platform#668) also landed in parallel — `TriageHandler` SPI, `LlmTriageHandler` implementation in the agency repo, and CBR execution chain linkage in neocortex. The remaining gap is engine#1264: applying the triage modifications to the playbook during re-execution, which closes the loop between "triage decides what to change" and "the change takes effect."
