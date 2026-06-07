---
layout: post
title: "Breaking static routing in humanTask bindings"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [java, quarkus, sealed-types, humanTask, async-testing, routing]
---

Until now a humanTask binding in a case definition had fixed routing. You'd write:

```yaml
humanTask:
  title: "IRB Committee Review"
  candidateGroups: [irb-committee]
```

And every WorkItem from that binding would go to `irb-committee`, regardless of which trial site, protocol, or jurisdiction the case involved. Clinical already had an `IrbCommitteeAssignmentPolicy` SPI resolving the right committee per trial and writing the result to the case context. The routing just ignored it.

Today's change adds JQ expression support for `candidateGroups` and `candidateUsers`:

```yaml
humanTask:
  candidateGroups: ".irb.candidateGroups"
```

The expression is evaluated against the case context when the task fires. If the policy has written `["committee-a"]`, the WorkItem goes to `committee-a`.

The evaluation follows the same pattern as `inputMapping`: `CaseContextChangedEventHandler` evaluates the JQ expression before publishing the event, and the handler receives an already-resolved `Set<String>`. The handler creates WorkItems — it shouldn't be evaluating case context expressions. That boundary matters.

The more interesting question was where the new type belongs. `ExpressionEvaluator` already exists — it's the marker interface for boolean predicates dispatched by `ExpressionEngine.evaluate()`. My first instinct was to add `LiteralListEvaluator implements ExpressionEvaluator`, parallel to the existing `JQExpressionEvaluator`. A code reviewer caught why that was wrong: `ExpressionEngine.evaluate()` returns `boolean`. A list producer in that hierarchy would reach `ExpressionEngineRegistry` with no handler and no sensible dispatch path.

`ListEvaluator` is a separate sealed interface:

```java
public sealed interface ListEvaluator permits ListEvaluator.StaticList, ListEvaluator.JQList {
  record StaticList(Set<String> values) implements ListEvaluator {}
  record JQList(String expression)      implements ListEvaluator {}
}
```

Two permitted records, no connection to `ExpressionEvaluator`. The switch is compiler-enforced exhaustive. When engine#439 adds dynamic `title` or `scope`, a `StringEvaluator` sealed type follows the same pattern.

One other finding worth recording: the right technique for asserting an event was never published. We had initially written `Thread.sleep(500)` followed by `assertThat(events).isEmpty()`. That only checks at one point after the sleep — an event published during the wait window would leave the test passing incorrectly. The correct pattern:

```java
Awaitility.await()
    .during(Duration.ofMillis(300))
    .atMost(Duration.ofMillis(500))
    .untilAsserted(() -> assertThat(events).isEmpty());
```

`during()` asserts the condition holds continuously for the full 300ms. That's what "this should never happen" actually means as a test.

Shipping the feature surfaced a bigger gap. The JQ approach works when routing logic fits in a path expression. It doesn't if you want to fire Drools rules against the case state to select a committee, or run an ML model against case history, or apply a policy engine. The sealed `ListEvaluator` type has no extension point for that.

And candidateGroups routing is not the only place this problem exists. Worker routing has `AgentRoutingStrategy`. SLA escalation has `EscalationPolicy`. Sub-case routing is static. Each is its own mechanism — none of them share a common strategy contract that harness authors can extend. A routing decision made in one part of the platform can't reuse a strategy from another.

What we need is a universal routing architecture: a consistent named-strategy SPI that all routing decision points use, documented in PLATFORM.md and the protocols index so every repo knows to reach for it. The JQ evaluator becomes the default strategy, not the only one. engine#442 tracks the design work. The `ListEvaluator` sealed type may turn out to be an implementation detail of the JQ strategy rather than a top-level API — that's one of the questions engine#442 needs to answer before engine#439 ships.
