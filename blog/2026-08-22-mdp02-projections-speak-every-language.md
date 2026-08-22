---
layout: post
title: "Projections speak every language"
date: 2026-08-22
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [expression-evaluator, projections, type-migration, casehub]
---

Boolean conditions got per-expression language override in engine#925 — a binding's `when` clause could be JQ, MVEL, or whatever the `ExpressionEngineRegistry` supports. But data transform projections (`inputProjection`, `outputProjection`, `inputProjectionOverride`) stayed hardcoded to JQ. Every runtime call site created a `JqTransformer` directly or called `jqEvaluator.eval()`. The YAML mapper read projection fields as plain strings. The type system didn't know projections had a language at all.

Engine#943 closes that gap. Three model types changed: `CapabilityTarget` expanded from a 1-field record to carry resolved `ExpressionEvaluator` objects for both input and output projections. `Binding.inputProjectionOverride` changed from `String` to `ExpressionEvaluator`. `SubCaseMapping.Expression` changed from wrapping a raw string to wrapping an `ExpressionEvaluator`.

The interesting constraint was threading evaluators through the YAML mapper without breaking backward compatibility. `Capability` is a foundation-tier type in `worker-api` — its `inputSchema()` and `outputSchema()` must stay as strings because the foundation tier has no dependency on `platform-api` where `ExpressionEvaluator` lives. So `CapabilityTarget` became the resolution boundary: it wraps the `Capability` and adds the resolved evaluators. The YAML mapper builds a `capTargetMap` alongside its existing `capabilityMap`, resolves projections through `resolveExpression()` from the raw JSON nodes, and passes the pre-built `CapabilityTarget` to the binding builder. Plain string projections resolve to `JQExpressionEvaluator` — zero behavioral change for existing definitions.

On the runtime side, every `evalJqAsMap()` and `evalJqAsJsonNode()` call site was replaced with `ExpressionEngineRegistry.transform()` via `transformAsMap()` helper methods. `CaseContextChangedEventHandler`, `WorkerScheduleEventHandler`, `DefaultWorkOrchestrator`, `SubCaseCompletionService`, `DefaultPersistentScope` — all now dispatch through the registry. The dead `evalJq*` methods were deleted. `AgentConverter` gained a 4-arg overload that resolves agent projections through the registry and wraps them in `inputTransformer`/`outputTransformer` lambdas, keeping the `Agent` itself a pure execution unit that never sees the expression engine.

The YAML syntax is the same `{lang: expr}` map that #925 established for conditions:

```yaml
capabilities:
  - name: analyse
    inputProjection: { mvel: "transaction" }
    outputProjection: ".result"
```

With this, every data path through the engine — capability projections, binding overrides, SubCase mappings, agent transformers — routes through `ExpressionEngineRegistry.transform()`. The expression language is a per-field choice, not a per-definition constraint.
