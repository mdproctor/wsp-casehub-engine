---
layout: post
title: "Teaching the Engine Who Should Handle This"
date: 2026-07-29
categories: casehub engine routing
---

Most workflow engines assign human tasks the same way they did twenty years ago: a queue, a round-robin, maybe a role-based filter. The task appears in someone's inbox. Whether that person is the right person — whether they've handled this kind of work before, whether they're already overloaded, whether the case context makes one reviewer more appropriate than another — is left to the humans to figure out.

We've been building the infrastructure to change that.

## What CBR routing actually does

Case-Based Reasoning has been part of the engine's agent routing for a while — when an AI worker needs to handle a task, the engine retrieves similar past cases and checks which agents performed well on them. The agent with the best historical success rate gets selected.

The question I wanted to answer: why should humans get a worse routing experience than AI agents?

The `HumanTaskRoutingStrategy` SPI already existed from earlier work. What it lacked was a concrete implementation that used the same CBR scoring infrastructure. The challenge was subtle: humanTask plan traces use `bindingName` as the matching key (not `capabilityName` like agent traces), and human outcomes include DECLINED — a signal that was being silently dropped because `RoutingOutcome` didn't have a value for it.

That silent drop is the kind of bug that doesn't show up in testing. A human who declines nine tasks but completes one would score 1.0 — perfect — because only the success was counted. Claude caught this during the adversarial design review, tracing the data flow from `CbrCaseRetainObserver` through `ExperienceAnalyser` to the scoring output.

## Beyond history: constraint-based routing

CBR answers "who has historically performed well on similar tasks?" but it can't answer "who is available right now?" or "does this case require a senior reviewer?"

The constraint-based strategy handles both. Context constraints are declarative rules — JQ or MVEL expressions evaluated against the case state — that filter or score candidates:

```yaml
humanTaskConstraints:
  context:
    - when: ".transaction.amount > 10000"
      preferGroups: ["senior-reviewers"]
      weight: 0.8
  workload:
    maxActiveTaskCount: 5
    loadBalanceWeight: 0.3
```

The workload side queries a `WorkloadDataProvider` SPI for each candidate's active task count. Users above a threshold are excluded; the rest are scored inversely to their load. The two dimensions — context fitness and operational availability — combine additively.

Every constraint uses the platform's `ExpressionEvaluator` SPI, so the same expression that gates a binding trigger can gate a routing constraint. And the dual YAML/Java DSL convention holds: anything you can declare in YAML, you can also write as a lambda in the Java builder.

## Why this matters

The work routing problem in most organisations is a quiet tax. Tasks sit in the wrong queue. Experienced people get overloaded while junior staff wait for work they could handle. Context that would make the routing decision obvious — the case type, the transaction amount, the customer tier — is visible in the case data but invisible to the assignment mechanism.

These two strategies make that context actionable. CBR learns from outcomes. Constraints encode domain knowledge. Neither requires a human to manually configure assignment rules for every case type — the system adapts as it accumulates experience, and the constraints provide guardrails around that adaptation.
