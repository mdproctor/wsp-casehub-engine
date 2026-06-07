---
layout: post
title: "The gate that doesn't exist in the code until it fires"
date: 2026-06-07
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [casehub-engine, ActionRiskClassifier, design, gate, WorkerResult]
---

The idea behind `ActionRiskClassifier` is simple: a worker is about to do something consequential — file a SAR, freeze an account, push to production — and before the engine commits that output to the case, a classifier gets to say "no, wait, a human needs to see this." The implementation is not simple.

The first question was how a worker even signals intent. Workers return `Map<String, Object>` — there's no hook. We looked at three options: a thread-local side channel (`WorkerContext.declareAction()`), a reserved key in the output map, or a new return type. The thread-local fails for workflow workers — they run asynchronously, the thread-local context doesn't survive. We went with the breaking change: all worker functions now return `WorkerResult`, which wraps the output map plus an optional `PlannedAction`. The Serverless Workflow workers return `WorkerResult.of(output)` with no action — same bytes, different wrapper.

For the gate mechanism itself, the question was Qhorus oversight channel or WorkItem. Qhorus is what we already use for routing escalation. But AML needs MLRO sign-off with a specific approver role and a 24-hour deadline. Clinical needs physician approval. Neither of those are properties of a Qhorus QUERY/RESPONSE interaction — they're properties of a WorkItem, which already has `candidateGroups`, `expiresIn`, and `scope`. We built around WorkItem.

The gate state — what we call `pendingActionGate` — lives on the `CaseInstance` in memory. That decision cost us time. The JPA entity has no `pending_action_gate` column, so `JpaCaseInstanceRepository.update()` silently ignores it. When we wrote the first version of `ActionGateApprovedHandler`, we loaded the case via `CrossTenantCaseInstanceRepository` and got null every time. The mismatch log fired, the approval was discarded, the case stalled. The fix is `CaseInstanceCache.get(caseId)` — the cache holds the live object with the gate still set. The repository gives you a fresh DB snapshot where in-memory fields don't exist.

The approval path has an elegant property once you understand the engine. When a gate is approved, the handler doesn't need to re-apply the worker's output, resume orchestration, notify listeners, complete the PlanItem, or fire the goal evaluator. It does one thing: re-publish `WorkflowExecutionCompleted` with `plannedAction=null`. `WorkflowExecutionCompletedHandler` runs the entire normal path from that event — same code, same machinery. The `null` `plannedAction` short-circuits the gate fork. Nothing is duplicated.

One thing we learned the hard way: `WorkerRetriesExhaustedEvent` is dual-use. It faults a PlanItem in the blackboard module and transitions the CaseInstance to FAULTED in the runtime module. Publishing it from gate rejection handlers caused cases to die rather than stay RUNNING for the rejection binding to react. That event means "worker is done and failed permanently at the engine level." Gate rejection is different — the worker ran, the action was refused, the case keeps going. We added `ACTION_GATE_WORKER_FAULTED` as a separate address that only the blackboard consumes.

The five consumer repos — aml, clinical, devtown, life, openclaw — each get the same SPI. They implement `ActionRiskClassifier` with `@RiskClassifier`, the engine chains them with most-restrictive-wins semantics, and the whole oversight path emerges from configuration rather than case definitions. A hospital with both AML and clinical classifiers registered gets both automatically, without either knowing about the other.
