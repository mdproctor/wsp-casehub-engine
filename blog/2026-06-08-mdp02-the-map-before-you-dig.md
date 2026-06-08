---
layout: post
title: "The Map Before You Dig"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [blackboard, registry, eviction, plan-execution, hybrid-execution]
---

Four small issues in a single batch branch. Three were exactly what they looked like. One had a finding inside.

The actor-state tests were straightforward: cover a contributor that writes partial state before throwing, and cover the deleted-channel race in the Qhorus contributor. The partial-write test documents something non-obvious — if a contributor calls `acc.trustScore(0.8)` then throws, that score stays in the response. No rollback. The engine doesn't enforce contributor atomicity; each contributor is responsible for its own consistency. The test is there so no one "fixes" this by adding a rollback, then discovers why the accumulator can't be transactional.

The registry lifecycle analysis is where things got interesting. My assumption going in was that evicting from BlackboardRegistry at WAITING state would require rebuilding the `completionIndex` on re-hydration. The completionIndex maps workerName → planItemId — it's populated at schedule time so completion handlers can route back to the right PlanItem. If we evict the case from the registry when it goes idle, we'd need to recover that map.

I asked Claude to trace through how WorkItem lifecycle events actually reach the registry. The answer changed the analysis. `WorkItemLifecycleAdapter` — the class that handles WorkItem COMPLETED/REJECTED/CANCELLED events — doesn't touch the completionIndex at all. It routes via callerRef: case ID and planItemId are embedded in the WorkItem's `callerRef` field, parsed by `CallerRef.parse()`. The adapter calls `registry.get(caseId)` to get the CasePlanModel, then locates the PlanItem by ID directly. The completionIndex is only needed by Quartz completion handlers, and at WAITING state there are no active Quartz workers — all active workers are DELEGATED HumanTask.

So Phase 1 of stateless-on-rest eviction lands for free. Evict when the case enters WAITING. Lazy hydration already restores DELEGATED PlanItems from PlanItemStore on the next access. No new persistence fields. No new SPI methods. Phase 2 — evicting while Quartz workers are still running — does need `workerName` persisted in PlanItemRecord. But Phase 2 is a separate future issue, and not a dependency.

The hybrid execution analysis had a similar shape but the other way around. I expected to design the FlowWorker → WorkOrchestrator bridge — a flow step dispatching a casehub worker and awaiting the result. We looked at the code first. `casehub-engine-flow` already shipped `CasehubDispatch` and `CasehubCallableTaskBuilder` earlier this year. `call: casehub:dispatch` in a YAML flow step does exactly what the issue describes. The issue was written before that module existed, never updated.

What remained from that issue was plan-based execution: an LLM or rules engine hands the engine an ordered list of workers, and the engine executes them sequentially with full `causedByEntryId` lineage. The design adds `Worker(Plan.of(...))` as a new function type alongside `Worker(Workflow)` and `Worker(Agent)`. Dynamic plans use `Plan.fromContext(".executionPlan")` — the plan is written to case context at runtime, the function reads and executes it.

Issue #187 was the simplest close. It tracked whether WorkerRegistry would grow complex enough to need an internal WorkerCandidateSource chain. WorkerRegistry never materialised — the architecture evolved differently, and WorkerProvisioner SPI together with CaseDefinitionRegistry are already the separate candidate sources the issue anticipated. The trigger condition never fired.

Three issues told me what to expect before I started. The fourth — the registry one — looked like a confirming exercise until it wasn't. Knowing that callerRef routing bypasses the completionIndex at WAITING state is what makes Phase 1 safe. You need to map the code before you draw the safety diagram.
