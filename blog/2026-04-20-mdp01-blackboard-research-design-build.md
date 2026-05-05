---
layout: post
title: "Blackboard: Research, Analysis, and Implementation"
date: 2026-04-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [blackboard, architecture, reactive, quarkus]
---

PR3 landed upstream as #85 on Thursday. By Friday the upstream had also merged
#72 (in-memory SPI), #74 (treblereel's concurrent signal handling), and a few
others. The engine is in reasonable shape. It was time to start the blackboard.

I wanted to do this properly. Before writing a line of code, I asked Claude to
pull recent academic papers on blackboard architectures — specifically LLM-based
systems. The 2024 papers were more useful than I expected: one showed blackboard
multi-agent systems outperforming static frameworks on five of six benchmarks,
explicitly attributing gains to dynamic agent selection and shared memory. Another
introduced memory stratification — working memory, episodic, semantic — as a
concrete improvement over flat key-value blackboards.

The research shaped the design in one critical way. The classical blackboard
control shell is synchronous. It takes eligible bindings, returns a list. That
means a `PlanningStrategy` can only reason from what's already in memory — it
can't query the EventLog, can't call an LLM scorer, can't reach any I/O without
blocking the Vert.x event loop. We changed `LoopControl.select()` to return
`Uni<List<Binding>>`. One interface change, four files, zero behaviour change for
`ChoreographyLoopControl`. That became the article published alongside:
[Four Things a Synchronous Blackboard Strategy Can't Do](https://mdproctor.github.io/casehub).

The spec and plan came next — 16 tasks, TDD throughout. We started executing via
subagent-driven development.

Task 2 — Maven module setup — surfaced something worth noting: a
`casehub-blackboard` module already existed in the upstream. A prior
implementation was already in place: generic `PlanItem<T>`, stages containing
workers directly, `CasePlanModelRegistry`, `SubCase`.

## Why a fresh implementation against the new specification

The immediate incompatibility was the `LoopControl` interface change — our own
Task 1. The existing `PlanningStrategyLoopControl` had been built against the
previous synchronous signature:

```java
@Override
public List<Binding> select(PlanExecutionContext context, List<Binding> eligible)
```

Our Task 1 change required `Uni<List<Binding>>`. The existing implementation
no longer compiled against the new interface.

Beyond the compilation break, the existing module reflected an earlier design
specification. Several decisions reached during the brainstorm session were not
part of that prior specification. Adapting the existing code to meet the new
specification involved six divergences, each requiring changes of comparable
scope to building against the new specification directly:

**1. Plan model scope.** `CasePlanModelRegistry` associated plan models with
`CaseDefinition` objects. The new specification requires a `CasePlanModel`
per running case UUID, so that concurrent instances of the same case type each
have independent agenda, stage state, and focus of attention. This was a
decision from the design session.

**2. `PlanItem<T>` as a unified generic type.** The new specification separates
the activation record (`PlanItem`, keyed to a `Binding`) from the lifecycle
container (`Stage`). This separation is required by the independent
completion-tracking design, which was new to the specification.

**3. Stage-to-worker relationship.** In the prior implementation, workers were
assigned to stages at definition time. The new specification has stages evaluate
entry and exit conditions against `CaseContext` state — stages gate which
Bindings can fire, not which workers are assigned. This reflects the design
session's decision on how capability matching should interact with stage activation.

**4. `PlanningStrategy` interface signature.** The new specification gives
strategies read-write access to `CasePlanModel` — focus of attention, resource
budget, extensible key-value state. The new signature
`(CasePlanModel, PlanExecutionContext, List<Binding>)` captures design decisions
not present in the prior specification.

**5. Plan item completion feedback.** The `PlanItemCompletionHandler` role —
marking items complete when workers finish and triggering stage autocomplete —
was introduced in the new specification and was not in scope for the prior
implementation.

**6. Stage lifecycle events.** The new specification publishes
`StageActivatedEvent`, `StageCompletedEvent`, and `StageTerminatedEvent` as
first-class EventBus events for observability and lineage integration. This
was a new requirement.

Given the interface incompatibility and the six specification divergences, an
implementation built directly against the new specification was the cleaner
path. The result: `plan/`, `stage/`, `event/`, `control/`, `registry/`,
`handler/` packages built throughout against the async `LoopControl` contract.

## On the synchronous nature of the original LoopControl

It is worth being precise about what "synchronous" meant here, because the
existing handler was already operating in a reactive context.

`CaseContextChangedEventHandler` is a `@ConsumeEvent` handler that returns
`Uni<Void>` — it participates in Vert.x's reactive model. However, within that
handler, the call to `loopControl.select()` was a blocking synchronous call
returning `List<Binding>`:

```java
// Before: synchronous call lodged inside an async chain
List<Binding> selected = loopControl.select(planCtx, eligible);
// ...then compose the Uni from results
return Uni.combine().all().unis(unis).discardItems();
```

The outer context was reactive; the selection point within it was not. Any code
inside `select()` that attempted I/O — querying the EventLog, calling an external
scoring service — would block the event loop thread directly. On Vert.x, blocking
the event loop thread is a correctness violation, not merely a performance concern.

After the change, `select()` returns a `Uni` and participates in the chain:

```java
// After: select() is a first-class participant in the reactive pipeline
return loopControl.select(planCtx, eligible)
    .chain(selected -> { ... });
```

The practical difference: a `PlanningStrategy` can now query the EventLog,
call an LLM scorer, or reach any non-blocking I/O before resolving its selection.
The async context and the selection point are now consistent throughout.

## Code review and findings

The code review returned 18 findings. Two were critical:

First: `PlanItem.priority` was mutable. A strategy calling `setPriority()` after
insertion into the `PriorityBlockingQueue` would silently corrupt the heap —
wrong ordering, no exception, no warning. `priority` was made `final`.

Second: `hasActivePlanItem()` was an O(n) scan of all `ConcurrentHashMap` values
with a TOCTOU gap between the check and the subsequent insert. We replaced it
with an `activeByBinding` index — a single map lookup, effectively atomic.

The six remaining Important findings were all fixed. Three Minor ones tightened
test coverage and the `PlanItemCompletionHandler` control flow.

## Integration and PRs

The upstream had diverged enough that a rebase of the branch produced conflicts
on commit 1 of 48. A fresh branch from `upstream/main` with the net diff applied
via `git diff upstream/main -- <files>` resolved this cleanly. One commit, 68
tests green.

The PR was split into three for review — #88 (async LoopControl, 4 files), #89
(data model, 36 tests), #90 (orchestration + integration, full 68 tests). All
open against `casehubio/engine:main`, merge in order.

The research-identified improvements — meta-control, private agent scratchpad,
memory stratification, hierarchical panels — are all tracked in issues #77–#84.
That's the next evolutionary layer. For now the foundation is in.
