---
title: "Plan Adaptation Design and the Compound Replacement Problem"
date: 2026-08-08
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [plan-adaptation, design-review, casehub-engine, htn-planning]
published: false
---

Static plans don't survive contact with reality. The engine's hierarchical planning (#802) decomposes agent goals into ordered capability sequences at case start, but once materialised, the plan is frozen. A worker discovers the data is structured differently than expected, or fails outright, and the remaining steps execute against invalidated assumptions.

Plan adaptation is the fix: after each worker completion, evaluate whether the plan still holds and revise if it doesn't. The design decomposes this into two orthogonal strategies. An `AdaptationTrigger` decides whether to replan — `EveryStepTrigger` fires on every completion, `OnFailureTrigger` only when something goes wrong. A `PlanRevisionStrategy` produces the replacement — `ForwardReplanRevision` re-invokes the LLM with completed step history and asks "what's left to do?" Both are `NamedStrategy` implementations, configurable per case definition, with YAML presets for common combinations (`adaptation: adaptive` maps to every-step + forward-replan).

The interesting problem was compound replacement. A four-dimension design review — coherence, structure, robustness, cross-cutting — found 29 issues, and the cross-cutting analysis identified the deepest one: the spec's premise of shared materialisation between initial decomposition and adaptation was architecturally unsound.

Initial decomposition calls `registerDefinition()` once with the complete compound — all scoped bindings, child definitions, and index structures populated atomically. Adaptation needs to replace an existing compound: unregister old children, remove pending PlanItems, re-register with updated bindings. These are structurally different operations. `addChild()` doesn't update the compound's immutable `scopedBindings`, so new steps are invisible to the dispatch loop. `removePlanItem()` doesn't clean up the definition layer, so orphaned definitions block compound completion. The shared abstraction hides a fundamental mismatch.

The fix is `CasePlanModel.replaceCompound()` — atomic compound replacement. Unregister old children from all five index structures (definitions, definitionStates, childrenIndex, parentIndex, scopedBindings), remove pending PlanItems while preserving completed and running ones, then re-register the entire compound with a fresh definition. The dispatch loop sees updated bindings immediately. Compound completion evaluates the revised children.

The other review finding worth noting: the SPI signature needs to be thin. Both call sites — `PlanItemCompletionHandler` (success) and `WorkerRetryExhaustionHandler` (failure) — have different data available. Rather than enriching both handlers with `CaseInstance`, `CaseDefinition`, and `MutableCaseContext`, the evaluator accepts only `(caseId, tenancyId, bindingName, TaskStatus)` and resolves the rest internally. Keeps the wiring minimal.

Three of six implementation tasks are done — SPI types, `replaceCompound()`, and the built-in strategies. The orchestrator and call site wiring are next.
