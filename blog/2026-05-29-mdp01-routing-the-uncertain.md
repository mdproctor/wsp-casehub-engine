---
layout: post
title: "Routing the Uncertain"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [routing, trust, reactive, cdi, sealed-types]
---

Two routing issues had been sitting on the backlog since the `AgentRoutingStrategy`
SPI shipped. One was about what to do when all candidates are borderline on trust —
the four-phase model says escalate to human oversight, but the code just returned
`noOp()`. The other was about semantic routing: matching agents to cases by embedding
similarity, not just trust scores and job counts.

I thought they were independent. They weren't. Both touch the same seam —
`AgentAssignment`, the return type from routing selection. Once I decided to fix
the nullability smell in that type, both issues had to be designed together or
I'd build the escalation path twice.

The sealed type was the right move. The old `AgentAssignment` record had `null
workerId` as the sentinel for "no worker" — a single discriminant collapsing three
semantically different outcomes. We replaced it with three variants:

```java
sealed interface AgentAssignment permits Assigned, Unresolvable, EscalateToOversight {}
```

`Unresolvable` when no candidates pass trust filters. `EscalateToOversight` when
they all pass but are borderline — uncertainty, not disqualification. The engine
handles each differently: `Unresolvable` tries dynamic provisioning;
`EscalateToOversight` posts a QUERY to the case's oversight channel and waits.

The `TrustCandidateClassifier` was the design detail that made semantic routing
clean. Both `TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` need
to classify candidates through the four trust phases before doing anything else.
If we put that logic in the trust strategy, the semantic strategy would have to
duplicate it or compose via the strategy interface — which would mean losing the
intermediate `ClassifiedCandidate` data the semantic re-ranking depends on.
An `@ApplicationScoped` CDI bean injected into both strategies keeps the
classification algorithm in one place, tested once.

The code review caught something I'd missed. Claude flagged that `embed()` is
blocking — an HTTP call to an embedding service — and `AgentRoutingStrategy.select()`
was synchronous. Calling blocking code from a `@ConsumeEvent` handler running on
the Vert.x IO thread is a hard violation, not a performance concern. The fix wasn't
to mark the handler as `blocking = true` (that would move all routing to a worker
thread, including the in-memory trust scoring that has no business being there).
The fix was to make the SPI itself return `Uni<AgentAssignment>` and let
`SemanticAgentRoutingStrategy` use `.emitOn(Infrastructure.getDefaultWorkerPool())`
before the embed calls.

The timing mattered. There were exactly two implementations at that point. Retrofitting
a synchronous SPI to reactive later is a breaking multi-repo change with no compile-time
signal — you discover it at runtime under load. We took the pain now when the migration
cost was minimal. The in-memory strategies trivially return
`Uni.createFrom().item(result)`; semantic gets to emit properly on a worker pool.

The rest was cascading breakage — fixing every caller of the old record API.
`AgentRoutingContext` got `caseContext` (the case data the semantic strategy needs
to embed). `AgentCandidate` got `agentDescriptor` (the vocabulary for embedding).
Both handlers that built candidates were duplicating the same health-probing loop,
so we extracted it to `AgentCandidateFactory` while we were touching them.
