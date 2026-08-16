---
layout: post
title: "When the Strategy Resolver Can't See Your Strategy"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [quarkus, cdi, goal-formation, agent-learning]
---

The final piece of the agent learning sub-epic landed today — goal formation. Agents can now discover new goals from accumulated experience, completing the lifecycle: goals are declared, revised based on outcomes, abandoned when infeasible, and now formed when reflection surfaces new objectives.

The mechanism is straightforward. After a reflection cycle produces insights, the `GoalFormationEvaluator` passes them — along with the agent's current goals and recent memories — to an LLM strategy. The strategy proposes new goals; the evaluator validates them (name length, description length, no duplicates, capacity check) and registers them on the agent's descriptor. A config-based approval gate lets production deployments log proposals without auto-registering.

The interesting part wasn't the design. It was what went wrong testing it.

## The invisible strategy

I wrote the integration test, started cases, waited for the formation pipeline to fire. Nothing happened. No errors, no logs, no registrations. The evaluator's `evaluate()` method has a virtual thread that catches all exceptions and logs at WARN — but no WARN appeared either.

Adding debug traces to each guard condition revealed: `Instance<AgentRegistry>.isResolvable()` was returning `false`. The `AgentRegistry` implementation existed as an inner test bean. It was `@ApplicationScoped`. It compiled. The direct injection worked. But `Instance<>` couldn't see it.

Two separate CDI problems were stacked on top of each other.

**Problem one:** the `EngineStrategyResolver` uses a catch-all `@Any Instance<NamedStrategy>` constructor parameter to discover strategies not explicitly listed. Quarkus ARC's build-time pruning doesn't reliably discover beans whose only `NamedStrategy` relationship is through a sub-interface chain. `LlmGoalFormationStrategy implements GoalFormationStrategy extends NamedStrategy` — invisible. The fix: add explicit `@Any Instance<GoalFormationStrategy>` as a constructor parameter. The catch-all had been silently broken for any SPI added after the resolver was written — it just happened that all previous SPIs had their own explicit parameters.

**Problem two:** two `@QuarkusTest` classes each defined an inner `@ApplicationScoped TestAgentRegistry implements AgentRegistry`. Quarkus discovers ALL test inner classes globally, regardless of which test profile is running. Two beans of the same type without `@Alternative` creates CDI ambiguity. `Instance<T>.isResolvable()` returns `false` on ambiguity — not just on absence. The symptom is identical: "nothing happens." The fix: `@Alternative @Priority(1)` on the beans that need to win.

Both problems share the same failure mode: silent no-op. The code returns early from a guard condition, the virtual thread never fires, and the catch block never runs because there's nothing to catch. You can stare at the code for an hour without finding it, because the code is correct — the CDI wiring around it isn't.

## What the sub-epic delivered

Four issues, three sessions. Plan adaptation — revise active plans based on worker completions. Goal revision — adjust priority and description from outcome signals. Goal formation — discover new goals from reflection insights. Goal discovery from memory — merged into the formation design since the memory retrieval was already part of the formation context.

The formation evaluator follows the same pattern as revision: `Instance<>` injection with `isResolvable()` guards, virtual thread for async work, `ConcurrentHashMap` for cooldown state, per-goal error isolation. The LLM strategy is identical in shape to `LlmGoalRevisionStrategy` — `Agent.builder()` with `ChatModelProvider`, structured JSON response parsing.

What makes goal formation different from revision is the trigger. Revision fires from accumulated outcome signals — a statistical threshold. Formation fires from reflection insights — synthesized observations about patterns across outcomes. The insights are the LLM's contribution; the validation and registration are the engine's.
