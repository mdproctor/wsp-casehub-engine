---
layout: post
title: "The Data Store Drools Actually Needs"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [blackboard, drools, architecture]
---

After PR #443 merged — dynamic candidateGroups/Users for humanTask — the question
turned to what's next. I'd been wanting to do Drools integration for a while.
Engine has issue #5 (DroolsExpressionEvaluator) and issue #207 (RulesRouter) already
filed, so I assumed it was mostly a matter of picking up the right issue.

Claude and I started mapping the dependency chain and hit the problem immediately.
Issue #207 has a `WorkingMemoryBridge` stub that was explicitly deferred. The
reason it was deferred is the same reason it can't be picked up now without
prerequisite work: `CaseContext` is a flat `Map<String, JsonNode>`.

Drools's working memory holds typed Java objects — facts. A rule evaluating whether
an account meets a SAR filing threshold expects an `Account` type with a `balance`
field and a `riskScore` field. What we have is a JSON blob with those keys as strings.
You can write DRL against untyped JSON, but then you've written rules tied to JSON
structure rather than your domain model, which defeats the point of a rules engine.

The fix is what I'd mentally filed under "future blackboard work" — issue #80
(memory stratification: working/episodic/semantic panels on CaseContext) and issue #81
(hierarchical blackboard panels). I'd been treating those as evolutionary improvements
for academic completeness. Drools makes them prerequisites.

Classical blackboard architectures — HEARSAY-II, BB1 — didn't store flat JSON. The
blackboard was a typed, structured data space divided into levels or sections, each
holding domain objects at a specific abstraction. Knowledge sources read and wrote
typed entries. We implemented the control shell correctly: reactive, async,
CMMN-gated. But we left the data store as a JSON map. That gap doesn't matter for
JQ expressions and context-change triggers, which are string-path-based. It matters
for anything that wants to reason over typed domain objects.

So the dependency chain for full Drools integration:

1. `ExpressionEvaluatorFactory` SPI (engine#289) — pluggable expression languages
   at the YAML definition level
2. Typed panels on `CaseContext` (#80 + #81) — the actual data store rules can reason over
3. `WorkingMemoryBridge` (engine#446, filed today) — bridges typed panel entries into
   Drools `WorkingMemory` as Java facts
4. `DroolsExpressionEvaluator` (engine#5) — stateless DRL for binding conditions
5. `RulesRouter` (engine#207) — full rules-based orchestration with `RULES_DECISION`
   lineage in the EventLog

We wrapped this under epic #445. The order matters: without typed panels,
`WorkingMemoryBridge` has nothing meaningful to bridge; without the bridge, the
`RulesRouter` is reasoning over JSON blobs.

One small thing shipped today: engine#447, adding `NoOpWorkerExecutionManager` as
a `@DefaultBean` to unblock casehub-workers. Pure pattern-following — the ninth
existing no-op in `engine/internal/worker/` made the tenth obvious.

The framing that shifted for me: engine#80 and #81 were filed under epic #77
("Blackboard Architecture Evolution") with the note "these improve the architecture
after the foundation is solid." That's slightly off. They're not improvements —
they're the part of the classical blackboard data model we didn't implement yet.
The foundation is a control shell with a JSON scratchpad attached. The typed data
store the architecture was built around is still missing. Drools just made that
visible.
