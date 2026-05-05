---
layout: post
title: "Wanted a Sketch, Got a Framework"
date: 2026-03-27
tags: [day-zero, architecture]
excerpt: "One session, 73 files, 14,003 lines of code — what started as a request for a sketch became a working framework."
---

# Wanted a Sketch, Got a Framework

**Date:** 2026-03-27
**Type:** day-zero

---

## The assignment I gave my colleague

I'd asked a colleague to look at case management in Kogito — the jBPM and SonataFlow family — specifically around flexible processes and CMMN. The KIE blog has a reasonable series on it. I wanted him to bring in choreography more strongly, and to take a fresh look at CMMN in the context of what we were building.

One instruction I was clear about: don't make it process-centric. Kogito's case management is good work, but it sits inside a process engine. Everything is framed through that lens. Case Management was never successful in the BPM world partly for this reason — business users couldn't shift their mental model away from process-centric thinking, even when the underlying concepts were a better fit for their problem. The Blackboard pattern and CMMN aren't process-centric. For Agentic AI — where you have multiple specialists reasoning over shared context, not a sequence of steps executing in order — that distinction matters. Approaching it with fresh eyes through a process-centric frame would be misleading.

## What I hadn't yet articulated

Here's what I didn't tell him, partly because I hadn't fully worked it out myself: the connection between the Blackboard Architecture and CMMN is closer than it first appears. Hayes-Roth's 1985 Blackboard model — shared workspace, independent knowledge sources, a control component — maps almost directly onto CMMN's CaseFile, TaskDefinitions, and PlanningStrategy. The terminology is different; the structure is largely the same.

I had a sense of this. I hadn't verified it. And I certainly hadn't investigated whether any standalone Java implementation of it existed.

## The gap that made it worth building

Before sketching anything, I'd done some investigation with Claude into what already existed. The answer was: not much. No standalone Java Blackboard implementation that wasn't already embedded inside a larger platform. Everything found was either a research prototype, tightly coupled to a specific framework, or buried inside something bringing far more than you'd want.

I wanted something different. Standalone — not a platform, no mandatory infrastructure. Quarkus-native, able to run lean and go native. And designed to work well with the ecosystem we already use: LangChain4J for AI agents, Quarkus Flow for workflow execution, Drools for rule-based reasoning. A Blackboard coordinator that integrated cleanly with all three would be genuinely useful.

## The specific thing I wanted to go deep on

Distributed context propagation. How do you track lineage across a hierarchy of cases and tasks? How does a budget flow from a parent case into child cases? How do you get W3C-compatible trace IDs threaded through everything without coupling every component to a tracing framework?

The plan: sketch the broader architecture, go deep on that one specific part. Something concrete enough that my colleague could react to.

## What Claude did with that

I started a session. Described the concept, the Blackboard pattern, what I was trying to show. I expected scaffolding. Something to react to.

Claude went further. It didn't just sketch the control loop — it built it. `CaseEngine`, `CasePlanModel`, `PlanItem`, `PlanningStrategy`. Then `CaseFile` with per-key versioning and change listeners. Then `TaskDefinition` with DFS cycle detection to prevent circular dependencies at registration time. Then a full resilience layer: `RetryPolicy`, `PoisonPillDetector`, `DeadLetterQueue`, `IdempotencyService`, `TimeoutEnforcer`, `ConflictResolver`. Then `TaskBroker`, `TaskScheduler`, `WorkerRegistry`. Two working example applications. LLM worker integration.

And the distributed context — `PropagationContext` carrying a W3C trace ID, inherited attributes, a deadline, a remaining budget. Exactly what I'd intended to sketch deeply. More complete than I would have produced alone.

A 2,400-line design document appeared alongside it. Working through it, I could see the Blackboard/CMMN overlap laid out in front of me — more clearly than I'd understood it before starting.

I hadn't asked for most of this. I didn't want to stop it.

## 73 files, 14,003 lines, 2:33am

[![Side project scope creep — CommitStrip](https://www.commitstrip.com/wp-content/uploads/2014/11/Strip-Side-project-650-finalenglish.jpg)](https://www.commitstrip.com/en/2014/11/25/west-side-project-story/)

One session. I'd gone in wanting to clarify my own thinking and give my colleague something concrete. What came out was a framework — and a clearer view of the architectural territory than I'd had when I started.

I let it run because I wanted to see how far it would go. The subsequent sessions would keep answering that question.
