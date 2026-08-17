---
layout: post
title: "The View That Wasn't Just a View"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [rest, snapshots, orchestration, workbench]
---

The issue said "no new logic — just a new view composing existing snapshots." Six endpoints already existed on `PlanResource` serving decomposition trees, DAG plans, execution results, and live plan models. The orchestration workbench needed all of that in one JSON shape. A composition endpoint.

The `ExecutionSnapshot` TypeScript contract was clear: `executionId`, `state`, `model` with a pattern and failure policy, `activeAgents`, `completedAgents` with per-agent status and duration, timing fields. The engine types were clear: `CasePlanModelSnapshot`, `DagPlanSnapshot`, `DagResultSnapshot`. Map one to the other. Done by lunch.

Except the mapping exposed what the existing snapshots didn't carry.

The TypeScript contract has five `AgentRefType` values: WORKER, HUMAN, EXTERNAL, COMPOSED, CHANNEL. The engine's `AgendaItemSnapshot` had four fields — `planItemId`, `bindingName`, `status`, `description`. No target type. Every agenda item would show as WORKER, even human tasks. The fix wasn't in the view — it was in `AgendaItemSnapshot` itself, threaded from `BindingTarget` through `PlanningCasePlanModelSnapshotProvider`.

The contract distinguishes TIMEOUT from FAILURE in `AgentResultStatus`. The engine doesn't — `WorkerOutcome.Expired` becomes FAULTED at the PlanItem level, indistinguishable from a connection error. No clean enum variant exists in `NodeState` for expiration. We went pragmatic: pattern-match on the failure reason string. If it contains "timeout", "timed out", or "expired", map to TIMEOUT. Not elegant, but correct for every case the engine produces today.

Per-agent duration: the contract expects milliseconds on each `AgentResult`. `DagResultSnapshot` had total elapsed time but nothing per-node. The data existed implicitly — `SnapshotCapturingDagEventListener` receives `onNodeDispatched` and `onNodeCompleted` callbacks for every node. We added dispatch-time tracking with `ConcurrentHashMap<String, Instant>`, computed duration at completion, and threaded a `nodeDurationsMs` map through `DagResultSnapshot`. One new field on the record, one backward-compatible constructor, the listener tracks what it already observed.

The `ExecutionModel` strategy fields — `routingStrategy`, `decompositionStrategy` — were null because the REST layer didn't have `CaseDefinition`. But `PlanResource` already injected `CaseService`, which has `CaseDefinitionRegistry`, which has `getCaseDefinition(CaseMetaModel)`. One new injection, one `try/catch` for cases where the definition is no longer registered, and the strategies populate from what the definition declares.

State refinement: IDLE, RUNNING, COMPLETE, FAULTED covered the common cases but missed three states the workbench knows about. DELEGATED agenda items — all active work handed to external actors — map to WAITING_FOR_AGENT. SUSPENDED items map to WAITING_FOR_EVENT. All-cancelled DAG nodes map to CANCELLED. Three stream checks in `deriveState()`, three new paths.

Pattern detection from DAG topology turned out cleaner than expected. All nodes independent → PARALLEL. Every node has at most one predecessor → SEQUENCE. Anything else → HTN. Single node or no DAG → SEQUENCE. Five lines of stream logic covering every current engine pattern.

The composition endpoint itself is the smallest part of the change. The real work was surfacing data the engine already had — target types, durations, strategy names, refined states — through snapshot types that were designed for a narrower audience.
