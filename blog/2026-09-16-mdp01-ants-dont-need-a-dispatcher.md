---
entry_type: note
subtype: diary
title: "Ants don't need a dispatcher"
series: issue-1104-hive-mind
projects: [casehubio/engine]
author: mdp
date: 2026-09-16
tags: [hive-mind, stigmergy, observation, spi, architecture]
---

CaseHub's dispatch model is reactive — a JQ expression evaluates to true, a binding fires, a worker runs. It's clean, it's predictable, and it's exactly wrong for self-organizing agents.

An ant doesn't wait for a dispatcher to tell it there's food nearby. It observes the environment — pheromone gradients, trail density, the behaviour of nearby ants — and makes its own decision about what to do next. The perception IS the coordination mechanism. That's what stigmergy means: coordination through the environment, not through a central controller.

Today I built the perception layer that makes this possible in CaseHub.

## The gap

The engine already has `ContextChangeTrigger` — a static JQ filter on each binding that evaluates on every context change. Boolean result: fire or don't. Stateless, no memory of what happened before. Good for reactive dispatch ("when the transaction amount exceeds the threshold, run the AML check"). Useless for perception ("the fraud score has been climbing for the last three evaluations while the entity resolution confidence is dropping — something's wrong").

What agents need for stigmergy is richer: multi-key correlation, temporal sequences, threshold crossings. And each agent needs its own view — two agents observing the same context may perceive different things based on their own state and goals.

## The design

`EnvironmentObserver` is the SPI. An observer declares what keys it watches, receives the current context snapshot plus a sliding window of recent history, and returns structured `Observation` results. Not a boolean dispatch decision — a perception: what pattern was detected, at what confidence, with what details.

```java
public interface EnvironmentObserver {
  String observerType();
  Set<String> watchedKeys();
  List<Observation> observe(ObservationContext ctx);
}
```

The interface is deliberately minimal. Classical implementations in the engine handle the mechanical patterns — threshold crossing, multi-key JQ correlation, temporal sequence detection. Blocks will provide LLM-backed observers later that can do semantic perception. The engine provides the eyes; blocks provides the interpretation.

Registration happens through `WorkerRuntime` — agents register observers during execution, scoped to their binding's lifecycle. A CASE-scoped observer lives for the entire case; a COMPOUND-scoped one dies when the compound completes. BINDING-scoped registration is rejected outright — there's no cleanup event for BINDING scope, so the observer would leak.

## Where it sits in the pipeline

Observation runs after `rules()` and `goals()` inside the `CaseEvaluationSerializer` gate — the same per-case lock that prevents concurrent dispatch evaluation. Each observer gets a 100ms timeout via `CompletableFuture.orTimeout()`. Key filtering skips observers whose watched keys don't overlap with what changed. Results go into an in-memory `ObservationRegistry`, not back into the `CaseContext` — writing observations to the context would trigger another `CONTEXT_CHANGED`, creating a feedback loop.

The one-cycle delay is intentional. Observations from cycle N inform local rules in cycle N+1. No circular dependency between dispatch and perception.

## The review caught real issues

The decision review surfaced the dependency direction problem I'd missed: `WorkerScope` (worker-api) can't reference `EnvironmentObserver` (engine-api) because the dependency flows the wrong way. Registration goes through `WorkerRuntime` instead — which already lives in engine-api. The spec review caught that `CompletableFuture.orTimeout()` doesn't actually interrupt the observer thread — it bounds the pipeline delay but the virtual thread continues running. For classical observers that complete in microseconds this is fine, but the spec now documents the real semantics instead of claiming interruption.

## What this opens up

The observation SPI is the foundation for the next three issues in the Hive Mind epic. Issue #1106 (signals/pheromones) gives agents something meaningful to observe — temporal signals with decay that mimic ant pheromones. Issue #1109 (local rules) consumes observations and produces decisions — the per-agent decision layer. Issue #1111 (stigmergy execution model) ties it all together into a coordination pattern where agents self-organize through the shared environment.

No competing framework does this. CrewAI, AutoGen, LangGraph — they all use predefined roles and handoff chains. CaseHub's observation layer lets agents discover what to do by watching what's happening around them, the same way ants build highways without a highway authority.
