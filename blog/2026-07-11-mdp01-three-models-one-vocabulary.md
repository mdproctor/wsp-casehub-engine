---
layout: post
title: "Three Models, One Vocabulary"
date: 2026-07-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [architecture, orchestration, type-unification]
series: issue-700-unify-orchestration-model
---

CaseHub has three coordination models: blackboard (engine), workflow (engine-flow), and agentic patterns (blocks). They all answer the same question — how do I coordinate work across multiple agents? — but they evolved independently, and the type hierarchies diverged.

Engine had `PlanItemStatus`. Blocks had `ExecutionState`. Engine had `AgentAssignment`. Both had "worker name as a string." The same concepts, different names, different shapes, different packages. Cross-model orchestration was impossible without adapters. Unified monitoring was impossible without per-model glue.

This branch fixes the foundation. Not a rewrite — an alignment.

## What Changed

`TaskStatus` replaces `PlanItemStatus`. Same nine states (PENDING through CANCELLED), relocated from `common/internal/model` to `api/model`. The name change matters: these aren't plan-item-specific lifecycle phases. They're coordination-level vocabulary that any orchestration model needs. Blocks' `ExecutionState` maps cleanly: `Idle` → PENDING, `Running` → RUNNING, `WaitingForAgent` → DELEGATED, and so on.

`TaskDescriptor` is the shared behavioral interface. Five methods: `id()`, `description()`, `executor()`, `status()`, `createdAt()`. PlanItem implements it. When blocks adopts it, PlannedTask will too. A `TaskSnapshot` record projects from any descriptor for monitoring — flat strings, no interface references, serializable across boundaries.

The routing layer got the same treatment. `RoutingResult` replaces `AgentAssignment` — a sealed interface with `Selected`, `Unresolvable`, and `Escalated` variants. `Assignment` replaces `Assigned`. `ExecutorRef` provides shared executor identity. `OutcomeKind` provides shared outcome taxonomy. The old types are deleted.

The biggest single change: PlanItem stores `ExecutorRef` instead of a bare `workerName` string. Every `PlanItem.create()` call now takes `ExecutorRef.of("worker-name")` instead of a raw string. That's ~80 test call sites. The persistence layer threads `executorName` and `executorDescription` through `PlanItemRecord`, `PlanItemSaveRequest`, and `PlanItemEntity`.

## The Design Tension That Shaped It

The hardest question was `TaskStatus`: enum or sealed variants?

Blocks' `ExecutionState` uses sealed records with data — `Running(int iteration)`, `WaitingForAgent(AgentRef agent)`. That's richer. But PlanItem's CAS transitions need identity comparison (`compareAndSet(PENDING, RUNNING)`), and sealed records with data fields break that — two `Running` instances with different iterations aren't equal. JPA `@Enumerated(STRING)` also assumes an enum.

The resolution: status is a phase label. Contextual data (which agent, what iteration, why it faulted) belongs on the task or in events, not on the status itself. The enum wins on first principles, not on convenience.

## What This Enables

Four issues filed for blocks adoption: `AgentRef extends ExecutorRef`, `PlannedTask implements TaskDescriptor`, `SubTaskStatus` → `TaskStatus`, and an engine-side issue for migrating downstream events from `executorName()` strings to full `ExecutorRef`. The shared Plan type is deferred to #694 (DAG plan structure) — it needs these task-level types as its foundation.
