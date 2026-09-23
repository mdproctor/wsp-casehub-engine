---
layout: post
title: "The Map Key Nobody Imported"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/engine]
tags: [codegen, cdi, evolution, conductor]
series: issue-1141-cdi-event-wiring
---

# The Map Key Nobody Imported

The conductor's CDI event wiring was mechanical — inject `Event<T>` into four beans, fire `fireAsync()` at the right transition points, write tests that capture what was fired. The interesting part was what happened when we turned to the YAML codegen.

The conductor types need YAML deserialization records for DSL case definitions. `GatePolicy` has a field typed `Map<ImprovementStage, GateMode>` — the first enum-keyed map in the codegen. The generated record compiled with one import missing: `ImprovementStage`. `GateMode` resolved fine.

The codegen's `collectTypeImports` method extracts type names from generic signatures to resolve imports. It called `extractBaseType`, which peels off the outermost generic wrapper and returns the *last* type parameter — designed for `List<Foo>` and `Map<String, Foo>` where the interesting type is always at the end. For comma-separated generics, a second pass extracted the part after the comma. Both paths converged on the same answer: the value type. The key type was invisible.

The fix was three lines: split on commas, iterate all parts, check each against the imports map. The kind of bug that can only exist when every prior use case had `String` as the map key — which, across fifty generated records, they all did.

One design choice worth noting: `EvolutionTicker` fires `TickEvaluatedEvent` only for notable outcomes — a gate blocked or a proposal generated. Quiet ticks where nothing reached consensus produce no event. The SSE stream that `EvolutionStreamBroadcaster` serves becomes a feed of things that actually happened, not a heartbeat of things that didn't.
