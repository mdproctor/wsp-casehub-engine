---
layout: post
title: "GOAP in a Case Engine: What Six Integration Tests Teach About Planning Under Uncertainty"
date: 2026-08-22
entry_type: article
subtype: diary
projects: [casehub-engine]
tags: [goap, planning, adaptive, integration-testing, a-star]
---

# GOAP in a Case Engine: What Six Integration Tests Teach About Planning Under Uncertainty

A case engine that dispatches workers based on trigger conditions doesn't need a planner. Bindings fire when their conditions are met. Context changes. More bindings fire. The choreography strategy handles this — it's reactive, event-driven, and sufficient for most case types.

But some case types have goals. Not "execute this binding when X is true" but "reach a state where Y holds, and figure out the path." That's where GOAP comes in.

## What GOAP Actually Does

GOAP — Goal-Oriented Action Planning — is an AI planning technique from game development. The original idea (Jeff Orkin, 2003) gave NPCs the ability to reason about how to achieve goals rather than following scripted behaviour trees. An agent that needs to kill a target doesn't follow a fixed sequence; it plans backward from the goal state, considering what actions are available and what preconditions each requires.

In a case engine, the translation is direct. Workers are actions. Each declares preconditions (what must be true in the context before it can run) and effects (what it produces). The goal is a set of context keys that must be true for the case to complete. The GOAP planner runs A* forward search to find the cheapest sequence of workers that reaches the goal from the current world state.

```java
CaseDefinition.builder()
    .planningStrategy("goap")
    .goapActions(List.of(
        new GoapAction("analyse", Map.of(), Map.of("analysisResult", true), 1.0),
        new GoapAction("assess", Map.of("analysisResult", true),
                       Map.of("riskAssessment", true), 1.0),
        new GoapAction("report", Map.of("riskAssessment", true),
                       Map.of("reportDone", true), 1.0)))
    .goalToEffectKey("done", Set.of("reportDone"))
```

Three actions. `analyse` has no preconditions — it can run immediately. `assess` needs `analysisResult`. `report` needs `riskAssessment`. The planner finds: analyse → assess → report. Declare them in any order in the definition — the planner doesn't care about declaration order. It cares about precondition chains.

## The Dispatch Loop

The GOAP strategy sits inside `CompoundStrategyDispatcher`, which calls it on every context-change cycle. The strategy builds a world state from the case's working layer (keys present = true, keys absent = false), resolves the goal conditions, runs A*, and returns one binding — the first step of the plan.

That binding executes. Its worker writes output to the context. Context changes. The dispatcher calls the strategy again. New world state, same goal, new plan. The first step of *that* plan fires next.

This is the critical insight: **GOAP doesn't produce a plan and execute it start to finish.** It plans from scratch on every cycle. The plan is ephemeral — it exists only long enough to select the next action. By the time that action completes and the world state evolves, the next plan might be completely different.

## Adaptive Replanning: When the World Surprises the Planner

The basic GOAP strategy replans from the current world state each cycle. The adaptive strategy (`AdaptivePlanningStrategy`) does the same thing but with a deliberate difference: it accounts for the fact that workers can produce output the planner didn't predict.

Consider three workers:
- `analyse` — declared effect: `{analysisResult: true}`. Actual output: `{analysisResult: true, fastPath: true}`
- `full-resolve` — precondition: `{analysisResult: true}`, cost: 2.0
- `fast-resolve` — preconditions: `{analysisResult: true, fastPath: true}`, cost: 0.5

The planner's initial plan is analyse → full-resolve. It doesn't know about `fastPath` because `analyse` doesn't declare it as an effect — GOAP plans from declared preconditions and effects, not from actual worker output. But when `analyse` runs, it writes `fastPath: true` to the context. The adaptive strategy replans from the actual world state. Now `fast-resolve` is applicable (both preconditions met) and cheaper. The new plan is just `fast-resolve`.

The execution trace: analyse, then fast-resolve. Never full-resolve. The planner adapted because the world state diverged from the declared effects — the worker produced more than it promised.

This matters for real case types. A medical triage that discovers the patient already has recent imaging doesn't need to order new scans. A compliance check that finds the entity is pre-approved skips the full assessment path. The adaptive strategy picks up these shortcuts automatically, without the case definition author having to enumerate every possible fast path.

## What Trigger Guards Do to Planning

Case bindings have trigger conditions — JQ expressions evaluated against the context that determine whether a binding is eligible. The GOAP planner only sees eligible bindings. This creates an interaction worth understanding.

If binding B has a trigger guard `.ready == true` and `ready` isn't in the initial context, B won't be in the eligible set. The GOAP planner can only plan with what it sees. If the goal requires B's effects, and B isn't eligible, the planner can't find a path to the goal. It returns empty. Nothing dispatches.

This is a fundamental property of A* forward search: the planner requires all actions needed for a complete path to be available at planning time. It doesn't do partial planning — "I can get partway there, so let me start." It either finds a complete solution or returns nothing.

For case definitions that use GOAP, this means: either all GOAP-planned bindings share the same trigger (so they're always eligible together), or the goal conditions match what the currently-eligible actions can achieve. Mixing trigger-based gating with GOAP-based planning creates an impedance mismatch the planner can't resolve.

The right pattern is to use GOAP preconditions — not trigger guards — for ordering. Preconditions are part of the planner's model; trigger guards are invisible to it.

## Cost-Optimal Path Selection

When two actions can reach the same goal, the planner picks the cheaper one. This is A* doing what A* does — but in a case engine, it has practical implications.

```java
new GoapAction("cheap-path", Map.of(), Map.of("done", true), 0.5)
new GoapAction("expensive-path", Map.of(), Map.of("done", true), 5.0)
```

Both produce `{done: true}`. The planner selects `cheap-path` (cost 0.5) over `expensive-path` (cost 5.0). In a case with learned costs from CBR traces, these cost values update based on historical success rates — an action that fails frequently gets a higher effective cost, steering the planner toward more reliable alternatives.

## The Case That Waits

Not every starting context has a viable path to the goal. A case might start with insufficient data — the preconditions for the only available action aren't met. The GOAP strategy returns empty. No workers dispatch. The case stays RUNNING.

Then a signal arrives. An external system pushes data into the context. The trigger fires, the strategy runs, and now the precondition is met. The planner finds a path. The worker dispatches.

This isn't a failure mode — it's intentional. The case is waiting for the world to provide what it needs. The planner tells you there's nothing productive to do yet, and it's right.

## Failure and Reroute

When a worker fails, the engine's reroute mechanism kicks in: the failed agent is excluded, and the binding re-fires with a different agent. GOAP doesn't handle this — agent selection is a routing concern, not a planning concern. The planner picks the right *action* (which binding to fire); the routing strategy picks the right *agent* (which worker handles it).

This separation matters. The planner's job is to find the cheapest path through precondition chains. The router's job is to find a capable, available agent for each step. A failed agent doesn't invalidate the plan — it just means a different agent needs to handle that step.

## What This Means for Case Definitions

GOAP planning turns a case definition from a reactive choreography into a goal-directed system. Instead of wiring bindings with trigger conditions that implicitly sequence the work, you declare preconditions and effects and let the planner figure out the ordering. The cost model steers it toward efficient paths. The adaptive strategy handles real-world surprises.

But the planner is a tool with specific assumptions: the action space is fully known, the world state is observable, and effects are deterministic. When those assumptions hold — and for most structured case types, they do — GOAP replaces manual sequencing with automatic planning. When they don't — dynamic eligibility, non-deterministic effects, partial observability — the choreography strategy is the right default.

The choice isn't GOAP or choreography universally. It's which case types benefit from goal-directed planning, and which are better served by reactive dispatch. The integration tests verify that the engine handles both correctly through the same dispatch pipeline.
