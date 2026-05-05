---
layout: post
title: "Phase 1: Into casehub-engine"
date: 2026-04-14
tags: [architecture, migration, testing, casehub-engine]
excerpt: "The merge direction reversed before a single line of code was written. casehub-engine becomes the home; Phase 1 lays the extension points incrementally, one PR at a time."
---

# Phase 1: Into casehub-engine

**Date:** 2026-04-14
**Type:** phase-update

---

## The direction changed before the first commit

The previous entry laid out a 9-phase merge plan — casehub as the base, casehub-engine's reactive infrastructure ported in. Sensible at the time. The co-worker was right: casehub-engine already has the working reactive event cycle, Quartz, EventLog, Signal mechanism. Porting all of that into casehub would mean a big-bang Phase 5 async refactor that everything before it sits on top of. Wrong order.

The better answer: casehub-engine is the home. casehub's CMMN/Blackboard features come in as Maven modules layered on top of the reactive core. One repo, clean module dependencies, the architecture enforced by the build.

I wanted to work incrementally — bit by bit, evaluate and re-spec each iteration. No grand upfront plan executed faithfully. Discover what's right by doing it.

## Four PRs, four extension points

We submitted four PRs this session. Each one small enough to review independently.

`LoopControl` SPI first. `CaseStateContextChangedEventHandler` evaluates JQ conditions, collects eligible bindings, and fires all of them — pure choreography by default. We extracted the "which bindings to fire" decision into an interface:

```java
public interface LoopControl {
    List<Binding> select(StateContext context, List<Binding> eligible);
}
```

`ChoreographyLoopControl` returns all eligible unchanged — existing behaviour, no regression. Any future orchestration layer plugs in here.

Next, unsealing the `ExpressionEvaluator`. It was `sealed permits JQExpressionEvaluator` — lambda conditions were impossible. We added `LambdaExpressionEvaluator(Predicate<StateContext>)` and the dispatch machinery:

```java
public interface ExpressionEngine {
    String type();
    boolean evaluate(ExpressionEvaluator evaluator, StateContext context);
    void validate(ExpressionEvaluator evaluator);
}
```

That `validate` method is the third PR: pre-validation on case registration. Bad JQ fails at startup with a clear message instead of a runtime NPE. Checking the backlog, the co-worker had planned most of this in issues #2–#9. Our `ExpressionEngine` SPI was what he'd called the expression registry in issue #3. Useful convergence.

The fourth PR completed the renames: `DispatchRule` → `Binding` throughout, `CaseHubDefinition` → `CaseDefinition`. YAML schema, generated classes, API methods — all aligned.

## What the gap analysis found

Before coding we did a systematic review of the original design spec against casehub-engine's codebase. Several things that looked like work turned out not to be.

`evalObjectTemplate()` was already on the `StateContext` interface — the co-worker's PR #28 handled it. `CaseFileItem` (per-key versioning) doesn't belong in casehub-engine at all — enriching the EventLog with key-level diffs is cleaner than adding a parallel tracking structure to `StateContextImpl`. And the `Milestone`→`ProgressMarker` rename was based on a collision that disappears once you accept that `Milestone` can serve both purposes from one class: lightweight fire-and-forget observability now, CMMN PENDING→ACHIEVED lifecycle tracking via `CasePlanModel` in Phase 2.

## The test coverage pass

At the end we ran a comprehensive test review. The suite went from 103 to 284 tests.

Claude found meaningful gaps: `StateContextImpl` had only `applyDiff` tested; `AllOfGoalExpression`/`AnyOfGoalExpression` had no tests despite driving case completion; model builder null-enforcement was entirely absent.

Claude caught the most concrete issue: `Milestone.builder().condition((String) null).build()` doesn't throw. The `String` overload wraps null in `new JQExpressionEvaluator(null)` without checking — `requireNonNull(condition)` sees a non-null object and passes. The null only surfaces as a runtime NPE at evaluation time. The tests document the behaviour. The fix is a one-line null check in the String overloads.

Four PRs open. Phase 2 — the new modules — waits for the co-worker's feedback.
