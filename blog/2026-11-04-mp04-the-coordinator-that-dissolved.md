# The Coordinator That Dissolved

Every multi-agent system ends up with a coordinator. A class that holds the routing strategy, the aggregation strategy, the provisioning logic, the budget pool, the prompt sections, the termination condition — a gravitational well for coordination concerns. Ours was called `LlmSwarmCoordinator`.

It worked. Nine fields, one method that assembled prompts, accessor methods for everything else. The builder had two paths: `build()` returned the coordinator, `buildTopology()` returned a `CentralisedTopology` that held the same components behind a `CoordinationTopology` interface. Two ways to build the same thing — a clear sign that one of them should not exist.

The question was which one.

## What the Topology Already Knew

`CentralisedTopology` had something the coordinator did not: structure. Its `configure()` method defined three `DecisionLoopDescriptor`s — team formation, collective reasoning, provisioning — each with an observation strategy, a decision scope, and a budget priority. The descriptors declared *what coordination should happen*. The coordinator just held *the things that do coordination*.

Meanwhile, the `AgentDecisionLoop` infrastructure — built in the previous iteration — provided a pipeline for turning observations into decisions. You give it a `ContentSummariser` (the reasoner) and the factory wraps it in prompt section injection, budget checking, and concurrency gating. Budget management, prompt enrichment, and in-flight deduplication — all composed, not coded.

![Decision loop pipeline — what you get for free](images/decision-loop-pipeline.svg)

The coordinator assembled prompts manually. The loop pipeline does it through `SectionAwareContentSummariser`. The coordinator checked budgets by calling `pool.tryAcquire()` in each strategy. The loop pipeline does it through `BudgetAwareSummariser`. The coordinator was reimplementing infrastructure that already existed.

## The Adapter That Made It Click

The gap between the LLM strategies and the loop infrastructure was a type mismatch. `SwarmRoutingStrategy` takes a `RoutingContext<SwarmContext>` and returns a `RoutingDecision`. The loop wants a `ContentSummariser<ObservationEvent, DecisionOutcome>`. Different worlds.

The adapter pattern bridges them. `RoutingReasoner` implements `ContentSummariser`, translates observation events into a routing context, calls the strategy, and wraps the result as a `DecisionOutcome.Route`. Same pattern for `AggregationReasoner` and `ProvisioningReasoner`. Each is under 40 lines. The existing strategies — with all their fallback logic, experience replay, prompt customisation — remain untouched.

![Before and after — coordinator dissolving into topology](images/coordinator-before-after.svg)

Three thin adapters turned three strategies into loop-compatible reasoners, and the topology became the single coordination path. The coordinator's `build()` method had zero callers. It was dead code hiding in plain sight.

## The Type That Prevented a Hack

Provisioning needed a home in the `DecisionOutcome` sealed hierarchy. The first instinct was `WriteBack` — deposit a signal, write a context key, done. Quick, no new types.

The decision review caught the flaw. `DecisionOutcomeExecutor` dispatches on outcome type. The `EventStreamBus` carries no source metadata — no loop name, no producer tag. If provisioning used `WriteBack`, the executor could not distinguish a provisioning write-back from a generic one. Loop-identity dispatching was architecturally impossible.

The fix was a `Provision` variant:

```java
record Provision(ProvisioningAdvice advice) implements DecisionOutcome {
    public Provision {
        Objects.requireNonNull(advice, "advice must not be null");
    }
}
```

One new type in the sealed hierarchy. The compiler enforces exhaustive handling in every executor — every topology that pattern-matches on outcomes now has a `case Provision` arm, or it does not compile. The convenient hack would have shipped silently. The type system would not.

## Why This Matters

The pattern here is not specific to swarms. Any time coordination logic is split across three concerns — routing, synthesis, feedback — and each concern has its own observation cadence, budget, and prompt context, a monolithic coordinator is fighting the infrastructure.

Topologies with decision loops give you independent observation pipelines per concern, composed budget management (not hand-wired per strategy), prompt section injection without manual string assembly, and the sealed outcome hierarchy as a type-safe dispatch mechanism. You write the reasoner. The infrastructure wraps it.

The coordinator didn't fail. It dissolved — its responsibilities moved into infrastructure that already existed, waiting for the adapters to connect them.
