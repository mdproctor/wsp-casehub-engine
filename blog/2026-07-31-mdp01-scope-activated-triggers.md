---
layout: post
title: "Scope-Activated Triggers — Workers That Start When Their World Does"
date: 2026-07-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [lifecycle-scopes, planning, dispatch]
series: issue-821-lifecycle-scope-wiring
---

Most case workers are reactive. A context change satisfies a condition, the binding fires, the worker runs. This is the choreography model — bindings listen for data, and dispatch follows naturally.

But some workers need a different relationship with time. A monitoring sidecar should start when its compound activates and run for the compound's lifetime. A case-level audit logger should start when the case starts, not when some arbitrary piece of data appears. These workers don't react to context — they react to scope.

That's what `ScopeActivatedTrigger` does. A binding declares `on: scopeActivated` instead of `on: contextChange`, and the engine dispatches it when the owning scope transitions to active — compound PENDING→RUNNING, or case STARTING→RUNNING. The `when` guard still applies, so activation can be conditional. But the trigger itself is lifecycle, not data.

The interesting architectural question was where this dispatch should live.

The obvious answer was a new `CompoundActivatedEventHandler` in the runtime module — the same pattern as every other event handler. `CompoundLifecycleEvaluator` already publishes a `CompoundActivatedEvent` when a compound activates. Wire up a consumer, do agent routing, publish the `WorkerScheduleEvent`. Done.

The problem is PlanItem creation. The engine tracks every dispatched binding through a `PlanItem` — it's how compound completion knows when all its children are done. PlanItems are created inside the planning module, via `LoopControl.select()`. The runtime module doesn't depend on the planning module. A runtime handler would need to reach back into planning for PlanItem creation — a circular dependency dressed up as an event.

The right answer is that scope activation is a planning decision, not a runtime event. The planning module already evaluates compound lifecycle inside `select()`. It already knows which compounds just activated. Adding scope-activated binding discovery at that point means the dispatch rides the same pipeline as every other binding — PlanItem creation, compound gating, agent routing, `WorkerScheduleEvent`. Zero new classes.

The implementation is small. `CompoundLifecycleEvaluator.evaluate()` now returns the list of activated compounds instead of void. `PlanningStrategyLoopControl.select()` uses that list to find `ScopeActivatedTrigger` bindings whose names appear in the activated compound's `scopedBindings()`. For case scope, `markConfigured()` already fires exactly once per case — CASE-scoped bindings collect there. Both paths merge into the eligible list before the existing gating and dispatch logic.

One decision worth calling out: scope-activated bindings dispatch with a null `signalId`. The signal that triggered the context change caused the compound to activate, which caused the scope-activated binding to dispatch — but tracking settlement through that causal chain means a signal's completion would depend on planning-level secondary effects. Wrong abstraction level. The signal tracks its direct effects; compound activation is the planning module's business.

Claude caught a factual error in the design spec during adversarial review — the edge case for repeatable compound re-activation claimed `addPlanItemIfAbsent()` would create a new PlanItem when the old one was COMPLETED. It doesn't. The method rejects new items when an existing one is COMPLETED or active. Repeatable compound support needs PlanItem recycling, which doesn't exist yet. The spec now says "not yet supported" instead of making a false claim about current behaviour.

What this opens up: the remaining lifecycle scope issues (#823–#826) now have a dispatch path. Scoped workers can be discovered, selected, and scheduled. What they can't yet do is register sessions in the `ScopedWorkerRegistry` (#823), accumulate state across reinvocations (#826), or spawn persistent virtual threads (#824). The dispatch was the foundation — the execution modes are the structure that goes on top.
