---
title: "One Convention, Not Seven"
date: 2026-07-04
author: Mark Proctor
type: diary
projects: [casehub-engine]
tags: [routing, spi, architecture, platform-coherence]
---

# One Convention, Not Seven

The casehub platform had seven different ways to select a routing strategy. CDI `@Priority`. CDI `@Alternative` with config fallback. `@Named` qualifier. Named `id()` lookup. Single `@DefaultBean` replacement. `@RiskClassifier` qualifier with chain composition. Config property switch. Each worked. None was aware of the others.

The problem showed up when I asked a simple question: can a harness author use a Drools-based candidateGroups resolver for human task routing, then share that same Drools session with worker routing? The answer was no — not because of a technical limitation, but because the two routing points used completely different selection mechanisms with incompatible extension models. `ListEvaluator` was a sealed type with two variants (static list, JQ expression). `AgentRoutingStrategy` was a CDI priority chain. You couldn't plug into one without knowing the other's internals.

The audit across five repos surfaced roughly 64 routing-related mechanisms. Most of them aren't routing strategies at all — they're data providers, access control gates, fan-out delivery. The actual "select one from many" routing decisions that a harness author needs to configure are concentrated in engine and work, and there are about eight of them. Those eight had no shared convention.

The design question was whether to unify them under a single generic `RoutingStrategy<I, O>` or create a family of domain-specific interfaces with a shared naming convention. The generic looked appealing on paper but fell apart under inspection — the input shapes (case context, breach context, binding list), output shapes (sealed assignment types, group sets, deadline computations), and cardinalities (select one, select many, produce the candidate set) are genuinely different. A `RoutingStrategy<AgentRoutingContext, AgentAssignment>` has nothing in common with `RoutingStrategy<SlaBreachContext, BreachDecision>` except the method signature shape. You can't write shared implementations because the domain semantics differ.

What IS shared is the convention: every strategy has a name, is CDI-discoverable, can be selected by that name in YAML, ships a `@DefaultBean` fallback, and resolves through a single `StrategyResolver`. That's `NamedStrategy` in platform-api — a marker interface with one method: `String id()`. The resolver lives alongside it. Both engine and work consume it without depending on each other.

The sealed `ListEvaluator` became `CandidateSetStrategy` — an open SPI returning `Uni<Set<String>>`. Static lists and JQ expressions are now value objects implementing the same interface. A Drools-based resolver or an LDAP lookup can implement it too, registered as a CDI bean, referenced by name in YAML. The hardcoded `AgentCandidateFactory` two-tier matching algorithm became `CandidateMatchingStrategy` — exact match and subsumption match are now pluggable implementations, not baked-in logic.

One thing I didn't expect: Quarkus ARC's `Instance<NamedStrategy>` doesn't discover beans implementing sub-interfaces. A bean implementing `AgentRoutingStrategy extends NamedStrategy` is invisible to `Instance<NamedStrategy>`. No error, no warning — just an empty iterator. The platform `DefaultStrategyResolver` couldn't work as designed. The workaround is an `EngineStrategyResolver` that injects `Instance<>` per domain type. Pragmatic but not elegant — every new strategy SPI requires updating the resolver's constructor.

The design review ran four rounds, raised eighteen issues, and every one was resolved. The most impactful: the reviewer caught that `CandidateSetStrategy.evaluate()` was synchronous, violating the platform's reactive convention. It also caught that `HumanTaskTarget` was about to hold a CDI proxy (`CandidateSetStrategy`) directly, breaking its pure-data-model contract. The fix — `CandidateSetSpec` as a sealed type with `Inline` (value object) and `Named` (deferred CDI reference) — is the kind of separation I should have seen from the start.

The work-side retrofit, consumer migration, and documentation are filed as follow-up issues. The engine-side architecture is landed.
