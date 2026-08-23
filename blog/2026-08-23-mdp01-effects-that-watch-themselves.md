---
layout: post
title: "The Effects That Watch Themselves"
date: 2026-08-23
entry_type: article
subtype: diary
projects: [casehub-engine]
tags: [goap, monitoring, adaptation, eventlog, closed-loop]
---

# The Effects That Watch Themselves

GOAP actions declare what a worker will produce. The engine now checks whether it actually did.

A `GoapAction` carries a `Map<String, Boolean>` of effects — the same map the A* planner uses to find paths from initial state to goal state. At planning time, these effects are promises: "after this action runs, `resolved` will be true, `scored` will be true." At execution time, the engine holds the worker to those promises.

After every worker completion, `ExpectationValidator` reads the GOAP action's declared effects for the binding that fired, then checks the actual working layer. A binding whose action declares `{resolved: true, scored: true}` expects both keys present in the context. If the worker only produces `resolved`, the validator records a divergence ratio of 0.5 — one violation out of two expected effects — and writes it to EventLog metadata:

```json
{
  "expectationValidation": {
    "totalExpectedEffects": 2,
    "violatedEffectCount": 1,
    "divergenceRatio": 0.5,
    "effectSource": "GOAP",
    "adaptationGeneration": 0,
    "violations": [
      { "key": "scored", "expected": true, "actual": "UNKNOWN" }
    ]
  }
}
```

"UNKNOWN" is deliberate. The ternary world state treats absent keys as unknowable. A key the worker promised to produce but didn't isn't false — the effect simply didn't happen.

This metadata is where monitoring connects to adaptation. `ProgressGatedTrigger` reads `WORKER_EXECUTION_COMPLETED` EventLog entries, filters by compound, and computes a windowed divergence average. When the average exceeds a configurable threshold, it fires adaptation — the plan evaluator re-invokes the GOAP planner with the failed action blacklisted, producing a revised plan that routes around the underperforming worker.

The same `Map<String, Boolean>` drives both ends. Declare effects → plan with A* → dispatch → validate effects → detect divergence → adapt → replan with A*. The planning metadata IS the monitoring metadata.

For bindings without GOAP actions, `Binding.producedKeys` provides a simpler fallback — a `Set<String>` of expected output keys, converted internally to all-true effects. Same validation pipeline, `effectSource: "PRODUCED_KEYS"` in the metadata. When per-completion divergence exceeds the threshold (distinct from the windowed average), the handler publishes an `ExpectationViolationEvent` on the event bus for dashboards and alerting.

The integration test validates this pipeline end-to-end. Start a case with GOAP-declared effects, dispatch a worker that partially satisfies them, verify the metadata block lands correctly on the EventLog. Separate scenarios cover zero divergence (all effects satisfied), the `producedKeys` fallback, and the violation event publication. What the test actually guards is less obvious: without the `expectationValidation` block, `ProgressGatedTrigger.matchesCompound()` returns false for every entry. The trigger never fires. The adaptation loop is silently dead. This metadata is the bridge between execution and adaptation — the test verifies it exists.
