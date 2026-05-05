---
layout: post
title: "The Architecture Behind CaseHub: Blackboard Meets CMMN"
date: 2026-03-28
tags: [architecture, cmmn]
excerpt: "Two patterns from very different traditions — Blackboard Architecture and CMMN — and why they belong together for agentic AI."
---

# The Architecture Behind CaseHub: Blackboard Meets CMMN

**Date:** 2026-03-28
**Type:** phase-update

---

## Two patterns, one problem

CaseHub is built on two ideas from very different traditions. Understanding why they belong together is the foundation for understanding every design decision that followed.

**The Blackboard Architecture** comes from AI research — Hayes-Roth, 1985. The original application was speech understanding: a hard problem where no single algorithm could solve the whole thing, but a collection of specialists working on the same data could converge on a solution. The model is simple. There is a shared workspace — the Blackboard — where all state lives. Independent Knowledge Sources observe the Blackboard and write back when they have something to contribute. A control component evaluates the current state and decides which Knowledge Source should fire next.

No Knowledge Source communicates directly with another. They only read and write the shared workspace. The control loop is the intelligence.

[![Blackboard Architecture — three components: knowledge sources, blackboard, control](https://upload.wikimedia.org/wikipedia/commons/1/18/Blackboad_pattern_system_structure.png)](https://en.wikipedia.org/wiki/Blackboard_(design_pattern))

**CMMN — Case Management Model and Notation** — comes from the business standards world, OMG 2014. A Case is work that is dynamic and unpredictable: you can't define the path in advance because it depends on what you find. CMMN gives you vocabulary for that: a CaseFile as the shared data model, Plan Items as work that activates when conditions are met, Stages as containers grouping related work with their own lifecycle, and Milestones as named achievement markers that signal meaningful progress without themselves doing any work.

[![CMMN diagram — CaseFile, Stages, Plan Items, Milestones](https://cdn-images.visual-paradigm.com/guide/cmmn/what-is-cmmn/02-cmmn-diagram-example.png)](https://www.visual-paradigm.com/guide/cmmn/what-is-cmmn/)

## Where they overlap

The structural similarity is closer than it first appears.

| Blackboard | CMMN |
|---|---|
| Blackboard (shared workspace) | CaseFile |
| Knowledge Source | TaskDefinition / Plan Item |
| Control Component | Sentry evaluation + Stage lifecycle |
| Activation condition | Entry criteria |

Both were designed for problems where the execution path is not predetermined. Both treat the shared state as the communication medium. Both support emergent behaviour: you define the specialists and the conditions, not the sequence.

## Where they differ

Blackboard is rooted in AI. Its central concept is the **control loop** — a reasoner that looks at current state and makes a deliberate decision about what to fire next. That control component can be sophisticated: it can prioritise, focus, reason about resources, apply pluggable strategy. That is where the intelligence lives.

CMMN is rooted in knowledge work. Its central concepts are the **lifecycle concepts** — Stages with entry and exit criteria, Milestones as progress markers, the sentry model activating plan items declaratively. It is richer in vocabulary and lifecycle expressiveness. But weaker in control reasoning. A CMMN implementation tells you what can happen; it doesn't tell you what should happen next.

## Orchestration, choreography, and why Agentic AI needs both

This is one of the most important distinctions in distributed systems — and most frameworks force you to choose.

| | Orchestration | Choreography |
|---|---|---|
| Control | Central coordinator directs workers | Workers observe and self-trigger |
| Analogy | Orchestra conductor | Ballet dancers following shared rules |
| Strength | Predictable, deliberate, easy to reason about | Scalable, resilient, no single point of control |
| Weakness | Brittle for autonomous agents | Hard to reason about globally |

[![Orchestration — a conductor directs the orchestra](https://camunda.com/wp-content/uploads/2023/01/orchestra-with-conductor.png)](https://camunda.com/blog/2023/02/orchestration-vs-choreography/)
[![Choreography — dancers follow shared rules without a conductor](https://camunda.com/wp-content/uploads/2023/01/Choreography.png)](https://camunda.com/blog/2023/02/orchestration-vs-choreography/)

For Agentic AI, neither alone is right.

Some agents need to be deliberately invoked at precisely the right moment — a reasoning agent that should only run after specific evidence has accumulated, or a decision agent that needs the full current state before acting. Those need orchestration. You want the `PlanningStrategy` to decide when they fire.

Others need to operate autonomously — a monitoring agent that watches an external system and reacts when it sees something significant, a continuous observation agent that runs throughout a case without being explicitly triggered. Those need choreography. You want them to self-initiate based on what they observe.

A real Agentic AI system has both. CaseHub supports both — in the same case, simultaneously.

**Directly orchestrated workers** are fired by the `PlanningStrategy`. It reads the `CaseFile`, evaluates the current state, and decides which worker should run next. The worker executes, writes its result back to the `CaseFile`, and the loop re-evaluates.

**Observed choreography** works differently. Workers observe the `CaseFile` via binding conditions that self-trigger when their conditions are met. Autonomous workers monitor external systems independently and write their findings directly to the `CaseFile` when something warrants attention. The engine observes all of this — it doesn't initiate it.

[![Hybrid approach — combining orchestration and choreography](https://camunda.com/wp-content/uploads/2023/01/Mixture-Choreography-Orchestration.png)](https://camunda.com/blog/2023/02/orchestration-vs-choreography/)

The blend is the point. A `PlanningStrategy` can make deliberate decisions about which analysis agents to invoke, while autonomous monitoring workers run continuously writing observations. Both contribute to the same `CaseFile`. Both participate in the same case. The `PlanningStrategy` can factor in what the autonomous workers have written when making its next decision.

## Why the Blackboard/CMMN combination enables this

The `PlanningStrategy` is the orchestration mechanism — sits at the heart of the Blackboard control loop, reasons about what should run next.

The `CaseFile` with its change listeners and condition evaluation is the choreography mechanism. Workers that observe state and self-trigger don't need to know the `PlanningStrategy` exists. They just write to the shared workspace.

CMMN's Stages and Milestones give both modes structural context. A Stage can scope which orchestrated workers are eligible. A Milestone can signal choreographed workers that were waiting for a particular condition. The lifecycle vocabulary serves both execution models.

## Where process engines fall short

Process engines — BPMN — are pure orchestration. The path is declared upfront. Agents are steps in a sequence. Handling genuinely autonomous behaviour requires bolting exception flows onto a fundamentally sequential model. It doesn't compose.

Case Management attempted to fix this within the BPM world. CMMN was closer — dynamic, condition-driven, non-sequential. But sitting inside process engines, carrying their frame, it never escaped the process-centric mental model. Business users couldn't shift. The concepts were right; the context was wrong.

For Agentic AI, there is no prior process model to unlearn. The Blackboard/CMMN combination can be approached on its own terms.

## What session 2 added to this foundation

Session 1 had produced a working Blackboard skeleton with the control loop and distributed context propagation. What it didn't have was the CMMN lifecycle machinery.

**Stages** — containers with entry and exit criteria, nested support, autocomplete when required items complete. Full lifecycle: `PENDING`, `ACTIVE`, `SUSPENDED`, `COMPLETED`, `TERMINATED`, `FAULTED`. A Stage doesn't execute — it creates a bounded context in which work can execute, whether orchestrated or choreographed.

**Milestones** — named achievement markers. Criteria: key presence on the `CaseFile`. Status: `PENDING` or `ACHIEVED`. No execution. The signal that a meaningful condition became true — from work completing, an external event arriving, or multiple things converging.

**Storage as a proper SPI** — `CaseFileStorageProvider`, `TaskStorageProvider`, `PropagationStorageProvider` extracted as interfaces. In-memory implementations in a separate module. Fast, no infrastructure, reset between test runs.

**The first Quarkus ecosystem connection** — `FlowWorker`, a bridge between casehub's task model and Quarkus Flow. It polls `WorkerRegistry` for tasks, looks up the corresponding workflow in `FlowWorkflowRegistry`, executes via `FlowExecutionContext`, and submits the result back. The pattern established: casehub coordinates, Quarkus Flow executes.

## End of session 2

Two sessions in. A Blackboard control loop with CMMN stage and milestone lifecycle, hybrid orchestration and choreography, pluggable storage, and a first connection to the Quarkus ecosystem. Session 3 would be about getting the architecture right rather than adding more.
