---
title: "The missing half of producedKeys"
author: mdp
date: 2026-09-06
entry_type: note
subtype: diary
series: issue-1047-compensation-viz-follow-ups
projects: [casehubio/engine]
tags: [binding-model, graph-projection, design-symmetry]
---

The engine's binding model had an asymmetry hiding in plain sight. `producedKeys` on a `Binding` declares what context keys a worker writes — the effects side. But nothing declares what a binding reads. The preconditions side was missing.

This matters because we're building a design-time viewer for the binding DAG. The viewer needs edges: which binding feeds which. Compensation edges come from `compensateRef`. Data flow edges come from `produces`/`consumes` channels. But trigger dependencies — binding A's output activating binding B — were implicit, buried in JQ conditions evaluated at runtime.

The obvious fix was to parse JQ expressions and extract referenced paths. I nearly went down that road. But JQ is a full programming language. `.transaction.amount > 1000` is easy. `[.items[] | select(.status == "completed")] | length > 0` is not. Static path extraction from arbitrary JQ is a compiler problem, not a string scan.

The actual fix was staring at the GOAP planner we already have. `GoapAction` carries both `effects` (what it produces) and `preconditions` (what it requires). The binding model had the effects side (`producedKeys`) but not the preconditions side. Adding `requiredKeys` — a `Set<String>` declaring which context keys a binding needs — gives us reliable trigger dependency edges through simple set intersection. No parsing. No fragility.

The `CompensationGraphProjection` from the saga branch only knew about compensation edges. With `requiredKeys` in place, I generalised it to `BindingGraphProjection` with three typed edges: `COMPENSATION` (from `compensateRef`), `DATA_FLOW` (channel matching), and `TRIGGER_DEPENDENCY` (key set intersection). The old compensation-specific types are gone. `CaseDefinitionType` exposes `bindingGraph` instead of `compensationGraph`.

The design question that keeps surfacing: when should inference be explicit? The engine's choreography model is built on implicit trigger evaluation — bindings fire when JQ conditions match. That's powerful for runtime flexibility. But for static analysis, visualisation, and tooling, implicit relationships are invisible. `requiredKeys` makes one class of implicit relationship explicit without constraining the runtime. The binding still fires from JQ evaluation. The keys are metadata for tooling, not a runtime gate.

The pattern might have legs beyond the graph viewer. Validation could warn when a binding declares `requiredKeys` that no other binding produces. The planner could use key dependencies for smarter scheduling. Documentation generators could show data flow without reading JQ. Each of those is a consumer of the same declaration — the symmetric half that was always missing.
