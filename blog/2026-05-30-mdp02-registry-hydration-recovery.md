---
layout: post
title: "Hydration and Recovery: Teaching the Engine to Survive a Restart"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [blackboard, registry, restart-recovery, humantask]
---

When a Quarkus app restarts, every in-memory registry starts empty. For `BlackboardRegistry`, that meant any `WorkItemLifecycleEvent` arriving before the next CONTEXT_CHANGED — a human completing a delegated task, say — would call `registry.get(caseId)`, get empty, log a debug line, and throw the completion away. The case would never progress.

That was engine#274. The second failure mode was subtler: a WorkItem could complete *while the JVM was down*, the lifecycle event would fire into a void, and the case would hang indefinitely with a DELEGATED PlanItem that would never transition.

Both belong in the same branch. They share a root cause — `BlackboardRegistry` has no recovery path — and the startup catch-up service depends on the registry being hydrated first.

## The Startup Service That Didn't Survive Contact with the Spec Review

I started with an eager approach: a `@Priority(10)` `StartupEvent` observer that pre-populated the registry before Quartz jobs ran. Working through it with Claude, three rounds of spec review produced the right correction: lazy loading in `get()` is simpler, more resilient to rolling restarts, and eliminates the startup priority ordering entirely. Once the registry hydrates on first miss, the ordering constraint disappears.

Two surprises came out of the review cycle.

First, `HumanTaskTarget.inline().build()` throws `IllegalStateException` if no title is set — even when building a recovery-only target to carry an outputMapping expression. The fix was a `[restored]` sentinel. Claude's spec review caught this before any code was written.

Second, adding a JPA call inside `BlackboardRegistry.get()` immediately broke four existing `@ConsumeEvent` methods. `PlanItemCompletionHandler`, `WorkerRetryExhaustionHandler`, `PlanItemFaultHandler` — all called `registry.get()` with no `blocking = true`. Previously that was fine; the call was just a map lookup. Once `get()` started querying JPA on miss, those handlers were making blocking database calls on the Vert.x IO thread. We added `blocking = true` to all four methods and converted their `Uni<Void>` returns to `void`.

I captured that as a protocol: any `@ConsumeEvent` handler in the blackboard module calling `registry.get()` must declare `blocking = true`. It's not obvious from reading the handler — the JPA call is buried in the registry implementation.

## Catching Up While the JVM Was Down

`HumanTaskRecoveryService` runs at `@Priority(25)` — after Quartz recovery at 20. At startup it scans for all DELEGATED PlanItems, finds their WorkItems via `callerRef`, and for each WorkItem already in a terminal state fires the catch-up transition. The registry hydrates lazily as those transitions land.

One thing we had to add first: `WorkItemService.findByCallerRef()`. It didn't exist in casehub-work. The implementation is a `scanAll().stream().filter()` — only called at startup, so a full scan is acceptable.

The shared completion logic — status transition, outputMapping, `CONTEXT_CHANGED` — was sitting in `WorkItemLifecycleAdapter` and would have had to be duplicated. We extracted it into `PlanItemCompletionApplier` first, so both callers use the same path.
