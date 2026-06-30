---
layout: post
title: "Workers Never Needed Capabilities"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Engine]
tags: [worker-api, casehub-engine, first-principles]
series: issue-509-binding-inputschema-yaml-augment
---

I started this session planning to add a template method to YamlCaseHub — a straightforward refactoring to eliminate duplicated double-checked locks across eight consumer subclasses. The kind of thing you file as "S / Low" and expect to close before lunch.

The interesting part was what happened when I traced the duplication to its root.

## The cap() smell

Every consumer CaseHub in casehub-life had a `cap()` helper:

```java
private static Capability cap(String name) {
    return Capability.builder().name(name).inputSchema(".").outputSchema(".").build();
}
```

Eight classes, identical method, building a four-field record just to carry a single string. The obvious fix is `Capability.passthrough(name)` — a factory method that says what it means. But Claude and I kept pulling at the thread: why does a Worker need a `Capability` instance at all?

## Workers only ever use names

We traced every production usage of `worker.capabilities()` across the engine. Six call sites. Every single one does the same thing — extracts the name:

```java
w.capabilities().stream().anyMatch(c -> c.name().equals(capabilityName))
```

The `inputSchema` and `outputSchema` on worker capabilities are never read. Schemas come from the binding's `CapabilityTarget`, not the worker. Workers carry `List<Capability>` purely as name labels.

This is a design over-specification baked in from the beginning. Workers don't declare *schemas* — they declare *support* for a capability. The right type is `Set<String>`, not `List<Capability>`.

## The fix eliminates the problem instead of naming it

`Capability.passthrough(name)` would have papered over the symptom. Changing `Worker` to carry `Set<String> capabilityNames` eliminates the root cause:

- The `cap()` helper disappears — workers declare `capabilityName("book-appointment")` on the builder
- `contains()` replaces stream-anyMatch at every call site
- `DeadLetterReplayService` resolves the authoritative `Capability` from `CaseDefinition` instead of picking an arbitrary one from the worker — a pre-existing imprecision that `Set` would have made non-deterministic

The template method landed too. `YamlCaseHub.getDefinition()` is now `final`. Subclasses override `augment(CaseDefinition)` — called inside the DCL between loading and caching. The three inconsistent augmentation patterns across the platform (casehub-life's DCL duplication, casehub-aml's `@PostConstruct` delegation, casehub-devtown's race-prone mutation) all collapse to a single hook.

The design review ran six rounds and caught genuine issues — `Set<String>` instead of `List<String>` for the capability names, the DLQ replay metadata resolution, and the need to document why `Capability` stays in casehub-worker-api after `Worker` no longer references it.

The downstream migration is mechanical. Four consumer repos need updating; all the issues are filed.
