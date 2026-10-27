---
layout: post
title: "Quorum: The Coordination Primitive Hiding in Plain Sight"
date: 2026-10-27
entry_type: article
subtype: diary
projects: [casehubio/engine]
tags: [coordination, dsl, planning, quorum, design]
---

# Quorum: The Coordination Primitive Hiding in Plain Sight

I thought we already had quorum coordination. Turns out I was half right — and the half that was wrong is the interesting part.

The blocks-ui gallery has been rendering quorum barriers for months — those "wait for 2 of 3 agents" phase items in the `CoordinationPipeline`. The coordination-diff-view renders before/after overlays when quorum thresholds change. It looks real. But there's nothing backing it in the engine. The gallery fabricates the data.

When I picked up the issue to close that gap, the first thing Claude and I did was audit what already existed. The assumption was straightforward: add a quorum primitive, wire it to the UI, done.

## Four Layers of Quorum That Already Existed

The audit surfaced M-of-N coordination at four distinct levels:

- **CompletionSemantics.MofN** on compounds — a sealed type variant that gates when a compound is *done*
- **SubCase group M-of-N** — `groupId`/`totalInGroup`/`requiredCount` for child case fan-out, used in production by casehub-life's family-vote pattern
- **QuorumConfig** on action gates — M-of-N multi-approver oversight for consequential worker actions
- **HumanTask quorum** — per-task approval gates via `YamlQuorumConfig`

The runtime for compound MofN works. `DefaultCasePlanModel.evaluateCompletion()` handles it correctly. The sealed interface is clean. Tests exist and pass across all three pathways — YAML, Java builder, annotations.

So the engine *does* have quorum. The issue's premise — "the engine has no equivalent" — was wrong. And yet the issue was right that something was missing.

## The Real Gap: Completion vs. Dispatch

`CompletionSemantics.MofN` answers one question: "when is this group of things done?" It doesn't answer a different question that looks similar but isn't: "when should the next thing fire?"

![Quorum coordination layers — before and after](images/quorum-coordination-layers.svg)

A compound with `completionSemantics: "2"` and three scoped bindings will mark itself complete when two of the three finish. That's the completion gate. But nothing in the compound model says "hold binding X from dispatching until N of these other bindings have completed." The compound doesn't control dispatch timing — that's the planning strategy's job.

Workers today either all run (choreography) or run one at a time (sequential). There is no "wait for N of M workers before firing the next one."

That's the actual gap. Not completion semantics — dispatch prerequisites. The distinction matters because it determines where the solution lives in the architecture.

## A Guard, Not a Trigger

The design question was where to put the barrier. Claude explored three options: a new trigger type (`quorumBarrier`), a binding property (`requires`), and named groups. I initially liked having both a trigger and a guard — the trigger for cases where the quorum IS the reason to fire, the guard for cases where it's additional gating on top of an existing trigger.

The design review pushed back hard on the trigger approach and it was the right call. All four existing trigger types — `contextChange`, `schedule`, `scopeActivated`, `cloudEvent` — respond to external events. A quorum barrier responds to internal plan model state. Putting it in the Trigger sealed interface would blur a distinction the platform keeps clean.

The result: `requires` as a binding-level guard, alongside `when`. It composes with any trigger type.

```yaml
bindings:
  - name: extract-a
    capability: extract
    on: { scopeActivated: {} }

  - name: extract-b
    capability: extract
    on: { scopeActivated: {} }

  - name: extract-c
    capability: extract
    on: { scopeActivated: {} }

  - name: aggregate
    capability: aggregate
    on: { scopeActivated: {} }
    requires:
      bindings: [extract-a, extract-b, extract-c]
      atLeast: 2
```

The `aggregate` binding's trigger fires on scope activation — same as the extractors. But `requires` holds it from dispatching until at least 2 of the 3 have completed. Clean separation: the trigger says *what* activates, the guard says *when* it's eligible.

![Requires in the dispatch pipeline](images/requires-dispatch-pipeline.svg)

The `requires` filter slots into `PlanningStrategyLoopControl.select()` after compound gating and before implementation routing. Bindings with unsatisfied prerequisites are filtered from the eligible set. When a referenced binding completes, the next planning cycle re-evaluates — no explicit notification needed, because worker completion already triggers a context change.

## A Silent Bug Along the Way

The audit also found that compound MofN from YAML was silently broken. `YamlCompound` has a `mofnRequired` field (via yaml-record-mappings), and `YamlCaseDefinitionConverter.convertCompound()` dutifully reads the YAML. Then it drops `mofnRequired` on the floor and passes only `completionSemantics` to the `CompoundDeclaration` constructor. The field existed, was parsed, and was ignored.

Worse: no `BlackboardPlanConfigurer` implementation existed to bridge YAML compounds into the runtime `PlanItemDefinition.Compound` model. The conversion chain was a dead end. You could declare a compound with `completionSemantics: "3"` in YAML, parse it successfully, and nothing would happen at runtime.

The programmatic path (Java builder, annotations) worked because tests construct `PlanItemDefinition.Compound` directly, bypassing the YAML converter entirely. The YAML path — the one most users would hit — was the broken one.

We fixed both: `convertCompound()` now resolves `mofnRequired` into the completion semantics string, and a new `CompoundBlackboardPlanConfigurer` registers YAML compounds into the plan model at case start.

## The Observer That Watches Itself

The trickiest part of the design was the threshold evaluator. When enough referenced bindings complete to meet the quorum, two things can happen depending on `onThresholdReached`: remaining in-flight bindings are cancelled (CANCEL) or left alone (KEEP). When enough referenced bindings *fault* that the quorum becomes unreachable, the gated binding itself is proactively faulted.

![Threshold evaluator — two paths from one observer](images/requires-threshold-cascade.svg)

Both paths use the same integration point: `@ObservesAsync PlanItemStateChangedEvent`. This is the existing pattern from `CaseCompensationServiceImpl` — a single event catches all completion and fault paths across at least seven different handlers. Wiring into individual handlers would be fragile and miss paths.

The interesting wrinkle: the evaluator observes the event *and fires it*. When it cancels a remaining PlanItem, it fires a new `PlanItemStateChangedEvent` for the cancellation. When it faults an unreachable gated binding, same thing. The evaluator receives events triggered by its own actions.

This self-observation is what enables requires-chain cascading — if binding C requires B, and B is faulted by the evaluator because B's own quorum became unreachable, the evaluator sees B's fault event and checks whether C's quorum is still viable. The cascade terminates because three properties hold simultaneously: requires chains are validated as acyclic at load time, PlanItem state transitions are CAS-guarded (you can't fault an already-faulted item), and `addPlanItemIfAbsent` is idempotent. Remove any one of these and the cascade either loops or double-fires.

The one-shot `thresholdTriggered` flag on `CasePlanModel` prevents redundant re-cancellation. It follows the `SubCaseGroup.policyTriggered()` pattern — the cancellation itself is idempotent (it filters to non-terminal items), so the flag is for efficiency, not correctness.

## Three Surfaces, One Primitive

The engine exposes case definitions through three pathways: YAML, the Java builder API, and annotations. Any feature that appears in one surface but not the others creates a parity gap that confuses users and breaks assumptions. `requires` needed to work everywhere.

YAML was the first surface — the `BindingRequires` schema definition and `YamlCaseDefinitionConverter` wiring. The Java builder came for free because `Binding.Builder.requires(BindingRequires)` is the shared layer that YAML conversion targets anyway.

The annotation surface required two new concepts. First, a `@Requires` annotation nested inside `@Bind`:

```java
@Bind(name = "aggregate", scopeActivated = true,
      requires = @Requires(
          bindings = {"extract-a", "extract-b"},
          atLeast = 2))
```

Second, a `name()` attribute on `@Bind` itself. Until now, binding names came from the method name. But `requires` references other bindings by name, and method names in Java don't have hyphens. Adding `name()` lets annotations match the naming conventions used in YAML definitions. A small change with a subtle consequence: the `BindingDescriptor` now carries both `name` (the declared or overridden name) and `methodName` (the Java method), because the worker lookup in `CaseDefinitionRecorder` matches on method name, not binding name.

## What Makes This Work

Three design properties that matter:

**Composability.** `requires` is orthogonal to trigger type. A cron-triggered binding with `requires` is valid — "run this nightly aggregation, but only if the day's extractions have completed." Each cron tick re-evaluates the guard. A context-change binding with `requires` is the most common case — the trigger and the guard evaluate independently.

**Unreachable quorum detection.** If enough referenced bindings fault that the threshold becomes unreachable, the gated binding is proactively faulted. The formula follows `SubCaseGroupPolicy`: `remaining = total - completed - faulted`, `needed = required - completed`, `remaining < needed` means the quorum can never be met. Without this, an unreachable quorum blocks forever — a silent deadlock that no monitoring would catch because the case appears to be waiting normally.

**Projection parity.** The `CoordinationPipelineProjection` now reads `Binding.getRequires()` and populates the `required` and `of` fields on `PhaseItemView`. The blocks-ui `CoordinationPipeline` component has had `required` and `of` properties on its phase items since the beginning — it was rendering fabricated values. Now the values come from the engine, and the gallery can show real coordination topology instead of a fiction that happens to look right.
