---
layout: post
title: "The Registry That Ate the Scheduler's Reputation"
date: 2026-06-12
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [cdi, quarkus, test-isolation, jq, registry]
---

Every symptom pointed at Quartz. Twelve tests timing out — cases start, workers never fire, `ConditionTimeoutException` after ten seconds of nothing. The contaminating class was `CaseFaultedStateTest`, which runs six cases through `AlwaysFailingCaseHubBean` with retry policies. When it ran first, everything after it died. Textbook cross-test scheduler pollution.

Except it wasn't.

We added diagnostic logging to the JQ expression evaluator — the engine evaluates binding trigger expressions against the case context to decide which workers to schedule. The expression `.working.status == "processing"` was evaluating against a context where `.working.status` was clearly `"processing"`. Result: `false`.

That's impossible. Unless the expression being evaluated isn't the one you think.

Claude traced it one level deeper: the actual expression reaching the evaluator was `.status == "processing"` — no `.working.` prefix. The context had the data under `.working.status` (panels convention), but the expression was from a flat-context era. The JQ evaluator was working correctly. It was evaluating the *wrong expression* against the *right data*.

**Two beans, one key**

`SimpleCaseHubBean` (Java) and `YamlSimpleCaseHubBean` (loading `simple.yaml`) both registered under the same `CaseDefinitionRegistry` key: `(test, Document Processing Test, 1.0.0)`. The Java definition used `.working.status == "processing"`. The YAML used `.status == "processing"`. The registry uses first-wins — whichever CDI bean's `getDefinition()` gets iterated first by `Instance<CaseHub>` determines the active definition.

Quarkus ARC doesn't guarantee `Instance<T>` iteration order across JVM instances. So the test suite wasn't test-ordering-dependent — it was CDI-ordering-dependent. Different JVM starts, different bean ordering, different winner. The Testcontainers startup delay in the hibernate profile (~8–12s) was masking it by giving the event bus time to drain — nothing to do with Quartz at all.

**The fix**

Three changes. First: give the YAML definition its own namespace (`test-yaml`) so the key collision can't happen. Second: `registerKnownDefinitions()` now scans all `CaseHub` beans for duplicate keys before registering and throws `IllegalStateException` naming both colliding beans — every `@QuarkusTest` in every module becomes an implicit collision detector. Third: `CaseKeyUniquenessTest` as an explicit regression guard, scanning all beans with a diagnostic failure message.

The registry's original "first wins, no warning" behaviour was the real bug. The YAML expression drift was the trigger. The Quartz theory was a dead end that cost an hour of investigation into scheduler internals, event bus saturation, and `LoopControl` state — none of which were involved.

The misleading symptoms are worth naming: `ConditionTimeoutException` in a shared-Quartz-instance test suite, worker scheduling that fails silently, and a contamination pattern that correlates with test class ordering. All three pointed at the scheduler. The actual cause was a map key collision in the definition registry, surfaced by non-deterministic CDI bean ordering, expressed through a JQ evaluator that was doing exactly what it was told.
