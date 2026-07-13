---
layout: post
title: "The Shim That Wasn't Dead"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [snapshot, dependency-management]
---

## The Shim That Wasn't Dead

We spent this session on issue triage — reviewing the state of the orchestration model unification (#700) and its trailing children. The epic itself was closed. The shared type foundation had landed: `TaskDescriptor`, `TaskStatus`, `ExecutorRef`, `TaskSnapshot`. `PlanItem` implements the shared interface. The parallel hierarchies between engine and blocks are gone.

Three child issues were still open: #702 (migrate event/handler `workerName` to `ExecutorRef`), #719 and #720 (clean up the `PlanItemStatus` shim). We checked the codebase. #702 is genuinely unfinished — `workerName` strings still thread through events. But #719 and #720 looked done. `PlanItemStatus` existed only as a deprecated compatibility shim with zero references anywhere in engine. The only consumer had been `casehub-engine-work-adapter`, which relocated to the work repo months ago.

So we deleted it. Build passed. Issues closed.

Within the hour, AML's build broke. `WorkAdapterPlanItemEntity` — still published as part of an old SNAPSHOT — references `PlanItemStatus`. The class had zero references *in engine's source tree*, but it had a live consumer in a transitive SNAPSHOT dependency that engine's own index couldn't see.

This is the SNAPSHOT coupling problem in miniature. A class can have zero references in your codebase and still be load-bearing for downstream consumers pulling your artifacts. `mvn install` and IntelliJ's index both confirm the same thing: nothing in *this repo* uses it. Neither tool knows what *other repos* resolved against your last published SNAPSHOT.

The fix is straightforward — publish a new work-adapter SNAPSHOT without the `PlanItemStatus` reference, or restore the shim until that happens. But the lesson is worth naming: in a multi-repo SNAPSHOT ecosystem, "zero references" is a statement about your repo, not about your artifact's consumers. Safe deletion requires checking downstream, not just upstream.

We also confirmed #724 (EngineStrategyResolver not indexing work strategy types) is already resolved — the catch-all `@Any Instance<NamedStrategy>` fallback discovers any `NamedStrategy` bean, and work#304 fixed the Jandex corruption that was preventing bean discovery in the first place.
