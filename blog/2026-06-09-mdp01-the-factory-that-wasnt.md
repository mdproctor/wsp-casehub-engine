---
layout: post
title: "The factory that wasn't"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [expression-evaluator, spi, cdi, quarkus, design]
---

`CaseDefinitionYamlMapper` had five call sites that all said the same wrong thing:

```java
new JQExpressionEvaluator(expression)
```

The runtime evaluation chain was already pluggable — `ExpressionEngine`, CDI discovery, `ExpressionEngineRegistry` dispatching by type. The only part that wasn't pluggable was parse time: the mapper that turns YAML strings into evaluator value objects. Engine #289 was supposed to fix that.

My first instinct was a new `ExpressionEvaluatorFactory` interface. It seemed obvious: a factory takes a string and a language identifier and produces the right evaluator. One CDI bean, one dispatch. We went through three rounds of critique before I understood why that was wrong.

The first critique pointed out that a single factory makes adding a new language a wholesale replacement — you either modify the default factory or override the entire thing with `@Alternative @Priority(1)`. That's the opposite of how `ExpressionEngine` works, where each language is a separate CDI bean and new ones are purely additive. Why have two competing patterns for the same pluggability story?

The cleaner answer: `ExpressionEngine` already declares `type()`, which IS the language identifier. Just add `create(String expression)` to it as a default method that throws `UnsupportedOperationException`. JQ overrides it; Lambda doesn't. Same CDI bean handles both creation and evaluation. The three-way invariant — `expressionLang` in YAML equals `evaluator.type()` equals `engine.type()` — is structurally enforced rather than documented and hoped for.

The second critique caught two blockers I'd introduced. `ExpressionEngineRegistry` was in `common/spi/` but all its dependencies were in `api` — I'd proposed injecting it into `YamlCaseHub` in `api`, which is a circular dependency that won't compile. And the convenience `load(InputStream)` overload used a lambda for `ExpressionEngineRegistry`, which has four abstract methods and isn't a functional interface.

Those fixes were structural rather than clever. `ExpressionEngineRegistry` moved to `api/engine/` alongside `ExpressionEngine` — where it architecturally belongs. `@YamlMapper` moved from `runtime/internal/marshaller/` to a new `api/marshaller/` package. A CDI qualifier has no business being buried inside `runtime/internal/` when the injection point it qualifies is in a different module.

The third critique was mostly implementation gaps: `convertBinding` and `convertTrigger` needed the registry and expressionLang threaded through as parameters, `scheduler-quartz/pom.xml` needed an explicit `casehub-engine-api` dependency, and `loadStream()` didn't exist.

By the time we implemented, the design was solid. The mapper lost its static `yamlMapper` field and `setObjectMapper()` — a method that had a TODO comment attached to it for months. `YamlCaseHub` now injects `ExpressionEngineRegistry` and `@YamlMapper ObjectMapper` directly via CDI instead of relying on an observer that fired too late to matter. `ObjectMapperInjector` is gone.

That observer was a good example of a workaround that works just well enough to hide that it's broken. `DefaultCaseDefinitionRegistry.onStart()` runs at `@Priority(10)` — meaning it fires first, loads all the case definitions. `ObjectMapperInjector.onStartup()` had no `@Priority`, which defaults to 2500 in CDI. It ran *after* the definitions were already loaded with the fallback plain `ObjectMapper`. The fallback happened to parse YAML fine, so nobody noticed for months.

The classifier-throws integration test (#434) was quicker. The fail-safe path in `WorkflowExecutionCompletedHandler` was already implemented in #402 — `.onFailure().recoverWithItem()` applies a `GateRequired` when the classifier throws instead of returning a decision. Adding `throwOnClassify` to the `CapturingClassifier` test stub and writing one test confirmed it works end to end.

One thing worth noting: the Drools framing got dropped partway through. The HANDOFF said #289 was "first in the Drools chain," and I initially assumed the factory would eventually produce `DroolsExpressionEvaluator` instances for binding conditions. It won't. Drools isn't a per-condition expression evaluator — it's a ruleset engine that fires holistically against working memory. The factory is for languages that actually fit the `evaluate(evaluator, context) → boolean` contract: JQ, SpEL, FEEL, JEXL. Drools integration, when it happens, will be at a different level entirely.
