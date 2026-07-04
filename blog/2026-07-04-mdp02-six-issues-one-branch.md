---
layout: post
title: "Six Issues, One Branch"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-engine]
tags: [concurrency, routing, worker-runtime]
---

The last session landed the universal routing strategy convention (#634) — `NamedStrategy`, `StrategyResolver`, five SPIs retrofitted. That work left a trail of follow-on issues: evaluation happening in the wrong place, non-deterministic defaults, a concurrency race nobody had noticed, and three methods that still threw `UnsupportedOperationException`.

I wanted to clear the deck. Six issues, all on one branch, ranging from a one-line annotation swap to a full case-lifecycle tracker.

## The Race That Was Always There

The PlanItem concurrency fix turned out to be more interesting than the issue title suggested. The `SequentialPlanningStrategy` integration test was flagged as "bindings fire concurrently" — which sounds like a test timing problem. It's not. It's a production race condition.

`PlanItem.status` was `volatile` with check-then-set transitions. Two concurrent `CONTEXT_CHANGED` evaluations — perfectly normal on the Vert.x worker pool — could both read PENDING, both call `markRunning()`, both dispatch. The fix was `AtomicReference` with a `tryMarkRunning()` CAS, plus merging the two-step filter-then-index into a single atomic `filterAndIndexForDispatch`. Only the thread that wins the CAS dispatches; the loser's binding quietly drops from the returned list.

The harder question was whether to also serialize all `CONTEXT_CHANGED` processing per case. We filed that separately — the CAS guard is correct for CapabilityTarget bindings, but non-CapabilityTarget bindings (HumanTask, SubCase) still have a TOCTOU gap because their handlers own the state transition. That gap needs its own analysis of re-entrant paths and deadlock surface.

## Spawning Cases From Workers

The `WorkerRuntime.spawnCase()`/`awaitCase()` implementation was the largest piece. The design needed three things: a way to find a case definition by name, a way to track when a child case completes, and careful handling of the race where the child finishes before the parent registers interest.

`CaseDefinitionRegistry.findByName()` scans by name and throws on ambiguity — in Tier 1 (same JVM), case names are almost always unique. `CaseCompletionTracker` is a `ConcurrentHashMap<UUID, CompletableFuture<CaseContext>>` with a deliberate asymmetry: `register()` creates entries, `complete()` only acts on existing entries. No orphans — if nobody is awaiting a case, the terminal notification is a no-op.

The race guard in `awaitCase()` handles the out-of-order case by checking `CaseInstanceCache` after registration. The cache has no eviction, so terminal instances are always present. The design review caught several things I'd have missed: `CaseContext` mutability (fixed with `snapshot()`), the need for `CaseTerminatedException` to propagate through `ExecutionException` unwrapped, and a tracker leak where `complete()` originally used `remove()` instead of `get()`.

## The Default That Wasn't Deterministic

`EngineStrategyResolver` used `putIfAbsent` to select the default strategy per type — which meant whichever CDI bean iterated first won. CDI iteration order is not deterministic. The fix uses Quarkus ARC's `InjectableBean.isDefaultBean()` to detect `@DefaultBean` strategies explicitly. `@DefaultBean` wins unconditionally; multiple `@DefaultBean` for the same type throws at startup. This is Quarkus-specific, but the resolver is already `@Alternative @Priority(1) @ApplicationScoped` — it's not pretending to be portable.

## Moving the Gate Evaluation

The gate evaluation fix was structurally clean. `ActionGateWorkItemHandler` was evaluating `CandidateSetStrategy` with null context — it lives in `work-adapter` and has no case context. Moving the evaluation to `WorkflowExecutionCompletedHandler.handleGate()`, where `CaseInstance` is in scope, and carrying the resolved `Set<String>` in the event, was the obvious fix. The design review added failure recovery — if the strategy evaluation fails, the gate still fires with empty candidate groups and a warning, rather than silently dropping.

The routing strategy convention protocol and PLATFORM.md updates landed as cross-repo commits to `casehubio/parent` and the garden. The convention now has a formal home: per-case selectable strategies extend `NamedStrategy`, declare `id()`, ship `@DefaultBean`, resolve via `StrategyResolver`.

Six issues, fourteen commits, two follow-on issues filed. The deck is clear.
