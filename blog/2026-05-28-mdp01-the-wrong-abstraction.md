---
layout: post
title: "The Wrong Abstraction"
date: 2026-05-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [cdi, routing, trust, architecture]
---

**Seven null fields**

One of the better diagnostic tests for a wrong abstraction: count how many fields you're passing as `null`.

`WorkerSelectionStrategy` and `WorkBroker` came from `casehub-work`, a module built for human task routing. The engine had borrowed them for agent worker selection because the interface looked close enough. But the actual call site:

```java
new SelectionContext(null, null, capability.getName(), null, null, null, null, null);
```

Seven out of eight fields null. `AssignmentTrigger.CREATED` passed even though agents aren't triggered by WorkItem creation — they're triggered by context change. `WorkerCandidate.activeWorkItemCount` counted WorkItems, not Quartz jobs. And `CasehubWorkloadProvider` wrapped `WorkerExecutionManager` (Quartz job counts — correct semantics) inside a `WorkloadProvider` SPI from casehub-work. Correct semantics, wrong abstraction. The consequence was a CDI ambiguity in the resilience test module: `JpaWorkloadProvider` and `CasehubWorkloadProvider` both implementing `WorkloadProvider`, resolved only by a stub class whose entire purpose was to suppress the collision.

We replaced the borrowed stack with an engine-owned `AgentRoutingStrategy` SPI. `AgentRoutingContext` carries `caseId` and `capabilityName` — nothing else. `AgentCandidate` carries the worker name, Quartz job count, and a pre-probed `AgentHealth` enum defined in the engine's own api module (so `casehub-eidos-api` stays out of the Tier 1 contract). Both `WorkOrchestrator` and `CaseContextChangedEventHandler` migrated. Three stub files and two pom dependencies deleted.

**The trust cache**

The bigger design question was how to get trust scores into the selection path. Query `ActorTrustScoreRepository` at selection time? Add the scores to `SelectionContext` (a cross-repo change to `casehub-work-api`)?

I went with neither.

`TrustScoreRoutingPublisher` in the ledger already fires `TrustScoreFullPayload` after every scoring cycle. Nothing was listening. We added a `TrustScoreCache` — `@Startup @PostConstruct` for startup hydration, `@Observes TrustScoreFullPayload` (not `@ObservesAsync`: the publisher uses synchronous `fire()`, and `BeanManager.resolveObserverMethods()` only returns sync observers, so `@ObservesAsync` would silently receive nothing). Two `ConcurrentHashMap`s for CAPABILITY and CAPABILITY_DIMENSION scores. Selection-time lookups are O(1). No cross-repo change needed.

The trust maturity model runs through four phases. Phase 0: no CAPABILITY history — availability routing, identical to the pre-trust baseline. Phase 1: sparse history — same. Phase 2: borderline candidates (score within `borderlineMargin` of threshold) get score 0.0 and are excluded; below-threshold candidates likewise. Phase 3: CAPABILITY_DIMENSION quality floors — if a dimension score is below its floor, excluded.

The mixed Phase 0/Phase 2 pool has an interesting property. A new agent with no trust history scores on workload only — availability routing, typically > 0. A borderline established agent scores 0.0. The new agent wins. I think that's correct: no track record is preferable to evidence of borderline quality.

One trap caught in testing: trust score 0.75, threshold 0.70, `borderlineMargin` 0.10. |0.75 − 0.70| = 0.05 ≤ 0.10 — borderline, excluded. The margin applies symmetrically. Easy to miss when writing tests that reason about "above threshold."

**The merge**

Dmitrii had done a package reorganization in the same window — `engine.internal.*` → `engine.common.internal.*`, `engine.spi.*` → `engine.common.spi.*` across the common module. Our `CaseLifecycleEvent` move crossed it. The rebase left conflicts in thirteen files. We resolved them with a Python script that accepted the upstream's `engine.common.*` structure for all imports but placed `CaseLifecycleEvent` at `engine.common.spi.event` — honouring both the package reorganization and the promotion to a stable package.

The reorganization fixed a genuine split-package problem. When two reasonable changes cross, you adapt. The result is cleaner than either would have been alone.