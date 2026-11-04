---
layout: post
title: "Six Topologies, One Primitive"
date: 2026-11-03
entry_type: article
subtype: diary
projects: [casehubio/agency]
tags: [coordination, topology, spi-design, multi-agent, decision-loop, graduated-autonomy, architecture]
series: issue-377-distributed-per-agent-coordination
---

# Six Topologies, One Primitive

Continued from [The Playbook That Replaces the Coordinator](2026-11-02-mdp04-the-playbook-that-replaces-the-coordinator.md).

The previous entry ended with a claim: topology is a factory for descriptors and an executor for outcomes. The intelligence stays in the loop; the coordination pattern stays in the playbook. That's a nice architectural sentence. Whether it holds up depends on whether you can actually build six different coordination strategies from the same primitive without any of them fighting the abstraction.

## The SPI that does almost nothing

The `CoordinationTopology` interface has three methods beyond the inherited `describeTopology()`:

```java
public interface CoordinationTopology extends TopologyProvider {
    CoordinationPlan configure(TopologyContext context);
    void onSwarmEvent(SwarmCoordinationEvent event);
    void onFeedback(CoordinationFeedback feedback);
}
```

`configure()` returns a `CoordinationPlan` — a list of `DecisionLoopDescriptor` playbooks plus a `DecisionOutcomeExecutor`. The topology says "create these loops" and "execute their outcomes like this." It doesn't run the loops, doesn't manage their lifecycle, doesn't subscribe to their output. All of that is the orchestration layer's concern.

The interesting design question was what goes into `TopologyContext`. The first instinct is to stuff everything in — swarm state, group hierarchy, graduation history, the full situational picture. But different topologies need different things. A distributed topology doesn't care about swarm progress. A graduated topology needs feedback history that no other topology uses. A hive topology needs to know which orchestration pattern to apply.

The answer was to keep TopologyContext universal — five fields that every topology needs — and constructor-inject the rest. `CentralisedTopology` takes a `SwarmContext` in its constructor. `GraduatedTopology` takes a `GraduationState`. `HierarchicalTopology` takes a parent map. The per-invocation context stays clean; the per-instance dependencies are type-safe and explicit.

```java
public record TopologyContext(
    UUID caseId,
    TopologyDescriptor descriptor,
    AgentProvider agentProvider,
    DecisionLoopFactory loopFactory,
    SwarmInvocationPolicy invocationPolicy
) { }
```

No `Optional<SwarmContext>`. No `Map<Class<?>, Object>` attribute bag. If a topology needs something, its constructor says so.

## Six ways to coordinate, same code shape

Each topology's `configure()` method tells the same story in different words: how many loops, what scope, what observation strategy, what communication mode. The differences are configuration, not structure.

| Topology | Loops | Scope | Key idea |
|----------|-------|-------|----------|
| Centralised | 3 (team, reasoning, provisioning) | FULL_AUTONOMY | The existing LlmSwarmCoordinator, decomposed into three separate decision loops |
| Distributed | N per-agent | FULL_AUTONOMY | Every agent decides independently via stigmergy |
| Hybrid | 1 global + N local | FULL / TASK_ADAPTATION | Global coordinator sets direction; local agents adapt within constraints |
| Hierarchical | Tree of loops | FULL → TASK → STIMULUS | Authority narrows as you go deeper in the tree |
| Graduated | N per-agent, mutable | Dynamic via GraduationState | Agents earn broader scope through sustained good decisions |
| Hive | 1 coordinator + N workers | FULL / TASK_ADAPTATION | Composes existing orchestration patterns (supervisor, debate, voting) |

The centralised topology is the one that validates the SPI. It's a refactoring of `LlmSwarmCoordinator` — the same routing, aggregation, and provisioning responsibilities, but decomposed into three separate loops instead of one monolithic coordinator class. If the SPI can't express this topology without fighting, the SPI is wrong. It can. The three loops map directly to three `DecisionLoopDescriptor` instances with different names and the same prompt sections.

## The graduated autonomy problem

Most topologies are static — the configuration is set at `configure()` time and doesn't change. Graduated topology breaks this. Agents start with narrow scope (`STIMULUS_DRIVEN`) and earn broader scope (`TASK_ADAPTATION`, then `FULL_AUTONOMY`) through sustained successful decisions. Scope violations cause immediate demotion.

The `DecisionScope` enum is immutable. I didn't want to change that — every other consumer of scope treats it as a fixed property. The graduated topology needs scope to move, but the model type shouldn't become mutable to accommodate one topology's needs.

`GraduationState` wraps the immutable enum in a mutable container:

```java
public class GraduationState {
    private final Map<String, DecisionScope> agentScopes;
    private final Map<String, List<FeedbackSignal>> recentSignals;
    private final int windowSize;
    // ...
}
```

It's a windowed feedback accumulator. Each `FeedbackSignal` (SUCCESS, PARTIAL, FAILURE, SCOPE_VIOLATION) goes into a sliding window. When 80% of the window is successes, scope upgrades one level. When 50% is failures, it downgrades. A single SCOPE_VIOLATION causes an immediate hard downgrade — no averaging, no window, straight down.

The topology's `configure()` reads the current scope from `GraduationState` for each agent. When `onFeedback()` adjusts a scope, the next `configure()` call returns a plan with the updated scope. The loop rebuilds its `DecisionContext` with the new scope, which changes what decisions it's permitted to make. The feedback loop closes: act → feedback → scope adjustment → new permissions → act differently.

This is the design I'm most interested in watching under real load. The thresholds (80% success, 50% failure, window size) are tuneable but arbitrary. Whether graduated autonomy produces better agent behaviour than just picking the right fixed scope remains an open empirical question.

## Composing what already exists

HiveTopology was the test of whether the SPI composes with existing code or requires parallel infrastructure. The agency already has three orchestration patterns — supervisor, debate, and voting — implemented as builders that produce their own coordination types. HiveTopology needs to wire these patterns into the `CoordinationTopology` SPI without reimplementing them.

`HivePatternAdapter` bridges the gap. It takes a `DecisionOutcome` from a loop and translates it into the appropriate pattern's execution model — supervisor routing for `Route` outcomes, debate synthesis for `Aggregate` outcomes. The topology's executor delegates to the adapter:

```java
DecisionOutcomeExecutor executor = (agentId, outcome, ctx) ->
    patternAdapter.execute(patternType, agentId, outcome, ctx);
```

The orchestration pattern (supervisor vs debate vs voting) is a `HiveTopology` configuration parameter, not a structural difference. Three different hive groups can use three different patterns. The topology configures loops for the coordinator and workers; the adapter handles the pattern-specific execution.

## What the topology doesn't do

The topology is not a coordinator. It doesn't hold conversation state, doesn't manage agent lifecycles, doesn't track which agents are busy. It's a factory that produces a recipe (the plan) and an executor that applies decisions. The intelligence — observing the environment, reasoning about what to do, producing decisions — stays in the `AgentDecisionLoop`. The coordination pattern — which loops exist, what scope they have, how their outputs are applied — stays in the `CoordinationPlan`.

This separation is load-bearing. When the topology is a factory, changing coordination strategy is a configuration change. When the topology is a coordinator, changing strategy is a rewrite.
