---
layout: post
title: "The Hand-Rolled Parser That Shouldn't Exist"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [jq, cdi, refactoring, architecture]
---

The bug was simple: `{ humanApproval: { status: .decision } }` in an `outputMapping` was producing a String literal in the case context instead of a nested Map. The fix should have been five lines. It turned out to be the wrong question.

The hand-rolled parser in `CaseContextImpl.evalObjectTemplate` only handled three cases: dot-path expressions, JSON literals, and bare strings. Nested `{ }` fell through to the string fallback. Adding a recursive call would have fixed the symptom.

I didn't want to fix the symptom. `JQEvaluator` already existed — a proper CDI-injectable jackson-jq wrapper with `$secret` and `$config` injection, pre-built scope, and the full jq 1.6 surface area. `evalObjectTemplate` was a partial, incorrect reimplementation of it. The question was why `evalObjectTemplate` existed at all.

Three reasons, none good: it predated `JQEvaluator`, it lived on the `CaseContext` interface (a data holder — the wrong place for expression evaluation), and it had been duplicated into `ContextUtils` as a static copy. Separately, `casehub-work/queues` had rolled its own `JqConditionEvaluator` because `JQEvaluator` lived in the Orchestration tier and Foundation-tier repos couldn't depend on it.

Three independent jq evaluator implementations. The right fix was to delete all of them.

We used IntelliJ's find-references to locate the nine production call sites — handlers, orchestrators, the Quartz job, the work adapter. Claude migrated each one to inject `JQEvaluator` and call `eval(expression, context.asJsonNode())`. The pattern is uniform:

```java
ValidationResult vr = jqEvaluator.eval(expression, context.asJsonNode());
if (!vr.ok() || vr.output() == null || vr.output().isEmpty()) return Map.of();
Map<String, Object> updates = MAPPER.convertValue(vr.output().get(0), MAP_TYPE);
```

The Quartz job was the awkward case. It depends on `casehub-engine-common`, not `casehub-engine` — moving `JQEvaluator` into the runtime module would have created a circular dependency. Moving it to `casehub-engine-common` solved that cleanly, and discovered that `SecretManager` and `ConfigManager` — the two injected dependencies — were already there.

There was one non-obvious Quarkus trap: moving a `@ApplicationScoped` bean from the application module (auto-indexed) to a library JAR (not indexed) silently breaks CDI discovery at test startup. No compile error, just "Unsatisfied dependency for type JQEvaluator" across every `@QuarkusTest` that needed it. Fix: add `quarkus.index-dependency.engine-common.artifact-id=casehub-engine-common` to four test `application.properties` files. Now in the garden as GE-20260522-adb5cd.

The tier problem is separate. `JQEvaluator` now lives in `casehub-engine-common` as an interim home — still Orchestration tier, still unreachable by `casehub-work`. The permanent fix is a `casehub-platform` expression module. Filed as platform#23, now being picked up.

`evalObjectTemplate` is gone. `ContextUtils` is gone. The nested template bug is gone, but that was never really the point.
