---
layout: post
title: "Four Things a Synchronous Blackboard Strategy Can't Do"
date: 2026-04-18
entry_type: article
projects: [casehub]
tags: [architecture, blackboard, reactive, vertx, multi-agent]
---

The blackboard architecture is 40 years old. Hayes-Roth described it in 1985
for HEARSAY-II speech recognition. The idea — a shared workspace where
independent knowledge sources self-organise around a problem, activated
opportunistically as data accumulates — is genuinely elegant, and it is
having a significant revival in multi-agent LLM systems.

But the canonical implementation has a problem nobody talks about, because
in 1985 it didn't matter: the control shell is synchronous.

In Hayes-Roth's model, the control shell evaluates which knowledge sources
are eligible, selects among them, and fires the winner. That selection is a
function call. It blocks. It returns. The rest of the system waits.

For most of the past four decades, that was fine. Today, if your knowledge
sources are LLM calls, if the control shell wants to consult historical
execution data before selecting, or if you're running on a reactive event
loop where blocking is a correctness violation — the synchronous shell is a
ceiling, not a foundation.

## What the Synchronous Shell Actually Prevents

Consider a planning strategy that makes a non-trivial selection decision. It
wants to look at the last five executions of this case type, check which
knowledge sources produced high-quality results, and bias the current
selection toward those. Reasonable. Useful. Exactly the kind of learned
control Hayes-Roth was gesturing at with the concept of "meta-level knowledge
sources."

In a synchronous shell, the strategy receives the current blackboard state
and returns a ranked list. That is all it has access to. The interesting data
— the execution history, confidence signals, external context — is in an
event log, which requires an I/O call. A synchronous strategy cannot make
that call without blocking the event loop. On Vert.x, blocking the event
loop is not a performance problem — it is a correctness failure.

The fix requires changing the LoopControl interface from a blocking contract
to a reactive one:

```java
// Classical — selection is synchronous, blocking
List<Binding> select(PlanExecutionContext context, List<Binding> eligible);

// Reactive — selection is a Uni, fully non-blocking
Uni<List<Binding>> select(PlanExecutionContext context, List<Binding> eligible);
```

The default `ChoreographyLoopControl` wraps its result in
`Uni.createFrom().item(eligible)` — no behaviour change, zero cost. But
`PlanningStrategyLoopControl` can now delegate to a strategy whose
`select()` method chains EventLog queries, calls an LLM scorer, or checks
any external system — all without blocking:

```java
public Uni<List<Binding>> select(PlanExecutionContext ctx, List<Binding> eligible) {
    CasePlanModel plan = getOrCreate(ctx.getCaseId());
    return planningStrategy.select(plan, ctx, eligible);  // fully async
}
```

That one interface change enables four things the synchronous model cannot do.

## Four Concrete Improvements

**1. Strategies that reason over history.** An async `PlanningStrategy` can
query the EventLog — which knowledge sources fired recently, which produced
valuable outputs, which ran and left the blackboard unchanged. Selection
based on observed outcomes rather than declared priority. This is what
Hayes-Roth meant by control knowledge sources learning from the problem-solving
record; the synchronous shell just made it impractical.

**2. Stage lifecycle as first-class events.** In the classical model, stage
activation and completion are internal state mutations — the control shell
updates its own data structures and moves on. In a reactive model, stage
transitions are published onto the event bus:

```java
eventBus.publish(STAGE_ACTIVATED, new StageActivatedEvent(caseId, stage));
eventBus.publish(STAGE_COMPLETED, new StageCompletedEvent(caseId, stage));
```

Other components — observability, lineage trackers, dashboards — can react
without coupling. The stage lifecycle becomes auditable and hookable without
touching the engine internals.

**3. PlanItem completion tracking without polling.** Worker execution
completion in casehub-engine is published via `eventBus.publish()` — a
fan-out, not a point-to-point send. The blackboard module adds a second
consumer on the same address. When a worker finishes, the `CasePlanModel`
updates immediately: the corresponding `PlanItem` is marked `COMPLETED`,
Stage autocomplete criteria are re-evaluated. No polling loop, no tight
coupling.

**4. Separation of domain state from control state.** The `CasePlanModel`
is Hayes-Roth's "control blackboard" — per-case, keyed by UUID, holding the
scheduling agenda, current focus of attention, resource budget, and stage
tracking. Strategies read and write it. The engine reads the selection result.
Domain state (`CaseContext`) and control state (`CasePlanModel`) never mix.
This is the architecture Hayes-Roth described; the synchronous shell made it
a formality rather than a functional separation.

## What Recent Research Confirms

The academic literature is catching up to these architectural directions.
A 2024 paper (arXiv [2510.01285](https://arxiv.org/abs/2510.01285))
demonstrates LLM-based blackboard multi-agent systems outperforming baseline
static frameworks on five of six benchmarks, attributing the gains explicitly
to dynamic agent selection and shared memory. A companion paper
([2507.01701](https://arxiv.org/abs/2507.01701)) shows the performance and
cost improvements come from two sources: the shared memory pool enabling
comprehensive information exchange, and dynamic agent selection ensuring only
suitable agents are activated for each blackboard state.

Both depend on control reasoning that is more than a priority sort over a
static list. The async shell is the mechanism that makes richer control
reasoning tractable.

## Opt-In Orchestration

The practical consequence for casehub-engine users: add the
`casehub-blackboard` module and you get pure choreography unchanged.
`ChoreographyLoopControl` fires all eligible bindings — the default behaviour.
Add a `PlanningStrategy` CDI bean and the `PlanningStrategyLoopControl`
alternative, and the engine switches to orchestration without changing a
line of application code or touching the engine core.

The upgrade path is opt-in, the control contract is reactive throughout,
and strategies have access to the full context the event loop provides —
including anything a non-blocking I/O call can reach.

## One Loop, One Hook Point

A common question: if the engine already has a control loop, what is the
blackboard's control loop doing? Are there two loops? Do they conflict?

There is one loop. The engine's reactive Vert.x EventBus cycle drives
everything — it always runs, with or without `casehub-blackboard` on the
classpath. What changes is what happens at a single hook point inside that
loop: `loopControl.select()`.

[![Engine reactive event loop showing the loopControl.select() hook point](/casehub/images/engine-reactive-loop.svg)](/casehub/images/engine-reactive-loop.svg)

On every `CONTEXT_CHANGED` event, `CaseContextChangedEventHandler` evaluates
trigger conditions and produces a list of eligible bindings. It then hands that
list to `loopControl.select()` and waits for the answer. That answer determines
which workers fire.

By default, `ChoreographyLoopControl` answers "all of them" — one line, the
identity function. `PlanningStrategyLoopControl` from `casehub-blackboard`
replaces it via `@Alternative @Priority(10)` and answers differently: it
evaluates stage lifecycle, consults the per-case plan model, then delegates to
`PlanningStrategy`, which can do anything a non-blocking Uni allows before
returning its selection.

[![ChoreographyLoopControl versus PlanningStrategyLoopControl at the hook point](/casehub/images/loopcontrol-comparison.svg)](/casehub/images/loopcontrol-comparison.svg)

The engine loop is unchanged. The selection function is replaced. That is the
entire integration surface.

## The Full Picture

Below is how all the blackboard components wire together with the engine on
every cycle.

[![Full casehub-blackboard architecture showing all components and their connections to the engine](/casehub/images/blackboard-architecture.svg)](/casehub/images/blackboard-architecture.svg)

`PlanningStrategyLoopControl` coordinates three things on each `select()` call:
it builds `PlanItem`s for newly eligible bindings (via `BlackboardRegistry`,
which owns the per-case `CasePlanModel`), runs `StageLifecycleEvaluator` to
activate and terminate stages and publish their lifecycle events, then delegates
to `PlanningStrategy` which returns the final selection.

Separately, two `@ConsumeEvent` handlers run on the engine's fan-out events:
`PlanItemCompletionHandler` listens to `WORKER_EXECUTION_FINISHED` and marks
plan items complete — triggering stage autocomplete when all required items
are done. `MilestoneAchievementHandler` listens to `MILESTONE_REACHED` and
promotes tracked milestones to ACHIEVED in the plan model.

Neither handler interferes with the engine's own processing of the same events.
Fan-out (`eventBus.publish()`) means both the engine and the blackboard receive
every event independently.

The synchronous control shell was a constraint of its era. It doesn't need
to be a constraint of ours.
