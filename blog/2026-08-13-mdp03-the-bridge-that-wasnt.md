---
layout: post
title: "The bridge that wasn't — goal decomposition meets engine dispatch"
date: 2026-08-13
entry_type: article
subtype: diary
projects: [casehub-engine]
tags: [planning, goal-decomposition, binding, architecture]
---

casehub's goal decomposition pipeline looked complete on paper. An LLM takes a
high-level goal, produces an ordered sequence of capability references, and the
engine materializes them as compound PlanItems with CHOREOGRAPHED dispatch. The
spec was clean. The types were right. The code compiled.

It had never actually worked.

The bridge between "what the LLM knows" (capabilities) and "what the engine
dispatches" (bindings) was missing entirely. Three bugs, all in the same method,
all invisible until someone tried to wire it into a real case definition.

## Three bugs, one root cause

A `GoalStep` carries a `capabilityName` — say, `"data-gathering"`. The binding
that targets this capability has its own name — say, `"gather"`. These are
different strings referencing the same thing at different abstraction levels.

`DefaultGoalDecomposer` used `step.capabilityName()` everywhere a binding name
was expected. Three consequences:

**Compound gating bypassed.** `compoundBuilder.binding("data-gathering")` puts
the capability name into `scopedBindings`. But `PlanningStrategyLoopControl`
gates on `Binding.getName()`, which is `"gather"`. The filter
`allScopedNames.contains(b.getName())` returns false. The binding passes through
unscoped — compound ordering is ignored, and every binding fires on every
context change.

**Null executor NPE.** `PlanItemDefinition.Primitive` requires a non-null
`ExecutorRef` in its compact constructor. The GoalDecomposer passed `null`. This
throws before the compound is even registered.

**PlanItems invisible to dispatch.** `PlanItemSaveRequest.primitive()` was called
with the capability name as the `bindingName` parameter.
`CasePlanModel.findPlanItemByBindingName()` looks up by actual binding name. The
PlanItems existed in the store but the dispatch loop could never find them.

The root cause: `CaseDefinition` had no reverse lookup from capability name to
binding. The GoalDecomposer had the right data but no way to translate it.

## What the planning literature says

The impedance mismatch between abstract planning and concrete dispatch turns out
to be well-studied. HTN planners decompose compound tasks into subtasks via
methods — method selection is governed by precondition satisfaction, not by the
decomposer. GOAP explores all actions producing the same effect and selects by
cost. The consensus across HTN, GOAP, CMMN, and modern agentic AI architecture
is the same: planning should be abstract; implementation binding should be late.

casehub already has this layered model. `ImplementationRoutingStrategy` exists
precisely for dispatch-time selection among bindings targeting the same
capability. The GoalDecomposer just wasn't using it — it was reaching across the
abstraction boundary and speaking the dispatch layer's language without a
translator.

## The fix and a deliberate constraint

The fix is three pieces: `CaseDefinition.findBindingsByCapability()` for the
reverse lookup, `BindingExecutorResolver` extracted as a shared utility from
`PlanningStrategyLoopControl`, and the GoalDecomposer rewritten to resolve
capability → binding before touching compound creation or PlanItem persistence.

The interesting design question was multi-binding: what happens when two bindings
target the same capability? The compound completion evaluator requires *all*
scoped bindings to reach terminal state. If `ImplementationRoutingStrategy`
dispatches only one of two scoped bindings, the other stays PENDING and the
compound never completes.

The proper fix — capability-level scoping on `Compound` with "any-binding-terminal"
completion semantics — is architecturally clean but has no concrete consumer yet.
We took the v1 constraint: one binding per capability in decomposed plans, first
in declaration order. The limitation is documented alongside the existing
"sequential plans only" constraint. Both can be lifted when a real multi-binding
use case materializes — but building the infrastructure without a consumer to
validate the semantics against risks getting it wrong.
